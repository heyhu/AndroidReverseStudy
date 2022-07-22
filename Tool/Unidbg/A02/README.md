<!-- @import "[TOC]" {cmd="toc" depthFrom=1 depthTo=6 orderedList=false} -->
<!-- code_chunk_output -->

* [奇技淫巧](#奇技淫巧)

  - [0x01. PatchCode](#0x01-PatchCode)

  * [0x02. Hook](#0x02-Hook)
    - [HookZz--参数位置](#HookZz--参数位置)
    - [HookZz--寄存器](#HookZz--寄存器)
    - [HookZz--InlineHook](#HookZz--InlineHook)
    - [UnicornHook](#UnicornHook)
    - [Xhook](#xhook)
    - [ConsoleDebugger](#ConsoleDebugger)
    - [监控内存访问](#监控内存访问)
    - [查看函数执行流的汇编长度](#查看函数执行流的汇编长度)
    - [searchData](#searchData)
  * [0x03. 主动调用](#0x03-主动调用)
    - [使用原生函数](#使用原生函数)
    - [封装的API](#Unidbg封装的API)
  * [0x04. Trace](#0x04-Trace)
    - [打印寄存器的值](#打印寄存器的值)
    - [traceRead](#traceRead)
    - [traceWrite](#traceWrite)
    - [traceFunctionCall](#traceFunctionCall)
  * [0x05. Unidbg使用反射补Java的类](#0x05-Unidbg使用反射补Java的类)
  * [0x06. 打开系统调用日志](#0x06-打开系统调用日志)
  * [0x07. 使用Unidbg打印函数参数5之后的值](#0x07-使用Unidbg打印函数参数5之后的值)
  * [0x08. 使用Unidbg打印jobject](#0x08-使用Unidbg打印jobject)
  * [0x09. 固定随机数与时间戳](#0x09-固定随机数与时间戳)
    - [lrand48](#lrand48)
    - [gettimeofday](#gettimeofday)
  * [0x0a. 数据格式转换](#0x0a-数据格式转换)
    - [bytesToHex](#bytesToHex)
    - [hexStringToBytes](#hexStringToBytes)

<!-- /code_chunk_output -->

## 奇技淫巧

### 0x01. PatchCode

报错日志：

```java
JNIEnv->FindClass(android/content/ContextWrapper) was called from RX@0x40002c4f[libutility.so]0x2c4f
JNIEnv->GetMethodID(android/content/ContextWrapper.getPackageManager()Landroid/content/pm/PackageManager;) => 0x53f2c391 was called from RX@0x40002c69[libutility.so]0x2c69
JNIEnv->FindClass(android/content/pm/PackageManager) was called from RX@0x40002c79[libutility.so]0x2c79
[14:04:03 080]  WARN [com.github.unidbg.linux.ARM32SyscallHandler] (ARM32SyscallHandler:469) - handleInterrupt intno=2, NR=0, svcNumber=0x11e, PC=unidbg@0xfffe0274, LR=RX@0x40002c8d[libutility.so]0x2c8d, syscall=null
com.github.unidbg.arm.backend.BackendException
	at com.github.unidbg.linux.android.dvm.DalvikVM$31.handle(DalvikVM.java:498)
	at com.github.unidbg.linux.ARM32SyscallHandler.hook(ARM32SyscallHandler.java:105)
	at com.github.unidbg.arm.backend.UnicornBackend$10.hook(UnicornBackend.java:323)
```

`sub_1C60`(这个函数会返回一个值，如果为真，就继续执行，为假，就返回0) --call--  `sub_2C3C`(PackageManager之流，联想到签名校验函数)

需要让`sub_2C3C`函数返回真正的值，不能返回0，否则`sub_1C60`也返回的是0，直接patch掉对`sub_2C3C`函数的调用，就是把这儿的函数跳转改成不跳转了。

```
.text:00001E7C F0 B5                       PUSH    {R4-R7,LR}
.text:00001E7E 11 1C                       MOVS    R1, R2
.text:00001E80 89 B0                       SUB     SP, SP, #0x24
.text:00001E82 1D 1C                       MOVS    R5, R3
.text:00001E84 04 1C                       MOVS    R4, R0
.text:00001E86 FF F7 EB FE                 BL      sub_1C60     PATCH掉对`sub_1C60`函数的调用
.text:00001E8A 05 95                       STR     R5, [SP,#0x38+var_24]
.text:00001E8C 00 28                       CMP     R0, #0
.text:00001E8E 00 D1                       BNE     loc_1E92
.text:00001E90 9D E0                       B       loc_1FCE
```

正常执行这个函数的话，如果校验没问题返回真，比如1，校验失败返回0。

根据ARM调用约定，入参前四个分别通过R0-R3调用，返回值通过R0返回，所以这儿可以通过“mov r0,1”实现我们的目标——不执行这个函数，并给出正确的返回值。除此之外还有一个幸运的地方在于，这个函数并没有产生一些之后需要使用的值或者中间变量，这让我们不需要管别的寄存器。

此处的机器码是FF F7 EB FE, 查看一下“mov r0,1”的机器码，这里我们使用[ARMConvert](https://armconverter.com/?code=mov r0,1)看一下，除此之外，使用别的工具查看汇编代码也是可以的。

![01.a](pic/01.png)

即把 FF F7 EB FE 替换成 4FF00100 即可。

Unidbg提供了`两种`方法打Patch，简单的需求可以调用Unicorn对虚拟内存进行修改，如下:

- 使用机器码，**这里的地址不需要+1，Thumb的+1只在运行和Hook时需要考虑**，打Patch不需要。

  ```java
      public void  patchVerify() {
          // arm指令为 `mov r0,1` 转换为机器码
          int patchCode = 0x4FF00100;
          // 不用加1，Thumb的+1只在运行和Hook时需要考虑
          emulator.getMemory().pointer(module.base + 0x1E86).setInt(0, patchCode);
      }
  ```

- 使用指令，先确认有没有找对地方，地址上是不是 FF F7 EB FE，再用Unicorn的好兄弟Keystone 把patch代码“mov r0,1"转成机器码，填进去，校验一下长度是否相等。

  ```java
      public void patchVerify1() {
          Pointer pointer = UnidbgPointer.pointer(emulator, module.base + 0x1E86);
          assert pointer != null;
          // 获取前4个机器码 FF F7 EB FE
          byte[] code = pointer.getByteArray(0, 4);
          // 判断是否相等，相等才操作
          if (!Arrays.equals(code, new byte[]{(byte) 0xFF, (byte) 0xF7, (byte) 0xEB, (byte) 0xFE})) {
              throw new IllegalStateException(Inspector.inspectString(code, "patch32 code=" + Arrays.toString(code)));
          }
          try (Keystone keystone = new Keystone(KeystoneArchitecture.Arm, KeystoneMode.ArmThumb)) {
              KeystoneEncoded encoded = keystone.assemble("mov r0,1");
              // 指令转为机器码
              byte[] patch = encoded.getMachineCode();
              if (patch.length != code.length) {
                  throw new IllegalStateException(Inspector.inspectString(patch, "patch32 length=" + patch.length));
              }
              pointer.write(0, patch, 0, patch.length);
          }
      }
  ```

- 直接跳过函数执行处。在调用处下了一个断点，inline hook 都是在调用前断下来了，所以在行前断了下来，然后加了BreakPointCallback回调，在这个时机做处理，我们将PC指针直接往前加了 5，PC指向哪里，函数就执行哪里，本来PC指向“blx log”这个地址，程序即将去执行log函数。但我们直 接将PC加了5，为什么加5? 我们知道这里的log是个坑，它长四个字节，我们要越过这个坑，但加4不够，我们是thumb模式，再 +1，所以就是+5。

  除此之外，OnHit命中断点时返回true，true表示不用断下来变成调试模式，继续往下走。

  ```java
      public void patchLog2() {
          emulator.attach().addBreakPoint(module.base + 0xABE, new BreakPointCallback() {
              @Override
              public boolean onHit(Emulator<?> emulator, long address) {
                  emulator.getBackend().reg_write(ArmConst.UC_ARM_REG_PC, (address) + 5);
                  return true;
              }
          });
      }
  ```
  
  ```java
      public void hookonegetstring() {
          emulator.attach().addBreakPoint(module.base + 0x51e51, new BreakPointCallback() {
              @Override
              public boolean onHit(Emulator<?> emulator, long address) {
                  String input = "d7b7d042-d4f2-4012-be60-d97ff2429c17";
                  MemoryBlock replaceBlock = emulator.getMemory().malloc(input.length(), true);
                  replaceBlock.getPointer().write(input.getBytes(StandardCharsets.UTF_8));
                  // 修改r0为指向新字符串的新指针
                  emulator.getBackend().reg_write(ArmConst.UC_ARM_REG_R0, replaceBlock.getPointer().peer);
                  emulator.getBackend().reg_write(ArmConst.UC_ARM_REG_PC, address + 3);
                  return true;
              }
          });
      }
  ```

### 0x02. Hook

#### HookZz--参数位置

```java
public void HookMDStringold() {
        // 加载HookZz
        IHookZz hookZz = HookZz.getInstance(emulator);
        // inline wrap导出函数
        hookZz.wrap(module.base + 0x1BD0 + 1, new WrapCallback<HookZzArm32RegisterContext>() {
            @Override
            // 类似于 frida onEnter
            public void preCall(Emulator<?> emulator, HookZzArm32RegisterContext ctx, HookEntryInfo info) {
                // 类似于Frida args[0]
                Pointer input = ctx.getPointerArg(0);
                System.out.println("input:" + input.getString(0));
            }

            @Override
            // 类似于 frida onLeave
            public void postCall(Emulator<?> emulator, HookZzArm32RegisterContext ctx, HookEntryInfo info) {
                Pointer result = ctx.getPointerArg(0);
                System.out.println("result:" + result.getString(0));
            }
        });
    }
```

函数Hook:

![](pic/02.png)

```java
    // HookZz hook Sc_EncryptWallEncode
    public void hookEncryptWallEncode() {
        // 获取HookZz对象，https://github.com/jmpews/HookZz
        IHookZz hookZz = HookZz.getInstance(emulator); // 加载HookZz，支持inline hook，文档看
        // enable hook
        hookZz.enable_arm_arm64_b_branch(); // 测试enable_arm_arm64_b_branch
        hookZz.wrap(module.base + 0xA284 + 1, new WrapCallback<HookZzArm32RegisterContext>() {
            Pointer buffer;
            @Override
            // 方法执行前
            public void preCall(Emulator<?> emulator, HookZzArm32RegisterContext ctx, HookEntryInfo info) {
                System.out.println("HookZz hook EncryptWallEncode");
                Pointer input1 = ctx.getPointerArg(0);
                Pointer input2 = ctx.getPointerArg(1);
                Pointer input3 = ctx.getPointerArg(2);
                // getString的参数i代表index,即input[i:]
                System.out.println("参数1：" + input1.getString(0));
                System.out.println("参数2：" + input2.getString(0));
                System.out.println("参数3：" + input3.getString(0));

                buffer = ctx.getPointerArg(3);
            }
            @Override
            // 方法执行后
            public void postCall(Emulator<?> emulator, HookZzArm32RegisterContext ctx, HookEntryInfo info) {
                // getByteArray参数1是起始index，参数2是长度，我们不知道结果多长，就先设置0x100吧
                byte[] outputhex = buffer.getByteArray(0, 0x100);
                Inspector.inspect(outputhex, "EncryptWallEncode output");
            }
        });
        hookZz.disable_arm_arm64_b_branch();
    }

```

#### HookZz--寄存器

- 编写对该函数的Hook，首先因为不确定三个参数是指针还是数值，所以先全部做为数值处理，作为long类型看待，防止整数溢出
- Inspector.inspect其效果类似于frida中hexdump
- getR0long == emulator.getBackend().reg_read(ArmConst.UC_ARM_REG_R0)

```java
public void hook65540(){
  // 加载HookZz
  IHookZz hookZz = HookZz.getInstance(emulator);

  hookZz.wrap(module.base + 0x65540 + 1, new WrapCallback<HookZzArm32RegisterContext>() { // inline wrap导出函数
    @Override
    // 类似于 frida onEnter
    public void preCall(Emulator<?> emulator, HookZzArm32RegisterContext ctx, HookEntryInfo info) {
      // 类似于Frida hook寄存器r0
      Inspector.inspect(ctx.getR0Pointer().getByteArray(0, 0x10), "Arg1");
      // 因为是长度，所以用getR1Long
      System.out.println(ctx.getR1Long());
      Inspector.inspect(ctx.getR2Pointer().getByteArray(0, 0x10), "Arg3");
      // push
      ctx.push(ctx.getR2Pointer());
    };

    @Override
    // 类似于 frida onLeave
    public void postCall(Emulator<?> emulator, HookZzArm32RegisterContext ctx, HookEntryInfo info) {
      // pop 取出
      Pointer output = ctx.pop();
      Inspector.inspect(output.getByteArray(0, 0x10), "Arg3 after function");
    }
  })
}
```

#### HookZz--InlineHook

- 在有些时候，函数可能在程序中运行许多次，但我们只想观察此次此地的执行情况，那我们可能会使用inline hook，需要注意，inline hook 的时机是目标指令执行前。

  ![](pic/03.png)

- 通过base+offset inline wrap内部函数，在IDA看到为sub_xxx那些

```java
    public void inlinehookEncryptWallEncode() {
        IHookZz hookZz = HookZz.getInstance(emulator);
        hookZz.enable_arm_arm64_b_branch();
        hookZz.instrument(module.base + 0x9d24 + 1, new InstrumentCallback<Arm32RegisterContext>() {
            @Override
            public void dbiCall(Emulator<?> emulator, Arm32RegisterContext ctx, HookEntryInfo info) {
                System.out.println("HookZz inline hook EncryptWallEncode");
                Pointer input1 = ctx.getPointerArg(0);
                Pointer input2 = ctx.getPointerArg(1);
                Pointer input3 = ctx.getPointerArg(2);
                buffer = ctx.getPointerArg(3);
            }
        });
        hookZz.instrument(module.base + 0x9d28 + 1, new InstrumentCallback<Arm32RegisterContext>() {
            @Override
            public void dbiCall(Emulator<?> emulator, Arm32RegisterContext ctx, HookEntryInfo info) {
                Inspector.inspect(buffer.getByteArray(0, 0x100), "inline hookEncryptWallEncode");
            }
        });
    }
```

#### xhook

```java
public void recordPopenByxHook(){
  IxHook xHook = XHookImpl.getInstance(emulator);
  xHook.register("libszstone.so", "popen", new ReplaceCallback() {
    @Override
    public HookStatus onCall(Emulator<?> emulator, HookContext context, long originFunction) {
      RegisterContext registerContext = emulator.getContext();
      String command = registerContext.getPointerArg(0).getString(0);
      emulator.set("command", command);
      return HookStatus.RET(emulator, originFunction);
    }
  }, true);
  // 使其生效
  xHook.refresh();
}
```

#### UnicornHook

- 原生的办法进行Hook，代码量不小，但很多时候，我们会选择它，因为HookZz等工具有时 候会遇到BUG，而且使用HookZz等hook框架时，样本可以较容易的检测到自身代码片段被Hook，而 Unicorn原生的Hook不容易被检测，相当于是CPU自身在打印寄存器。

```java
public void hookByUnicorn(){
  emulator.getBackend().hook_add_new(new CodeHook() {
    @Override
    public void onAttach(UnHook unHook) {
    }

    @Override
    public void detach() {
    }

    @Override
    public void hook(Backend backend, long address, int size, Object user) {
      if(address == (module.base+0x9d24)){
        System.out.println("Hook By Unicorn");
        RegisterContext ctx = emulator.getContext();
        Pointer input1 = ctx.getPointerArg(0);
        Pointer input2 = ctx.getPointerArg(1);
        Pointer input3 = ctx.getPointerArg(2);
        // getString的参数i代表index,即input[i:]
        System.out.println("参数1："+input1.getString(0));
        System.out.println("参数2："+input2.getString(0));
        System.out.println("参数3："+input3.getString(0));

        buffer = ctx.getPointerArg(3);
      }
      if(address == (module.base+0x9d28)){
        Inspector.inspect(buffer.getByteArray(0,0x100), "Unicorn hook EncryptWallEncode");
      }
    }
  },module.base + 0x9d24, module.base + 0x9d28, null);
}
```

#### ConsoleDebugger

```java
import com.github.unidbg.debugger.Debugger;
// 在类的构造函数中添加
Debugger debugger = emulator.attach();
debugger.addBreakPoint(module.base + 0x1ecc + 1);

//下面列举常见的断点命令
1. mr0查看r0所指向的内存块，它等同于Frida native hook中的hexdump(this.context.r0)。
2. 按c继续运行
3. 输入bt指令回车（bt即backtrace缩写）打印调用栈。
4. 查看函数离开后r0寄存器的值，r0此场景为buffer。
   a）暂停到断点处的函数
   b）mr0查看r0寄存器的值，此时r0寄存器的地址为0xbffff660。
   c) 输入blr，此命令用于在函数返回时设置一个一次性断点
   d）然后运行c运行函数，它此时在返回处断下来。有个问题，mr0这时候并不代表入参时的r0了，但没关系，记住mr0的address即可。
   e）m0xbffff660 查看0xbffff660
   f）c继续执行
5. 基于Unicorn的Console Debugger同样不用因为thumb模式+1，会自己做转换。
```

```java
 // 程序中下断点
public void HookMemcmp(){
  emulator.attach().addBreakPoint(module.findSymbolByName("memcmp").getAddress(), new BreakPointCallback() 				{
    @Override
    public boolean onHit(Emulator<?> emulator, long address) {
      System.out.println("call memcmp 作比较");
      RegisterContext registerContext = emulator.getContext();
      UnidbgPointer arg1 = registerContext.getPointerArg(0);
      UnidbgPointer arg2 = registerContext.getPointerArg(1);
      int size = registerContext.getIntArg(2);
      Inspector.inspect(arg1.getByteArray(0, size), "arg1");
      Inspector.inspect(arg2.getByteArray(0, size), "arg2");

      if(arg1.getString(0).equals("Context")){
        emulator.attach().debug();
      }
      return true;
    }
  });
}
```

> JNIEnv->NewStringUTF  was called from RX@0x4000336f[libmakeurl.so]0x336f

如果提示调用处在0x336f，这个地址实际上是LR（返回地址），所以NewStringUTF函数调用是 0x336f 的上一条 0x336C，对这个地址打断点。

![](pic/06.png)

#### 监控内存访问

使用场景:因为白盒加密的主要实现方式是查表法，所以加密主体就是大量的内存访问。那么记录函数对内存的访问以及发起访问的地址（PC指针），绘制成折线图，就可以较好的反映加密流程。使用Unidbg的ReadHook

```java
public void traceAESRead(){
    emulator.getBackend().hook_add_new(new ReadHook() {
        @Override
        public void hook(Backend backend, long address, int size, Object user) {
            long now = emulator.getBackend().reg_read(ArmConst.UC_ARM_REG_PC).intValue();
            if((now>module.base) & (now < (module.base+module.size))){
                System.out.println(now - module.base);
            }
        }

        @Override
        public void onAttach(UnHook unHook) {

        }

        @Override
        public void detach() {

        }
    }, module.base, module.base+module.size, null);
}
```

规则如下：监控整个SO地址范围内的内存读取操作，记录其发起地址，减去了SO基地址，只打印偏移，呈现效果更好。

将这几千条记录拷贝出来，保存在trace.txt 中，在Python中做可视化，这十分方便。需要安装matplotlib以及numpy库。

```python
import matplotlib.pyplot as plt
import numpy

input = numpy.loadtxt("trace.txt", int)

plt.plot(range(len(input)), input)
plt.show()
```

运行后生成折线图，将其放大是如下效果:

![](pic/04.png)

X轴的计数单位是次数，表示当前是第几次内存访问，如图，在程序的运行过程中，发生了1400余次对SO内存的读操作，Y轴是发起访问的偏移地址。需要注意，X与Y轴的数值表示为十进制。图上可得，Y主要在80000-100000之间，我们修改Y轴范围，增强呈现效果。

```python
import matplotlib.pyplot as plt
import numpy

input = numpy.loadtxt("trace.txt", int)

# 限制Y
plt.ylim(85000, 90000)
plt.plot(range(len(input)), input)
plt.show()
```

运行后:

![](pic/05.png)

首先，可以比较明显的看到，存在十个重复的模式，这代表了十轮运算。这一点是有用的，可用于区分AES-128/192/256，分别对应10/12/14 轮。

除此之外，我们发现每轮运算的起点是一个较低的地址，具体在86000附近左右，转成十六进制就是0x14FF0附近，正是所分析的wbShiftRows中。

#### 查看函数执行流的汇编长度

确认函数执行流的汇编长度，如果上千万甚至上亿行，那我们就徐徐图之，如果几十万行，那就重拳出击。

```java
public void traceLength(){
    emulator.getBackend().hook_add_new(new CodeHook() {
        int count = 0;
        @Override
        public void hook(Backend backend, long address, int size, Object user) {
            count += 1;
            System.out.println(count);
        }

        @Override
        public void onAttach(UnHook unHook) {

        }

        @Override
        public void detach() {

        }
    }, module.base, module.size+module.base, null);
}
```

不超过100w行的执行流，要么程序没怎么混淆，要么逻辑不太复杂。两者任意一个复杂度高一些，都不会只有100w行汇编以内。

#### searchData

在SO加载前，因为模块监听器得在SO加载前才能拦截到。

```java
memory.addModuleListener(new SearchData(data, soName, intervals));
```

SearchData 构造函数的参数1是待搜索数据的16进制字符串形式，会自动省略空格。

目标数据是e70b75d06455d041df9bb125eaa2cf17，[cyberchef](https://gchq.github.io/CyberChef/#recipe=To_Hex('Space',0)&input=ZTcwYjc1ZDA2NDU1ZDA0MWRmOWJiMTI1ZWFhMmNmMTc) 中转化为十六进制形式，下面两种代码等价。

```java
memory.addModuleListener(new SearchData("65 37 30 62 37 35 64 30 36 34 35 35 64 30 34 31 64 66 39 62 62 31 32 35 65 61 61 32 63 66 31 37", "libnet_crypto.so", 100));
```

或

```java
memory.addModuleListener(newSearchData("6537306237356430363435356430343164663962623132356561613263663137", "libnet_crypto.so", 100));
```

结果可能已经做了tohex，所以直接把结果当十六进制来搜索

```java
memory.addModuleListener(new SearchData("e70b75d06455d041df9bb125eaa2cf17", "libnet_crypto.so", 100));
```

如果搜索不到，再试上面的。

参数2是SO完整的名字，参数3是搜索间隔，比如目前间隔是100个基本块单位。

![](pic/07.png)

断了下来，而且看到“find target xxx”字眼，这说明检索到了数据，我们对tohex的猜想没有错。因为我们按照100个基本块的间隔做搜索，因此当前位置离真实的数据产生位置，可能还有不少偏差。

下面使用SearchData的另一个构造函数，多一个参数，它代表着搜索的起点，搜索间隔也被我调整成了1，即逐基本块搜索。

```java
memory.addModuleListener(new SearchData("e70b75d06455d041df9bb125eaa2cf17", "libnet_crypto.so", 1,0x2466c));
```

直接搜索到![](pic/08.png)

IDA跳转到0x63615，正是之前来回跳转才找到的最终地址。

图示两次检索的效果，第一次检索![](pic/09.png)

比较粗糙，发现数据后，工具会打印减去一个间隔后所处于的基本块数，即 **find target at xxx blocks**

第二次检索，以一种更细的粒度做检索

![](pic/10.png)

我们这个样本的汇编量比较小，对于大一些的样本，比如几千万行汇编，第一次检索间隔为1000-10000个基本块比较好，第二次检索间隔为50-500即可，可以再配合内存数据搜索。

### 0x03. 主动调用

#### 使用原生函数

```java
public void callMd5(){
        List<Object> list = new ArrayList<>(10);
        // arg1
        String input = "r0ysue";
        // malloc memory
        MemoryBlock memoryBlock1 = emulator.getMemory().malloc(16, false);
        // get memory pointer
        UnidbgPointer input_ptr=memoryBlock1.getPointer();
        // write plainText on it
        input_ptr.write(input.getBytes(StandardCharsets.UTF_8));

        // arg2
        int input_length = input.length();

        // arg3 -- buffer
        MemoryBlock memoryBlock2 = emulator.getMemory().malloc(16, false);
        UnidbgPointer output_buffer=memoryBlock2.getPointer();

        // 填入参入
        list.add(input_ptr);
        list.add(input_length);
        list.add(output_buffer);
        // run
        module.callFunction(emulator, 0x65540 + 1, list.toArray());

        // print arg3
        Inspector.inspect(output_buffer.getByteArray(0, 0x10), "output");
    };
```

#### Unidbg封装的API

```java
    public void callByAddress(){
        // args list
        List<Object> list = new ArrayList<>(10);
        // jnienv
        list.add(vm.getJNIEnv());
        // jclazz
        list.add(0);
        // str1
        list.add(vm.addLocalObject(new StringObject(vm, "str1")));
        // str2
        list.add(vm.addLocalObject(new StringObject(vm, "str2")));
        // str3
        list.add(vm.addLocalObject(new StringObject(vm, "str3")));
        // str4
        list.add(vm.addLocalObject(new StringObject(vm, "str4")));
        // str5
        list.add(vm.addLocalObject(new StringObject(vm, "str5")));
        // strArr 假设字符串包含两个字符串
        // str6_1
        StringObject str6_1 = new StringObject(vm, "str6_1");
        vm.addLocalObject(str6_1);
        // str6_2
        StringObject str6_2 = new StringObject(vm, "str6_2");
        vm.addLocalObject(str6_2);

        ArrayObject arrayObject = new ArrayObject(str6_1,str6_2);
        list.add(vm.addLocalObject(arrayObject));

        // 最后的int
        list.add(1);

        Number number = module.callFunction(emulator, 0x2301, list.toArray())[0];
        ArrayObject resultArr = vm.getObject(number.intValue());
        System.out.println("result:"+resultArr);
    };

    public void callByAPI(){
        DvmClass RequestCryptUtils = vm.resolveClass("com/meituan/android/payguard/RequestCryptUtils");

        StringObject str6_1 = new StringObject(vm, "str6_1");
        vm.addLocalObject(str6_1);
        StringObject str6_2 = new StringObject(vm, "str6_2");
        vm.addLocalObject(str6_2);
        ArrayObject arrayObject = new ArrayObject(str6_1,str6_2);
        ArrayObject result = RequestCryptUtils.callStaticJniMethodObject(emulator, "encryptRequestWithRandom()", "str1","str2", "str3","str4","str5",arrayObject,1);
        System.out.println(result);
    };
```

### 0x04. Trace

#### 打印寄存器的值

- trace出来的结果几万行应该不存在高度的OLLVM混淆，也说明运算逻辑不会太复杂，否则应该百万行起步

```Java
// emulator.traceCode(module.base, module.base + module.size);
// 保存的path
String traceFile = "unidbg-android/src/test/java/com/lession5/qxstrace.txt";
PrintStream traceStream = new PrintStream(new FileOutputStream(traceFile), true);
emulator.traceCode(module.base, module.base+module.size).setRedirect(traceStream);
```

增加寄存器值信息:

- AbstractARMEmulator.java

  ```java
  // 添加值显示
  private void printAssemble(PrintStream out, Capstone.CsInsn[] insns, long address, boolean thumb) {
      StringBuilder sb = new StringBuilder();
      for (Capstone.CsInsn ins : insns) {
          sb.append("### Trace Instruction ");
          sb.append(ARM.assembleDetail(this, ins, address, thumb));
          // 打印每条汇编指令里参与运算的寄存器的值
          Set<Integer> regset = new HashSet<Integer>();
  
          Arm.OpInfo opInfo = (Arm.OpInfo) ins.operands;
          for(int i = 0; i<opInfo.op.length; i++){
              regset.add(opInfo.op[i].value.reg);
          }
          String RegChange = ARM.SaveRegs(this, regset);
          sb.append(RegChange);
          sb.append('\n');
          address += ins.size;
      }
      out.print(sb);
  }
  ```

- src/main/java/com/github/unidbg/arm/ARM.java 中，新建SaveRegs方法，实际上就是showregs的代码，只不过从print改成return回来而已

  ```java
  public static String SaveRegs(Emulator<?> emulator, Set<Integer> regs) {
          Backend backend = emulator.getBackend();
          StringBuilder builder = new StringBuilder();
          builder.append(">>>");
          Iterator it = regs.iterator();
          while(it.hasNext()) {
              int reg = (int) it.next();
              Number number;
              int value;
              switch (reg) {
                  case ArmConst.UC_ARM_REG_R0:
                      number = backend.reg_read(reg);
                      value = number.intValue();
                      builder.append(String.format(Locale.US, " r0=0x%x", value));
                      break;
                  case ArmConst.UC_ARM_REG_R1:
                      number = backend.reg_read(reg);
                      value = number.intValue();
                      builder.append(String.format(Locale.US, " r1=0x%x", value));
                      break;
                  case ArmConst.UC_ARM_REG_R2:
                      number = backend.reg_read(reg);
                      value = number.intValue();
                      builder.append(String.format(Locale.US, " r2=0x%x", value));
                      break;
                  case ArmConst.UC_ARM_REG_R3:
                      number = backend.reg_read(reg);
                      value = number.intValue();
                      builder.append(String.format(Locale.US, " r3=0x%x", value));
                      break;
                  case ArmConst.UC_ARM_REG_R4:
                      number = backend.reg_read(reg);
                      value = number.intValue();
                      builder.append(String.format(Locale.US, " r4=0x%x", value));
                      break;
                  case ArmConst.UC_ARM_REG_R5:
                      number = backend.reg_read(reg);
                      value = number.intValue();
                      builder.append(String.format(Locale.US, " r5=0x%x", value));
                      break;
                  case ArmConst.UC_ARM_REG_R6:
                      number = backend.reg_read(reg);
                      value = number.intValue();
                      builder.append(String.format(Locale.US, " r6=0x%x", value));
                      break;
                  case ArmConst.UC_ARM_REG_R7:
                      number = backend.reg_read(reg);
                      value = number.intValue();
                      builder.append(String.format(Locale.US, " r7=0x%x", value));
                      break;
                  case ArmConst.UC_ARM_REG_R8:
                      number = backend.reg_read(reg);
                      value = number.intValue();
                      builder.append(String.format(Locale.US, " r8=0x%x", value));
                      break;
                  case ArmConst.UC_ARM_REG_R9: // UC_ARM_REG_SB
                      number = backend.reg_read(reg);
                      value = number.intValue();
                      builder.append(String.format(Locale.US, " sb=0x%x", value));
                      break;
                  case ArmConst.UC_ARM_REG_R10: // UC_ARM_REG_SL
                      number = backend.reg_read(reg);
                      value = number.intValue();
                      builder.append(String.format(Locale.US, " sl=0x%x", value));
                      break;
                  case ArmConst.UC_ARM_REG_FP:
                      number = backend.reg_read(reg);
                      value = number.intValue();
                      builder.append(String.format(Locale.US, " fp=0x%x", value));
                      break;
                  case ArmConst.UC_ARM_REG_IP:
                      number = backend.reg_read(reg);
                      value = number.intValue();
                      builder.append(String.format(Locale.US, " ip=0x%x", value));
                      break;
                  case ArmConst.UC_ARM_REG_SP:
                      number = backend.reg_read(reg);
                      value = number.intValue();
                      builder.append(String.format(Locale.US, " SP=0x%x", value));
                      break;
                  case ArmConst.UC_ARM_REG_LR:
                      number = backend.reg_read(reg);
                      value = number.intValue();
                      builder.append(String.format(Locale.US, " LR=0x%x", value));
                      break;
                  case ArmConst.UC_ARM_REG_PC:
                      number = backend.reg_read(reg);
                      value = number.intValue();
                      builder.append(String.format(Locale.US, " PC=0x%x", value));
                      break;
              }
          }
          return builder.toString();
      }
  ```

#### traceRead

- 打印对某连续地址进行读操作的相关信息

- 举例：查看程序对内存地址0xbffff660的值的引用

  此时0xbffff660的值为：BD 39 85 65 07 4D F8 3A 3B 84 E1 4B 4E A0 F0 B5 EA

```java
emulator.traceRead(0xbffff620L, 0xbffff620L+20L);
```

trace的结果：

```java
### Memory READ at 0xbffff633, data size = 1, data value = 0xea pc=RX@0x40003c3e[libnative-lib.so]0x3c3e lr=RX@0x400fe617[libc.so]0x58617
### Memory READ at 0xbffff62a, data size = 4, data value = 0xa04e4be1 pc=RX@0x40003c56[libnative-lib.so]0x3c56 lr=RX@0x400fe617[libc.so]0x58617
```

上述结果的意思为，在后续运算中，结果只有五个字节被使用到了分别是：0x3c3e地址使用了最后一个值0xEA 以及 0x3c56地址使用了0xa04e4be1

#### traceWrite

- 打印对某连续地址进行写操作的相关信息

```java
JNIEnv->SetByteArrayRegion([B@77167fb7, 0, 7, unidbg@0xbffff5f8) was called from RX@0x4000185b[libnative-lib.so]0x185b
JNIEnv->NewObject(class java/lang/String, <init>([B@77167fb7, "UTF-8") => "JCD2D38") was called from RX@0x4000186d[libnative-lib.so]0x186d
```

```java
emulator.traceWrite(0xbffff5f8L, 0xbffff5f8L+7L);
```

trace的结果：

说明在0x3cba等地方对想要监控的内存地址进行了操作，操作值为data value。

```java
### Memory WRITE at 0xbffff5fc, data size = 4, data value = 0x373635 pc=RX@0x40003c9c[libnative-lib.so]0x3c9c lr=RX@0x400fe617[libc.so]0x58617
### Memory WRITE at 0xbffff5f8, data size = 4, data value = 0x34333231 pc=RX@0x40003ca0[libnative-lib.so]0x3ca0 lr=RX@0x400fe617[libc.so]0x58617
### Memory WRITE at 0xbffff5f8, data size = 1, data value = 0x4a pc=RX@0x40003cba[libnative-lib.so]0x3cba lr=RX@0x40003cb1[libnative-lib.so]0x3cb1
### Memory WRITE at 0xbffff5f9, data size = 1, data value = 0x43 pc=RX@0x40003cba[libnative-lib.so]0x3cba lr=RX@0x40003cb1[libnative-lib.so]0x3cb1
### Memory WRITE at 0xbffff5fa, data size = 1, data value = 0x44 pc=RX@0x40003cba[libnative-lib.so]0x3cba lr=RX@0x40003cb1[libnative-lib.so]0x3cb1
### Memory WRITE at 0xbffff5fb, data size = 1, data value = 0x32 pc=RX@0x40003cba[libnative-lib.so]0x3cba lr=RX@0x40003cb1[libnative-lib.so]0x3cb1
### Memory WRITE at 0xbffff5fc, data size = 1, data value = 0x44 pc=RX@0x40003cba[libnative-lib.so]0x3cba lr=RX@0x40003cb1[libnative-lib.so]0x3cb1
### Memory WRITE at 0xbffff5fe, data size = 1, data value = 0x38 pc=RX@0x40003d4e[libnative-lib.so]0x3d4e lr=RX@0x40003d3f[libnative-lib.so]0x3d3f
### Memory WRITE at 0xbffff5fd, data size = 1, data value = 0x33 pc=RX@0x40003d56[libnative-lib.so]0x3d56 lr=RX@0x40003d3f[libnative-lib.so]0x3d3f
```

PC即发生写入操作的第一现场，LR即目标函数的返回地址。可以看到，对数据的写入发生在 libc 标准库中，这一般意味着使用了memcpy 等库函数对内存块做了拷贝操作，我们可以验证一下，在IDA中看LR地址，其上一条指令就是具体发生的调用。

```java
qmemcpy(v8, a2, v5);
v10 = -1302498787;// LR
v6 = &v8[v5];
```

具体调用就是qmemcpy 函数，不用管前缀，它就是memcpy。

```
4AA76                 BLX             __aeabi_memcpy
```

#### traceFunctionCall

基本的函数追踪代码

```java
debugger.traceFunctionCall(module, new FunctionCallListener() {
    @Override
    public void onCall(Emulator<?> emulator, long callerAddress, long functionAddress) {}
    @Override
    public void postCall(Emulator<?> emulator, long callerAddress, long functionAddress, Number[] args) {
        System.out.println("onCallFinish caller=" + UnidbgPointer.pointer(emulator, callerAddress) + ", function=" + UnidbgPointer.pointer(emulator, functionAddress));
    }
});
```

```java
// aes起始处
emulator.attach().addBreakPoint(module.base + 0x142c8, new BreakPointCallback() {
    UnidbgPointer buffer;
    int length;
    @Override
    public boolean onHit(Emulator<?> emulator, long address) {
        // 使用register获取r1寄存器的内存地址
        UnidbgPointer plainTextPtr = UnidbgPointer.register(emulator, ArmConst.UC_ARM_REG_R1);
        //把hex转为bytes数组
        byte[] p = hexStringToByteArray("30313233343536373839616263646501");
        for(int i=0;i<p.length;i++){
            // 写入到r1寄存器的地址
            plainTextPtr.setByte(i, p[i]);
        }
      
        //获取SP寄存器初始内存块，也就是参数5
        UnidbgPointer spPtr = UnidbgPointer.register(emulator, ArmConst.UC_ARM_REG_SP);
        // 设置值
        spPtr.setInt(0, 0x10);
        
        // 查看操作是否生效
        RegisterContext registerContext = emulator.getContext();
        UnidbgPointer plainText = registerContext.getPointerArg(1);
        buffer = registerContext.getPointerArg(3);
        length = registerContext.getIntArg(4);
				
      	//打印plainText
        Inspector.inspect(plainText.getByteArray(0, length), "input");
				
      	// 开启函数追踪, 函数里执行 caller:调用函数的地址，function:调用的是什么函数
        emulator.attach().traceFunctionCall(module, new FunctionCallListener() {
            @Override
            public void onCall(Emulator<?> emulator, long callerAddress, long functionAddress) {
            }
            @Override
            public void postCall(Emulator<?> emulator, long callerAddress, long functionAddress, Number[] args) {
                System.out.println("onCallFinish caller=" + UnidbgPointer.pointer(emulator, callerAddress) + ", function=" + UnidbgPointer.pointer(emulator, functionAddress));
            }
        });
				// 在0x142c8函数结束返回处断下
        emulator.attach().addBreakPoint(registerContext.getLRPointer().peer, new BreakPointCallback() {
            @Override
            public boolean onHit(Emulator<?> emulator, long address) {
                // 打印buffer的值
                Inspector.inspect(buffer.getByteArray(0, length), "output");
                return false;
            }
        });
      
        // hook xorwithiv
        emulator.attach().addBreakPoint(module.base+0x13bc0);
        // hook First addroundkey
        emulator.attach().addBreakPoint(module.base+0x1187c);
        return false;
    }
});
```

### 0x05. Unidbg使用反射补Java的类

- 涉及的环境缺失是JAVA环境，具体地说，主要就是com.bilibili.nativelibrary.SignedQuery这个类的问题。

  可以直接JADX中复制这个类，直接拿过来用，Unidbg提供了另外一种模拟Native调用JAVA的方式——缺啥补啥，其原理是JAVA的反射。

- 主要两点改变，运行后会报错找不到缺少的那个类，补上就可以了。

  - LibBili1 不继承自AbstractJni
  - vm.setJni(this);改成 vm.setDvmClassFactory(new ProxyClassFactory());

- [原文传送](https://blog.csdn.net/qq_38851536/article/details/117923970)

### 0x06. 打开系统调用日志

- 方法一

```java
public static void main(String[] args){
  Logger.getLogger("com.github.unidbg.linux.ARM32SyscallHandler").setLevel(Level.DEBUG);
  Logger.getLogger("com.github.unidbg.unix.UnixSyscallHandler").setLevel(Level.DEBUG);
  Logger.getLogger("com.github.unidbg.AbstractEmulator").setLevel(Level.DEBUG);
  Logger.getLogger("com.github.unidbg.linux.android.dvm.DalvikVM").setLevel(Level.DEBUG);
  Logger.getLogger("com.github.unidbg.linux.android.dvm.BaseVM").setLevel(Level.DEBUG);
  Logger.getLogger("com.github.unidbg.linux.android.dvm").setLevel(Level.DEBUG);
  demo2 test = new demo2();
  System.out.println("call demo2");
  System.out.println(test.call());
}
```

- 方法二

   src/test/resources/log4j.properties中**INFO**全配置成**DEBUG**

### 0x07. 使用Unidbg打印函数参数5之后的值

[常见函数调用约定](https://bbs.pediy.com/thread-224583.htm)

假如一个函数有九个参数，根据ATPCS调用约定，参数1~参数4 分别保存到 R0~R3 寄存器中 ，剩下的参数从右往左依次入栈，被调用者实现栈平衡，返回值存放在 R0 中。后五个参数在栈中。

首先查看堆栈SP寄存器的内存：

```java
debugger break at: 0x4000b300
>>> r0=0x401af000 r1=0x401b0000 r2=0x401b1000 r3=0x5 r4=0x0 r5=0x40228000 r6=0x40218060 r7=0xbffff710 r8=0x0 sb=0x0 sl=0x0 fp=0x0 ip=0x40039b68
>>> SP=0xbffff6d8 LR=RX@0x4000a313[libSCoreTools.so]0xa313 PC=RX@0x4000b300[libSCoreTools.so]0xb300 cpsr: N=0, Z=0, C=1, V=0, T=1, mode=0b10000
>>> d0=0xffffffffffffffff(NaN) d1=0x736f6c7261437369(1.0985571808136345E248) d2=0x3530653161316361(1.711747594598551E-52) d3=0x6532326661316331(2.949539641781722E179) d4=0x2030203020302030(1.2027122125173386E-153) d5=0x2030203020302030(1.2027122125173386E-153) d6=0x2030203020302030(1.2027122125173386E-153) d7=0x2030203020302030(1.2027122125173386E-153)
>>> d8=0x0(0.0) d9=0x0(0.0) d10=0x0(0.0) d11=0x0(0.0) d12=0x0(0.0) d13=0x0(0.0) d14=0x0(0.0) d15=0x0(0.0)
```

msp查看内存信息，每`四个字节`就代表栈中的一个值，且为`小端序`， 所以参数5就是0x40228000，参数6即0x40203000，参数7是0x40218040，参数8是00 00 00 00，参数9是0x00 00 00 40。

```java
msp
>-----------------------------------------------------------------------------<
[17:05:38 660]sp=unidbg@0xbffff6d8, md5=c8023b520df3ea0aaaaa8918adcb4e9c, hex=00802240003020404080214000000000400000000100000000101b4000001b40408021400030204000f01a40a012feff00101b400030204048f7ffbf299d0040a402000000f01a4000001b40f4ffffff00a0034048a10340000000004c00214000000000000000000000000000000000
size: 112
0000: 00 80 22 40 00 30 20 40 40 80 21 40 00 00 00 00    .."@.0 @@.!@....
0010: 40 00 00 00 01 00 00 00 00 10 1B 40 00 00 1B 40    @..........@...@
0020: 40 80 21 40 00 30 20 40 00 F0 1A 40 A0 12 FE FF    @.!@.0 @...@....
0030: 00 10 1B 40 00 30 20 40 48 F7 FF BF 29 9D 00 40    ...@.0 @H...)..@
0040: A4 02 00 00 00 F0 1A 40 00 00 1B 40 F4 FF FF FF    .......@...@....
0050: 00 A0 03 40 48 A1 03 40 00 00 00 00 4C 00 21 40    ...@H..@....L.!@
0060: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00    ................
^-----------------------------------------------------------------------------^
```

### 0x08. 使用Unidbg打印jobject

[原文传送](https://blog.csdn.net/qq_38851536/article/details/118122592)

有个jobject对象，想查看它的内容，最常见的就是jbyteArray。

Frida中可以这么操作：`hexdump(ptr(Java.vm.tryGetEnv().getByteArrayElements(args[0])))`
Unidbg中当然也可以，以hookZz中为例：

```java
public void preCall(Emulator<?> emulator, HookZzArm32RegisterContext ctx, HookEntryInfo info) {
    UnidbgPointer jbytearrayptr = ctx.getPointerArg(2);
    DvmObject<?> dvmbytes = vm.getObject(jbytearrayptr.toIntPeer());
    // 取出byte
    byte[] result = (byte[]) dvmbytes.getValue();
    // 转换成String 或者按需转成其他
    System.out.println(new String(result));
};

```

### 0x09. 固定随机数与时间戳

#### lrand48

```java
HookZz hookZz = HookZz.getInstance(emulator);
Symbol lrand48 = module.findSymbolByName("lrand48");
hookZz.replace(lrand48, new ReplaceCallback() {
  @Override
  public void postCall(Emulator<?> emulator, HookContext context) {
    EditableArm32RegisterContext ctx = emulator.getContext();
    ctx.setR0(7);
  }
}, true);
```

```java
public void hookRandom(){
    emulator.attach().addBreakPoint(module.findSymbolByName("lrand48", true).getAddress(), new BreakPointCallback() {
        @Override
        public boolean onHit(Emulator<?> emulator, long address) {
            System.out.println("call lrand48");
            emulator.getUnwinder().unwind();
            emulator.attach().addBreakPoint(emulator.getContext().getLRPointer().peer, new BreakPointCallback() {
                @Override
                public boolean onHit(Emulator<?> emulator, long address) {
                    emulator.getBackend().reg_write(ArmConst.UC_ARM_REG_R0, 0x12345678);
                    return true;
                }
            });
            return true;
        }
    });
}
```

#### gettimeofday

```java
Symbol gettimeofday = module.findSymbolByName("gettimeofday");
hookZz.replace(gettimeofday, new ReplaceCallback() {
  UnidbgPointer tv_ptr = null;
  @Override
  public HookStatus onCall(Emulator<?> emulator, HookContext context, long originFunction) {
    tv_ptr = context.getPointerArg(0);
    return super.onCall(emulator, context, originFunction);
  }
  @Override
  public void postCall(Emulator<?> emulator, HookContext context) {
    EditableArm32RegisterContext ctx = emulator.getContext();
    if(tv_ptr != null){
      ByteBuffer tv = ByteBuffer.allocate(8);
      tv.order(ByteOrder.LITTLE_ENDIAN);
      tv.putInt(0,1626403551);
      tv.putInt(4, 151606);
      byte[] data = tv.array();
      tv_ptr.write(0,data,0,8);
    }
  }
}, true);
```

### 0x0a. 数据格式转换

#### bytesToHex

```java
private static final char[] HEX_ARRAY = "0123456789ABCDEF".toCharArray();

public static String bytesToHex(byte[] bytes) {
  char[] hexChars = new char[bytes.length * 2];
  for (int j = 0; j < bytes.length; j++) {
    int v = bytes[j] & 0xFF;
    hexChars[j * 2] = HEX_ARRAY[v >>> 4];
    hexChars[j * 2 + 1] = HEX_ARRAY[v & 0x0F];
  }
  return new String(hexChars);
}
```

```java
public static String bytesTohexString(byte[] bytes) {
  StringBuffer sb = new StringBuffer();
  for(int i = 0; i < bytes.length; i++) {
    String hex = Integer.toHexString(bytes[i] & 0xFF);
    if(hex.length() < 2){
      sb.append(0);
    }
    sb.append(hex);
  }
  return sb.toString();
}
```

#### hexStringToBytes

```java
public static byte[] hexStringToBytes(String hexString) {
  if (hexString.isEmpty()) {
    return null;
  }
  hexString = hexString.toLowerCase();
  final byte[] byteArray = new byte[hexString.length() >> 1];
  int index = 0;
  for (int i = 0; i < hexString.length(); i++) {
    if (index  > hexString.length() - 1) {
      return byteArray;
    }
    byte highDit = (byte) (Character.digit(hexString.charAt(index), 16) & 0xFF);
    byte lowDit = (byte) (Character.digit(hexString.charAt(index + 1), 16) & 0xFF);
    byteArray[i] = (byte) (highDit << 4 | lowDit);
    index += 2;
  }
  return byteArray;
}
```
