## 1.1 环境搭建

学习工具：

```
Android Studio、Vscode
NDK、ADB、IDA(keypatch)
```

学习目标：

```
1. 用户模式下(数据移动、基本整形运算、分支和调用)
2. 同步了解指令和机器码
```

常用命令：

```
C头文件信息目录：Library/Android/sdk/ndk/22.0.7026061/toolchains/llvm/prebuilt/darwin-x86_64/sysroot/usr/include
模拟器文件路径：/Users/xxx/.android/avd/
启动模拟器：emulator -avd Pixel_XL_API_16
ida调试：./android-server & （模拟器采取快照模式，使用后台启动，每次就不用进adb启动了）
端口映射：adb forward tcp:23946 tcp:23946
```

NDK编译：

```makefile
#Android.mk

LOCAL_PATH := $(call my-dir)
include $(CLEAR_VARS)
LOCAL_ARM_MODE := arm #采用arm还是thumb
LOCAL_MODULE := hello
LOCAL_SRC_FILES := hello.c
include $(BUILD_EXECUTABLE) #不编译成so,编译为可执行文件
```

```makefile
#Application.mk

APP_ABI := armeabi-v7a
APP_BUILD_SCRIPT := Android.mk
APP_PLATFORM := android-16
```

```
ndk-build NDK_PROJECT_PATH=. NDK_APPLICATION_MK=Application.mk
生成libs和obj文件:
1.libs会有一个hello的可执行文件
2.obj也会有一个hello的可执行文件，是debug版本
	adb push hello可执行文件到手机里进行调试
3./proc/2077/maps 内存分布
```

