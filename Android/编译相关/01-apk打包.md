
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
//windows在path中添加
F:\IDE\sdk\build-tools\28.0.3

//mac(本文是在Mac系统下操作的，如果是windows系统，需要注意路径分隔符区别)
> vim ~/.bash_profile
添加：export PATH=/Users/openXu/Library/Android/sdk/build-tools/28.0.3:$PATH
> :wq
> source ~/.bash_profile
```

## 1. 编译资源

使用aapt2的compile命令传递一个资源文件作为输入，会解析该文件并生成一个扩展名为.flat的中间二进制文件放入指定目录。可以使用--dir参数标记多个资源文件的资源目录。

```xml
//进入项目根目录下
> cd CompileApk/

//使用compile命令编译单个资源文件，-o指定编译后中间产物存放目录
> aapt2 compile app/src/main/res/mipmap-xhdpi/ic_launcher.png -o compileDir
> aapt2 compile app/src/main/res/layout/activity_main.xml -o compileDir

//编译指定的多个资源文件（\表示输入未结束）
> aapt2 compile \
app/src/main/res/mipmap-xhdpi/ic_launcher.png \
app/src/main/res/layout/activity_main.xml \
-o compileDir

//--dir标记指定资源目录，但这样做会重新编译目录中的所有文件，将其打包到一个res.zip文件中（由于是编译整个项目，所以采用这种方式）
> aapt2 compile --dir app/src/main/res/ -o compileDir/res.zip
```

## 2. 链接

链接阶段aapt2会合并在编译阶段生成的所有中间文件（如资源表、二进制XML文件和处理过的PNG文件），并将它们打包成一个apk(为了做区分，后缀用ap_)。此外，在此阶段还会生成其他辅助文件，如R.java和ProGuard规则文件。不过，生成的APK不包含DEX字节码且未签名，所以不能运行。

```xml
//-I指定链接时需要使用的外部资源路径，比如android.jar
//--manifest指定项目清单文件路径
//--java指定R文件生成目录，R文件生成会自带包名
//-o指定apk输出
//-v提高输出详细程度 
> aapt2 link compileDir/res.zip -I /Users/openXu/Library/Android/sdk/platforms/android-28/android.jar --java app/src/main/java --manifest app/src/main/AndroidManifest.xml --min-sdk-version 15 -o compileDir/resources.ap_ -v
```

> 链接过程中可能出现`error: resource style/Theme.AppCompat.Light.DarkActionBar (aka com.openxu.fix:style/Theme.AppCompat.Light.DarkActionBar) not found.`这是因为这些资源是引用的三方库资源没有包含进来，所以找不到。如果使用gradle是合并所有资源统一编译不会出现此问题。为了解决此问题，可以修改项目中对三方资源的引用，比如`<style name="AppTheme" parent="Theme.AppCompat.Light.DarkActionBar">`修改为`android:Theme.Holo.Light.DarkActionBar`

aapt命令相关详细参数配置请参考[aapt2](https://developer.android.google.cn/studio/command-line/aapt2?authuser=19)

## 3. 根据aidl生成java文件

由于本案例中没有aidl需求，就随便创建了一个，编译了一下

```xml
//-I指定aidl目录，注意后面不要有空格
//-p指定系统类aidl路径，如果使用到android.os.Bundle系统的类，一定要设置sdk的framework.aidl路径
//-o生成java文件的目录
//最后是需要编译的aidl文件
> aidl -Iapp/src/main/aidl -p/Users/openXu/Library/Android/sdk/platforms/android-28/framework.aidl -oapp/src/main/aidl app/src/main/aidl/com/openxu/compile/IAidlTest.aidl
```

## 4. 编译Java文件

编译Java文件时需要注意，我们项目里面有没有依赖其他库（.jar），如果有则需要在javac命令中指定，否则会找不到类错误。

```xml
//由于javac指定Java文件目录只支持一级目录(不支持子目录)，可将所有java文件全路径保存到compile\all.txt文件中。/s查找当前目录以及所有子目录下的文件
> dir /b/s app/src/main/*/*.java > compileDir\all.txt

//上述方法Mac不支持，只能手动创建all.txt添加所有Java文件路径，然后编译
> javac  -encoding utf-8 -target 1.8 -bootclasspath /Users/openXu/Library/Android/sdk/platforms/android-28/android.jar -d compileDir @compileDir/all.txt

//或者直接指定Java文件路径
> javac -encoding utf-8 -target 1.8 -bootclasspath /Users/openXu/Library/Android/sdk/platforms/android-28/android.jar -d compileDir app/src/main/java/com/openxu/compile/*.java app/src/main/aidl/com/openxu/compile/*.java
```

编译完成后，所有Java文件对应的class字节码输出在compileDir文件夹中。由于Android虚拟机中运行的字节码是dex的，所以下一步是将所有class文件编译为dex字节码。

## 5. 编译class文件为dex

编译dex使用的是2017年发布的d8编译器，之前的dx编译器会逐渐被取代，d8编译器相比dx编译器会更快，生成的dex文件更小，运行性能更佳。2018年4月，d8已成为Android Studio 3.1中的默认选项。

d8相关更多信息请参考[d8](https://developer.android.google.cn/studio/command-line/d8?authuser=19)

```xml
//老的dx编译器：--output指定输出，最后的参数指定class文件所在路径（compileDir目录下包含包名在内）
//dx --dex --output=compileDir/classes.dex compileDir

//d8编译器，输入字节码可以是 *.class 文件或容器（例如 JAR、APK 或 ZIP 文件）的任意组合。您还可以添加 DEX 文件作为 d8 的输入，以将这些文件合并到 DEX 输出中，这在包含增量构建的输出时很有用。
//--output 指定dex输出路径，默认是当前工作目录
> d8 compileDir/com/openxu/*/*.class --output compileDir
```

## 6. 生成apk


```xml

