---
layout: post
title: Android 12 - WMS 层级结构 && DisplayAreaGroup 引入
tags: [Android,Android 12,Android S]
categories: 
- Android
---


# 1. 简介

在 Android 窗口管理中，所有的窗口都是以树形数据结构进行组织管理的，认知这棵 WMS 的树有助于我们理解窗口的管理和显示，同时，WMS 的层级也决定了其在 `SurfaceFlinger` 的层级结构，这恰恰决定了它的显示规则。

# 2. WMS 顶层层级构建

在 Android 12 中，所有窗口树形管理都继基于 `WindowContainer，` 每个 `WindowContainer` 都有一个父节点和若干个子节点，我们先看看框架中 `WindowContainer` 都有哪些类型：

![2021-11-01-10-50-30](http://image.hanschen.site/master/2021-11-01-10-50-30.png)

 - RootWindowContainer： 最顶层的管理者，直接管理 DisplayContent
 - DisplayContent： 代表着一个真实或者虚拟的显示设备，在普遍场景中，系统中只存在一个 DisplayContent
 - TaskDisplayArea： 是系统中所有应用任务的父节点，用于管理 Task
 - Task： 代表着一个任务
 - ActivityRecord： 代表一个 Activity 节点
 - WallpaperWindowToken： 代表壁纸节点
 - ...


<!-- more -->

在开始之前大概整理了一下系统中各个节点之间的关系： 

![2021-11-01-11-35-41](http://image.hanschen.site/master/2021-11-01-11-35-41.png)


从上图可以看到，节点之间的嵌套关系还是比较复杂的（ 而且这还是不包括下面章节中提到的引入 Feature 之后的层级关系），层级的最顶端就是 `RootWindowContainer，` 而它的子节点只能是： `DisplayContent`

```java
// RootWindowContainer.java
void setWindowManager(WindowManagerService wm) {
    ...

    final Display[] displays = mDisplayManager.getDisplays();
    for (int displayNdx = 0; displayNdx < displays.length; ++displayNdx) {
        final Display display = displays[displayNdx];
        // 为每一个 Display 挂载一个 DisplayContent 节点
        final DisplayContent displayContent = new DisplayContent(display, this);
        addChild(displayContent, POSITION_BOTTOM);
        if (displayContent.mDisplayId == DEFAULT_DISPLAY) {
            mDefaultDisplay = displayContent;
        }
    }
    ...
}
```

再来看看 `DisplayContent` 的构造方法，核心逻辑就只有一句，依靠 `DisplayAreaPolicy` 进行层级初始化

```java
// DisplayContent.java
DisplayContent(Display display, RootWindowContainer root) {
    super(root.mWindowManager, "DisplayContent", FEATURE_ROOT);
    ...

    // 构造子节点层级，默认策略是使用 DisplayAreaPolicy.DefaultProvider
    mDisplayAreaPolicy = mWmService.getDisplayAreaPolicyProvider().instantiate(
            mWmService, this /* content */, this /* root */, mImeWindowsContainer);

    ...
}
```

```java
// DisplayAreaPolicy.java
static final class DefaultProvider implements DisplayAreaPolicy.Provider {
    @Override
    public DisplayAreaPolicy instantiate(WindowManagerService wmService,
            DisplayContent content, RootDisplayArea root,
            DisplayArea.Tokens imeContainer) {
        // 创建 TaskDisplayArea 节点，注意，这里是允许创建多个 TaskDisplayArea 并添加的
        final TaskDisplayArea defaultTaskDisplayArea = new TaskDisplayArea(content, wmService,
                "DefaultTaskDisplayArea", FEATURE_DEFAULT_TASK_CONTAINER);
        final List<TaskDisplayArea> tdaList = new ArrayList<>();
        tdaList.add(defaultTaskDisplayArea);

        final HierarchyBuilder rootHierarchy = new HierarchyBuilder(root);
        rootHierarchy.setImeContainer(imeContainer).setTaskDisplayAreas(tdaList);
        if (content.isTrusted()) {
            // 配置 Feature 及它所能影响的层级
            configureTrustedHierarchyBuilder(rootHierarchy, wmService, content);
        }

        // 根据配置的 Feature  生成并挂载各个节点，建造层级
        return new DisplayAreaPolicyBuilder().setRootHierarchy(rootHierarchy).build(wmService);
    }
}
```

在 Android 12 上，`Feature` 正式派上用场了，原生添加了以下 `Feature：`

 - WindowedMagnification: 屏幕放大功能，通過 SystemUI mirrorSurface 该节点实现内容拷贝，详见 `WindowMagnificationGestureHandler#toggleMagnification`
 - HideDisplayCutout
 - OneHandedBackgroundPanel
 - OneHanded
 - FullscreenMagnification： 屏幕放大功能，通过无障碍服务 `FullScreenMagnificationController.SpecAnimationBridge#setMagnificationSpecLocked` 最后调用 `DisplayContent#applyMagnificationSpec` 方法实现节点放大。不过源码中并不是通过这个 Feature 来实现相关层级放大的，改造得还不彻底
 - ImePlaceholder

我们知道，Android 系统是有 Z 轴概念的，不同的窗口有不同的高度，所有的窗口类型对应到 WMS 都会有一个 layer 值，layer 越大，显示在越上面，WMS 规定 1~36 层级，每一个 `Feature` 都指定了它所能影响到的 layer 层。这里用颜色对不同 `Feature` 能影响 layer 图层进行颜色标记：


![2021-11-01-14-35-46](http://image.hanschen.site/master/2021-11-01-14-35-46.png)


标记完之后，就需要根据图表生成窗口层级了，首先对标记好的图表进行上移，上移规则： 如果色块上方是空白的，则可以上移，直至上方是颜色块（不知道大家有没有玩过 2048 这款游戏，上移逻辑是一样的~）

![2021-11-01-14-45-24](http://image.hanschen.site/master/2021-11-01-14-45-24.png)

上移之后，我们得到了最终的图表，接下来用以下规则进行层级构建：

 - 同一行相邻的同色块变成一个 `Feature` 节点，从左到右根据颜色不断生成节点，同一行所有节点挂在同一个父节点下
 - 父节点就是垂直正上方一行的色块对应的节点
 - 为最末端所有 `Feature` 节点再添加一个节点，根据子节点代表的 layer 不一样，最后添加的节点也不一样
    - 除了 layer 是 2、15 和 16 外，挂载 `DisplayArea.Tokens`（这类节点后续只能添加 `WindowToken` 节点）
    - layer = 2 （也就是 `APPLICATION_LAYER`）的节点，挂载 `TaskDisplayArea`
    - layer 等于 15 和 16 的节点，挂载 `ImeContainer` 

通过上述构建规则后，我们可以获得一个树形的层级，并且这棵树有以下特点： 

 - 树的最末端节点对应一个 layer 范围，同一个 layer 值只有一个末端节点与之对应
 - 为所有 `Feature` 都生成了对应的父节点，用以控制其所能影响的 layer

生成了这棵树后，我们会保存两样东西：

 - 所有 layer 值对应的最末端节点，方便我们后续根据窗口类型添加节点
 - 以 `Map<Feature, List<DisplayArea>>`  形式保存的所有 `Feature` 节点，方便我们后取出某 `Feature` 对应的所有节点

现在，虽说我们的 WMS 层级是构建好了，但对于这些 `Feature` 有何作用还完全没有涉及，这块打算放在 `WM Shell` 专题里进行说明~~


# 3. DisplayAreaGroupture

通过上面 `Feature` 的说明可以知道，不同的 `Feature` 是父子节点的关系，那如果我想划分一个逻辑显示区域，对这块区域配置不同的 `Feature` 该如何呢？ 这时候就可以使用 `DisplayAreaGroup` 了，框架允许我们添加多个 `DisplayAreaGroup，` 并为其配置不同的 `Feature`。

就像原生提供的 demo 一样，我们可以创建两个 `DisplayAreaGroup` 并将屏幕一分为二分别放置这两个，这两个区域都是可以作为应用容器的，和分屏不一样的是，这两块区域可以有不同的 Feature 规则以及其他特性，比如设置不同的 `DisplayArea#setIgnoreOrientationRequest` 值


![2021-11-01-15-35-28](http://image.hanschen.site/master/2021-11-01-15-35-28.png)

`DisplayAreaGroup` 和 `DisplayContent` 都是 `RootDisplayArea` 的直接子类，`DisplayAreaGroup` 可以认为是一个 Display 划分出的多个`逻辑 Display` 吧。当然，AOSP 虽然引入了这个概念和代码，但其实并未使用，我们只能从测试代码 `DualDisplayAreaGroupPolicyTest` 中略窥一二了~      

# 4. 小结

WMS 相关的内容体系实在太多，本文也仅仅是分析 WMS 窗口层级最顶层的结构，对于具体的窗口添加移除管理这些尚未涉及，同样，原生新增的 Feature 节点使用也没有涉及（这大部分都被打包进 WM Shell 中去了）