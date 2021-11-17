## 1.1 基础配置

### 1.1.1 前置

```java
package com.lession1;

// 导入通用且标准的类库
import com.github.unidbg.linux.android.dvm.AbstractJni;
import com.github.unidbg.AndroidEmulator;
import com.github.unidbg.Module;
import com.github.unidbg.linux.android.AndroidEmulatorBuilder;
import com.github.unidbg.linux.android.AndroidResolver;
import com.github.unidbg.linux.android.dvm.*;
import com.github.unidbg.linux.android.dvm.array.ByteArray;
import com.github.unidbg.memory.Memory;

import java.io.File;

// 继承AbstractJni类
public class oasis extends AbstractJni{
    private final AndroidEmulator emulator;
    private final VM vm;
    private final Module module;

    oasis() {
        // 创建模拟器实例,进程名建议依照实际进程名填写，可以规避针对进程名的校验
        emulator = AndroidEmulatorBuilder.for32Bit().setProcessName("com.sina.oasis").build();
        // 获取模拟器的内存操作接口
        final Memory memory = emulator.getMemory();
        // 设置系统类库解析
        memory.setLibraryResolver(new AndroidResolver(23));
        // 创建Android虚拟机,传入APK，Unidbg可以替我们做部分签名校验的工作
        // 如果创建Android虚拟机时，选择不传入APK，填入null，那么样本在JNI OnLoad中所做的签名校验，就需要手动补环境校验
        vm = emulator.createDalvikVM(new File("unidbg-android/src/test/java/com/lession1/lvzhou.apk"));
        // 加载目标SO, 加载so到虚拟内存
        DalvikModule dm = vm.loadLibrary(new File("unidbg-android/src/test/java/com/lession1/liboasiscore.so"), true); 
        //获取本SO模块的句柄,后续需要用它
        module = dm.getModule();
        // 设置JNI
        vm.setJni(this); 
        //打印日志
        vm.setVerbose(true);
        // 调用JNI OnLoad,可以看到JNI中做的事情，比如动态注册以及签名校验等。如果没有使用可以注释掉。
        dm.callJNI_OnLoad(emulator); 
    };

    public static void main(String[] args) {
        oasis test = new oasis();
    }
}
```



## 1.2 执行目标函数--参数构造

- 字节数组需要裹上unidbg的包装类，并加到本地变量里，两件事缺一不可。
- 除了基本类型，比如int，long等，其他的对象类型一律要手动 addLocalObject。
- vm.getObject(number.intValue()).getValue().toString() 解释：在DalvikVM中有Map存储了jni交互的对象，key是该对象的hash，value是该对象。你这个intValue就是这个对象的hash，通过vm.getObject方法，来取出这个hash对应的Object。

### 1.2.1 字节数组以及布尔值

```java
    public String getS(){
        // args list
        List<Object> list = new ArrayList<>(10);
        // arg1 env
        list.add(vm.getJNIEnv());
        // arg2 jobject/jclazz 一般用不到，直接填0
        list.add(0);
        // arg3 bytes
        String input = "aid=01A-khBWIm48A079Pz_DMW6PyZR8" +
                "uyTumcCNm4e8awxyC2ANU.&cfrom=28B529501" +
                "0&cuid=5999578300&noncestr=46274W9279Hr1" +
                "X49A5X058z7ZVz024&platform=ANDROID&timestamp" +
                "=1621437643609&ua=Xiaomi-MIX2S__oasis__3.5.8_" +
                "_Android__Android10&version=3.5.8&vid=10190135" +
                "94003&wm=20004_90024";
        byte[] inputByte = input.getBytes(StandardCharsets.UTF_8);
        ByteArray inputByteArray = new ByteArray(vm,inputByte);
        list.add(vm.addLocalObject(inputByteArray));
        // arg4 ,boolean false 填入0
        list.add(0);
        // 参数准备完成
        // call function
        Number number = module.callFunction(emulator, 0xC365, list.toArray())[0];
        String result = vm.getObject(number.intValue()).getValue().toString();
        return result;
    }
```

### 1.2.2 context以及字符串类型

