---
layout: post
title: Android Q 黑暗模式（Dark Mode）源码解析
tags: [Android]
categories: 
- Android
---

# 1. 简介

随着 Android Q 发布，「黑暗模式」或者说是「夜间模式」终于在此版本中得到了支持，官方介绍见：[https://developer.android.com/guide/topics/ui/look-and-feel/darktheme](https://developer.android.com/guide/topics/ui/look-and-feel/darktheme)，再看看效果图：

![2019-10-21-17-21-50.png](https://gitee.com/hanschencoder/Images/raw/master/2019-10-21-17-21-50.png)

其实这个功能魅族在两年前就已支持，不得不说 Android 有点落后了，今天我们就来看看原生是怎么实现全局夜间模的吧

# 2. 打开与关闭
从文档上我们可以可知，打开夜间模式有三个方法：

 - 设置 -> 显示 -> 深色主题背景
 - 下拉通知栏中开启
 - Pixel 手机开启省点模式时会自动激活夜间模式

# 3. 如何适配

打开后，我们会发现，除原生几个应用生效外，其他应用依然没有变成深色主题，那么应用该如何适配呢？官方提供了下面两种方法：

## 3.1. 让应用主题继承 `DayNight` 主题

```xml
<style name="AppTheme" parent="Theme.AppCompat.DayNight">
```

或者继承自

```xml
<style name="AppTheme" parent="Theme.MaterialComponents.DayNight">
```

继承后，如果当前开启了夜间模式，系统会自动从 night-qualified 中加载资源，所以应用的颜色、图标等资源应尽量避免硬编码，而是推荐使用新增 attributes 指向不同的资源，如

```xml
?android:attr/textColorPrimary
?attr/colorControlNormal
```

另外，如果应用希望主动切换夜间/日间模式，可以通过 `AppCompatDelegate.setDefaultNightMode()` 接口主动切换

## 3.2. 通过 forceDarkAllowed 启用

如果应用不想自己去适配各种颜色，图标等，可以通过在主题中添加 `android:forceDarkAllowed="true"` 标记，这样系统在夜间模式时，会强制改变应用颜色，自动进行适配（这个功能也是本文主要探讨的）。不过如果你的应用本身使用的就是 `DayNight` 或 `Dark Theme`，forceDarkAllowed 是不会生效的。

另外，如果你不希望某个 view 被强制夜间模式处理，则可以给 view 添加 `android:forceDarkAllowed="false"` 或者 [view.setForceDarkAllowed(false)](https://developer.android.com/reference/android/view/View.html#setForceDarkAllowed(boolean))，设置之后，即使打开了夜间模式且主题添加了 forceDarkAllowed，该 view 也不会变深色。比较重要的一点是，这个接口只能关闭夜间模式，不能开启夜间模式，也就是说，如果主题中**没有**显示声明 forceDarkAllowed，`view.setForceDarkAllowed(true)` 是没办法让 view 单独变深色的。如果 view 关闭了夜间模式，那么它的子 view 也会强制关闭夜间模式

总结如下：
 - 主题若添加 forceDarkAllowed=false，无论 view 是否开启 forceDarkAllowed 都不会打开夜间模式
 - 主题若添加 forceDarkAllowed=true，view 可以通过 forceDarkAllowed 关闭夜间模式，一旦关闭，子 view 的夜间模式也会被关闭
 - 如果父 view 或主题设置了 forceDarkAllowed=false，子 view 无法通过 forceDarkAllowed=true 单独打开夜间模式为
 - 若使用的是 `DayNight` 或 `Dark Theme` 主题，则所有 forceDarkAllowed 都不生效


# 4. 实现原理

通过继承主题适配夜间模式的原理本质是根据 ui mode 加载 night-qualified 下是资源，这个并非 Android Q 新增的东西，我们这里不再描述。现在主要来看看 forceDarkAllowed 是如何让系统变深色的。

既然一切的源头都是 `android:forceDarkAllowed` 这个属性，那我们就从它入手吧，首先我们要知道，上面我们说的 `android:forceDarkAllowed` 其实是分为两个用处，它们分别的定义如下：

frameworks/base/core/res/res/values/attrs.xml
```xml
<declare-styleable name="View">
        <!-- <p>Whether or not the force dark feature is allowed to be applied to this View.
            <p>Setting this to false will disable the auto-dark feature on this View draws
            including any descendants.
            <p>Setting this to true will allow this view to be automatically made dark, however
            a value of 'true' will not override any 'false' value in its parent chain nor will
            it prevent any 'false' in any of its children. -->
    <attr name="forceDarkAllowed" format="boolean" />
</declare-styleable>

 <declare-styleable name="Theme">
        <!-- <p>Whether or not the force dark feature is allowed to be applied to this theme.
             <p>Setting this to false will disable the auto-dark feature on everything this
             theme is applied to along with anything drawn by any children of views using
             this theme.
             <p>Setting this to true will allow this view to be automatically made dark, however
             a value of 'true' will not override any 'false' value in its parent chain nor will
             it prevent any 'false' in any of its children. -->
        <attr name="forceDarkAllowed" format="boolean" />
    </declare-styleable>
```

一个是 View 级别的，一个是 Theme 级别的。

## 4.1. Theme 级别 forceDarkAllowed
从上面的总结来看，Theme 级别的开关优先级是最高的，控制粒度也最大，我们看看源码里面使用它的地方

```java
    // frameworks/base/core/java/android/view/ViewRootImpl.java
    private void updateForceDarkMode() {
        // 渲染线程为空，直接返回
        if (mAttachInfo.mThreadedRenderer == null) return;

        // 系统是否打开了黑暗模式
        boolean useAutoDark = getNightMode() == Configuration.UI_MODE_NIGHT_YES;

        if (useAutoDark) {
            // forceDarkAllowed 默认值，开发者模式是否打开了强制 smart dark 选项
            boolean forceDarkAllowedDefault =
                    SystemProperties.getBoolean(ThreadedRenderer.DEBUG_FORCE_DARK, false);
            TypedArray a = mContext.obtainStyledAttributes(R.styleable.Theme);
            // useAutoDark = 使用浅色主题 && 主题中声明的 forceDarkAllowed 值
            useAutoDark = a.getBoolean(R.styleable.Theme_isLightTheme, true)
                    && a.getBoolean(R.styleable.Theme_forceDarkAllowed, forceDarkAllowedDefault);
            a.recycle();
        }

        // 关键代码，设置是否强制夜间模式
        if (mAttachInfo.mThreadedRenderer.setForceDark(useAutoDark)) {
            // TODO: Don't require regenerating all display lists to apply this setting
            invalidateWorld(mView);
        }
    }

    // frameworks/base/graphics/java/android/graphics/HardwareRenderer.java
    public boolean setForceDark(boolean enable) {
        if (mForceDark != enable) {
            mForceDark = enable;
            // native 代码，mNativeProxy 其实是  RenderThread 代理类的指针
            nSetForceDark(mNativeProxy, enable);
            return true;
        }
        return false;
    }
```

这段代码还是比较简单，判断系统：
 - 是否打开了夜间模式
 - 是否使用浅色主题
 - Theme_forceDarkAllowed 是否为 true

三者同时为 true 时才会设置夜间模式，而 updateForceDarkMode 调用的时机分别是在 `ViewRootImpl#setView` 和 `ViewRootImpl#updateConfiguration`，也就是初始化和夜间模式切换的时候都会调用，确保夜间模式能及时启用和关闭。继续跟踪 `HardwareRenderer#setForceDark` 发现，这是一个 native 方法，所以接下来让我们进入 native 世界，nSetForceDark 对应的实现位于



```cpp
// frameworks/base/core/jni/android_view_ThreadedRenderer.cpp
static void android_view_ThreadedRenderer_setForceDark(JNIEnv* env, jobject clazz,
        jlong proxyPtr, jboolean enable) {
    RenderProxy* proxy = reinterpret_cast<RenderProxy*>(proxyPtr);
    proxy->setForceDark(enable);
}

// frameworks/base/libs/hwui/renderthread/RenderProxy.cpp
void RenderProxy::setForceDark(bool enable) {
    mRenderThread.queue().post([this, enable]() { mContext->setForceDark(enable); });
}

// frameworks/base/libs/hwui/renderthread/CanvasContext.h
class CanvasContext : public IFrameCallback {
public:
  
    ...
    
    void setForceDark(bool enable) { mUseForceDark = enable; }

    bool useForceDark() {
        return mUseForceDark;
    }

    ...

private:
   
    ...
    // 默认关闭强制夜间模式
    bool mUseForceDark = false;
   
    ...
};
```

最终就是设置了一个 CanvasContext 的变量值而已，什么都还没有做，那么这个变量值的作用是什么，什么时候生效呢？我们进一步查看使用的地方：

```cpp
// frameworks/base/libs/hwui/TreeInfo.cpp
TreeInfo::TreeInfo(TraversalMode mode, renderthread::CanvasContext& canvasContext)
        : mode(mode)
        , prepareTextures(mode == MODE_FULL)
        , canvasContext(canvasContext)
        , damageGenerationId(canvasContext.getFrameNumber())
        // 初始化 TreeInfo 的 disableForceDark 变量，注意变量值意义的变化，0 代表打开夜间模式，>0 代表关闭夜间模式
        , disableForceDark(canvasContext.useForceDark() ? 0 : 1)
        , screenSize(canvasContext.getNextFrameSize()) {}

}
```

进一步看看 disableForceDark 使用的地方
```cpp
// frameworks/base/libs/hwui/RenderNode.cpp
/**
 * 这个可以说是核心方法了，handleForceDark 方法调用栈如下：
 * - RenderNode#prepareTreeImpl
 * - RenderNode#pushStagingDisplayListChanges
 * - RenderNode#syncDisplayList
 * - RenderNode#handleForceDark
 * 
 * 而 RenderNode#prepareTree 是绘制的必经之路，每一个节点都会走一遍这个流程
 */
void RenderNode::handleForceDark(android::uirenderer::TreeInfo *info) {
    // 若没打开强制夜间模式，直接退出
    if (CC_LIKELY(!info || info->disableForceDark)) {
        return;
    }

    // 根据是否有文字、是否有子节点、子节点数量等情况，得出当前 Node 属于 Foreground 还是 Background
    auto usage = usageHint();
    const auto& children = mDisplayList->mChildNodes;
    if (mDisplayList->hasText()) {
        usage = UsageHint::Foreground;
    }
    if (usage == UsageHint::Unknown) {
        if (children.size() > 1) {
            usage = UsageHint::Background;
        } else if (children.size() == 1 &&
                children.front().getRenderNode()->usageHint() !=
                        UsageHint::Background) {
            usage = UsageHint::Background;
        }
    }
    if (children.size() > 1) {
        // Crude overlap check
        SkRect drawn = SkRect::MakeEmpty();
        for (auto iter = children.rbegin(); iter != children.rend(); ++iter) {
            const auto& child = iter->getRenderNode();
            // We use stagingProperties here because we haven't yet sync'd the children
            SkRect bounds = SkRect::MakeXYWH(child->stagingProperties().getX(), child->stagingProperties().getY(),
                    child->stagingProperties().getWidth(), child->stagingProperties().getHeight());
            if (bounds.contains(drawn)) {
                // This contains everything drawn after it, so make it a background
                child->setUsageHint(UsageHint::Background);
            }
            drawn.join(bounds);
        }
    }

    // 根据 UsageHint 设置变色策略：Dark（压暗）、Light（提亮）
    mDisplayList->mDisplayList.applyColorTransform(
            usage == UsageHint::Background ? ColorTransform::Dark : ColorTransform::Light);
}
```

```cpp
// frameworks/base/libs/hwui/RecordingCanvas.cpp
void DisplayListData::applyColorTransform(ColorTransform transform) {
    // transform: Dark 或 Light
    // color_transform_fns 是一个对应所有绘制指令的函数指针数组，主要是对 op 的 paint 变色或对 bitmap 添加 colorfilter
    this->map(color_transform_fns, transform);
}

template <typename Fn, typename... Args>
inline void DisplayListData::map(const Fn fns[], Args... args) const {
    auto end = fBytes.get() + fUsed;
    // 遍历当前的绘制的 op
    for (const uint8_t* ptr = fBytes.get(); ptr < end;) {
        auto op = (const Op*)ptr;
        auto type = op->type;
        auto skip = op->skip;
        // 根据 type 找到对应的 fn，根据调用关系，我们知道 fns 数组对应 color_transform_fns，这个数组其实是一个函数指针数组，下面看看定义
        if (auto fn = fns[type]) {  // We replace no-op functions with nullptrs
            // 执行 
            fn(op, args...);        // to avoid the overhead of a pointless call.
        }
        ptr += skip;
    }
}

#define X(T) colorTransformForOp<T>(),
static const color_transform_fn color_transform_fns[] = {
        X(Flush)
        X(Save)
        X(Restore)
        X(SaveLayer)
        X(SaveBehind)
        X(Concat)
        X(SetMatrix)
        X(Translate)
        X(ClipPath)
        X(ClipRect)
        X(ClipRRect)
        X(ClipRegion)
        X(DrawPaint)
        X(DrawBehind)
        X(DrawPath)
        X(DrawRect)
        X(DrawRegion)
        X(DrawOval)
        X(DrawArc)
        X(DrawRRect)
        X(DrawDRRect)
        X(DrawAnnotation)
        X(DrawDrawable)
        X(DrawPicture)
        X(DrawImage)
        X(DrawImageNine)
        X(DrawImageRect)
        X(DrawImageLattice)
        X(DrawTextBlob)
        X(DrawPatch)
        X(DrawPoints)
        X(DrawVertices)
        X(DrawAtlas)
        X(DrawShadowRec)
        X(DrawVectorDrawable)
};
#undef X

```

color_transform_fn 宏定义展开
```cpp
template <class T>
constexpr color_transform_fn colorTransformForOp() {
    if
        // op 变量中是否同时包含 paint 及 palette 属性，若同时包含，则是绘制 Image 或者 VectorDrawable 的指令
        // 参考：frameworks/base/libs/hwui/RecordingCanvas.cpp 中各 Op 的定义
        constexpr(has_paint<T> && has_palette<T>) {
        
            return [](const void* opRaw, ColorTransform transform) {
                const T* op = reinterpret_cast<const T*>(opRaw);
                // 关键变色方法，根据 palette 叠加 colorfilter
                transformPaint(transform, const_cast<SkPaint*>(&(op->paint)), op->palette);
            };
        }
    else if
        // op 变量中是否包含 paint 属性，普通绘制指令
        constexpr(has_paint<T>) {
            return [](const void* opRaw, ColorTransform transform) {
                const T* op = reinterpret_cast<const T*>(opRaw);
                // 关键变色方法，对 paint 颜色进行变换
                transformPaint(transform, const_cast<SkPaint*>(&(op->paint)));
            };
        }
    else {
        // op 变量不包含 paint 属性，返回空
        return nullptr;
    }
}

static const color_transform_fn color_transform_fns[] = {
        // 根据 Flush、Save、DrawImage等不同绘制 op，返回不同的函数指针
        colorTransformForOp<Flush>
        ...
};
```

让我们再一次看看 map 方法
```cpp
template <typename Fn, typename... Args>
inline void DisplayListData::map(const Fn fns[], Args... args) const {
    auto end = fBytes.get() + fUsed;
    for (const uint8_t* ptr = fBytes.get(); ptr < end;) {
        auto op = (const Op*)ptr;
        auto type = op->type;
        auto skip = op->skip;
        if (auto fn = fns[type]) {  // We replace no-op functions with nullptrs
            // 对 op 的 paint 进行颜色变换或叠加 colorfilter
            fn(op, args...);        // to avoid the overhead of a pointless call.
        }
        ptr += skip;
    }
}
```

贴了一大段代码，虽然代码中已经包含了注释，但还是可能比较晕，我们先来整理下：
 - CanvasContext.mUseForceDark 只会影响 TreeInfo.disableForceDark 的初始化
 - TreeInfo.disableForceDark 若大于 0，RenderNode 在执行 handleForceDark 就会直接退出
 - handleForceDark 方法里会根据 UsageHint 类型，对所有 op 中的 paint 颜色进行变换，如果是绘制图片，则叠加一个反转的 colorfilter。变换策略有：Dark、Light

接下来让我们来看 paint 和 colorfilter 的变色实现
```cpp
bool transformPaint(ColorTransform transform, SkPaint* paint) {
    applyColorTransform(transform, *paint);
    return true;
}

static void applyColorTransform(ColorTransform transform, SkPaint& paint) {
    if (transform == ColorTransform::None) return;

    // 对画笔颜色进行颜色变换
    SkColor newColor = transformColor(transform, paint.getColor());
    paint.setColor(newColor);

    if (paint.getShader()) {
        SkShader::GradientInfo info;
        std::array<SkColor, 10> _colorStorage;
        std::array<SkScalar, _colorStorage.size()> _offsetStorage;
        info.fColorCount = _colorStorage.size();
        info.fColors = _colorStorage.data();
        info.fColorOffsets = _offsetStorage.data();
        SkShader::GradientType type = paint.getShader()->asAGradient(&info);

        if (info.fColorCount <= 10) {
            switch (type) {
                case SkShader::kLinear_GradientType:
                    for (int i = 0; i < info.fColorCount; i++) {
                        // 对 shader 中的颜色进行颜色变换
                        info.fColors[i] = transformColor(transform, info.fColors[i]);
                    }
                    paint.setShader(SkGradientShader::MakeLinear(info.fPoint, info.fColors,
                                                                 info.fColorOffsets, info.fColorCount,
                                                                 info.fTileMode, info.fGradientFlags, nullptr));
                    break;
                default:break;
            }

        }
    }

    if (paint.getColorFilter()) {
        SkBlendMode mode;
        SkColor color;
        // TODO: LRU this or something to avoid spamming new color mode filters
        if (paint.getColorFilter()->asColorMode(&color, &mode)) {
            // 对 colorfilter 中的颜色进行颜色变换
            color = transformColor(transform, color);
            paint.setColorFilter(SkColorFilter::MakeModeFilter(color, mode));
        }
    }
}
```

逻辑很简单，就是对颜色进行变换，进一步看看变色逻辑：
```cpp
// 提亮颜色
static SkColor makeLight(SkColor color) {
    // 转换成 Lab 模式
    Lab lab = sRGBToLab(color);
    // 对明度进行反转，明度越高，反转后越低
    float invertedL = std::min(110 - lab.L, 100.0f);
    if (invertedL > lab.L) {
        // 反转后的明度高于原明度，则使用反转后的明度
        lab.L = invertedL;
        return LabToSRGB(lab, SkColorGetA(color));
    } else {
        return color;
    }
}

// 压暗颜色
static SkColor makeDark(SkColor color) {
    // 转换成 Lab 模式
    Lab lab = sRGBToLab(color);
    // 对明度进行反转，明度越高，反转后越低
    float invertedL = std::min(110 - lab.L, 100.0f);
    if (invertedL < lab.L) {
        // 反转后的明度低于原明度，则使用反转后的明度
        lab.L = invertedL;
        // 使用 rgb 格式返回
        return LabToSRGB(lab, SkColorGetA(color));
    } else {
        // 直接返回原颜色
        return color;
    }
}

static SkColor transformColor(ColorTransform transform, SkColor color) {
    switch (transform) {
        case ColorTransform::Light:
            return makeLight(color);
        case ColorTransform::Dark:
            return makeDark(color);
        default:
            return color;
    }
}
```

到此，对 paint 的变换结束，看来无非就是反转明度。

再来看看对图片的变换：
```cpp
bool transformPaint(ColorTransform transform, SkPaint* paint, BitmapPalette palette) {
    // 根据 palette 和 colorfilter 判断图片是亮还是暗的
    palette = filterPalette(paint, palette);
    bool shouldInvert = false;
    if (palette == BitmapPalette::Light && transform == ColorTransform::Dark) {
        // 图片本身是亮的，但是要求变暗，反转
        shouldInvert = true;
    }
    if (palette == BitmapPalette::Dark && transform == ColorTransform::Light) {
        // 图片本身是暗的，但是要求变亮，反转
        shouldInvert = true;
    }
    if (shouldInvert) {
        SkHighContrastConfig config;
        config.fInvertStyle = SkHighContrastConfig::InvertStyle::kInvertLightness;
        // 叠加一个亮度反转的 colorfilter
        paint->setColorFilter(SkHighContrastFilter::Make(config)->makeComposed(paint->refColorFilter()));
    }
    return shouldInvert;
}
```
终于，bitmap 的变换也分析完了，呼~

## 4.2. View 级别 forceDarkAllowed

但是，还没完呢~~~还记得我们最开始说的，除了 Theme 级别，还有一个 View 级别的 forceDarkAllowed，通过 View 级别 forceDarkAllowed 可以关掉它及它的子 view 的夜间模式开关。依然从 java 层看下去哈

```java
// rameworks/base/core/java/android/view/View.java
public class View implements Drawable.Callback, KeyEvent.Callback,
        AccessibilityEventSource {

    public View(Context context, @Nullable AttributeSet attrs, int defStyleAttr, int defStyleRes) {
        final TypedArray a = context.obtainStyledAttributes(
                attrs, com.android.internal.R.styleable.View, defStyleAttr, defStyleRes);
        final int N = a.getIndexCount();
        for (int i = 0; i < N; i++) {
            int attr = a.getIndex(i);
            switch (attr) {
                case R.styleable.View_forceDarkAllowed:
                    // 注意，这个默认是 true 的
                    mRenderNode.setForceDarkAllowed(a.getBoolean(attr, true));
                    break;
            }
        }
    }
}

// frameworks/base/graphics/java/android/graphics/RenderNode.java
public final class RenderNode {
    public boolean setForceDarkAllowed(boolean allow) {
        // 又是 native 方法
        return nSetAllowForceDark(mNativeRenderNode, allow);
    }
}
```

```cpp
// frameworks/base/core/jni/android_view_RenderNode.cpp
static jboolean android_view_RenderNode_setAllowForceDark(jlong renderNodePtr, jboolean allow) {
    return SET_AND_DIRTY(setAllowForceDark, allow, RenderNode::GENERIC);
}

// frameworks/base/libs/hwui/RenderProperties.h
class ANDROID_API RenderProperties {
public:
    bool setAllowForceDark(bool allow) {
        // 设置到 mPrimitiveFields.mAllowForceDark 变量中
        return RP_SET(mPrimitiveFields.mAllowForceDark, allow);
    }

    bool getAllowForceDark() const {
        return mPrimitiveFields.mAllowForceDark;
    }
}  
```

和 Theme 级别的一样，仅仅只是设置到变量中而已，关键是要看哪里使用这个变量，经过查找，我们发现，它的使用同样在 RenderNode 的 prepareTreeImpl 中：
```cpp
void RenderNode::prepareTreeImpl(TreeObserver& observer, TreeInfo& info, bool functorsNeedLayer) {

     ...
    // 1. 如果 view 关闭了夜间模式，会在这里让 info.disableForceDark 加 1
    // 2. info.disableForceDark 正是 handleForceDark 中关键变量，还记得吗？
    // 3. nfo.disableForceDark 大于 0 会让此 RenderNode 跳过夜间模式处理
    // 4. 如果 info.disableForceDark 本身已经大于 0 了，view.setForceDarkAllowed(true) 也毫无意义
    if (!mProperties.getAllowForceDark()) {
        info.disableForceDark++;
    }

    prepareLayer(info, animatorDirtyMask);
    if (info.mode == TreeInfo::MODE_FULL) {
        // 这里面会调用 handleForceDark 方法处理夜间模式
        pushStagingDisplayListChanges(observer, info);
    }

    if (mDisplayList) {
        info.out.hasFunctors |= mDisplayList->hasFunctor();
        // 递归调用子 Node 的 prepareTreeImpl 方法
        bool isDirty = mDisplayList->prepareListAndChildren(
                observer, info, childFunctorsNeedLayer,
                [](RenderNode* child, TreeObserver& observer, TreeInfo& info,
                   bool functorsNeedLayer) {
                    child->prepareTreeImpl(observer, info, functorsNeedLayer);
                });
        if (isDirty) {
            damageSelf(info);
        }
    }

    ...
    // 重要，把 info.disableForceDark 恢复回原来的值，不让它影响 Tree 中同级的其他 RenderNode
    // 但是本 RenderNode 的子节点还是会受影响的，这就是为什么父 view 关闭了夜间模式，子 view 也会受影响的原因
    // 因为还原 info.disableForceDark 操作是在遍历子节点之后执行的
    if (!mProperties.getAllowForceDark()) {
        info.disableForceDark--;
    }
    ...
}
```

# 5. 总结
本文到目前为止，总算把 Android Q 夜间模式实现原理梳理了一遍，总的来说实现不算复杂，说白了就是把 paint 中的颜色转换一下或者叠加一个 colorfilter，虽然中间还有关联知识没有细说，如 RenderThread、DisplayList、RenderNode 等图形相关的概念，限于文章大小，请读者自行了解

另外，由于水平有限，难免文中有错漏之处，若哪里写的不对，请大家及时指出，蟹蟹啦~









