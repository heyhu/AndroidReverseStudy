## Hook Native

### 1.1 输出打印

#### 1.1 hexdump

hexdump，其含义:打印内存中的地址，target参数可以是ArrayBuffer或者NativePointer,
而options参数则是自定义输出格式可以填这几个参数offset、lengt、header、ansi。

```js
 var libc = Module.findBaseAddress('libnative-lib.so');
 console.log(libc)
 console.log(hexdump(libc, {
     offset: 0,
     length: 64,
     header: true,
     ansi: true
 }));
 /*
 0x78472c1000
              0  1  2  3  4  5  6  7  8  9  A  B  C  D  E  F  0123456789ABCDEF
 78472c1000  7f 45 4c 46 A01 A01 A01 00 00 00 00 00 00 00 00 00  .ELF............
 78472c1010  A02 00 b7 00 A01 00 00 00 c0 fe 00 00 00 00 00 00  ................
 78472c1020  40 00 00 00 00 00 00 00 f8 a1 A02 00 00 00 00 00  @...............
 78472c1030  00 00 00 00 40 00 38 00 08 00 40 00 19 00 18 00  ....@.8...@.....
 左面：内存地址。中间：数据的hex16进制表示 右边：数据的字符串显示 0X45 -> E
 */
```
 和ida显示的一摸一样     
 ![](pic/01.a.png)


### 1.2 声明变量类型   

| **索引** |       **API**        |                           **含义**                           |
| :------: | :------------------: | :----------------------------------------------------------: |
|    1     |     new Int64(v)     | 定义一个有符号Int64类型的变量值为v，参数v可以是字符串或者以0x开头的的十六进制值 |
|    2     |    new UInt64(v)     | 定义一个无符号Int64类型的变量值为v，参数v可以是字符串或者以0x开头的的十六进制值 |
|    3     | new NativePointer(s) |                  定义一个指针，指针地址为s                   |
|    4     |       ptr(“0”)       |                             同上                             |

代码示例：
```js
 Java.perform(function () {
    console.log("");
    console.log("new Int64(1):"+new Int64(1));
    console.log("new UInt64(1):"+new UInt64(1));
    console.log("new NativePointer(0xEC644071):"+new NativePointer(0xEC644071));
    console.log("new ptr('0xEC644071'):"+new ptr(0xEC644071));
});
/*
输出效果如下：    
new Int64(1):1 
new UInt64(1):1    
new NativePointer(0xEC644071):0xec644071   
new ptr('0xEC644071'):0xec644071*/ 
```

frida也为Int64(v)提供了一些相关的API： 

| 索引 |                       API                       |               含义               |
| :--: | :---------------------------------------------: | :------------------------------: |
|  1   | add(rhs)、sub(rhs)、and(rhs)、or(rhs)、xor(rhs) |         加、减、逻辑运算         |
|  2   |                 shr(N)、shl(n)                  |  向右/向左移位n位生成新的Int64   |
|  3   |                  Compare(Rhs)                   |         返回整数比较结果         |
|  4   |                   toNumber()                    |            转换为数字            |
|  5   |              toString([radix=10])               | 转换为可选基数的字符串(默认为10) |

代码如下：   
```js
  function hello_type() {
    Java.perform(function () {
        console.log("");
        //8888 + 1 = 8889
        console.log("8888 + 1:"+new Int64("8888").add(1));
        //8888 - 1 = 8887
        console.log("8888 - 1:"+new Int64("8888").sub(1));
        //8888 << 1 = 4444
        console.log("8888 << 1:"+new Int64("8888").shr(1));
        //8888 == 22 = 1 1是false
        console.log("8888 == 22:"+new Int64("8888").compare(22));
        //转string
        console.log("8888 toString:"+new Int64("8888").toString());
    });
}
```
代码执行效果:     
![](pic/02.a.png)   



### 1.3 Process对象

#### 1.3.1 Process.id

```
Process.id`：返回附加目标进程的`PID
```

#### 1.3.2 Process.isDebuggerAttached()

`Process.isDebuggerAttached()`：检测当前是否对目标程序已经附加

