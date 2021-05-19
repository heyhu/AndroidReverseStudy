## 1.1 基础开发

### 1.1.1 创建一个新的c文件

1. 在 `app/src/main/cpp`新建一个C/C++ Source File，type选择 `.c `。填入代码，点击Sync Now，同步代码。

   ```c
   #include <string.h>
   #include <jni.h>
   #include <stdio.h>
   #include <stdlib.h>
   #include <string.h>
   #include <stdint.h>
   #include <android/log.h>
   
   #define  TAG    "heyTag"
   
   #define LOGI(...) __android_log_print(ANDROID_LOG_INFO,TAG,__VA_ARGS__)
   
   /* This is a trivial JNI example where we use a native method
    * to return a new VM String. See the corresponding Java source
    * file located at:
    *
    *   hello-jni/app/src/main/java/com/example/hellojni/HelloJni.java
    */
   
   JNIEXPORT jstring JNICALL
   Java_com_heyhu_openso_MainActivity_stringFromJNI(JNIEnv *env, jobject thiz) {
   #if defined(__arm__)
   #if defined(__ARM_ARCH_7A__)
   #if defined(__ARM_NEON__)
   #if defined(__ARM_PCS_VFP)
   #define ABI "armeabi-v7a/NEON (hard-float)"
   #else
   #define ABI "armeabi-v7a/NEON"
   #endif
   #else
   #if defined(__ARM_PCS_VFP)
   #define ABI "armeabi-v7a (hard-float)"
   #else
   #define ABI "armeabi-v7a"
   #endif
   #endif
   #else
   #define ABI "armeabi"
   #endif
   #elif defined(__i386__)
   #define ABI "x86"
   #elif defined(__x86_64__)
   #define ABI "x86_64"
   #elif defined(__mips64)  /* mips64el-* toolchain defines __mips__ too */
   #define ABI "mips64"
   #elif defined(__mips__)
   #define ABI "mips"
   #elif defined(__aarch64__)
   #define ABI "arm64-v8a"
   #else
   #define ABI "unknown"
   #endif
   
       return (*env)->NewStringUTF(env, "Hello from JNI !  Compiled with ABI " ABI ".");
   }
   
   ```

2. 在CmakeLists.txt 声明新建的 `heyhu.c`文件。

   ```txt
   add_library( # Sets the name of the library.
                heyhu
   
                # Sets the library as a shared library.
                # 这里共享, 别的app就可以load此so文件
                SHARED
   
                # Provides a relative path to your source file(s).
                heyhu.c )
                
                
   target_link_libraries( # Specifies the target library.
                          heyhu
   
                          # Links the target library to the log library
                          # included in the NDK.
                          ${log-lib} )
   ```

3. 在 Java文件中声明加载的文件, 运行在手机上就可以看到效果了。

   ```java
       static {
           System.loadLibrary("heyhu");
       }
   ```

   

### 1.1.2  C语言md5[交叉]编译

编译本地版

