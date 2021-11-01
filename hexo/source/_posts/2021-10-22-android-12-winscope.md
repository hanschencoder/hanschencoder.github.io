---
layout: post
title: Android 12 - 跟踪利器 WinScope 
tags: [Android,Android 12,Android S]
categories: 
- Android
---

# 1. 简介

在开发过程中，经常会遇到各种各样的窗口问题，比如动画异常、窗口异常、闪屏、黑屏、错位显示..

以前对于这些问题，我们可以通过添加日志，调试分析代码等手段去解决，但这些 UI 问题往往出现在一瞬间，很难把握出现的时机，录制下来的日志往往也是巨大的，从海量的日志中提取有效的信息是一个枯燥且繁琐的事情。

Android 也意识到了这个问题，`WinScope ` 的出现有效的帮助我们跟踪窗口和显示问题。它向开发者提供一个可视化的工具，让开发者能使用工具跟踪整个界面的变化过程， 让我们可以观察到细微的变化。 迭代了几个版本后， Android 12 上 WinScope 变得更好用了，下面来看看大概的效果：

![2021-10-22-14-26-57](http://image.hanschen.site/master/2021-10-22-14-26-57.png)

<!-- more -->

# 2. 工具获取

Android 12 平台的 `WinScope` 工具可以通过源码编译获得， 具体也可以查阅 `development/tools/winscope` 目录下的 README.md 文档， 这里提供一个 Ubuntu 平台的编译步骤：

```bash
1. cd development/tools/winscope
2. sudo apt install nodejs npm
3. npm install -g yarn
4. yarn install
5. yarn build
```

编译过程中遇到一个问题， 看上去是在执行 kotlin 优化的时候， 报了个内存不足的问题：

```bash
Caused by: java.lang.OutOfMemoryError: GC overhead limit exceeded
	at com.google.gwt.dev.js.ScopeContext.referenceFor(ScopeContext.kt:68)
	at com.google.gwt.dev.js.JsAstMapper.mapAsPropertyNameRef(JsAstMapper.java:247)
	at com.google.gwt.dev.js.JsAstMapper.mapGetProp(JsAstMapper.java:608)
	at com.google.gwt.dev.js.JsAstMapper.mapWithoutLocation(JsAstMapper.java:138)
	at com.google.gwt.dev.js.JsAstMapper.map(JsAstMapper.java:47)
	at com.google.gwt.dev.js.JsAstMapper.mapExpression(JsAstMapper.java:466)
	at com.google.gwt.dev.js.JsAstMapper.mapBinaryOperation(JsAstMapper.java:304)
	at com.google.gwt.dev.js.JsAstMapper.mapAssignmentVariant(JsAstMapper.java:258)
	at com.google.gwt.dev.js.JsAstMapper.mapWithoutLocation(JsAstMapper.java:102)
	at com.google.gwt.dev.js.JsAstMapper.map(JsAstMapper.java:47)

```

可以在执行 yarn build 前通过 `export JAVA_OPTS="-XX:-UseGCOverheadLimit"` 禁用掉 `GC overhead limit exceeded` 检测

编译完之后，在当前目录下会一个 `dist` 目录， 再把 `adb_proxy/winscope_proxy.py`(一个帮我们开启 trace 抓取命令的脚本， 这样我们就可以告别繁琐的命令啦)， 文件也拷贝进 `dist` 目录方便我们后面使用

# 3. 使用

使用比较简单了， 连接手机后： 

 - 双击打开 `dist` 目录下的 `index.html` 文件
 - 在终端执行 `python3 winscope_proxy.py`


PS： 这里建议大家可以设置个 alias 一键使用，比如：

```bash
alias winscope_s="xdg-open ~/tools/winscope_s/index.html && python3 ~/tools/winscope_s/winscope_proxy.py"
```

输入 python 命令后，终端可能会生成一个 `token` ，把它复制到浏览器即可

接下来就会出现下面这个界面：

![2021-10-22-14-32-16](http://image.hanschen.site/master/2021-10-22-14-32-16.png)

在选择 `START TRACE` 之后， 就可以在手机端进行录制操作，操作完后结束录制即可

# 4. 功能改进

相比与 Android 11， 新的 `WinScope` 工具在界面上更友好了，重要的改进如下：

 - 时间线控制更加方便了，可以选择不同类型的时间线用于控制，否则像以前录制了 `ProtoLog` 后，通过箭头控制时间推移简直是太难用了
 - 新的 `Transaction` 和 `Log` 浏览界面
 - 支持 Diff 功能， 不得不说每一帧的参数太多了， 如果没有 diff 功能，实在很难一眼看出那些参数发生了变化
 - 新增了 IME 的录制， 可以录制 `InputMethodService`、`InputMethodManagerService` 和 `Client` 的事件
 - 录屏界面可以以 PIP 形式显示在浏览器界面上
 - ...

不得不说确实比以前好用一些，赶紧尝尝鲜吧~