#### 1.3.3 Process.enumerateModules()

枚举当前加载的模块，返回模块对象的数组。
`Process.enumerateModules()`会枚举当前所有已加载的`so`模块，并且返回了数组`Module`对象。

```js
function frida_Process() {
    Java.perform(function () {
        var process_Obj_Module_Arr = Process.enumerateModules();
        for(var i = 0; i < process_Obj_Module_Arr.length; i++) {
            console.log("",process_Obj_Module_Arr[i].name);
        }
    });
}
setImmediate(frida_Process,0);
```

我来们开看看这段`js`代码写了啥：在`js`中能够直接使用`Process`对象的所有`api`，调用了`Process.enumerateModules()`方法之后会返回一个数组，数组中存储N个叫Module的对象，既然已经知道返回了的是一个数组，很简单我们就来`for`循环它便是，这里我使用下标的方式调用了`Module`对象的`name`属性，`name`是`so`模块的名称。终端输出了所有已加载的so：	

```
 app_process64_xposed
 libcutils.so
 libutils.so
 liblog.so
 libbinder.so
 libandroid_runtime.so
 libselinux.so
 libhwbinder.so
 libwilhelm.so
 libc++.so
 libc.so
 libm.so
 libdl.so
```

#### 1.3.4 Process.enumerateThreads()

`Process.enumerateThreads()`：枚举当前所有的线程，返回包含以下属性的对象数组：

| 索引 | 属性    | 含义                                                         |
| :--- | :------ | :----------------------------------------------------------- |
| 1    | id      | 线程id                                                       |
| 2    | state   | 当前运行状态有running, stopped, waiting, uninterruptible or halted |
| 3    | context | 带有键pc和sp的对象，它们是分别为ia32/x64/arm指定EIP/RIP/PC和ESP/RSP/SP的NativePointer对象。也可以使用其他处理器特定的密钥，例如eax、rax、r0、x0等。 |

使用代码示例如下：

```js
function frida_Process() {
    Java.perform(function () {
       var enumerateThreads =  Process.enumerateThreads();
       for(var i = 0; i < enumerateThreads.length; i++) {
        console.log("");
        console.log("id:",enumerateThreads[i].id);
        console.log("state:",enumerateThreads[i].state);
        console.log("context:",JSON.stringify(enumerateThreads[i].context));
        }
    });
}
setImmediate(frida_Process,0);
```

获取当前是所有线程之后返回了一个数组，然后循环输出它的值，如下。

```
id: 8264
state: waiting
context: {"pc":"0x78e326c37c","sp":"0x7fe5043fd0","x0":"0xfffffffffffffffc","x1":"0x7fe5044058","x2":"0x10","x3":"0xffffffff","x4":"0x0","x5":"0x8","x6":"0x71e0ea7c","x7":"0x0","x8":"0x16","x9":"0x469d7947143ba856","x10":"0xd","x11":"0x786160d870","x12":"0x0","x13":"0x22","x14":"0xcba","x15":"0x9","x16":"0x78e32ce498","x17":"0x78e321f654","x18":"0x8","x19":"0x78e630ba40","x20":"0x7855431768","x21":"0xffffffff","x22":"0xffffffff","x23":"0x78554316c0","x24":"0x28","x25":"0xc","x26":"0x78e630ba40","x27":"0x786163eb40","x28":"0x13680060","fp":"0x7fe5043ff0","lr":"0x78e321f68c"}

id: 8273
state: waiting
context: {"pc":"0x78e321d82c","sp":"0x78591ff380","x0":"0x7861642670","x1":"0x0","x2":"0x25","x3":"0x0","x4":"0x0","x5":"0x0","x6":"0x0","x7":"0xffffffffffffffff","x8":"0x62","x9":"0x469d7947143ba856","x10":"0x0","x11":"0x1","x12":"0x1","x13":"0x256","x14":"0x73","x15":"0x8","x16":"0x78615f5670","x17":"0x78e321d810","x18":"0x0","x19":"0x7861642660","x20":"0x785920f000","x21":"0x25","x22":"0x7861642670","x23":"0x7861642660","x24":"0x786147b430","x25":"0x78591ff588","x26":"0x1","x27":"0x16","x28":"0x7861642609","fp":"0x78591ff3e0","lr":"0x78610a7a98"}

```

