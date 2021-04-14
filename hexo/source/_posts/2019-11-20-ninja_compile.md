---
layout: post
title: 使用 Ninja 提升模块编译速度
tags: [Android]
categories: 
- Android
---

# 1. 简介

从 Android 7 开始，Android 源码编译时默认使用 Ninja，编译时，会先把 makefile 和 bp 转换成 ninja 再进行编译。这个转换过程非常慢（需要遍历处理所有关联的 makefile、bp 文件），即使只是通过 `mm` 或 `mmm` 编译某个模块，也会有很多因素触发 ninja 文件的重新生成，而这对基于源码开发的模块很不友好，编译好慢！

<!-- more -->

# 2. 初识 ninja

AOSP 在源码中已经内置了一个 ninja 执行文件，路径为：`./prebuilts/build-tools/linux-x86/bin/ninja`

我们先看看它的 help：

```bash
➜  ~ ./prebuilts/build-tools/linux-x86/bin/ninja -h
usage: ninja [options] [targets...]

if targets are unspecified, builds the 'default' target (see manual).

options:
  --version      print ninja version ("1.9.0")
  -v, --verbose  show all command lines while building

  -C DIR   change to DIR before doing anything else
  -f FILE  specify input build file [default=build.ninja]

  -j N     run N jobs in parallel (0 means infinity) [default=10 on this system]
  -k N     keep going until N jobs fail (0 means infinity) [default=1]
  -l N     do not start new jobs if the load average is greater than N
  -n       dry run (don't run commands but act like they succeeded)

  -d MODE  enable debugging (use '-d list' to list modes)
  -t TOOL  run a subtool (use '-t list' to list subtools)
    terminates toplevel options; further flags are passed to the tool
  -w FLAG  adjust warnings (use '-w list' to list warnings)
```

简单使用的话，我们关注它的两个参数就行了
 - `-f`：这个参数指定的就是输入文件，也就是 makefile 和 bp 转换后的 ninja 文件，一般位于 `./out` 目录，后面会说
 - `targets`：目标，这个和 makefile 是类似的，就是我们最终需要的产物，例如：Launcher3QuickStep、SystemUI。那么这些 targets 名是哪里定义的呢？要知道对应模块的对应的 target 名，只需要：
    - 若模块使用的是 Android.mk：查找 `LOCAL_PACKAGE_NAME` 或 `LOCAL_MODULE` 等对应的值
    - 若模块使用的是 Android.bp：查找 module 中 name 对应的值


举个栗子：

```bash
➜ android-10.0.0_r11 ./prebuilts/build-tools/linux-x86/bin/ninja -f out/combined-aosp_walleye.ninja Launcher3QuickStep
[3/13] Target Java: out/target/common/obj/JAVA_LIBRARIES/Launcher3QuickStepLib_intermediates/classes-full-debug.jar
注: 某些输入文件使用或覆盖了已过时的 API。
注: 有关详细信息, 请使用 -Xlint:deprecation 重新编译。
注: 某些输入文件使用了未经检查或不安全的操作。
注: 有关详细信息, 请使用 -Xlint:unchecked 重新编译。
[13/13] Install: out/target/product/walleye/system/product/priv-app/Launcher3QuickStep/Launcher3QuickStep.apk
```

就这样，不需要通过 `mm` 或者 `mmm` 命令，目标产物同样生成了。我们看看耗时：
```
➜ android-10.0.0_r11 time ./prebuilts/build-tools/linux-x86/bin/ninja -f out/combined-aosp_walleye.ninja Launcher3QuickStep
[3/13] Target Java: out/target/common/obj/JAVA_LIBRARIES/Launcher3QuickStepLib_intermediates/classes-full-debug.jar
注: 某些输入文件使用或覆盖了已过时的 API。
注: 有关详细信息, 请使用 -Xlint:deprecation 重新编译。
注: 某些输入文件使用了未经检查或不安全的操作。
注: 有关详细信息, 请使用 -Xlint:unchecked 重新编译。
[13/13] Install: out/target/product/walleye/system/product/priv-app/Launcher3QuickStep/Launcher3QuickStep.apk

real	0m18.994s
user	1m20.548s
sys	0m2.872s
```

可以看到，整个编译在 18s 完成了，相比动辄七八分钟的 `mmm`，效率提升还是很可观的。

## 3. 注意事项

虽然 ninja 很方便，但要用它来编译单个模块，还是有一些限制和注意事项的：

 - 使用前需把对应模块编译一遍，用于生产 ninja 文件（全编或 `mmm` 都可以）
 - 全编后，生成的 ninja 文件为：`./out/combined-[TARGET-PRODUCT].ninja`
 - `mmm` 编译后，生成的 ninja 文件为：`./out/combined-[TARGET-PRODUCT]-_[path_to_your_module_makefile].ninja`，比如：`./out/combined-aosp_walleye-_packages_apps_Launcher3_Android.mk.ninja`
 - 如果修改了 Android.mk 或 Android.bp，需使用传统的 make 命令进行编译以重新生成包含新依赖规则的 ninja 文件
 - 可以把 ninja 放到 PATH 环境变量中，这样就不用每次都敲 ./prebuilts/build-tools/linux-x86/bin/ninja 这个路径了

## 4. 最后

为 Launcher 和 SystemUI 准备一份开箱即用的指令，尽情玩耍吧~

Launcher：
```
./prebuilts/build-tools/linux-x86/bin/ninja -f out/combined-qssi-_packages_apps_Launcher3_Android.mk.ninja Launcher3QuickStep
```

SystemUI：
```
./prebuilts/build-tools/linux-x86/bin/ninja -f out/combined-qssi-_frameworks_base_packages_SystemUI_Android.mk.ninja SystemUI
```








