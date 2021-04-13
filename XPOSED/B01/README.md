### Xposed Api

[Xposed 源码地址](https://api.xposed.info/reference/packages.html)

[以下hook目标app源码](https://github.com/heyhu/xposeProject)


- [Hook 案例toastMessage方法](https://github.com/heyhu/xposeProject/blob/main/app/src/main/java/com/example/xposed1/HookMessage.java)  
    内涵技术栈：  
        1. hook指定目标函数加载 前/后以及执行顺序  
        2. 参数/返回值打印  
        3. 参数/返回值替换  
        4. 调用栈打印    
        5. 通过param.thisObject 来获取实例          
        6. 获取类的两种方法: 

           1. XposedHelpers.findClass
           2. loadPackageParam.classLoader.loadClass(className)   
  
- [Hook XposedBridge hookAllMethods方法](https://github.com/heyhu/xposeProject/blob/main/app/src/main/java/com/example/xposed1/practice/HookAllMethod.java)   
    内涵技术栈：  
        1. 使用hookAllMethods Hook住指定方法   
        2. 通过hook类的实例方法获得类的实例   
        3. 主动调用动态方法 

- [Hook 带壳App](https://github.com/heyhu/xposeProject/blob/main/app/src/main/java/com/example/xposed1/practice/HookApplication.java) 
    xpose 不能直接hook到带壳app下的类。    
    内涵技术栈：  
        1. Hook壳原理  
        2. Hook到目标类 
 
- [Hook 构造方法](https://github.com/heyhu/xposeProject/blob/main/app/src/main/java/com/example/xposed1/practice/HookApplication.java)  
    内涵技术栈：  
        1. Hook构造方法并挂钩  
        2. 获取实例 
        3. 返回给定对象实例中对象字段的值  

- [复现XposedHelpers中某些方法](https://github.com/heyhu/xposeProject/blob/main/app/src/main/java/com/example/xposed1/practice/HookHelpers.java)   
    内涵技术栈：  
        1. callMethod   
        2. getObjectField   
        3. getStaticIntField    

- [复现loadPackageParam的属性](https://github.com/heyhu/xposeProject/blob/main/app/src/main/java/com/example/xposed1/practice/HookLp.java)    
    内涵技术栈：  
        1. 过滤系统版本   
        2. 打印进程名    
        3. 打印包名    
        4. 获取ClassLoader,用于查找包名等。   

- [Hook 多dex apk](https://github.com/heyhu/xposeProject/blob/main/app/src/main/java/com/example/xposed1/practice/HookMoreDex.java)      
   内涵技术栈：   
       1. hook 多dex问题   
       2. findAndHookMethod可以指定参数类型指定hook特定的重载方法    

- [主动调用内存爆破](https://github.com/heyhu/xposeProject/blob/main/app/src/main/java/com/example/xposed1/practice/HookVerifier.java)  
  内涵技术栈：    
      1. XposedHelpers.callStaticMethod 主动调用    
      2. 反射主动调用   
      3. context构造  
   
- [NanoHttpd 算法主动暴露](https://github.com/heyhu/xposeProject/blob/main/app/src/main/java/com/example/xposed1/practice/HookVerifier.java)  
   内涵技术栈：   
      1. 使用http服务暴露算法   
      
- 获取实例的三种方式
   1. Hook一个实例方法来获取实例
   2. 获取构造方法来获取实例
      > 还可以findConstructorExact获得构造函数，然后.newInstance() 获取实例
           mNotificationLightConstructor = XposedHelpers.findConstructorExact(CLASS_NOTIFICATION_RECORD+".Light", mContext.getClassLoader(),int.class, int.class, int.class);
           return mNotificationLightConstructor.newInstance(color, onMs, offMs);
   3. xposed.newInstance(clazz) 获取实例

- 为什么HookMessage类在 com.example.xposed1包下，查看包下面所有的类却没有HookMessage？
    > 因为 HookMessage implements IXposedHookLoadPackage， 此类实现了接口，所以搜索不到，他不属于原本的包。

- 查找 HookMessage 为什么会出现两个？
  ![](pic/01.a.png)     
  > HookMessage下的XC_MethodHook 就是 HookMessage$1 ,他是匿名的内部类，它有 beforeHookedMethod和 afterHookedMethod 两个方法，如果想hook XC_MethodHook 函数，那么就需要hook  HookMessage$1，
    假如一个类有多个hook 方法比如多个XC_MethodHook ，那么就具有多个HookMessage内部类，比如HookMessage$1、HookMessage$2等。

- 如何定位className、XC_MethodHook 是属于哪个进程的内部类。  
  > if (loadPackageParam.packageName.equals("com.example.xposed1")) {    
        查看进程名等于谁，就可以找到。hook 的包名为com.example.xposed1

- 构造函数解释    
  ![](pic/02.a.png)         
   java初始化设置属性的时候，是用构造函数来设置的。new DemoHelper("123") 给类传参数。
   XposedHelpers.findConstructorExact(CLASS_SHOW_STACK_ACTION_BUTTON_EVENT, cl, boolean.class).newInstance(true));