#### 1.3.5 Process.getCurrentThreadId()

`Process.getCurrentThreadId()`：获取此线程的操作系统特定 `ID` 作为数字



### 1.4 Module对象

上章节中`Process.EnumererateModules()`方法返回了就是一个`Module`对象，咱们这里来详细说说`Module`对象，先来瞧瞧它都有哪些属性。

#### 1.4.1 Module对象的属性

| 索引 | 属性 |                含义                 |
| :--: | :--: | :---------------------------------: |
|  1   | name |              模块名称               |
|  2   | base | 模块地址，其变量类型为NativePointer |
|  3   | size |                大小                 |
|  4   | path |          完整文件系统路径           |

##### 1.4.2 Module对象的API

| 索引 |                             API                              |                   含义                   |
| :--: | :----------------------------------------------------------: | :--------------------------------------: |
|  1   |                        Module.load()                         |    加载指定so文件，返回一个Module对象    |
|  2   |                      enumerateImports()                      | 枚举所有Import库函数，返回Module数组对象 |
|  3   |                      enumerateExports()                      | 枚举所有Export库函数，返回Module数组对象 |
|  4   |                      enumerateSymbols()                      | 枚举所有Symbol库函数，返回Module数组对象 |
|  5   | Module.findExportByName(exportName)、Module.getExportByName(exportName) |     寻找指定so中export库中的函数地址     |
|  6   |  Module.findBaseAddress(name)、Module.getBaseAddress(name)   |              返回so的基地址              |

#### 1.4.3 Module.load()

在`frida-12-5`版本中更新了该`API`，主要用于加载指定`so`文件，返回一个`Module`对象。

使用代码示例如下：

```js
function frida_Module() {
    Java.perform(function () {
         //参数为so的名称 返回一个Module对象
         const hooks = Module.load('libnative-lib.so');
         //输出
         console.log("模块名称:",hooks.name);
         console.log("模块地址:",hooks.base);
         console.log("大小:",hooks.size);
         console.log("文件系统路径",hooks.path);
    });
}
setImmediate(frida_Module,0);

/*
输出如下：
模块名称: libnative-lib.so
模块地址: 0x78472c1000
大小: 245760
文件系统路径 /data/app/com.example.demoso1-v1xfTV75c5A89ZIUfqmeQg==/lib/arm64/libnative-lib.so */
```

#### 1.4.4 Process.EnumererateModules()

咱们这一小章节就来使用`Module`对象，把上章的`Process.EnumererateModules()`对象输出给它补全了，代码如下。

```js
function frida_Module() {
    Java.perform(function () {

        var process_Obj_Module_Arr = Process.enumerateModules();
        for(var i = 0; i < process_Obj_Module_Arr.length; i++) {
            if(process_Obj_Module_Arr[i].path.indexOf("hello")!=-1)
            {
                console.log("模块名称:",process_Obj_Module_Arr[i].name);
                console.log("模块地址:",process_Obj_Module_Arr[i].base);
                console.log("大小:",process_Obj_Module_Arr[i].size);
                console.log("文件系统路径",process_Obj_Module_Arr[i].path);
            }
         }
    });
}
setImmediate(frida_Module,0);

/*
输出如下：
模块名称: libhello.so
模块地址: 0xdf2d3000
大小: 24576
文件系统路径 /data/app/com.roysue.roysueapplication-7adQZoYIyp5t3G5Ef5wevQ==/lib/arm/libhello.so */
```

这边如果去除判断的话会打印所有加载的`so`的信息，这里我们就知道了哪些方法返回了`Module`对象了，然后我们再继续深入学习`Module`对象自带的`API`。

#### 1.4.5 enumerateImports()

该API会枚举模块中所有中的所有Import函数，示例代码如下。