1. [编写md5的c代码到本地](https://github.com/pod32g/MD5), 在github中找一个即可。
2. 到对应目录下执行gcc -o md5 md5.c`命令进行编译。
3. `./md5 heyhu` 得到`heyhu`的md5值。

将NDK与其他构建系统配合使用

1. `/Android/sdk/ndk/21.0.6113669/toolchains/llvm/prebuilt/darwin-x86_64/bin/clang -target aarch64-linux-android21  md5.clang`，会生成一个`a.out`可执行文件。注意c++ 需要使用 `clang ++`。
2. 因为我们选择的ABI位arm64-v8a，需要讲此文件push到手机上进行执行。对应关系详见[文档](https://developer.android.google.cn/ndk/guides/other_build_systems)。

移植到代码中: [heyhu.c](https://github.com/heyhu/openso/blob/master/app/src/main/cpp/heyhu.c)

Tips `如果安装后结果不对，可以删除/app/build/outputs/apk下的文件，修改成32位ABI即可`



### 1.1.3  指定32位

```java
android {
    compileSdkVersion 30
    buildToolsVersion "30.0.2"

    defaultConfig {
        applicationId "com.heyhu.openso"
        minSdkVersion 16
        targetSdkVersion 30
        versionCode 1
        versionName "1.0"

        testInstrumentationRunner "androidx.test.runner.AndroidJUnitRunner"
        externalNativeBuild {
            cmake {
                cppFlags ""
            }
            // 现在基本32位
            ndk {
                abiFilters 'armeabi-v7a'
            }

        }
    }
```



## 1.2 llvm组件lldb

llvm可以进行汇编、反汇编，机器码反汇编成16进制、指令集(disassemble)。llvm可以把C\C++代码编译成一个SO文件。

调试第三方App满足以下任意一种方式即可，推荐第二种方式：

 	1. app以debug模式、apk包中debuggable==true（得重打包、或者xposed/frida去hook）
 	2. 手机是aosp系统，编译成userdebug模式（n5x、sailfish）



### 1.2.1 Android Studio Debug调试原理

bebug的时候 会把llvm的组件lldb中的start_lldb_server.sh push到手机的`/data/local/tmp/`中，调试的时候会启动lldb服务。

如果之前使用Android Studio Debug过，它会自动帮我们push。

![](/Android/A03/pic/01.a.png)

如果没有需要手动把文件push到手机中，其路径为`/Users/zhaoshouxin/Library/Android/sdk/ndk/21.0.6113669/toolchains/llvm/prebuilt/darwin-x86_64/lib64/clang/9.0.8/lib/linux/aarch64或者arm以及x86_64`。aarch64位64位、arm为32位，以app进程为基准，不可混用。push完成后运行`./lldb-server`看是否可用。

lldb路径：`/Android/sdk/ndk/21.0.6113669/toolchains/llvm/prebuilt/darwin-x86_64/bin/lldb` 与clang在同一目录下，可以把目录设置为环境变量。

在Android Studio bebug模式下，旁边也有lldb切换卡，也可以操作lldb。



## 1.2.2 lldb命令

> 手机下启动lldb服务 ./lldb-server platform --listen "*:10086" --server

首先你需要在你的linux系统上安装lldb，然后执行lldb。进入后执行以下命令

```
platform select remote-android

#ENU7N16709000458 是adb devices显示的设备名称
platform connect connect://ENU7N16709000458:10086
attach -p 30827
```

详细文章[入口](https://my.oschina.net/u/4263893/blog/4349408)。



## 1.3 Frida Instruction

Capstone/Keystone(反)汇编器

作用：动态反汇编。原理内置Capstone

```javascript
function dis(address, number) {
    for (var i = 0; i < number; i++) {
        var ins = Instruction.parse(address);
        console.log("address:" + address + "--dis:" + ins.toString());
        address = ins.next;
    }
}
setImmediate(function(){
    var stringFromJniaddr = Module.findExportByName("libroysue.so","Java_com_roysue_easyso1_MainActivity_stringFromJNI")
    dis(stringFromJniaddr,10);
})
```

和lldb打印的一样。

![](/Android/A03/pic/02.a.png)



## 1.4 HyperPwn调试入门

### 1.4.1 app设置为可调式

```
CMakeLists.txt 最下面添加

# gdb 调试
SET(CMAKE_BUILD_TYPE "Debug")
SET(CMAKE_CXX_FLAGS_DEBUG "$ENV{CXXFLAGS} -O0 -Wall -g2 -ggdb")
SET(CMAKE_CXX_FLAGS_RELEASE "$ENV{CXXFLAGS} -O3 -Wall")
```

```
app下的build.gradle中android添加

    packagingOptions{
        doNotStrip "*/armeabi/*.so"
        doNotStrip "*/armeabi-v7a/*.so"
        doNotStrip "*/x86/*.so"
    }
```

编译成功后，so文件会增加100k左右。



### 1.4.2 环境搭建

GDB 为C/S的调试架构:

> 服务端地址为：`Android/sdk/ndk/22.0.7026061/prebuilt/android-arm/gdbserver`
>
> 客户端使用前人编译好的[Pwndocker](https://github.com/skysider/pwndocker)
>
> 进入docker后运行gdb-multiarch，要不然会报错。



### 1.4.3 调试

```
$ 手机内启动64/32gdbserver: ./gdbserver 0.0.0.0:23946 --attach target_pip

$ 端口转发: adb forward tcp:23946 tcp:23946

$ docker内运行客户端gdb-multiarch，进入pwndbg

$ 连接服务端target remote 手机ip:23946 

$ pwndbg设置架构: set arch arm, set arm fallback-mode thumb 

$ c（运行到下一个断点） r（继续运行） ctrl+c暂停  info share查看所有加载的so  

$ nm -s so_name 查看所有符号

$ 本地运行: nm -s so_name |grep method 查看 "*method*"方法地址或者使用objection、frida查看

$ 查看该地址汇编: x/20i base_addr 
	cat /proc/10700/maps |grep heyhu 基址为：d1241000
	d1241000-d1247000 r-xp 00000000 103:13 917882

$ b *(base_addr)设置断点  info b 查看断点 

$ del 编号 删除断点

$ 步入：f7 步进：f8

$ finish 跳出方法  

$ frida主动调用方法和pwndbg同时使用的话需要先启动frida附加（注入器注入之后ptracepid=0）然后再启动gdbserver，在主动调用时候先用pwndbg暂停，
  然后frida主动调用, 然后f8就可以一步一步调试了。

$ objection查看so基址：memory list modules

$ memory list exports so_name 查看so的所有符号基址

$ 下两个断点切换状态 ctrl+shift+pageup

$ frida和gdb为什么不能一起调试？
		其中一个程序attach目标进程时，TracerPid会被占用，别的进程就附加不上。
		cat /proc/10700/status   TracerPid:	25096   -> gdb
 		
$ 为什么frida先附加后，gdb可以进行调试？
 		谁进行调试 TracerPid就是谁的进程，frida注入后 TracerPid为0，因为注入器注入后就dettch了，然后就可以用gdb了。
```



### 1.4.4 判断app使用的架构

```
Objection：Process Architecture arm

查看目标进程的父进程
root 3316 1 4213072 48928 poll_schedule_timeout 7135b2b518 S zygote64
root 3317 1 1552068 40104 poll_schedule_timeout e72ebc4c S zygote
```



## 1.5 参考链接

https://developer.android.google.cn/ndk/guides/other_build_systems

https://my.oschina.net/u/4263893/blog/4349408

https://blog.csdn.net/wangyiyungw/article/details/81069631

https://github.com/heyhu/openso

https://lldb.llvm.org/use/map.html

---

https://github.com/skysider/pwndocker

https://www.cnblogs.com/zhwer/p/12494317.html

https://bbs.pediy.com/thread-257344.htm

https://blog.csdn.net/l460133921/article/details/52931328/
