---
layout: post
title: Android 12 - Letterbox 模式
tags: [Android,Android 12,Android S]
categories: 
- Android
---

# 1. 简介

随着越来越多大屏和折叠屏设备出现，很多应用并未对不同尺寸的设备进行 UI 适配，这时候应用选择以特定的宽高比显示（**虽然 Google 不建议这这样做，官方还是希望开发者可以对不同的屏幕尺寸进行自适应布局~**），当应用的宽高比和它的容器比例不兼容的时候，就会以 Letterbox 模式打开。

![2021-10-21-18-17-07](http://image.hanschen.site/master/2021-10-21-18-17-07.png)

Letterbox 模式下界面会以指定的比例显示，周围空白区域可以填充壁纸或者颜色。至于 Letterbox 的外观可受以下因素影响：

 - `config_letterboxActivityCornersRadius`： 界面圆角大小
 - `config_letterboxBackgroundType`： 背景填充类型，分别有： 
   - `LETTERBOX_BACKGROUND_APP_COLOR_BACKGROUND`： 颜色受 `android:colorBackground` 影响
   - `LETTERBOX_BACKGROUND_APP_COLOR_BACKGROUND_FLOATING`： 颜色受 `android:colorBackgroundFloating` 影响
   - `LETTERBOX_BACKGROUND_SOLID_COLOR`： 颜色受 `config_letterboxBackgroundColor` 影响
   - `LETTERBOX_BACKGROUND_WALLPAPER`： 显示壁纸，此选项和 `FLAG_SHOW_WALLPAPER` 类似，会导致壁纸窗口显示
 - `config_letterboxBackgroundWallpaperBlurRadius`： 壁纸模糊程度
 - `config_letterboxBackgroundWallaperDarkScrimAlpha`： 壁纸变暗程度

<!-- more -->

# 2. 何时触发

Letterbox 的触发条件一般有：

 - `android:resizeableActivity=false` 且应用声明的宽高比与容器不兼容的时候(如屏幕宽高超出 `android:maxAspectRatio`)
 - `setIgnoreOrientationRequest(true)` 系统设置忽略屏幕方向后，以横屏模式下打开一个强制竖屏的界面


# 3. 实现方案

Letterbox 显示的实现并不复杂，Android 12 在 `ActivityRecord` 中增加了 `LetterboxUiController` 用以控制 `Letterbox` 的布局和显示，先来看看处于 Letterbox 模式时 SurfaceFlinger 状态：

![2021-10-22-10-33-26](http://image.hanschen.site/master/2021-10-22-10-33-26.png)

可以看到，跟正常情况相比，除了界面本身的大小和位置被缩放到指定比例外，四周还多了两个 Layer，挂在 ActiviRecord 节点下面，这两个 Layer 可根据配置进行指定的颜色填充，如果背景是壁纸的话，还可以设置壁纸的 dim 值和模糊程度，这些都可以通过 SurfaceControl 接口轻松实现。

下面简单分析一下代码：

```java
// LetterboxUiController.java
void updateLetterboxSurface(WindowState winHint) {
    final WindowState w = mActivityRecord.findMainWindow();
    if (w != winHint && winHint != null && w != null) {
        return;
    }
    // 对界面四周需要显示的 Layer 进行位置计算
    layoutLetterbox(winHint);
    if (mLetterbox != null && mLetterbox.needsApplySurfaceChanges()) {
        // 对 Surface 执行创建、参数设置等操作
        mLetterbox.applySurfaceChanges(mActivityRecord.getSyncTransaction());
    }
}

void layoutLetterbox(WindowState winHint) {
    final WindowState w = mActivityRecord.findMainWindow();
    if (w == null || winHint != null && w != winHint) {
        return;
    }
    updateRoundedCorners(w);
    updateWallpaperForLetterbox(w);
    // 是否进入 Letterbox 模式的关键判断
    if (shouldShowLetterboxUi(w)) {
        if (mLetterbox == null) {
            // 把具体逻辑委托给 Letterbox 
            mLetterbox = new Letterbox(() -> mActivityRecord.makeChildSurface(null),
                    mActivityRecord.mWmService.mTransactionFactory,
                    mLetterboxConfiguration::isLetterboxActivityCornersRounded,
                    this::getLetterboxBackgroundColor,
                    this::hasWallpaperBackgroudForLetterbox,
                    this::getLetterboxWallpaperBlurRadius,
                    this::getLetterboxWallpaperDarkScrimAlpha);
            mLetterbox.attachInput(w);
        }
        mActivityRecord.getPosition(mTmpPoint);
        // Get the bounds of the "space-to-fill". The transformed bounds have the highest
        // priority because the activity is launched in a rotated environment. In multi-window
        // mode, the task-level represents this. In fullscreen-mode, the task container does
        // (since the orientation letterbox is also applied to the task).
        final Rect transformedBounds = mActivityRecord.getFixedRotationTransformDisplayBounds();
        final Rect spaceToFill = transformedBounds != null
                ? transformedBounds
                : mActivityRecord.inMultiWindowMode()
                        ? mActivityRecord.getRootTask().getBounds()
                        : mActivityRecord.getRootTask().getParent().getBounds();
        // 位置计算
        mLetterbox.layout(spaceToFill, w.getFrame(), mTmpPoint);
    } else if (mLetterbox != null) {
        mLetterbox.hide();
    }
}
```

```java
// Letterbox.LetterboxSurface.java
public void applySurfaceChanges(SurfaceControl.Transaction t) {
    if (!needsApplySurfaceChanges()) {
        // Nothing changed.
        return;
    }
    mSurfaceFrameRelative.set(mLayoutFrameRelative);
    if (!mSurfaceFrameRelative.isEmpty()) {
        if (mSurface == null) {
            // 创建挂在 ActivityRecord 节点下的 Surface，设置为 ColorLayer 类型
            createSurface(t);
        }
        // 设置颜色、位置、裁剪
        mColor = mColorSupplier.get();
        t.setColor(mSurface, getRgbColorArray());
        t.setPosition(mSurface, mSurfaceFrameRelative.left, mSurfaceFrameRelative.top);
        t.setWindowCrop(mSurface, mSurfaceFrameRelative.width(),
                mSurfaceFrameRelative.height());

        // 对壁纸背景设置透明度和模糊度
        mHasWallpaperBackground = mHasWallpaperBackgroundSupplier.get();
        updateAlphaAndBlur(t);

        t.show(mSurface);
    } else if (mSurface != null) {
        t.hide(mSurface);
    }
    if (mSurface != null && mInputInterceptor != null) {
        mInputInterceptor.updateTouchableRegion(mSurfaceFrameRelative);
        t.setInputWindowInfo(mSurface, mInputInterceptor.mWindowHandle);
    }
}
```

# 4. 小结

本文只是简单分析了下 Letterbox 模式的触发条件和显示的大概逻辑，还有很多细节没有涉及，比如详细的触发逻辑判断可以查看 `LetterboxUiController#shouldShowLetterboxUi` 方法
