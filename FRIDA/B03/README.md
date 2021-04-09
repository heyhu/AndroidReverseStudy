## 1.1 Interceptor对象

该对象功能十分强大，函数原型是`Interceptor.attach(target, callbacks)`:参数`target`是需要拦截的位置的函数地址，也就是填某个`so`层函数的地址即可对其拦截，`target`是一个`NativePointer`参数，用来指定你想要拦截的函数的地址，`NativePointer`我们也学过是一个指针。需要注意的是对于`Thumb`函数需要对函数地址`+1`，`callbacks`则是它的回调函数，分别是以下两个回调函数：

### 1.1.1 Interceptor.attach

`onEnter：`函数（`args`）：回调函数，给定一个参数`args`，可用于读取或写入参数作为 `NativePointer` 对象的数组。

`onLeave：`函数（`retval`）：回调函数给定一个参数 `retval`，该参数是包含原始返回值的 `NativePointer` 派生对象。可以调用 `retval.replace（1337）` 以整数 `1337` 替换返回值，或者调用 `retval.replace（ptr（"0x1234"））`以替换为指针。请注意，此对象在 `OnLeave` 调用中回收，因此不要将其存储在回调之外并使用它。如果需要存储包含的值，请制作深副本，例如：`ptr（retval.toString（））`。

我们来看看示例代码~

```js
//使用Module对象getExportByNameAPI直接获取libc.so中的导出函数read的地址，对read函数进行附加拦截
Interceptor.attach(Module.getExportByName('libc.so', 'read'), {
  //每次read函数调用的时候会执行onEnter回调函数
  onEnter: function (args) {
    this.fileDescriptor = args[0].toInt32();
  },
  //read函数执行完成之后会执行onLeave回调函数
  onLeave: function (retval) {
    if (retval.toInt32() > 0) {
      /* do something with this.fileDescriptor */
    }
  }
});
```

通过我们对`Interceptor.attach`函数有一些基本了解了~它还包含一些属性。

| 索引 | 属性          | 含义                                                         |
| ---- | ------------- | ------------------------------------------------------------ |
| 1    | returnAddress | 返回地址，类型是`NativePointer`                              |
| 2    | context       | 上下文：具有键`pc`和`sp`的对象，它们是分别为`ia32/x64/arm`指定`EIP/RIP/PC`和`ESP/RSP/SP的NativePointer`对象。其他处理器特定的键也可用，例如`eax、rax、r0、x0`等。也可以通过分配给这些键来更新寄存器值。 |
| 3    | errno         | 当前`errno`值                                                |
| 4    | lastError     | 当前操作系统错误值                                           |
| 5    | threadId      | 操作系统线程ID                                               |
| 6    | depth         | 相对于其他调用的调用深度                                     |

我们来看看示例代码。

```js
function frida_Interceptor() {
    Java.perform(function () {
        //对So层的导出函数getSum进行拦截
        Interceptor.attach(Module.findExportByName("libhello.so" , "Java_com_roysue_roysueapplication_hellojni_getSum"), {
            onEnter: function(args) {
                //输出
                console.log('Context information:');
                //输出上下文因其是一个Objection对象，需要它进行接送、转换才能正常看到值
                console.log('Context  : ' + JSON.stringify(this.context));
                //输出返回地址
                console.log('Return   : ' + this.returnAddress);
                //输出线程id
                console.log('ThreadId : ' + this.threadId);
                console.log('Depth    : ' + this.depth);
                console.log('Errornr  : ' + this.err);
            },
            onLeave:function(retval){
            }
        });
    });
}
setImmediate(frida_Interceptor,0);
```

我们注入脚本之后来看看执行之后的效果以及输出的这些都是啥，执行的效果图`1-9`。