```js
function frida_Module() {
    Java.perform(function () {
        const hooks = Module.load('libhello.so');
        var Imports = hooks.enumerateImports();
        for(var i = 0; i < Imports.length; i++) {
            //函数类型
            console.log("type:",Imports[i].type);
            //函数名称
            console.log("name:",Imports[i].name);
            //属于的模块
            console.log("module:",Imports[i].module);
            //函数地址
            console.log("address:",Imports[i].address);
         }
    });
}
setImmediate(frida_Module,0);

/*
输出如下：
[Google Pixel::com.roysue.roysueapplication]-> type: function
name: __cxa_atexit
module: /system/lib/libc.so
address: 0xf58f4521
type: function
name: __cxa_finalize
module: /system/lib/libc.so
address: 0xf58f462d                                                                                                                                           
type: function
name: __stack_chk_fail
module: /system/lib/libc.so
address: 0xf58e2681 
... */
```

#### 1.4.6 enumerateExports()

该API会枚举模块中所有中的所有`Export`函数，示例代码如下。

```js
function frida_Module() {
    Java.perform(function () {
        const hooks = Module.load('libhello.so');
        var Exports = hooks.enumerateExports();
        for(var i = 0; i < Exports.length; i++) {
            //函数类型
            console.log("type:",Exports[i].type);
            //函数名称
            console.log("name:",Exports[i].name);
            //函数地址
            console.log("address:",Exports[i].address);
         }
    });
}
setImmediate(frida_Module,0);

/*
输出如下：
[Google Pixel::com.example.demoso1]-> type: function
name: Java_com_example_demoso1_MainActivity_myfirstjniJNI
module: undefined
address: 0x78472d1448
type: variable
name: _ZTIDu
module: undefined
address: 0x78472fb568
... */
```

#### 1.4.7 enumerateSymbols()

代码示例如下。

```js
function frida_Module() {
    Java.perform(function () {
        const hooks = Module.load('libc.so');
        var Symbol = hooks.enumerateSymbols();
        for(var i = 0; i < Symbol.length; i++) {
            console.log("isGlobal:",Symbol[i].isGlobal);
            console.log("type:",Symbol[i].type);
            console.log("section:",JSON.stringify(Symbol[i].section));
            console.log("name:",Symbol[i].name);
            console.log("address:",Symbol[i].address);
         }
    });
}
setImmediate(frida_Module,0);

/*
输出如下：
isGlobal: true
type: function
section: {"id":"13.text","protection":"r-x"}
name: _Unwind_GetRegionStart
address: 0xf591c798
isGlobal: true
type: function
section: {"id":"13.text","protection":"r-x"}
name: _Unwind_GetTextRelBase
address: 0xf591c7cc
... */
```

#### 1.4.8 Module.findExportByName(exportName), Module.getExportByName(exportName)

返回`so`文件中`Export`函数库中函数名称为`exportName`函数的绝对地址。

代码示例如下。

```js
function frida_Module() {
    Java.perform(function () {
        Module.getExportByName('libhello.so', 'c_getStr')
        console.log("Java_com_roysue_roysueapplication_hellojni_getStr address:",Module.findExportByName('libhello.so', 'Java_com_roysue_roysueapplication_hellojni_getStr'));
        console.log("Java_com_roysue_roysueapplication_hellojni_getStr address:",Module.getExportByName('libhello.so', 'Java_com_roysue_roysueapplication_hellojni_getStr'));
    });
}
setImmediate(frida_Module,0);

/*
输出如下：
Java_com_roysue_roysueapplication_hellojni_getStr address: 0xdf2d413d
Java_com_roysue_roysueapplication_hellojni_getStr address: 0xdf2d413d */
```

#### 1.4.9 Module.findBaseAddress(name)、Module.getBaseAddress(name)

返回`name`模块的基地址。

代码示例如下。

```js
function frida_Module() {
    Java.perform(function () {
        var name = "libhello.so";
        console.log("so address:",Module.findBaseAddress(name));
        console.log("so address:",Module.getBaseAddress(name));
    });
}
setImmediate(frida_Module,0);

/*
输出如下：
so address: 0xdf2d3000
so address: 0xdf2d3000 */
```



