错误方案一
File Encodings 改为UTF-8？
没用！

错误方案二
build.gradle 添加如下代码？
```xml
tasks.withType(JavaCompile) {
    options.encoding = "UTF-8"
}
```
没用！ 这是解决System.out.print输出的中文乱码问题的！


正确解决办法
双击`Shift`，输入`vmoption`,，选择`Edit Custom CM Options`
如果之前没有配置过，会弹出窗口问是否创建配置文件，点击Create
输入
```xml
-Dfile.encoding=UTF-8
```
保存，重启就可以了！


其实就是修改：C:\Users\admin\.AndroidStudio3.6\config\studio64.exe.vmoptions 
-Dfile.encoding=UTF-8

**注意保存配置文件编码格式为ANSI，而不是UTF-8，否则导致Android Studio启动不起来或者卡死**