[![img](https://p3.ssl.qhimg.com/t016ac097aac6fd0971.png)](https://p3.ssl.qhimg.com/t016ac097aac6fd0971.png)

图1-9 终端执行

### 1.1.2 Interceptor.detachAll

简单来说这个的函数的作用就是让之前所有的`Interceptor.attach`附加拦截的回调函数失效。

### 1.1.3 Interceptor.replace

相当于替换掉原本的函数，用替换时的实现替换目标处的函数。如果想要完全或部分替换现有函数的实现，则通常使用此函数。，我们也看例子，例子是最直观的！代码如下。

```js
function frida_Interceptor() {
    Java.perform(function () {
       //这个c_getSum方法有两个int参数、返回结果为两个参数相加
       //这里用NativeFunction函数自己定义了一个c_getSum函数
       var add_method = new NativeFunction(Module.findExportByName('libhello.so', 'c_getSum'), 
       'int',['int','int']);
       //输出结果 那结果肯定就是 3
       console.log("result:",add_method(1,2));
       //这里对原函数的功能进行替换实现
       Interceptor.replace(add_method, new NativeCallback(function (a, b) {
           //h不论是什么参数都返回123
            return 123;
       }, 'int', ['int', 'int']));
       //再次调用 则返回123
       console.log("result:",add_method(1,2));
    });
}
```

我来看注入脚本之后的终端是是不是显示了`3`和`123`见下图`1-10`。

[![img](https://p2.ssl.qhimg.com/t013fb138fe1c77d80c.png)](https://p2.ssl.qhimg.com/t013fb138fe1c77d80c.png)

图1-10 终端执行

 

## 1.2 NativePointer对象

同等与C语言中的指针

### 1.2.1 new NativePointer(s)

声明定义NativePointer类型

```js
function frida_NativePointer() {
    Java.perform(function () {
        //第一种字符串定义方式 十进制的100 输出为十六进制0x64
        const ptr1 = new NativePointer("100");
        console.log("ptr1:",ptr1);
        //第二种字符串定义方式 直接定义0x64 同等与定义十六进制的64
        const ptr2 = new NativePointer("0x64");
        console.log("ptr2:",ptr2);        
        //第三种定数值义方式 定义数字int类型 十进制的100 是0x64
        const ptr3 = new NativePointer(100);
        console.log("ptr3:",ptr3);
    });
}     
setImmediate(frida_NativePointer,0);

/*
输出如下，都会自动转为十六进制的0x64
ptr1: 0x64
ptr2: 0x64
ptr3: 0x64 */
```

### 1.2.2 运算符以及指针读写API

它也能调用以下运算符
[![img](https://p3.ssl.qhimg.com/t0135a7319c873d8d52.png)](https://p3.ssl.qhimg.com/t0135a7319c873d8d52.png)
[![img](https://p3.ssl.qhimg.com/t01821e14d1331f0aef.png)](https://p3.ssl.qhimg.com/t01821e14d1331f0aef.png)

看完API含义之后，我们来使用他们，下面该脚本是readByteArray()示例~

```js
function frida_NativePointer() {
    Java.perform(function () {
       console.log("");
        //拿到libc.so在内存中的地址
        var pointer = Process.findModuleByName("libc.so").base;
        //读取从pointer地址开始的16个字节
        console.log(pointer.readByteArray(0x10));
    });
}     
setImmediate(frida_NativePointer,0);

/*
输出如下：
           0  1  2  3  4  5  6  7  8  9  A  B  C  D  E  F  0123456789ABCDEF
00000000  7f 45 4c 46 01 01 01 00 00 00 00 00 00 00 00 00  .ELF............ */
```

首先我先来用`readByteArray`函数来读取`libc.so`文件在内存中的数据，这样我们方便测试，我们从`libc`文件读取`0x10`个字节的长度，肯定会是`7F 45 4C 46...`因为`ELF`文件头部信息中的`Magic`属性。

### 1.2.3 readPointer()

咱们直接从`API`索引11开始玩readPointer()，定义是从此内存位置读取`NativePointer`，示例代码如下。省略`function`以及`Java.perform`~

```js
    var pointer = Process.findModuleByName("libc.so").base;
    console.log(pointer.readByteArray(0x10));
    console.log("readPointer():"+pointer.readPointer());
    
    /*
    输出如下。
    readPointer():0x464c457f */
```

也就是将`readPointer`的前四个字节的内容转成地址产生一个新的`NativePointer`。

### 1.2.4 writePointer(ptr)

读取ptr指针地址到当前指针

```js
        //先打印pointer指针地址
        console.log("pointer :"+pointer);
        //分配四个字节的空间地址
        const r = Memory.alloc(4);
        //将pointer指针写入刚刚申请的r内
        r.writePointer(pointer);
        //读取r指针的数据
        var buffer = Memory.readByteArray(r, 4);
        //r指针内放的pointer指针地址
        console.log(buffer);

/*
输出如下。
//console.log("pointer :"+pointer); 这句打印的地址 也就是libc的地址
pointer :0xf588f000
//console.log(buffer); 输出buffer 0xf588f000在内存数据会以00 f0 88 f5方式显示
           0  1  2  3  4  5  6  7  8  9  A  B  C  D  E  F  0123456789ABCDEF
00000000  00 f0 88 f5                                      .... */
```

### 1.2.5 readS32()、readU32()

从该内存位置读取有符号或无符号`8/16/32/etc`或浮点数/双精度值，并将其作为数字返回。这里拿`readS32()、readU32()`作为演示.

```js
    //从pointer地址读4个字节 有符号
    console.log(pointer.readS32());
    //从pointer地址读4个字节 无符号
    console.log(pointer.readU32());

/*
输出如下。
           0  1  2  3  4  5  6  7  8  9  A  B  C  D  E  F  0123456789ABCDEF
00000000  7f 45 4c 46 01 01 01 00 00 00 00 00 00 00 00 00  .ELF............
1179403647 == 0x464c457f
1179403647 == 0x464c457f */
```

### 1.2.6 writeS32()、writeU32()

将有符号或无符号`8/16/32/`等或浮点数/双精度值写入此内存位置。

```js
    //申请四个字节的内存空间
    const r = Memory.alloc(4);
    //将0x12345678写入r地址中
    r.writeS32(0x12345678);
    //输出
    console.log(r.readByteArray(0x10));
    // writeS32()、writeU32()输出的也是一样的，只是区别是有符号和无符号

/*
输出如下。
           0  1  2  3  4  5  6  7  8  9  A  B  C  D  E  F  0123456789ABCDEF
00000000  78 56 34 12 00 00 00 00 00 00 00 00 00 00 00 00  xV4............. */
```

### 1.2.7 readByteArray(length))、writeByteArray(bytes)

`readByteArray(length))`连续读取内存`length`个字节，、`writeByteArray`连续写入内存`bytes`。

```js
       //先定义一个需要写入的字节数组
       var arr = [ 0x72, 0x6F, 0x79, 0x73, 0x75, 0x65];
       //这里申请以arr大小的内存空间
       const r = Memory.alloc(arr.length);
       //将arr数组字节写入r
       Memory.writeByteArray(r,arr);
       //读取arr.length大小的数组
       var buffer = Memory.readByteArray(r, arr.length);
       console.log("Memory.readByteArray:");
       console.log(hexdump(buffer, {
            offset: 0,
            length: arr.length,
            header: true,
            ansi: false
        }));

/*
输出如下。       
Memory.readByteArray:
           0  1  2  3  4  5  6  7  8  9  A  B  C  D  E  F  0123456789ABCDEF
00000000  72 6f 79 73 75 65                                roysue */
```

### 1.2.8 readCString([size = -1])、writeUtf8String(str)

`readCString`功能是读取指针地址位置的字节字符串，对应的`writeUtf8String`是写入指针地址位置的字符串处。（这里的`r`是接着上面的代码的变量）。

```js
        //在这里直接使用readCString读取会把上面的'roysue'字符串读取出来
        console.log("readCString():"+r.readCString());
        //这里是写入字符串 也就是 roysue起始位置开始被替换为haha
        const newPtrstr = r.writeUtf8String("haha");
        //替换完了之后再继续输出 必然是haha
        console.log("readCString():"+newPtrstr.readCString());
```

咱们来看看执行的效果~~见下图1-11。

[![img](https://p2.ssl.qhimg.com/t01e8013edc788aa585.png)](https://p2.ssl.qhimg.com/t01e8013edc788aa585.png)

图1-11 终端执行

 

## 1.3 NativeFunction对象

创建新的`NativeFunction`以调用`address`处的函数(用`NativePointer`指定)，其中`rereturn Type`指定返回类型，`argTypes`数组指定参数类型。如果不是系统默认值，还可以选择指定`ABI`。对于可变函数，添加一个‘.’固定参数和可变参数之间的`argTypes`条目，我们来看看官方的例子。

```js
// LargeObject HandyClass::friendlyFunctionName();
//创建friendlyFunctionPtr地址的函数
var friendlyFunctionName = new NativeFunction(friendlyFunctionPtr,
    'void', ['pointer', 'pointer']);
//申请内存空间    
var returnValue = Memory.alloc(sizeOfLargeObject);
//调用friendlyFunctionName函数
friendlyFunctionName(returnValue, thisPtr);
```

我来看看它的格式，函数定义格式为`new NativeFunction(address, returnType, argTypes[, options])，`参照这个格式能够创建函数并且调用`！returnType和argTypes[，]`分别可以填`void、pointer、int、uint、long、ulong、char、uchar、float、double、int8、uint8、int16、uint16、int32、uint32、int64、uint64`这些类型，根据函数的所需要的type来定义即可。

在定义的时候必须要将参数类型个数和参数类型以及返回值完全匹配，假设有三个参数都是`int`，则`new NativeFunction(address, returnType, ['int', 'int', 'int'])`，而返回值是`int`则`new NativeFunction(address, 'int', argTypes[, options])`，必须要全部匹配，并且第一个参数一定要是函数地址指针。

 

## 1.4 NativeCallback对象

`new NativeCallback(func，rereturn Type，argTypes[，ABI])：`创建一个由`JavaScript`函数`func`实现的新`NativeCallback`，其中`rereturn Type`指定返回类型，`argTypes`数组指定参数类型。您还可以指定`ABI`(如果不是系统默认值)。有关支持的类型和Abis的详细信息，请参见`NativeFunction`。注意，返回的对象也是一个`NativePointer`，因此可以传递给`Interceptor#replace`。当将产生的回调与`Interceptor.replace()`一起使用时，将调用func，并将其绑定到具有一些有用属性的对象，就像`Interceptor.Attach()`中的那样。我们来看一个例子。如下，利用`NativeCallback`做一个函数替换。

```js
    Java.perform(function () {
       var add_method = new NativeFunction(Module.findExportByName('libhello.so', 'c_getSum'), 
       'int',['int','int']);
       console.log("result:",add_method(1,2));
       //在这里new一个新的函数，但是参数的个数和返回值必须对应
       Interceptor.replace(add_method, new NativeCallback(function (a, b) {
            return 123;
       }, 'int', ['int', 'int']));
       console.log("result:",add_method(1,2));
    });
```

