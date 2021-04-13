## 1.1 反射

查找调用各种API接口、jni、frida/xposed原理一部分。frida的好多接口都是用到了反射。[以下提到的`源码`的地址](https://github.com/heyhu/demoso1)

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



## 1.2  Java反射举例

```java
// 一个Test类
package com.noguess.demoso1;
import android.util.Log;

public class Test {
    
    // 构造函数
    public String flag = null;
    public Test(){
        flag = "Test()";
    }
    public Test(String arg){
        flag = "Test(String arg)";
    }
    public Test(String arg, int arg2){
        flag = "Test(String arg, int arg2)";
    }

    public static String publicStaticFiled = "I am a publicStaticFiled";
    public String publicFiled = "I am a publicFiled";
    private static String privateStaticFiled = "I am a privateStaticFiled";
    private String privateFiled = "I am a privateFiled";

    public static void publicStaticFunc(){
        Log.i("r0reflection", "I`m from publicStaticFunc");
    }

    public void publicFunc(){
        Log.i("r0reflection", "I`m from publicFunc");
    }

    private static void privateStaticFunc(){
        Log.i("r0reflection", "I`m from privateStaticFunc");
    }

    public void privateFunc(){
        Log.i("r0reflection", "I`m from privateFunc");
    }
}
```

### 1.2.1 获取类的三种方法

```java
public void testField(){
        // 获取类的三种方法

        Class testClazz = null;
        try {
            testClazz = MainActivity.class.getClassLoader().loadClass("com.noguess.demoso1.Test");
            Log.i("r0reflection","loadClass->" + testClazz);
        } catch (ClassNotFoundException e) {
            e.printStackTrace();
        }

        Class testClazz2 = null;
        try {
            testClazz2 = Class.forName("com.noguess.demoso1.Test");
        } catch (ClassNotFoundException e) {
            e.printStackTrace();
        }
        Log.i("r0reflection","Class.forName->" + testClazz2);

        Class testClazz3 = Test.class;
        // getName 获得类的完整路径名字
        Log.i("r0reflection","Test.class->" + testClazz3.getName());

}
```

output：

```
I/r0reflection: loadClass->class com.noguess.demoso1.Test
I/r0reflection: Class.forName->class com.noguess.demoso1.Test
I/r0reflection: Test.class->com.noguess.demoso1.Test
```

### 1.2.2 Field代表类的成员变量

成员变量也称为类的属性

#### 1.2.2.1 获得某个属性对象：getDeclaredField(String name)

```java
import java.lang.reflect.Field;
// 获取 publicStaticFiled属性对象
try {
    Field publicStaticFiled_filed = testClazz3.getDeclaredField("publicStaticFiled");
    Log.i("r0reflection","publicStaticFiled_filed->" + publicStaticFiled_filed);
} catch (NoSuchFieldException e) {
    e.printStackTrace();
}
```

output：

```
I/r0reflection: publicStaticFiled_filed->public static java.lang.String com.noguess.demoso1.Test.publicStaticFiled
```

#### 1.2.2.2 获得所有属性对象：getDeclaredFields()

```java
// 返回属性数组
Field[] fields = testClazz3.getDeclaredFields();
for(Field i : fields){
    Log.i("r0reflection","getDeclaredFields_i->" + i);
}
```

output：

```
公有、私有、构造
I/r0reflection: getDeclaredFields_i->public java.lang.String com.noguess.demoso1.Test.flag
I/r0reflection: getDeclaredFields_i->private java.lang.String com.noguess.demoso1.Test.privateFiled
I/r0reflection: getDeclaredFields_i->public java.lang.String com.noguess.demoso1.Test.publicFiled
I/r0reflection: getDeclaredFields_i->private static java.lang.String com.noguess.demoso1.Test.privateStaticFiled
I/r0reflection: getDeclaredFields_i->public static java.lang.String com.noguess.demoso1.Test.publicStaticFiled
```

#### 1.2.2.3 获得所有公有的属性对象：getFields()

```java
Field[] fields_2 = testClazz3.getFields();
for(Field i : fields_2){
    Log.i("r0reflection","getFields->" + i);
}
```

output:

```
I/r0reflection: getFields->public java.lang.String com.noguess.demoso1.Test.flag
I/r0reflection: getFields->public java.lang.String com.noguess.demoso1.Test.publicFiled
I/r0reflection: getFields->public static java.lang.String com.noguess.demoso1.Test.publicStaticFiled
```

#### 1.2.2.4 获得obj中对应的属性值：get(Object obj)

```java
// 获取公有属性
try {
    Field publicStaticFiled_filed = testClazz3.getDeclaredField("publicStaticFiled");
    Log.i("r0reflection","publicStaticFiled_filed->" + publicStaticFiled_filed);
    // 静态属性传入null
    String value = (String) publicStaticFiled_filed.get(null);
    Log.i("r0reflection","Static value->" + value);

    Field publicFiled_field = testClazz3.getDeclaredField("publicFiled");
    Log.i("r0reflection","publicFiled_filed->" + publicFiled_field);
    // 创建一个实例obj
    Object objectTest = testClazz3.newInstance();
    // 非静态属性传入一个实例obj
    String value1 = (String) publicFiled_field.get(objectTest);
    Log.i("r0reflection","value->" + value1);
} catch (NoSuchFieldException | IllegalAccessException | InstantiationException e) {
    e.printStackTrace();
}
```

output：

```
I/r0reflection: publicStaticFiled_filed->public static java.lang.String com.noguess.demoso1.Test.publicStaticFiled
I/r0reflection: Static value->I am a publicStaticFiled
I/r0reflection: publicFiled_filed->public java.lang.String com.noguess.demoso1.Test.publicFiled
I/r0reflection: value->I am a publicFiled
```

```java
// 获取私有属性
try {
    Field privateStaticFiled_filed = testClazz3.getDeclaredField("privateStaticFiled");
    Log.i("r0reflection","privateStaticFiled_filed->" + privateStaticFiled_filed);
    // 反射私有属性
    privateStaticFiled_filed.setAccessible(true);
    // 静态属性传入null
    String value_private = (String) privateStaticFiled_filed.get(null);
    Log.i("r0reflection","Static value_private->" + value_private);

    Field privateFiled_filed = testClazz3.getDeclaredField("privateFiled");
    Log.i("r0reflection","privateFiled_filed->" + privateFiled_filed);
    // 反射私有属性
    privateFiled_filed.setAccessible(true);
    // 动态属性传入obj
    Object objectTest = testClazz3.newInstance();
    String value_private_1 = (String) privateFiled_filed.get(objectTest);
    Log.i("r0reflection","value_private->" + value_private_1);
} catch (NoSuchFieldException | IllegalAccessException | InstantiationException e) {
    e.printStackTrace();
}
```

output：

```
I/r0reflection: privateStaticFiled_filed->private static java.lang.String com.noguess.demoso1.Test.privateStaticFiled
I/r0reflection: Static value_private->I am a privateStaticFiled
I/r0reflection: privateFiled_filed->private java.lang.String com.noguess.demoso1.Test.privateFiled
I/r0reflection: value_private->I am a privateFiled
```

#### 1.2.2.5 设置obj中对应属性值：set(Object obj, Object value)

```java
// 以下为公有方法
try {
    Field publicStaticFiled_filed = testClazz3.getDeclaredField("publicStaticFiled");
    // 静态属性传入null
    publicStaticFiled_filed.set(null, "I am shouxin zhao Static");
    String value1 = (String) publicStaticFiled_filed.get(null);
    Log.i("r0reflection","value->" + value1);

    Field publicFiled_field = testClazz3.getDeclaredField("publicFiled");
    // 动态属性传入obj
    Object objectTest = testClazz3.newInstance();
    publicFiled_field.set(objectTest, "I am shouxin zhao");
    String value2 = (String) publicFiled_field.get(objectTest);
    Log.i("r0reflection","value->" + value2);
} catch (NoSuchFieldException | IllegalAccessException | InstantiationException e) {
    e.printStackTrace();
}
```

output：

```
I/r0reflection: Test.class->com.noguess.demoso1.Test
I/r0reflection: value->I am shouxin zhao Static
I/r0reflection: value->I am shouxin zhao
```

```java
// 以下为私有方法
try {
    Field privateStaticFiled_filed = testClazz3.getDeclaredField("privateStaticFiled");
    Log.i("r0reflection","privateStaticFiled_filed->" + privateStaticFiled_filed);
    // 反射私有属性
    privateStaticFiled_filed.setAccessible(true);
    privateStaticFiled_filed.set(null, "I am shouxin zhao Static");
    // 静态属性传入null
    String value_private = (String) privateStaticFiled_filed.get(null);
    Log.i("r0reflection","Static value_private->" + value_private);

} catch (NoSuchFieldException | IllegalAccessException  e) {
    e.printStackTrace();
}
```

output:

```
I/r0reflection: Test.class->com.noguess.demoso1.Test
I/r0reflection: privateStaticFiled_filed->private static java.lang.String com.noguess.demoso1.Test.privateStaticFiled
I/r0reflection: Static value_private->I am shouxin zhao Static
```

### 1.2.3 Method类代表类的方法

#### 1.2.3.1 获得类中的所有方法

```java
Class testClazz = Test.class;
// 获得该类所有方法（一般使用这个）
Method[] methods = testClazz.getDeclaredMethods();
for(Method i: methods){
    Log.i("r0reflection ", "getDeclaredMethods i->" + i);
}
// 获得该类所有公有的方法、包括父类以及超类的方法
Method[] methods1 = testClazz.getMethods();
for(Method i: methods1){
    Log.i("r0reflection ", "methods i->" + i);
}
```

```
I/r0reflection: getDeclaredMethods i->private static void com.noguess.demoso1.Test.privateStaticFunc()
I/r0reflection: getDeclaredMethods i->public static void com.noguess.demoso1.Test.publicStaticFunc()
I/r0reflection: getDeclaredMethods i->public void com.noguess.demoso1.Test.privateFunc()
I/r0reflection: getDeclaredMethods i->public void com.noguess.demoso1.Test.publicFunc()
I/r0reflection: methods i->public boolean java.lang.Object.equals(java.lang.Object)
I/r0reflection: methods i->public final java.lang.Class java.lang.Object.getClass()
I/r0reflection: methods i->public int java.lang.Object.hashCode()
I/r0reflection: methods i->public final native void java.lang.Object.notify()
I/r0reflection: methods i->public final native void java.lang.Object.notifyAll()
I/r0reflection: methods i->public void com.noguess.demoso1.Test.privateFunc()
I/r0reflection: methods i->public void com.noguess.demoso1.Test.publicFunc()
I/r0reflection: methods i->public static void com.noguess.demoso1.Test.publicStaticFunc()
I/r0reflection: methods i->public java.lang.String java.lang.Object.toString()
I/r0reflection: methods i->public final native void java.lang.Object.wait() throws java.lang.InterruptedException
I/r0reflection: methods i->public final void java.lang.Object.wait(long) throws java.lang.InterruptedException
I/r0reflection: methods i->public final native void java.lang.Object.wait(long,int) throws java.lang.InterruptedException

