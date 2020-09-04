
我们使用Android Studio开发Android项目，如果需要将项目安装到手机上，只需要点一个运行按钮即可，打包的过程开发人员是不需要关心的，但是随着技术的发展，各种新的技术层出不清，比如热修复、插件化等等，这些都需要我们清楚的知道打包过程。也许你大概在别人的博客上或者技术论坛看过打包的流程，但是没有真正操作实现都不属于自己的，就像我自己。这篇博客主要记录apk打包流程。

[aapt2](https://developer.android.google.cn/studio/command-line/aapt2?authuser=19)

Android打包主要有如下几个步骤：

1. 编译资源文件
2. 将编译后的资源链接到一个apk中
3. 根据aidl生成java文件
4. 编译java文件为class字节码
5. 将class字节码打包为dex
6. 生成apk
7. 签名
8. 安装

在正式操作之前，我们需要配置build-tool环境，这里我是用的Build Tools版本是28.0.3。Android Gradle插件3.2.0以上要求SDK Build Tools 28.0.3或更高版本（当然手动打包跟gradle没关系）。在环境变量中添加Build Tools目录：

```xml
F:\IDE\sdk\build-tools\28.0.3
```

## 1. 编译资源

编译是将资源文件编译为二进制格式的文件，放到指定的目录，下面的命令首先cmd进入到项目根目录下，执行命令后编译的资源文件中间产物.flat文件将被放在compile目录下。

```xml
//使用compile命令编译单个资源文件，-o指定编译后的目标目录
> aapt2 compile app\src\main\res\mipmap-xhdpi\ic_launcher.png -o compile
> aapt2 compile app\src\main\res\layout\activity_main.xml -o compile

//编译指定的多个资源文件（Linux使用\表示输入未结束），可以实现增量编译
> aapt2 compile \
app\src\main\res\mipmap-xhdpi\ic_launcher.png \
app\src\main\res\layout\activity_main.xml \
-o compile

// --dir标记指定资源目录，但这样做会重新编译目录中的所有文件，将其打包到一个res.zip文件中
> aapt2 compile --dir app\src\main\res\ -o compile/res.zip
```

## 2. 链接

在链接阶段，AAPT2会合并在编译阶段生成的所有中间文件（如资源表、二进制XML文件和处理过的PNG文件），并将它们打包成一个APK。此外，在此阶段还会生成其他辅助文件，如R.java和ProGuard规则文件。不过，生成的APK不包含DEX字节码且未签名。
aapt package -f -m -J build -S res -A assets -M AndroidManifest.xml -I D:/Android/Sdk/platforms/android-27/android.jar  -F ./build/res.zip

```xml
//-I指定SDK路径
//--manifest指定项目清单文件路径
//--java指定R文件生成目录
//-o指定apk输出
//-v提高输出详细程度 
> aapt2 link compile\res.zip -I F:\IDE\sdk\platforms\android-28\android.jar --java app\src\main\java --manifest app\src\main\AndroidManifest.xml --min-sdk-version 15 --java app\src\main\java -o compile\resources.apk -v
```

> 链接过程中可能出现`error: resource style/Theme.AppCompat.Light.DarkActionBar (aka com.openxu.fix:style/Theme.AppCompat.Light.DarkActionBar) not found.`这是因为这些资源是引用的三方库资源没有包含进来，所以找不到。如果使用gradle是合并所有资源统一编译不会出现此问题。为了解决此问题，可以修改项目中对三方资源的引用，比如`<style name="AppTheme" parent="Theme.AppCompat.Light.DarkActionBar">`修改为`android:Theme.Holo.Light.DarkActionBar`


其实上面两步可以使用`aapt package`命令合并执行


## 3. 根据aidl生成java文件

由于本案例中没有aidl需求，就随便创建了一个，编译了一下

```xml
//-I指定aidl目录，注意后面不要有空格
//-p指定系统类aidl路径，如果使用到android.os.Bundle系统的类，一定要设置sdk的framework.aidl路径
//-o生成java文件的目录
//最后是需要编译的aidl文件
> aidl -Iapp/src/main/aidl -pF:\IDE\sdk\platforms\android-28\framework.aidl -oapp\src\main\aidl app\src\main\aidl\com\openxu\fix\IAidlTest.aidl
```

## 4. 编译Java

```xml
//1 将当前目录下所有java文件全路径保存到compile\all.txt文件中
> dir /b/s *.java > compile\all.txt

javac  -encoding utf-8 -target 1.8 -bootclasspath F:\IDE\sdk\platforms\android-28\android.jar -d compile @compile\all.txt
```
javac -d build -cp %ANDROID_HOME%\platforms\android-28\android.jar
src\main\java\com\qsc\hello\*.java


javac -target 1.8 -bootclasspath D:/Android/Sdk/platforms/android-27/android.jar -d ./build @all.txt

## 5. dex

dx.bat --dex --output=classes.dex build

https://blog.csdn.net/zhuoxiuwu/article/details/100518186
















































