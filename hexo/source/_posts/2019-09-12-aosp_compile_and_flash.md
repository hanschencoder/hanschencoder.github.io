---
layout: post
title: AOSP 编译和烧写
tags: [Android]
categories: 
- Android
---
# 1. 简介
很多 Android 开发者都会希望编译 Android 源码并刷进自己的手机里面，但网上教程很多都仅仅是告诉你 lunch、make 等等，但你手里有一台设备时却发现，你编译出的镜像由于驱动关系是不能直接烧进手机的。这里整理了一下步骤，帮助大家可以按照流程编译并烧写镜像。

本篇文章以 **Pixel 2 && Android 10** 为例

# 2. 环境准备
这块没啥说，官方教程就够了，参考：[https://source.android.com/setup/build/initializing](https://source.android.com/setup/build/initializing) 就行了

# 3. 源码下载

 1. 根据 [https://developers.google.com/android/drivers](https://developers.google.com/android/drivers) 选择一个设备对应 Android 版本号和驱动，比如我们选择：Android 10.0.0 (QP1A.190711.020)，下载驱动，记住 **Build 号**
 ![2019-9-12-16-53-40.png](https://gitee.com/hanschencoder/Images/raw/master/2019-9-12-16-53-40.png)


 2. 在 [https://source.android.com/setup/start/build-numbers](https://source.android.com/setup/start/build-numbers) 查找 QP1A.190711.020 对应的分支：android-10.0.0_r2，记住**分支名**
![2019-9-12-16-56-41.png](https://gitee.com/hanschencoder/Images/raw/master/2019-9-12-16-56-41.png)

 3. 下载 AOSP 源码
注意在下载 aosp 前要安装 repo 工具，参考：[https://source.android.com/setup/build/downloading](https://source.android.com/setup/build/downloading)
```bash
mkdir Pixel2
cd Pixel2
repo init -u https://android.googlesource.com/platform/manifest -b android-10.0.0_r2 --depth=1
repo sync -j8
repo start android-10.0.0_r2 --all
```

 4. 把步骤1中选中的两个驱动下载到 aosp 源码根目录并解压
 5. 分别执行解压后的文件，注意，执行后要同意 License，确保正确解压到 aosp 根目录的 vendor 目录
```bash
./extract-qcom-walleye.sh
./extract-google_devices-walleye.sh
```

# 4. 源码编译
 1. 在 aosp 源码根目录执行：source build/envsetup.sh（注意，执行前终端请选bash，不要使用zsh等，在终端键入bash回车即可）
 2. 在 aosp 源码根目录执行：lunch
 3. 选择对应的版本，比如 Pixel2 就选择：aosp_walleye-userdebug
 4. 执行：make -j8

# 5. 镜像烧写
 1. 编译完后，执行：`export ANDROID_PRODUCT_OUT=/home/chenhang/source/Pixel2/out/target/product/walleye`
 2. 执行：`fastboot flashall -w`
 3. 烧写完成后，执行：`fastboot reboot`

# 6. Gapps 安装
编译出来的 aosp 默认没有 google 全家桶，可以通过以下方式进行安装

 1. 在 [https://opengapps.org/](https://opengapps.org/) 根据系统版本、芯片类型选择需要的 Gapps 全家桶，可以选 stock 版本
 2. 下载后把全家桶 push 到手机 sdcard（不用解压）
 3. 在 [https://twrp.me/devices/](https://twrp.me/devices/) 搜索你的设备，如： [https://twrp.me/google/googlepixel2.html](https://twrp.me/google/googlepixel2.html)
![2019-9-12-17-3-22.png](https://gitee.com/hanschencoder/Images/raw/master/2019-9-12-17-3-22.png)
 4. 下载 twrp.img 后根据截图中的命令，把 twrp 加载到手机， 选择 install 刷入 twrp.zip (这是一个 recovery 版本)，重启后，通过 adb reboot recovery 进入 twrp 的recovery 系统
 5. 在手机上选择 install， 选择步骤2中 push 到手机 sdcard 的全家桶，安装结束后选择擦除 dalvik cache 和 cache，重启即可