```

#### 1.2.3.2 获取公有、私有、静态、非静态方法、以及方法的调用

```java
public void testMethod(){
    Class testClazz = Test.class;
   
    try {
        // 公有静态
        Method  publicStaticFunc_method = testClazz.getDeclaredMethod("publicStaticFunc");
        Log.i("r0reflection ", "publicStaticFunc_method ->" + publicStaticFunc_method);
        // 调用方法
        publicStaticFunc_method.invoke(null);
        // 公有动态
        Method  publicFunc_method = testClazz.getDeclaredMethod("publicFunc");
        Log.i("r0reflection ", "publicFunc_method ->" + publicFunc_method);
        // 调用方法
        Object objectTest = testClazz.newInstance();
        publicFunc_method.invoke(objectTest);

        // 私有静态
        Method privateStaticFunc_method = testClazz.getDeclaredMethod("privateStaticFunc");
        Log.i("r0reflection ", "privateStaticFunc_method ->" + privateStaticFunc_method);
        privateStaticFunc_method.setAccessible(true);
        // 调用方法
        privateStaticFunc_method.invoke(null);
        // 私有动态
        Method privateFunc_method = testClazz.getDeclaredMethod("privateFunc");
        Log.i("r0reflection ", "privateFunc_method ->" + privateFunc_method);
        // 调用方法
        privateFunc_method.setAccessible(true);
        privateFunc_method.invoke(objectTest);


    } catch (NoSuchMethodException e) {
        e.printStackTrace();
    } catch (IllegalAccessException e) {
        e.printStackTrace();
    } catch (InvocationTargetException e) {
        e.printStackTrace();
    } catch (InstantiationException e) {
        e.printStackTrace();
    }
}
```

output:

```
I/r0reflection: publicStaticFunc_method ->public static void com.noguess.demoso1.Test.publicStaticFunc()
I/r0reflection: I`m from publicStaticFunc
I/r0reflection: publicFunc_method ->public void com.noguess.demoso1.Test.publicFunc()
I/r0reflection: I`m from publicFunc
I/r0reflection: privateStaticFunc_method ->private static void com.noguess.demoso1.Test.privateStaticFunc()
I/r0reflection: I`m from privateStaticFunc
I/r0reflection: privateFunc_method ->public void com.noguess.demoso1.Test.privateFunc()
I/r0reflection: I`m from privateFunc

