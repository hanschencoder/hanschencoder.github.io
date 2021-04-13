---
layout: post
title: 如何顺滑地查看 Android Native 代码
tags: [Android]
categories: 
- Android
---

# 1. 简介
使用 Android Studio 查看 Android Framework 代码体验非常好，无论是索引还是界面都让人很满意，但是当你跟踪代码，发现进入 native 逻辑时，就会发现 Android Studio 对 native 代码的支持非常不好，不能索引不支持符号搜索不能跳转等，这些让人非常抓狂。那么如何能在 IDE 愉快地查看 native 代码呢？在 Windows 上，Source Insight 的表现也很好，但苦于只有 Windows 平台支持且界面不好，经过一番折腾，还真是找到了方法，下面我们将一步一步打造丝滑的  native 代码阅读环境。

先看一下效果：

![2019-10-11-15-02-40.gif](http://image.hanschen.site/master/2019-10-11-15-02-40.gif)


# 2. CMake

能让 IDE 正确地建立索引，我们需要让 IDE 能正确地知道源文件、头文件、宏定义等各种数据，庆幸的是，我们发现 AOSP 在编译过程中，可以帮我们生成这些数据，详见：[http://androidxref.com/9.0.0_r3/xref/build/soong/docs/clion.md](http://androidxref.com/9.0.0_r3/xref/build/soong/docs/clion.md)

通过文档我们可知，只需要按照以下步骤完成一次编译，即可自动生成各模块对应的 CMake 文件。至于 Cmake 文件是什么，这里就不做赘述了，大家可以自行了解。

1. 打开以下两个开关，CMakeLists.txt 就会根据编译环境自动生成

```bash
export SOONG_GEN_CMAKEFILES=1
export SOONG_GEN_CMAKEFILES_DEBUG=1
```

2. 启动编译

```bash
make -j16
```

或者只编译你需要的模块

```bash
make frameworks/native/service/libs/ui
```

生成的文件存放在 out 目录，比如刚刚编译的 libui 模块对应的路径为：
```bash
out/development/ide/clion/frameworks/native/libs/ui/libui-arm64-android/CMakeLists.txt
```

3. 合并多个模块

生成了 CMake 后，我们发现，CMake 文件是按模块生成的。这样的话，会导致 IDE 只能单独导入一个模块，而我们平时不可能只看一个模块的代码，如果把多个模块都 include 进来呢？
我们可以在 `out/development/ide/clion` 路径新建一个 `CMakeLists.txt` 文件，并添加一下内容：

```CMake
# 指定 CMake 最低版本
cmake_minimum_required(VERSION 3.6)
# 指定工程名，随意
project(aosp)
# 把你需要的模块通过 add_subdirectory 添加进来，注意子目录必须也包含 CMakeLists.txt 文件
add_subdirectory(frameworks/native)
#add_subdirectory(frameworks/base/core/jni/libandroid_runtime-arm64-android)
```

这样，我们就把多个模块合并在一起了，用 IDE 去打开这个总的 CMake 文件即可


# 3. 导入 IDE

只要生成 CMake 文件后，剩下的事情就好办了，现在能识别 CMake 工程的 IDE 非常多，大家可以根据个人喜好选择，如：

 - CLion
 - Eclipse
 - Visual Studio
 - ...

这里以 CLion 为例讲一下如何导入
  1. 打开 CLion
  2. 选择「New CMake Project from Sources」
  3. 指定包含 `CMakeLists.txt` 的目录，如我们在上一个步骤中说的 `out/development/ide/clion`（这个目录的 CMakeLists.txt 包含了多个模块，还记得吗？）
  4. 选择「Open Existing Project」
  5. Enjoy your journey ...

当然，CLion 也有一个缺点，收费！！如何能免费使用就看大家各显神通了

# 4. 遇到的一些问题

 - 生成的 CMakeLists.txt 里指定路径可能会使用绝对路径，如： `set(ANDROID_ROOT /Volumes/AndroidSource/M1882_QOF7_base)`，这里大家要注意，如果把 CMakeLists.txt 拷贝到别的工程使用，记得修正一下路径
 - Mac 用户留意，如果你的 CMakeLists.txt 是从 linux 平台生成拷贝过来的，生成的 CMakeLists.txt 里指定的 c++ 编译器 `set(CMAKE_CXX_COMPILER "${ANDROID_ROOT}/prebuilts/clang/host/linux-x86/clang-3977809/bin/clang++")` 这里指定的是 linux-x86 的编译器，记得替换成 `darwin-x86`，如果对应目录下没有 clang++，那就从 AOSP 源码拷一个吧
 - 如果 CMake 中列出的源文件在工程中找不到，会导致 CLion 停止索引，如果出现不一致的时候，移除 CMake 中源文件的声明即可

如果使用遇到其他问题，欢迎联系告知，谢谢

# 5. 总结

所谓工欲善其事，必先利其器。通过这种方法建立的索引包含了 AOSP 所有模块，最重要是它还会根据编译环境，把相关 FLAGS 和宏都设置好。












