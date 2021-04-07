### Tips

* 到Activity下，记得看oncreate

* getprop ro.product.cpu.abi 查看系统CPU处理器

* whoami 查看当前用户

* 看雪xctf题可以看别人怎么做，或者在星球直接搜

* adb shell input text  "点击输入密码"

* adb logcat | grep run  查看日志

* Termux  安卓使用linux命令

* du -h *  查看此目录所有文件大小

* file * 查看文件类型

* jnettop 监控流量的使用

* tree -NCfhl | grep -i jni.h 查找***文件在哪

* frida 代码提示：npm install --save @types/frida-gum

* jadx -e xxx.apk  导出一个gradle项目

* 签名头部都是3082

* siege
    > 压力测试  siege -c10 -r10 "http://192.168.2.6:8899/encrypt POST zsx.json " 

* objection 批量hook
    > 先search所有的类保存到txt文件，在每行的行首加上android hook watch class ： sed -i -e 's/^/prefix/' file.txt，使用objection进行批量hook。

* uname -a 显示系统信息
    >  Linux localhost 3.4.0-g0e4eb55 #1 SMP PREEMPT Wed Jun 8 18:45:24 UTC 2016 armv7l

* nps
   > 使用nps内网渗透到公网，frida_server映射到公网，Xposed配合Nanohttpd映射到公网

* grep -ril "MainActivity" .  
    > 查看此目录脱下来的dex哪个包含指定字符串,可以查看脱下来的壳哪个是正确的。

* md5sum *
    > 对内容生成md5而不是名字，用于比对文件

* grep -o 60874635828   2020-11-16.json | wc -l
    > 输出文件内包含符合RE字符串的数量

* objection 加构造函数
    >  return clazz["class"].getDeclaredMethods().concat(clazz.class.getDeclaredConstructors()).map(function (method) {return method.toGenericString();})

* curl 
    >  curl -s -X POST "http://192.168.2.6:8899/encrypt"  -H "Content-Type: application/x-www-form-urlencoded" -d "r0ysue"

* 手机端口映射
    > adb forward tcp:8889 tcp:8889 手机8889映射电脑8889

* Wallbreaker
    > 搜索实例的时候不一定用类名，可以使用objection 搜索出的实例地址。

* objection 加载插件
    > 1) objection -g com.app.name explore -P ~/.objection/plugins
    > 2) plugin load /Users/zhaoshouxin/code/Wallbreaker

### Tool
```
$ jnitrace：jni
$ frida-trace：libc
$ Wallbreaker：内存漫游
$ ZenTracer：批量hook  
$ DEXDump/FART/Youpk：脱壳
```