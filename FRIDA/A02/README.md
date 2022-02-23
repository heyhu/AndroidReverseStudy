### Objection
```
$ 用于配合frida移动探索、默认attach上，如果app没有启动 就默认spawn模式启动
$ 只要objection hook到，frida就可以，底层是调用frida的接口
$ frida-server使用标准端口: objection就可以objection -g com.example.demoso1 explore启动，不用加host、端口
$ 使用网络连接objection：objection -N -h 192.168.199.237 -p 8888 -g com.android.settings explore
$ 启动方式：先尝试直接attach启动，失败后在spwan以新进程的方式来启动
$ hook的时机：有的方法是oncreate方法app初始化执行的，所以用spwan的方式，先敲击命令，objection 先尝试直接attach，失败后在spwan以新进程的方式来启动。在点开软件，就可以hook到初始化的方法
```

1. plugin load
    1. objection -g com.app.name explore -P ~/.objection/plugins
    2. plugin load /Users/xxx/code/Wallbreaker
    
2. Options：

    1. 命令行模式:
        1. -N 使用网络代替usb
        2. -h 指定ip
        3. -s 指定设备号
        4. -d debug模式
        5. -g 安卓没有root权限，可以把app进行重新打包，即使安卓没有root权限，也可以连接frida
        
    2. 交互模式:
        1. env: app常用目录
        2. jobs list: 查看目前当前作业系统，例如列出之前hook的类和方法  
        3. jobs kill 编号：杀死job，结束hook操作等    
        4. memory list modules: 进入当前程序加载的so，如果系统自己加载的也会显示出来  
            > so：Service Object(SO)是容器管理的一组对象，完成系统中的业务功能。Service Object完成的工作可能是一系列对数据库操作，文件系统，内存操作的集合，so库   
        5. memory list exports libsssl.so --json json.txt：查看so库的导出函数并导出到txt文件中  
        6. memory search --string --offsets-only  2d 2d 73 74 72 69 6e 67(dex头)：内存搜索
        7. android hooking list classes：列出内存中所有的类    
        8. android hooking search classes 类名/关键字: 列出包含指定关键字的类名，可以直接搜索想hook的类名     
        9. android hooking list class_methods 类名：列出此类名的所有方法名    
        10. android hooking search methods 方法名/关键字 : 从所有的类中搜寻包含此关键字的方法名     
        11. android hooking watch class 类名 --dump-args --dump-return --dump-backtrace: hook住类里面的所有函数，app刷新就会打印
            > (agent) [yvoqld7tvmm] Called android.util.Base64.encodeToString([B, int)，Called就是被调用  
        12. android hooking watch class_method android.util.Base64.encodeToString --dump-args --dump-backtrace --dump-return: hook住类的指定方法，直接打印出调用栈、返回值、参数, 自动hook所有的重载
        13. android heap search instances 类名  --fresh: 搜索此类的实例  
        14. android heap execute  实例地址  方法名（不加括号）: 直接调用类实例的方法   
        15. android hooking list activities：列出所有activity   
        16. android intent launch_activity com.bk.base.auth.AuthCenterActivity：直接进入某个activity下，可以绕过密码，进入activity   
        17. android hooking list services: 列出所有组件
        18. android intent launch_service com.igexin.sdk.PushServic: 可以开启组件，比如设置里面的蓝牙   
        19. android hooking generate simple 类名：直接自动生成hook的代码，包含类里面函数hook的方法(不全)
        20. android hooking generate class：自动生成类的hook代码。自动生成的一般不用   
        21. objection -g 包名 explore --startup-command "命令": objection在启动时执行命令
            > 用于hook例如oncreate这种一瞬即逝的方法 
        22. hook一个包下面所有的类
            1. 把hook类的命令全部写到一个txt文件下，然后执行objection -g package_name  explore  -c n.txt。（批量执行命令） 
            2. 先把函数都搜索出来保存到txt文件中，在每行前面加入android hook watch class/method，然后执行objection -g package_name  explore  -c n.txt。  
        23. hook 构造函数，在命令行中需要转义
        ![](pic/01.a.png)