### 1.5 Memory对象

`Memory`的一些`API`通常是对内存处理，譬如`Memory.copy()`复制内存，又如`writeByteArray`写入字节到指定内存中，那我们这章中就是学习使用`Memory API`向内存中写入数据、读取数据。

#### 1.5.1 Memory.scan搜索内存数据

其主要功能是搜索内存中以`address`地址开始，搜索长度为`size`，需要搜是条件是`pattern，callbacks`搜索之后的回调函数；此函数相当于搜索内存的功能。

我们来直接看例子，然后结合例子讲解，如下图`1-5`。

[![img](https://p0.ssl.qhimg.com/t012b2f2c69c430786a.png)](https://p0.ssl.qhimg.com/t012b2f2c69c430786a.png)

​																			图1-5 IDA中so文件某处数据

如果我想搜索在内存中`112A`地址的起始数据要怎么做，代码示例如下。

```js
function frida_Memory() {
    Java.perform(function () {
        //先获取so的module对象
        var module = Process.findModuleByName("libhello.so"); 
        //??是通配符
        var pattern = "A02 49 ?? 50 20 44";
        //基址
        console.log("base:"+module.base)
        //从so的基址开始搜索，搜索大小为so文件的大小，搜指定条件03 49 ?? 50 20 44的数据
        var res = Memory.scan(module.base, module.size, pattern, {
            onMatch: function(address, size){
                //搜索成功
                console.log('搜索到 ' +pattern +" 地址是:"+ address.toString());  
            }, 
            onError: function(reason){
                //搜索失败
                console.log('搜索失败');
            },
            onComplete: function()
            {
                //搜索完毕
                console.log("搜索完毕")
            }
          });
    });
}
setImmediate(frida_Memory,0);
```

先来看看回调函数的含义，`onMatch：function(address，size)`：使用包含作为`NativePointer`的实例地址的`address`和指定大小为数字的`size`调用，此函数可能会返回字符串`STOP`以提前取消内存扫描。`onError：Function(Reason)`：当扫描时出现内存访问错误时使用原因调用。`onComplete：function()`：当内存范围已完全扫描时调用。

我们来来说上面这段代码做了什么事情：搜索`libhello.so`文件在内存中的数据，搜索以`pattern`条件的在内存中能匹配的数据。搜索到之后根据回调函数返回数据。

我们来看看执行之后的效果图`1-6`。

[![img](https://p1.ssl.qhimg.com/t0162e674a6690c141c.png)](https://p1.ssl.qhimg.com/t0162e674a6690c141c.png)

​																			图1-6 终端执行

我们要如何验证搜索到底是不是图`1-5`中`112A`地址，其实很简单。`so`的基址是`0xdf2d3000`，而搜到的地址是`0xdf2d412a`，我们只要`df2d412a-df2d3000=112A`。就是说我们已经搜索到了！在ida中搜索也需要使用`112A`才能搜到。

#### 1.5.2 搜索内存数据Memory.scanSync

功能与`Memory.scan`一样，只不过它是返回多个匹配到条件的数据。
代码示例如下。

```js
function frida_Memory() {
    Java.perform(function () {
        var module = Process.findModuleByName("libhello.so"); 
        var pattern = "A02 49 ?? 50 20 44";
        var scanSync = Memory.scanSync(module.base, module.size, pattern);
        console.log("scanSync:"+JSON.stringify(scanSync));
    });
}
setImmediate(frida_Memory,0);

/*
输出如下，可以看到地址搜索出来是一样的
scanSync:[{"address":"0xdf2d412a","size":6}] */
```

#### 1.5.3 内存分配Memory.alloc

在目标进程中的堆上申请`size`大小的内存，并且会按照`Process.pageSize`对齐，返回一个`NativePointer`，并且申请的内存如果在`JavaScript`里面没有对这个内存的使用的时候会自动释放的。也就是说，如果你不想要这个内存被释放，你需要自己保存一份对这个内存块的引用。

使用案例如下

```js
function frida_Memory() {
    Java.perform(function () {
        const r = Memory.alloc(10);
        console.log(hexdump(r, {
            offset: 0,
            length: 10,
            header: true,
            ansi: false
        }));
    });
}
setImmediate(frida_Memory,0);
```

以上代码在目标进程中申请了`10`字节的空间~我们来看执行脚本的效果图`1-7`。

[![img](https://p2.ssl.qhimg.com/t01315da1bd55915455.png)](https://p2.ssl.qhimg.com/t01315da1bd55915455.png)

​															    图1-7 终端执行

可以看到在`0xdfe4cd40`处申请了`10`个字节内存空间~

也可以使用：
`		Memory.allocUtf8String(str)` 分配utf字符串
`Memory.allocUtf16String` 分配utf16字符串
`Memory.allocAnsiString` 分配ansi字符串

#### 1.5.4 内存复制Memory.copy

如同`c api memcp`一样调用，使用案例如下。

```js
function frida_Memory() {
    Java.perform(function () {
        //获取so模块的Module对象
        var module = Process.findModuleByName("libhello.so"); 
        //条件
        var pattern = "A02 49 ?? 50 20 44";
        //搜字符串 只是为了将so的内存数据复制出来 方便演示~
        var scanSync = Memory.scanSync(module.base, module.size, pattern);
        //申请一个内存空间大小为10个字节
        const r = Memory.alloc(10);
        //复制以module.base地址开始的10个字节 那肯定会是7F 45 4C 46...因为一个ELF文件的Magic属性如此。
        Memory.copy(r,module.base,10);
        console.log(hexdump(r, {
            offset: 0,
            length: 10,
            header: true,
            ansi: false
        }));
    });
}
setImmediate(frida_Memory,0);

/*
输出如下。
           0  1  2  3  4  5  6  7  8  9  A  B  C  D  E  F  0123456789ABCDEF
e8142070  7f 45 4c 46 A01 A01 A01 00 00 00                    .ELF......  */
```

从`module.base`中复制`10`个字节的内存到新年申请的`r`内

#### 1.5.5 写入内存Memory.writeByteArray

将字节数组写入一个指定内存，代码示例如下:

```js
function frida_Memory() {     
    Java.perform(function () {
        //定义需要写入的字节数组 这个字节数组是字符串"roysue"的十六进制
        var arr = [ 0x72, 0x6F, 0x79, 0x73, 0x75, 0x65];
        //申请一个新的内存空间 返回指针 大小是arr.length
        const r = Memory.alloc(arr.length);
        //将arr数组写入R地址中
        Memory.writeByteArray(r,arr);
        //输出
        console.log(hexdump(r, {
            offset: 0,
            length: arr.length,
            header: true,
            ansi: false
        }));  
    });
}
setImmediate(frida_Memory,0);

/*
输出如下。
           0  1  2  3  4  5  6  7  8  9  A  B  C  D  E  F  0123456789ABCDEF
00000000  72 6f 79 73 75 65                                roysue */
```

#### 1.5.6 读取内存Memory.readByteArray

将一个指定地址的数据，代码示例如下:

```js
function frida_Memory() {     
    Java.perform(function () {
        //定义需要写入的字节数组 这个字节数组是字符串"roysue"的十六进制
        var arr = [ 0x72, 0x6F, 0x79, 0x73, 0x75, 0x65];
        //申请一个新的内存空间 返回指针 大小是arr.length
        const r = Memory.alloc(arr.length);
        //将arr数组写入R地址中
        Memory.writeByteArray(r,arr);
        //读取r指针，长度是arr.length 也就是会打印上面一样的值
        var buffer = Memory.readByteArray(r, arr.length);
        //输出
        console.log("Memory.readByteArray:");
        console.log(hexdump(buffer, {
            offset: 0,
            length: arr.length,
            header: true,
            ansi: false
        }));
      });
}
setImmediate(frida_Memory,0);

/*
输出如下。
[Google Pixel::com.roysue.roysueapplication]-> Memory.readByteArray:
           0  1  2  3  4  5  6  7  8  9  A  B  C  D  E  F  0123456789ABCDEF
00000000  72 6f 79 73 75 65                                roysue */
```
