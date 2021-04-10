## 1.1 Symbols Hook
[以下提到的`源码`的地址](https://github.com/heyhu/demoso1)

### 1.1.1 IDA结构化JNIEnv

 		IDA 分析Android so 文件时，因为缺少JNIEnv结构定义，反编译后看起来很不友好，需要导入`jni.h`使JNIEnv 的结构定义让反编译代码看起来更轻松，就可以看到函数名，例如：_JNIEnv::GetStringUTFLength();

  	找到本机的`jni.h`，在`Android/sdk`下，`tree -NCfhl | grep -i jni.h`，打开ida，点`File->Load file->Parse C header file` 找到jni.h所在位置导入。如果没有识别，按y键。

```c++
// ida导入jni.h后显示内容
{
  __int64 v4; // [xsp+30h] [xbp-20h]
  __int64 v5; // [xsp+38h] [xbp-18h]
  _JNIEnv *v6; // [xsp+48h] [xbp-8h]

  v6 = a1;
  v5 = a3;
  v4 = _JNIEnv::GetStringUTFChars(a1, a3, 0LL);
  _JNIEnv::GetStringUTFLength(v6, v5);
  if ( v4 != 0 )
  {
    __android_log_print(4LL, "r0add", "now a is %s", v4);
    __android_log_print(4LL, "r0add", "now content is %s", v5);
  }
  _JNIEnv::ReleaseStringUTFChars(v6, v5, v4);
  return _JNIEnv::NewStringUTF(v6, "Hello I`m from myfirstjnienv!");
}
```



##### Native source code view:

```java
 Log.i("r0add", MainActivity.myfirstjniJNI("fromJava"));

 public static native String myfirstjniJNI(String context);
