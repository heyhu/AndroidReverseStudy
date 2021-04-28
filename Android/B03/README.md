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

### 沙箱自吐Http请求与响应

在源码中搜索java.net.SocketInputStream\java.net.SocketOutputStream

![05.a](/Android/B03/pic/05.a.png)

修改内容如下：

```java
// java.net.SocketInputStream

package java.net;

import java.io.FileDescriptor;
import java.io.FileInputStream;
import java.io.IOException;
import java.lang.reflect.InvocationTargetException;
import java.lang.reflect.Method;
import java.nio.channels.FileChannel;

import dalvik.system.BlockGuard;
import sun.net.ConnectionResetException;

    private int socketRead(FileDescriptor fd,
                           byte b[], int off, int len,
                           int timeout)
        throws IOException {
        int result = socketRead0(fd, b, off, len, timeout);

        if(result>0){
            byte[] input = new byte[result];
            System.arraycopy(b,off,input,0,result);

            String inputString = new String(input);
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
                loge.invoke(null,"r0ysueSOCKETresponse","Socket is => "+this.socket.toString());
                loge.invoke(null,"r0ysueSOCKETresponse","buffer is => "+inputString);
                Exception e = new Exception("r0ysueSOCKETresponse");
                e.printStackTrace();
            } catch (IllegalAccessException e) {
                e.printStackTrace();
            } catch (InvocationTargetException e) {
                e.printStackTrace();
            }

        }
        return result;
    }
```

```java
// java.net.SocketOutputStream

package java.net;

import java.io.FileDescriptor;
import java.io.FileOutputStream;
import java.io.IOException;
import java.lang.reflect.InvocationTargetException;
import java.lang.reflect.Method;
import java.nio.channels.FileChannel;

import dalvik.system.BlockGuard;

    private void socketWrite(byte b[], int off, int len) throws IOException {
        if (len <= 0 || off < 0 || len > b.length - off) {
            if (len == 0) {
                return;
            }
            throw new ArrayIndexOutOfBoundsException("len == " + len
                    + " off == " + off + " buffer length == " + b.length);
        }

        FileDescriptor fd = impl.acquireFD();
        try {
            BlockGuard.getThreadPolicy().onNetwork();
            socketWrite0(fd, b, off, len);


            if(len>0){
                byte[] input = new byte[len];
                System.arraycopy(b,off,input,0,len);

                String inputString = new String(input);
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
                    loge.invoke(null,"r0ysueSOCKETrequest","Socket is => "+this.socket.toString());
                    loge.invoke(null,"r0ysueSOCKETrequest","buffer is => "+inputString);
                    Exception e = new Exception("r0ysueSOCKETrequest");
                    e.printStackTrace();
                } catch (IllegalAccessException e) {
                    e.printStackTrace();
                } catch (InvocationTargetException e) {
                    e.printStackTrace();
                }

            }


        } catch (SocketException se) {
            if (se instanceof sun.net.ConnectionResetException) {
                impl.setConnectionResetPending();
                se = new SocketException("Connection reset");
            }
            if (impl.isClosedOrPending()) {
                throw new SocketException("Socket closed");
            } else {
                throw se;
            }
        } finally {
            impl.releaseFD();
        }
    }
```

### 沙箱自吐Https请求与响应

在源码中搜索SslWrapper.java

![06.a](/Users/zhaoshouxin/code/AndroidReverseStudy/Android/B03/pic/06.a.png)

修改内容如下：

```java
// TODO(nathanmittler): Remove once after we switch to the engine socket.
    int read(FileDescriptor fd, byte[] buf, int offset, int len, int timeoutMillis)
            throws IOException {
        int result = NativeCrypto.SSL_read(ssl, fd, handshakeCallbacks, buf, offset, len, timeoutMillis) ;
        if(result>0){
            byte[] input = new byte[result];
            System.arraycopy(buf,offset,input,0,result);

            String inputString = new String(input);
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
                loge.invoke(null,"r0ysueSOCKETresponse","SSL is =>"+this.handshakeCallbacks.toString());
                loge.invoke(null,"r0ysueSOCKETresponse","buffer is => "+inputString);
                Exception e = new Exception("r0ysueSOCKETresponse");
                e.printStackTrace();
            } catch (IllegalAccessException e) {
                e.printStackTrace();
            } catch (InvocationTargetException e) {
                e.printStackTrace();
            }

        }
        return result;
    }

    // TODO(nathanmittler): Remove once after we switch to the engine socket.
    void write(FileDescriptor fd, byte[] buf, int offset, int len, int timeoutMillis)
            throws IOException {


        if(len>0){
            byte[] input = new byte[len];
            System.arraycopy(buf,offset,input,0,len);

            String inputString = new String(input);
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
                loge.invoke(null,"r0ysueSSLrequest","SSL is => "+this.handshakeCallbacks.toString());
                loge.invoke(null,"r0ysueSSLrequest","buffer is => "+inputString);
                Exception e = new Exception("r0ysueSSLrequest");
                e.printStackTrace();
            } catch (IllegalAccessException e) {
                e.printStackTrace();
            } catch (InvocationTargetException e) {
                e.printStackTrace();
            }
        }

        NativeCrypto.SSL_write(ssl, fd, handshakeCallbacks, buf, offset, len, timeoutMillis);
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



## 参考链接  

> https://bbs.pediy.com/thread-264283.htm