```

### 1.3  Native反射举例

为什么对于frida来讲只要注入一个so，就可以对java层的属性调用和赋值？`so库使用反射`

##### Java source code view:

```java
package com.noguess.demoso1;

import androidx.annotation.RequiresApi;
import androidx.appcompat.app.AppCompatActivity;

import android.os.Build;
import android.os.Bundle;
import android.util.Log;
import android.widget.TextView;

import java.lang.reflect.Field;
import java.lang.reflect.InvocationTargetException;
import java.lang.reflect.Method;

public class MainActivity extends AppCompatActivity {

    // Used to load the 'native-lib' library on application startup.
    static {
        System.loadLibrary("native-lib");
    }

    @RequiresApi(api = Build.VERSION_CODES.KITKAT)
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        // Example of a call to a native method
        TextView tv = findViewById(R.id.sample_text);
        tv.setText(stringFromJNI());
        Log.i("r0add", MainActivity.stringFromJNI());
    }

    @RequiresApi(api = Build.VERSION_CODES.KITKAT)

    /**
     * A native method that is implemented by the 'native-lib' native library,
     * which is packaged with this application.
     */
    public static native String stringFromJNI();
}

```

##### Native source code view:

```c++
#include <jni.h>
#include <string>
#include <android/log.h>

#define  LOG_TAG  "native-lib"