```java
    public String calculateS(){
        List<Object> list = new ArrayList<>(10);
        // 第一个参数是env
        list.add(vm.getJNIEnv()); 
        // 第二个参数，实例方法是jobject，静态方法是jclazz，直接填0，一般用不到。
        list.add(0); 
        // 通过虚拟机创建一个context实例，其类型为DvmObject
        DvmObject<?> context = vm.resolveClass("android/content/Context").newObject(null);
        list.add(vm.addLocalObject(context));
        list.add(vm.addLocalObject(new StringObject(vm, "12345")));
        list.add(vm.addLocalObject(new StringObject(vm, "r0ysue")));
		    // 因为代码是thumb模式，别忘了+1
        Number number = module.callFunction(emulator, 0x1E7C + 1, list.toArray())[0];
        String result = vm.getObject(number.intValue()).getValue().toString();
        return result;
    };
```

### 1.2.3 treemap

```java
    public void s(){
        TreeMap<String, String> keymap = new TreeMap<String, String>();
        keymap.put("appkey", "1d8b6e7d45233436");
        keymap.put("autoplay_card", "11");
        keymap.put("banner_hash", "10687342131252771522");
        keymap.put("build", "6180500");
        keymap.put("c_locale", "zh_CN");
        keymap.put("channel", "shenma117");
        keymap.put("column", "2");
        keymap.put("device_name", "MIX2S");
        keymap.put("device_type", "0");
        keymap.put("flush", "6");
        keymap.put("ts", "1612693177")
    };

```

- 可以照着StringObject重新写一个，也可以不这么麻烦，直接返回一个“空壳”，Native中对treemap做了操作再补对应的方法，这样比较经济实惠。
- 代码中补齐了treeMap的继承关系：map→AbstractMap→TreeMap，这么做是必要的，否则在有些情况下会报错，具体讨论此处略过。

```java
 public void s(){
        List<Object> list = new ArrayList<>(10);
        list.add(vm.getJNIEnv()); // 第一个参数是env
        list.add(0); // 第二个参数，实例方法是jobject，静态方法是jclazz，直接填0，一般用不到。

        TreeMap<String, String> keymap = new TreeMap<String, String>();
        keymap.put("ad_extra", "E1133C23F36571A3F1FDE6B325B17419AAD45287455E5292A19CF51300EAF0F2664C808E2C407FBD9E50BD48F8ED17334F4E2D3A07153630BF62F10DC5E53C42E32274C6076A5593C23EE6587F453F57B8457654CB3DCE90FAE943E2AF5FFAE78E574D02B8BBDFE640AE98B8F0247EC0970D2FD46D84B958E877628A8E90F7181CC16DD22A41AE9E1C2B9CB993F33B65E0B287312E8351ADC4A9515123966ACF8031FF4440EC4C472C78C8B0C6C8D5EA9AB9E579966AD4B9D23F65C40661A73958130E4D71F564B27C4533C14335EA64DD6E28C29CD92D5A8037DCD04C8CCEAEBECCE10EAAE0FAC91C788ECD424D8473CAA67D424450431467491B34A1450A781F341ABB8073C68DBCCC9863F829457C74DBD89C7A867C8B619EBB21F313D3021007D23D3776DA083A7E09CBA5A9875944C745BB691971BFE943BD468138BD727BF861869A68EA274719D66276BD2C3BB57867F45B11D6B1A778E7051B317967F8A5EAF132607242B12C9020328C80A1BBBF28E2E228C8C7CDACD1F6CC7500A08BA24C4B9E4BC9B69E039216AA8B0566B0C50A07F65255CE38F92124CB91D1C1C39A3C5F7D50E57DCD25C6684A57E1F56489AE39BDBC5CFE13C540CA025C42A3F0F3DA9882F2A1D0B5B1B36F020935FD64D58A47EF83213949130B956F12DB92B0546DADC1B605D9A3ED242C8D7EF02433A6C8E3C402C669447A7F151866E66383172A8A846CE49ACE61AD00C1E42223");
        keymap.put("appkey", "1d8b6e7d45233436");
        keymap.put("autoplay_card", "11");
        keymap.put("banner_hash", "10687342131252771522");
        keymap.put("build", "6180500");
        keymap.put("c_locale", "zh_CN");
        keymap.put("channel", "shenma117");
        keymap.put("column", "2");
        keymap.put("device_name", "MIX2S");
        keymap.put("device_type", "0");
        keymap.put("flush", "6");
        keymap.put("ts", "1612693177");
   
        DvmClass Map = vm.resolveClass("java/util/Map");
        DvmClass AbstractMap = vm.resolveClass("java/util/AbstractMap",Map);
        DvmObject<?> input_map = vm.resolveClass("java/util/TreeMap", AbstractMap).newObject(keymap);
        list.add(vm.addLocalObject(input_map));
        Number number = module.callFunction(emulator, 0x1c97, list.toArray())[0];
        DvmObject result = vm.getObject(number.intValue());
    };
```

