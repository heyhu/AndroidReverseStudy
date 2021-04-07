### 介绍
```
$ 用于配合frida移动探索、默认attach上，如果app没有启动 就默认spawn模式启动。
只要objection hook到，frida就可以，底层是调用frida的接口。
frida-server使用标准端口，objection就可以objection  -g com.example.demoso1  explore启动，不用加host、端口。
使用objection操作手机：objection -N -h 192.168.199.237 -p 8888 -g com.android.settings explore
```