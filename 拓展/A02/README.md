## 1.1 IDA静态调试硬编码破解端口检测反调试

### 1.1.1 常见Frida反调试

1. 遍历连接手机所有端口发送`D-bus`消息，如果返回"REJECT"这个特征则认为存在frida-server 
2. 直接调用`openat`的`syscall`的检测在`text`节表中搜索`frida-gadget*.so / frida-agent*.so`字符串，避免了`hook`   `libc`来anti-anti的方法。 [frida-detection-demo/native-lib.cpp at master · b-...](https://github.com/b-mueller/frida-detection-demo/blob/master/AntiFrida/app/src/main/cpp/native-lib.cpp) 
3. 内存中存在`frida rpc`字符串，认为有frida-server [AntiFrida/detect.cpp at master · qtfreet00/AntiFri...](https://github.com/qtfreet00/AntiFrida/blob/master/app/src/main/cpp/detect.cpp)



### 1.1.2 某反调试代码实例

```c++
while (1) {

    /*
         * 1: Frida Server Detection.
         */
	
    for(i = 0 ; i <= 65535 ; i++) {
		// 开启一个socket
        sock = socket(AF_INET , SOCK_STREAM , 0);
        // 拿到socket的port
        sa.sin_port = htons(i);
		// 连接端口
        if (connect(sock , (struct sockaddr*)&sa , sizeof sa) != -1) {
            memset(res, 0 , 7);
 			// 发送 "\x00"、 "AUTH\r\n"给socket
            send(sock, "\x00", 1, NULL);
            send(sock, "AUTH\r\n", 6, NULL);

            usleep(100); // Give it some time to answer
			// 等待返回内容
            if ((ret = recv(sock, res, 6, MSG_DONTWAIT)) != -1) {
                // 检测原理 res跟REJECT对比，如果相同就代表检测到了frida
                // 可以hook strcmp函数，让他们返回永远不为0，也可也过反调试。
                // 需要动静态分析
                if (strcmp(res, "REJECT") == 0) {
                    __android_log_print(ANDROID_LOG_VERBOSE, APPNAME,  "FRIDA DETECTED [1] - frida server running on port %d!", i);
                }
            }
        }

        close(sock);
    }
    
extern "C"
JNIEXPORT void JNICALL Java_sg_vantagepoint_antifrida_MainActivity_init(JNIEnv *env, jobject thisObj) {
    pthread_t t;
    // 开启一个线程进入detect_frida_loop 检测frida loop
    // 也可以hook pthread_create。让他不创建线程。
    pthread_create(&t, NULL, detect_frida_loop, (void *)NULL);
}
```

TracePid fgets 反调试

![](/A02/pic/01.a.png)



### 1.1.3 硬编码修改反调试

idaNew打开so文件，找到判断的地方，因为是和'reject'做对比检测，找到reject。

![](/A02/pic/01.b.png)

选中, 进入hex view，把硬编码修改, 右键选择edit, 43和54任意修改，reject变成别的字就可以，可以使用[hex to ascii](https://www.rapidtables.com/convert/number/hex-to-ascii.html)查看，右键选择 apply changes保存。

全局修改：edit-> patch program -> apply patches to input file 保存。(同时保存备份文件，用md5对修改前、修改后两个so文件做对比，看是否修改保存成功。)

![](/A02/pic/01.c.png)

objection hook住app，找到native-lib.so(代码所在的so库，不一定是native-lib.so)所在的地址，使用adb shell 进入地址，找到对应so文件。把修改后的so文件push到原native-lib.so所在的地址，替换之前的so文件。

直接保存so文件，运行app，修改后的代码是不生效的，需要找到要修改的so库的所在地址，删除原文件，修改后的so文件重新push进去。

**总结**：需要动静态分析。可以把它用来检测、匹配的特征 全部硬编码替换掉，只要程序正常运行。

