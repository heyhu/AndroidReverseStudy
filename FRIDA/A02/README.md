### 介绍
```
$ 用于配合frida移动探索、默认attach上，如果app没有启动 就默认spawn模式启动。
$ 只要objection hook到，frida就可以，底层是调用frida的接口。
$ frida-server使用标准端口: objection就可以objection -g com.example.demoso1 explore启动，不用加host、端口。
$ 使用网络连接objection：objection -N -h 192.168.199.237 -p 8888 -g com.android.settings explore
$ 启动方式：先尝试直接attach启动，失败后在spwan以新进程的方式来启动。
$ hook的时机：有的方法是oncreate方法app初始化执行的，所以用spwan的方式，先敲击命令，objection 先尝试直接attach，失败后在spwan以新进程的方式来启动。在点开软件，就可以hook到初始化的方法。
```