### Hook java

1. 输出bytes数组
   ![](pic/01.a.png)
   ```
    ByteString.of是用来把byte[]数组转成hex字符串的函数, Android系统自带ByteString类
    var ByteString = Java.use("com.android.okhttp.okio.ByteString");
    var j = Java.use("c.business.comm.j");
    j.x.implementation = function() {
        var result = this.x();
        console.log("j.x:", ByteString.of(result).hex());
        return result;
    };
    
    j.a.overload('[B').implementation = function(bArr) {
        this.a(bArr);
        console.log("j.a:", ByteString.of(bArr).hex());
    };
    ```