// 定义info信息

#define  LOGI(...)  __android_log_print(ANDROID_LOG_INFO, LOG_TAG, __VA_ARGS__)
#define LOGE(...)  __android_log_print(ANDROID_LOG_ERROR, LOG_TAG, __VA_ARGS__)
#define LOGI(...)  __android_log_print(ANDROID_LOG_INFO, LOG_TAG, __VA_ARGS__)

extern "C" JNIEXPORT jstring JNICALL
Java_com_noguess_demoso1_MainActivity_stringFromJNI(
        JNIEnv *env,
        jclass clazz) {

    // 找到class
    jclass testClass = env->FindClass("com/noguess/demoso1/Test");
    // 获得属性id 传入参数：类、属性名、属性的签名
    jfieldID publicStaticFiled = env->GetStaticFieldID(testClass, "publicStaticFiled",
                                                       "Ljava/lang/String;");
    // 获取属性值 传入参数：类 属性id
    jstring publicStaticFiled_value = (jstring)env->GetStaticObjectField(testClass, publicStaticFiled);
    // 转成c++的对象
    // 无法直接打印jstring 必须转成char *
    const char* value_ptr = env->GetStringUTFChars(publicStaticFiled_value, nullptr);
    // LOGI('value_ptr %s', value_ptr);
    // jmethodID
    return publicStaticFiled_value;
}
```