### 1.2.4 对象数组

```java
public String main203(){
        List<Object> list = new ArrayList<>(10);
        list.add(vm.getJNIEnv());
        list.add(0);
        list.add(203);
        StringObject input2_1 = new StringObject(vm, "9b69f861-e054-4bc4-9daf-d36ae205ed3e");
        ByteArray input2_2 = new ByteArray(vm, "GET /aggroup/homepage/display __r0ysue".getBytes(StandardCharsets.UTF_8));
        DvmInteger input2_3 = DvmInteger.valueOf(vm, 2);
        vm.addLocalObject(input2_1);
        vm.addLocalObject(input2_2);
        vm.addLocalObject(input2_3);
        // 完整的参数2
        list.add(vm.addLocalObject(new ArrayObject(input2_1, input2_2, input2_3)));
        Number number = module.callFunction(emulator, 0x5a38d, list.toArray())[0];
        return vm.getObject(number.intValue()).getValue().toString();
    };

指定一个空对象的对象数组：
public void main111(){
        List<Object> list = new ArrayList<>(10);
        list.add(vm.getJNIEnv());
        list.add(0);
        list.add(111);
        DvmObject<?> obj = vm.resolveClass("java/lang/object").newObject(null);
        vm.addLocalObject(obj);
        ArrayObject myobject = new ArrayObject(obj);
        vm.addLocalObject(myobject);
        list.add(vm.addLocalObject(myobject));
        module.callFunction(emulator, 0x5a38d, list.toArray());
    };
```



## 1.3 奇技淫巧

### 1.3.1 PatchCode

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

![01.a](pic/01.a.png)

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

  

### 1.3.2 Hook

#### 1.3.2.1 HookZz--参数位置

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

#### 1.3.2.2 HookZ--寄存器

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

#### 1.3.2.3 主动调用

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

#### 1.3.2.4 Inline hook

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

#### 1.3.2.5 call 参数传递，Unidbg封装的API

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

#### 1.3.2.6 Unicorn hook

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

### 1.3.3 Trace

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

### 1.3.4 console debugger

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

### 1.3.5 traceRead

- 对内存读写/访问的trace

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

### 1.3.6 traceWrite

- 对结果的trace，此时程序最后结果为：

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

### 1.3.7 补一个完整类

- 涉及的环境缺失是JAVA环境，具体地说，主要就是com.bilibili.nativelibrary.SignedQuery这个类的问题。

  可以直接JADX中复制这个类，直接拿过来用，Unidbg提供了另外一种模拟Native调用JAVA的方式——缺啥补啥，其原理是JAVA的反射。

- 主要两点改变，运行后会报错找不到缺少的那个类，补上就可以了。

  - LibBili1 不继承自AbstractJni
  - vm.setJni(this);改成 vm.setDvmClassFactory(new ProxyClassFactory());

### 1.3.8 打开系统调用日志

在main下面写:

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

### 1.3.9 Unidbg VirtualModule

如果SO的依赖项中有Unidbg不支持的系统SO怎么办 ?