```

```c++
extern "C" JNIEXPORT jstring JNICALL
Java_com_example_demoso1_MainActivity_myfirstjniJNI(
        JNIEnv* env,
        jclass,
        jstring content ) {
    // java.lang.String对应的JNI类型是jstring，但本地代码只能通过GetStringUTFChars这样的JNI函数来访问字符串的内容
    const char* a = env->GetStringUTFChars(content, nullptr);
    int content_size = env->GetStringUTFLength(content);
    if(a!=0){
        LOGI("now a is %s",a);
        LOGI("now content is %s",content);
    }
    env->ReleaseStringUTFChars(content,a);
    jstring result = env->NewStringUTF("Hello I`m from myfirstjnienv!");
    return result;
}
```



### 1.1.2 HOOK JAVA层，myfirstjniJNI函数

```javascript
function hook_java() {
    Java.perform(function x() {
        Java.use('com.example.demoso1.MainActivity').myfirstjniJNI.implementation = function (x) {
            var result = this.myfirstjniJNI('from hook java function');
            console.log('result', x, result);
            return result;
        };
        Java.choose('com.example.demoso1.MainActivity', {
            onMatch: function (instance) {
                instance.init();
            },
            onComplete() {
            }
        })
    })
}
```



### 1.1.3 HOOK NATIVE层，myfirstjniJNI函数

Java_com_example_demoso1_MainActivity_myfirstjniJNI函数，so库为自己开发的库。

```javascript
function hook_native() {
    // module: Process.enumerateModules() 枚举立即加载的模块，并返回一个Module对象数组。
    // JSON.stringify 打印object对象
    var modules = JSON.stringify(Process.enumerateModules());
    // 打印so库基地址
    var libnative_addr = Module.findBaseAddress('libnative-lib.so');
    console.log("libnative", libnative_addr);
    // 找到符号地址
    if (libnative_addr) {
        var JNI_address = Module.findExportByName('libnative-lib.so', 'Java_com_example_demoso1_MainActivity_myfirstjniJNI');
        // 偏移量 = 符号地址 - so地址 = start（ida 最左边function_name 右拉 ）
        console.log("JNI_address", JNI_address)
    }

    // 符号 hook  Interceptor:拦截器
    Interceptor.attach(JNI_address, {
        onEnter: function (args) {
            // tryGetEnv == getEnv
            // GetStringUTFChars: java.lang.String对应的JNI类型是jstring，但本地代码只能通过GetStringUTFChars这样的JNI函数来访问字符串的内容
            // newStringUtf、GetStringUTFChars 需要和api对应 （星球）https://github.com/frida/frida-java-bridge/blob/master/lib/env.js
            // 如果api中有才能使用JNI API
            // 第一个参数为JniEnv，第二个参数为Jclass，第三个才是自定义参数
            var content = Java.vm.getEnv().getStringUtfChars(args[2], null).readCString();
            console.log("content", content)
        },
        onLeave: function (retval) {
            // 只能在native层使用
            console.log("retval", Java.vm.getEnv().getStringUtfChars(retval, null).readCString());
            // 新建返回值替换原有值
            var new_retval = Java.vm.getEnv().newStringUtf("new retval from hook");
            // ptr: 从包含以十进制或十六进制（如果以'0x'为前缀）内存地址的字符串创建一个新的NativePointers。
            // so 操作的都是指针
            return retval.replace(ptr(new_retval));
        }
    })
}
```



### 1.1.4  HOOK ART

以libart.so举例，系统库。

```javascript
function hook_art() {
    // hook GetStringUTFChars、NewStringUTF等env调用的一些函数。
    // 寻找libart.so在不在
    var modules = JSON.stringify(Process.enumerateModules());
    // 因为name mangling的原因我们无法找到它的符号，拿到模块枚举所有的符号，然后过滤。
    var Symbols = Process.findModuleByName("libart.so").enumerateSymbols();
    // console.log(JSON.stringify(Symbols));
    var GetString_ADD;
    for (var i = 0; i < Symbols.length; i++) {
        var symbol = Symbols[i].name;
        if ((symbol.indexOf('CheckJNI') == -1) && symbol.indexOf('JNI') >= 0) {
            if (symbol.indexOf('GetStringUTFChars') >= 0) {
                // 打印符号名 // _ZN3art3JNI17GetStringUTFCharsEP7_JNIEnvP8_jstringPh 和ida显示的一样
                // 也可以ida中查看, 搜索的哪个库，就在哪个库找
                console.log("Symbols[i].name:", Symbols[i].name);
                // 打印符号地址
                console.log("Symbols[i].address:", Symbols[i].address);
                GetString_ADD = Symbols[i].address;
            }
        }
    }
    console.log("GetString Address:", GetString_ADD);
    Interceptor.attach(GetString_ADD, {
        onEnter: function (args) {
            var content = Java.vm.getEnv().getStringUtfChars(args[1], null).readCString();
            console.log("content", content);
            // console.log("args[0]",hexdump(args[0].readPointer()));
        },
        onLeave: function (retval) {
            console.log("retval", ptr(retval).readCString())
            // console.log("retval", Memory.readCString(retval))
        }
    })
}
```



### 1.1.5  HOOK Libc

libc.so 里面的函数名就是原有的，不会混淆， 例如pthread_create。

 source code view:

```java
public native int init();
```

```c++
extern "C" JNIEXPORT void JNICALL
Java_com_example_demoso1_MainActivity_init(
        JNIEnv* env,
        jobject clazz) {

    pthread_t t;
    pthread_create(&t,NULL,detect_frida_loop,(void*)NULL);
    LOGI("frida server detect loop started");
}
```

```javascript
function hook_libc() {
    // 函数名就是原有的，不会混淆
    var Symbols = Process.findModuleByName("libc.so").enumerateSymbols();
    var pthread = null;
    for (var i = 0; i < Symbols.length; i++) {
        var symbol = Symbols[i].name;
        if (symbol.indexOf('pthread_create') >= 0) {
            //console.log("Symbols[i].name:", Symbols[i].name);
            //console.log("Symbols[i].address:", Symbols[i].address);
            pthread = Symbols[i].address;
        }
    }
    // hook pthread 需要主动调用hook_java()，主动调用init(), 才能找到pthread_create
    console.log("pthread_address",pthread);
    Interceptor.attach(pthread, {
        onEnter: function(args) {
            // args[2]就是detect_frida_loop的地址，可以在nativelib.so的导出函数找到，因为此函数是在libnative-lib.so声明的。
            console.log("pthread args", args[0], args[1], args[2], args[3])
        },
        onLeave: function(retval) {
            console.log("ret", retval)
        }
    });
}

function main() {
    hook_java();
    hook_native();
    hook_art();
    hook_libc()
}
// 可以在交互页面主动调用上面的函数。
setImmediate(main);
```



### 1.1.6  调用栈的打印

```
frida打印.so中堆栈信息

console.log("open" + ' called from:\n' +
Thread.backtrace(this.context, Backtracer.ACCURATE).map(DebugSymbol.fromAddress).join('\n') + '\n');
打印当前`so`的调用栈。`Backtracer.ACCURATE`会准确些，但是有时候会为空，主要看`so`有没有符号了。如果用`Backtracer.FUZZY`参数则得到的结果不一定准确。
```



