## 1.1 反射

查找调用各种API接口、jni、frida/xposed原理一部分。frida的好多接口都是用到了反射。

### 1.1.1 反射介绍

​	 [相关文档](https://www.jianshu.com/p/9be58ee20dee)

​    [JAVA反射机制](https://baike.baidu.com/item/JAVA反射机制/6015990)是在运行状态中，对于任意一个类，都能够知道这个类的所有属性和方法；对于任意一个对象，都能够调用它的任意方法和属性；这种动态获取信息以及动态调用对象方法的功能称为java语言的反射机制。


### 1.1.2  JNIENV

```
1.JNIEnv是一个线程相关的结构体，该结构体代表了Java在本线程的执行环境。
2.调用Java函数：JNIEnv代表了Java执行环境，能够使用JNIEnv调用Java中的代码
3.操作Java代码：Java对象传入JNI层就是jobject对象，需要使用JNIEnv来操作这个Java对象
4.JavaVM：JavaVM是Java虚拟机在JNI层的代表，JNI全局仅仅有一个JNIEnv。JavaVM 在线程中的代码，每个线程都有一个，JNI可能有非常多个JNIEnv；
```

### 1.1.3  JNIENV