```java
vm = emulator.createDalvikVM(new File("unidbg-android/src/test/java/com/lession8/demo2.apk"));
// 注册libandroid.so虚拟模块
new AndroidModule(emulator, vm).register(memory);

DalvikModule dm = vm.loadLibrary(new File("unidbg-android/src/test/java/com/lession8/readassets.so"), true);
module = dm.getModule();
```

需要注意，一定要在样本SO加载前加载它，道理也很简单，系统SO肯定比用户SO加载的早。VirtualModule并不是一种真正意义上的加载SO，它本质上也是Hook，只不过实现了SO中少数几个函数罢了。

### 1.3.10 常见函数调用约定

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

### 1.3.11 Unidbg 报错时如何分析

比如报错memory failed 

- src/test/resources/log4j.properties中**INFO**全配置成**DEBUG**

- 输入bt查看调用栈，ida中跳到对应函数地址`0x00abf`查看结果。

  ```java
  [15:22:35 859] DEBUG [net.fornwall.jelf.ArmExIdx] (ArmExIdx:274) - finish
  [0x40000000][0x40000abf][   libwtf.so][0x00abf] Java_com_sichuanol_cbgc_util_SignManager_getSign + 0x18e
  [15:22:35 860] DEBUG [net.fornwall.jelf.ArmExIdx] (ArmExIdx:153) - 
  >-----------------------------------------------------------------------------<
  ```

  

## 1.4 补环境

### 1.4.1 构造最基本Context实例

```java
@Override
public DvmObject<?> callStaticObjectMethodV(BaseVM vm, DvmClass dvmClass, String signature, VaList vaList) {
  switch (signature) {
    case "com/izuiyou/common/base/BaseApplication->getAppContext()Landroid/content/Context;":
      // 链接一个android.content.Context的类并通过构造方法创建一个实例
      return vm.resolveClass("android/content/Context").newObject(null);
  }
  return super.callStaticObjectMethodV(vm, dvmClass, signature, vaList);
}
```

### 1.4.2 Context实例的getClass方法

```java
@Override
public DvmObject<?> callObjectMethodV(BaseVM vm, DvmObject<?> dvmObject, String signature, VaList vaList) {
  switch (signature) {
    case "android/content/Context->getClass()Ljava/lang/Class;":{
      // 此时的dvmObject就是Context实例,.getObjectType方法获取类型
      // Context为主体所以用dvmObject获取
      return dvmObject.getObjectType();
    }
    case "java/util/UUID->toString()Ljava/lang/String;":{
      // 此时的dvmObject就是包裹Java.util.UUID实例
       String uuid = dvmObject.getValue().toString();
       return new StringObject(vm, uuid);
    }
    case "java/lang/Class->getSimpleName()Ljava/lang/String;":{
      // 使用Wallbreaker查看com.izuiyou.common.base.BaseApplication的getClass的getSimpleName
      return new StringObject(vm, "AppController");
    }
    case "android/content/Context->getFilesDir()Ljava/io/File;":
    case "java/lang/String->getAbsolutePath()Ljava/lang/String;":
      {
        return new StringObject(vm, "/data/user/0/cn.xiaochuankeji.tieba/files");
      }
  }
  return super.callObjectMethodV(vm, dvmObject, signature, vaList);
};
```

### 1.4.3 返回PID

```java
@Override
public int callStaticIntMethodV(BaseVM vm, DvmClass dvmClass, String signature, VaList vaList) {
  switch (signature){
    case "android/os/Process->myPid()I":{
      return emulator.getPid();
    }
  }
  throw new UnsupportedOperationException(signature);
}
```

### 1.4.4 使用java的api返回一个对象

- Java.util.UUID为Java的工具类，可以直接引用并构造。
- 为什么不使用context那样构造，因为那个类不是Java的工具类，是和app有关，不可用直接引用。

```java
@Override
public DvmObject<?> callStaticObjectMethodV(BaseVM vm, DvmClass dvmClass, String signature, VaList vaList) {
  switch (signature) {
    case "java/util/UUID->randomUUID()Ljava/util/UUID;":{
      return dvmClass.newObject(UUID.randomUUID());
    }
  }
  return super.callStaticObjectMethodV(vm, dvmClass, signature, vaList);
};
```

