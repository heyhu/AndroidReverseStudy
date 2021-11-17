<!-- @import "[TOC]" {cmd="toc" depthFrom=1 depthTo=6 orderedList=false} -->

<!-- code_chunk_output -->

- [基础配置](#基础配置)
  - [初始化](#初始化)
  - [执行目标函数--参数构造](#执行目标函数--参数构造)
    - [字节数组以及布尔值](#字节数组以及布尔值)
    - [context以及字符串类型](#context以及字符串类型)
    - [treemap](#treemap)
    - [对象数组](#对象数组)

<!-- /code_chunk_output -->

## 基础配置

### 初始化

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

### 执行目标函数--参数构造

- 字节数组需要裹上unidbg的包装类，并加到本地变量里，两件事缺一不可。
- 除了基本类型，比如int，long等，其他的对象类型一律要手动 `addLocalObject`。
- vm.getObject(number.intValue()).getValue().toString() 解释：在DalvikVM中有Map存储了jni交互的对象，key是该对象的hash，value是该对象。这个intValue就是这个对象的hash，通过vm.getObject方法，来取出这个hash对应的Object。

#### 字节数组以及布尔值

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

#### context以及字符串类型

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

#### treemap

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
- 代码中补齐了treeMap的继承关系：map→AbstractMap→TreeMap，这么做是必要的，否则在有些情况下会报错。

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

#### 对象数组

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
