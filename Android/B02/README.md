## 编译目标Android 7.1.2

> 目的：aosp源码修改MessageDigest.java 文件，实现入参和返回值的自吐。
>
> 环境：以下编译前环境准备、刷机都同于[AOSP源码编译篇](/Android/B01/README.md)
> 
> 系统产出：无root模式



## Android Studio导入AOSP源码

- 进入源码根目录，运行如下命令

  > source build/envsetup.sh
  >
  > mmm development/tools/idegen/
  >
  > development/tools/idegen/idegen.sh

- 导入Android Studio

  > 打开 Android Studio，选择 Open an existing Android Studio project，找到源码目录，点击 Android.ipr，Open，大约等 6 分钟，导入完毕。


## 修改源码

首先要找到对应目标文件在源码哪个目录下,在[源码](http://aosp.opersys.com/xref/android-8.1.0_r81/search?q=MessageDigest.java&project=libcore中搜索MessageDigest.java)中查找，可以看到对应路径。

<img src="/Android/B02/pic/01.a.png" style="zoom:40%;" />

修改内容如下：

```java

// 添加库
import java.lang.reflect.InvocationTargetException;
import java.lang.reflect.Method;

import java.util.*;
import java.lang.*;
import java.io.IOException;
import java.io.ByteArrayOutputStream;
import java.io.PrintStream;
import java.io.InputStream;
import java.io.ByteArrayInputStream;

import java.nio.ByteBuffer;


public abstract class MessageDigest extends MessageDigestSpi {

    /**
     * byte数组转hex
     * @param bytes
     * @return
     */
    public static String byteToHex(byte[] bytes){
        String strHex = "";
        StringBuilder sb = new StringBuilder("");
        for (int n = 0; n < bytes.length; n++) {
            strHex = Integer.toHexString(bytes[n] & 0xFF);
            sb.append((strHex.length() == 1) ? "0" + strHex : strHex); // 每个字节由两个字符表示，位数不够，高位补0
        }
        return sb.toString().trim();
    }

    /**
     * Updates the digest using the specified array of bytes.
     *
     * @param input the array of bytes.
     */
    public void update(byte[] input) {
        String inputString = new String(input);
        Class logClass = null;
        try {
            // 本文件没有引用log类，需要使用反射
            logClass = this.getClass().getClassLoader().loadClass("android.util.Log");
        } catch (ClassNotFoundException e) {
            e.printStackTrace();
        }
        Method loge = null;
        try {
            loge = logClass.getMethod("e",String.class,String.class);
        } catch (NoSuchMethodException e) {
            e.printStackTrace();
        }
        try {
            loge.invoke(null,"r0ysue","input is => "+inputString);
            Exception e = new Exception("r0ysueINPUT");
            e.printStackTrace();
        } catch (IllegalAccessException e) {
            e.printStackTrace();
        } catch (InvocationTargetException e) {
            e.printStackTrace();
        }

        engineUpdate(input, 0, input.length);
        state = IN_PROGRESS;
    }

    /**
     * Completes the hash computation by performing final operations
     * such as padding. The digest is reset after this call is made.
     *
     * @return the array of bytes for the resulting hash value.
     */
//    public byte[] digest() throws ClassNotFoundException, NoSuchMethodException, InvocationTargetException, IllegalAccessException {
    public byte[] digest(){
        /* Resetting is the responsibility of implementors. */
        byte[] result = engineDigest();
        state = INITIAL;
        String resultString = byteToHex(result);
//        Log.e("r0ysueDigest","result is => "+ resultString);
        Class logClass = null;
        try {
            logClass = this.getClass().getClassLoader().loadClass("android.util.Log");
        } catch (ClassNotFoundException e) {
            e.printStackTrace();
        }
        Method loge = null;
        try {
            loge = logClass.getMethod("e",String.class,String.class);
        } catch (NoSuchMethodException e) {
            e.printStackTrace();
        }
        try {
            loge.invoke(null,"r0ysue","result is => "+resultString);
            Exception e = new Exception("r0ysueRESULT");
            e.printStackTrace();
        } catch (IllegalAccessException e) {
            e.printStackTrace();
        } catch (InvocationTargetException e) {
            e.printStackTrace();
        }

        return result;
    }

```



## 开始编译

```
# cd /root/Desktop/COMPILE/aosp/
# export LC_ALL=C
# source build/envsetup.sh
# lunch选择对应的设备，因为要选择无root版，选项中也没有，手动输入lunch aosp_sailfish-user
# make update-api 更新改动
# make -j4
```
编译成功后刷入手机。


## 效果如下

![](/Android/B02/pic/02.a.png)



## 参考链接  

> http://wuxiaolong.me/2018/08/15/AOSP3/
>
> http://aosp.opersys.com/xref/android-8.1.0_r81/xref/libcore/openjdk_java_files.mk#61
>
> http://aosp.opersys.com/
>
> https://cs.android.com/
> 
> https://source.android.com/setup/build/building