java -classpath /Users/openXu/Library/Android/sdk/tools/lib/sdklib-26.0.0-dev.jar com.android.sdklib.build.ApkBuilderMain compileDir/source.apk -v -u -z compileDir/resources.ap_ -f compileDir/classes.dex
```

## 7. apk签名


https://blog.csdn.net/qq_32115439/article/details/55520012


apk签名有两种方式，一种是使用jdk提供的签名工具，早期Android就是通过它签名的，但是这种签名方式在高版本Android系统验证不通过。Android SDK构建工具24.0.3及更高版本中提供了apksigner工具为APK签名，并确保APK的签名能够在APK支持的所有版本的Android平台上成功通过验证。签名工具位于Android SDK/build-tools/version/apksigner.bat

```xml
//生成签名文件
//key.jks , 这个是生成jks的名字
//-validity 10000 , 中的10000,单位是天
//openxuKey 是别名
//123456 , 是你的密码
keytool -genkey -v -keystore compileDir/key.jks -keyalg RSA -keysize 2048 -validity 10000 -alias openxuKey -storepass 123456
```

- 使用jarsigner签名（不推荐）

jarsigner是JDK提供的针对jar包签名的通用工具，位于JDK/bin/jarsigner.exe。

```xml
//老的签名方式，这里使用debug签名
jarsigner -verbose -keystore ~/.android/debug.keystore -storepass android -keypass android compileDir/source.apk androiddebugkey
```

- 使用apksigner签名

关于签名更多信息参考[apksigner](https://developer.android.google.cn/studio/command-line/apksigner?authuser=19)

```xml
//--ks 选项指定密钥库文件
//--out 保存已签名APK的位置。如未提供此选项，则APK将就地签名并替换输入的APK文件
//--ks-pass 签名者私钥的密码
//--in 指定输入apk
> apksigner sign --ks compileDir/key.jks --ks-pass pass:123456 --out compileDir/source-signed.apk  --in compileDir/source.apk 
```

## 8. 安装

```xml
//安装
> adb install compileDir/source-signed.apk 

> 
```


6、优化apk

zipalign -v 4 compileDir/source-signed.apk compileDir/source-signed-align.apk

7、apk签名验证



https://blog.csdn.net/zhuoxiuwu/article/details/100518186

https://blog.csdn.net/fdd11119/article/details/51543571?utm_medium=distribute.pc_relevant.none-task-blog-BlogCommendFromMachineLearnPai2-1.channel_param&depth_1-utm_source=distribute.pc_relevant.none-task-blog-BlogCommendFromMachineLearnPai2-1.channel_param














































