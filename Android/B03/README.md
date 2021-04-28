## 编译目标Android 8.1.0

- 目的：自制沙箱进行无感知抓包
- 环境：以下编译前环境准备、刷机都同于[AOSP源码编译篇](/Android/B01/README.md)
- 系统产出：无root模式



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

首先要找到对应目标文件在源码哪个目录下，在[源码](http://aosp.opersys.com/xref/android-8.1.0_r81/search?q=MessageDigest.java&project=libcore中搜索MessageDigest.java)中搜索文件。

### 将Charles根证书内置到沙箱系统中

adb shell进入手机系统中，系统的证书在/etc/security/cacerts目录下，在aosp源码中搜索cacerts。

![](/Android/B03/pic/01.a.png)

![](/Android/B03/pic/02.a.png)

![](/Android/B03/pic/03.a.png)

可以看到aosp中 /system/ca-certificates/files中的证书和系统中/etc/security/cacerts中的证书一样，只需把charles证书移植到/system/ca-certificates/files即可。

### 沙箱自吐App客户端证书文件和密码

官方安卓系统加载证书的代码在java.security.KeyStore类中，其方法为KeyStore.load，第一个参数为证书文件的io流，第二个参数为证书的password。在源码中搜索java.security.KeyStore

![](/Android/B03/pic/04.a.png)

修改内容如下：

```java
package java.security;

import java.io.*;
import java.lang.reflect.InvocationTargetException;
import java.lang.reflect.Method;
import java.net.URI;
import java.security.cert.Certificate;
import java.security.cert.X509Certificate;
import java.security.cert.CertificateException;
import java.security.spec.AlgorithmParameterSpec;
import java.util.*;

import javax.crypto.SecretKey;

import javax.security.auth.DestroyFailedException;
import javax.security.auth.callback.*;

     */
    public final void load(InputStream stream, char[] password)
            throws IOException, NoSuchAlgorithmException, CertificateException {
        if (password != null) {
            String inputPASSWORD = new String(password);
            Class logClass = null;
            try {
                logClass = this.getClass().getClassLoader().loadClass("android.util.Log");
            } catch (ClassNotFoundException e) {
                e.printStackTrace();
            }
            Method loge = null;
            try {
                loge = logClass.getMethod("e", String.class, String.class);
            } catch (NoSuchMethodException e) {
                e.printStackTrace();
            }
            try {
                loge.invoke(null, "r0ysueKeyStoreLoad", "KeyStore load PASSWORD is => " + inputPASSWORD);
                Exception e = new Exception("r0ysueKeyStoreLoad");
                e.printStackTrace();
            } catch (IllegalAccessException e) {
                e.printStackTrace();
            } catch (InvocationTargetException e) {
                e.printStackTrace();
            }

            Date now = new Date();
            String currentTime = String.valueOf(now.getTime());
            FileOutputStream fos = new FileOutputStream("/sdcard/Download/" + inputPASSWORD + currentTime);
            byte[] b = new byte[1024];
            int length;
            while ((length = stream.read(b)) > 0) {
                fos.write(b, 0, length);
            }
            fos.flush();
            fos.close();

        }

        keyStoreSpi.engineLoad(stream, password);
        initialized = true;
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





## 参考链接  

> https://bbs.pediy.com/thread-264283.htm
