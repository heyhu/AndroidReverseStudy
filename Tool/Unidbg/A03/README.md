<!-- @import "[TOC]" {cmd="toc" depthFrom=1 depthTo=6 orderedList=false} -->
<!-- code_chunk_output -->

* [补环境](#奇技淫巧)
  * [0x01. JNI以及相关API](#0x01-JNI以及相关API)
    - [构造最基本Context实例](#构造最基本Context实例)
    - [Context实例的getClass方法](#Context实例的getClass方法)
    - [返回PID](#返回PID)
    - [使用java的api返回一个对象](#使用java的api返回一个对象)
    - [VarArg的使用](#VarArg的使用)
    - [VaList的使用](#VaList的使用)
  * [0x02. 文件访问](#0x02-文件访问)
    * [实体路径](#实体路径)
    * [代码方式](#代码方式)

<!-- /code_chunk_output -->



## 补环境

### 0x01. JNI以及相关API

#### 构造最基本Context实例

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

#### Context实例的getClass方法

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

#### 返回PID

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

#### 使用java的api返回一个对象

- Java.util.UUID为Java的工具类，可以直接引用并构造。
- 为什么不使用context那样构造，因为context个类不是Java的工具类，是和app有关，不可用直接引用。

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

#### VarArg的使用

![](pic/01.png)

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

#### VaList的使用

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

### 0x02. 文件访问

#### 实体路径

文件访问的情况各种各样，比如从app的某个xml文件中读取key，读取某个资源文件的图片做运算，读取proc/self 目录下的文件反调试等等。当样本做文件访问时，Unidbg重定向到本机的某个位置，进入 src/main/java/com/github/unidbg/file/BaseFileSystem.java，打印一下路径

```java
[15:14:10 318]  INFO [com.github.unidbg.linux.ARM32SyscallHandler] (ARM32SyscallHandler:1890) - openat dirfd=-100, pathname=/data/app/com.sankuai.meituan-TEfTAIBttUmUzuVbwRK1DQ==/base.apk, oflags=0x20000, mode=0
```

接下来我们按照要求，在data目录下新建对应文件夹`/data/app/com.sankuai.meituan-TEfTAIBttUmUzuVbwRK1DQ==`，并把我们的apk复制进去，改名成base.apk，就可以了。

#### 代码方式

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