### 1.4.5 VarArg的使用

![](pic/01.b.png)

- 主体: dvmObject.getValue是获取treeMap的对象，参数: key的对象用VarArg获取(从app中)不是自己构造。

```java
    @Override
    public DvmObject<?> callObjectMethod(BaseVM vm, DvmObject<?> dvmObject, String signature, VarArg varArg) 		{
        switch (signature) {
            case "java/util/Map->get(Ljava/lang/Object;)Ljava/lang/Object;":
                StringObject keyobject = varArg.getObjectArg(0);
                String key = keyobject.getValue();
                // getValue得到的就是本身对象，可以各种使用对应的api
                TreeMap<String, String> treeMap = (TreeMap<String, String>)dvmObject.getValue();
                String value = treeMap.get(key);
                return new StringObject(vm, value);
        }
        return super.callObjectMethod(vm, dvmObject, signature, varArg);
		}
```

```java
    @Override
    public DvmObject<?> callStaticObjectMethod(BaseVM vm, DvmClass dvmClass, String signature, VarArg varArg) {
        switch (signature){
            // SignedQuery的r方法自己定义
            case "com/bilibili/nativelibrary/SignedQuery->r(Ljava/util/Map;)Ljava/lang/String;":{
                DvmObject<?> mapObject = varArg.getObjectArg(0);
                // getValue 获取对象本身
                TreeMap<String, String> mymap = (TreeMap<String, String>) mapObject.getValue();
                String result = utils.r(mymap);
                return new StringObject(vm, result);
            }
        }
        return super.callStaticObjectMethod(vm, dvmClass, signature, varArg);
    }
```

### 1.4.6 VaList

```java
    @Override
    public DvmObject<?> callObjectMethodV(BaseVM vm, DvmObject<?> dvmObject, String signature, VaList vaList) {
        switch (signature) {
            case "android/content/Context>getSharedPreferences(Ljava/lang/String;I)Landroid/content/SharedPreferences;":
                return vm.resolveClass("android/content/SharedPreferences").newObject(vaList.getObject(0));
        }
        return super.callObjectMethodV(vm, dvmObject, signature, vaList);
    }
```

### 1.4.7 文件访问

- 第一种方式：

  文件访问的情况各种各样，比如从app的某个xml文件中读取key，读取某个资源文件的图片做运算，读取proc/self 目录下的文件反调试等等。当样本做文件访问时，Unidbg重定向到本机的某个位置，进入 src/main/java/com/github/unidbg/file/BaseFileSystem.java，打印一下路径

```java
[15:14:10 318]  INFO [com.github.unidbg.linux.ARM32SyscallHandler] (ARM32SyscallHandler:1890) - openat dirfd=-100, pathname=/data/app/com.sankuai.meituan-TEfTAIBttUmUzuVbwRK1DQ==/base.apk, oflags=0x20000, mode=0
```

接下来我们按照要求，在data目录下新建对应文件夹`/data/app/com.sankuai.meituan-TEfTAIBttUmUzuVbwRK1DQ==`，并把我们的apk复制进去，改名成base.apk，就可以了。

- 第二种方式：

  1. public class NBridge extends AbstractJni implements IOResolver 

  2. emulator.getSyscallHandler().addIOResolver(this)

  3. 
  ```java
     @Override
         public FileResult resolve(Emulator emulator, String pathname, int oflags) {
             if (("proc/"+emulator.getPid()+"/cmdline").equals(pathname)) {
                 // 填入想要重定位的文件
                 return FileResult.success(new SimpleFileIO(oflags, newFile("D:\\unidbg-teach\\unidbg-android\\src\\test\\java\\com\\lession1\\cmdline"), pathname));
             }
             return null;
         }
  ```

  或者：

  ```java
	  @Override
	  public FileResult resolve(Emulator emulator, String pathname, int oflags) {
	    if (("proc/"+emulator.getPid()+"/cmdline").equals(pathname)) {
	      return FileResult.success(new ByteArrayFileIO(oflags, pathname, "ctrip.android.view".getBytes()));
	    }
	    return null;
	  }
  ```
