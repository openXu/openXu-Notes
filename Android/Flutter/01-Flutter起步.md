https://book.flutterchina.club/chapter1/mobile_development_intro.html


# 1. 移动开发技术简介

## 1.1 原生开发和跨平台技术

### 原生开发

原生应用程序指某一个移动平台（Android或IOS）特有的应用，使用相应平台支持的开发工具和语言，并直接调用系统提供的SDK API。

主要优势：

- 可访问平台全部功能（GPS、摄像头）
- 速度快、性能高、可以实现复杂动画及绘制，整体用户体验好

主要缺点：

- 平台特定，不同代码，人力成本大，开发测试成本高
- 内容固定，动态化弱，大多数情况下，有新功能更新时只能发版

随着移动互联网发展，项目越来越复杂，传统原生开发的缺点导致不能满足日益增长的业务需求，所以诞生了一些跨平台动态化框架。


### 跨平台技术简介

目前跨平台框架主要分为三类

- H5+原生（Cordova、Ionic、微信小程序）
- JavaScript开发+原生渲染（React Native、Weex、快应用）
- 自绘UI+原生(QT for mobile、Flutter)

### H5+原生混合开发

这类框架主要原理就是将APP的一部分需要动态变动的内容通过H5来实现，通过原生的网页加载控件WebView (Android)或WKWebView（iOS）来加载。H5代码是运行在WebView中，而WebView实质上就是一个浏览器内核，其JavaScript依然运行在一个权限受限的沙箱中，所以对于大多数系统能力都没有访问权限，如无法访问文件系统、不能使用蓝牙等。所以，对于H5不能实现的功能，都需要原生去做。而混合框架一般都会在原生代码中预先实现一些访问系统能力的API， 然后暴露给WebView以供JavaScript调用，这样一来，WebView就成为了JavaScript与原生API之间通信的桥梁，主要负责JavaScript与原生之间传递调用消息，而消息的传递必须遵守一个标准的协议，它规定了消息的格式与含义，我们把依赖于WebView的用于在JavaScript与原生之间通信并实现了某种消息传输协议的工具称之为WebView JavaScript Bridge, 简称 JsBridge，它也是混合开发框架的核心。

**混合应用无非就是在第一步中预先实现一系列API供JavaScript调用，让JavaScript有访问系统的能力**，看到这里，我相信你也可以自己实现一个混合开发框架了。

**混合应用的优点是动态内容是H5，web技术栈，社区及资源丰富，缺点是性能不好，对于复杂用户界面或动画，WebView不堪重任。**


### React Native、Weex及快应用

JavaScript解释器， 映射为原生控件树。

JavaScript开发+原生渲染的方式主要优点如下：

- 采用Web开发技术栈，社区庞大、上手快、开发成本相对较低。
- 原生渲染，性能相比H5提高很多。
- 动态化较好，支持热更新。

不足：

- 渲染时需要JavaScript和原生之间通信，在有些场景如拖动可能会因为通信频繁导致卡顿。
- JavaScript为脚本语言，执行时需要JIT(Just In Time)，执行效率和AOT(Ahead Of Time)代码仍有差距。
- 由于渲染依赖原生控件，不同平台的控件需要单独维护，并且当系统更新时，社区控件可能会滞后；除此之外，其控件系统也会受到原生UI系统限制，例如，在Android中，手势冲突消歧规则是固定的，这在使用不同人写的控件嵌套时，手势冲突问题将会变得非常棘手。


### 自绘UI+原生

通过在不同平台实现一个统一接口的渲染引擎来绘制UI，而不依赖系统原生控件，所以可以做到不同平台UI的一致性。注意，自绘引擎解决的是UI的跨平台问题，如果涉及其它系统能力调用，依然要涉及原生开发。这种平台技术的优点如下：

- 性能高；由于自绘引擎是直接调用系统API来绘制UI，所以性能和原生控件接近。
- 灵活、组件库易维护、UI外观保真度和一致性高；由于UI渲染不依赖原生控件，也就不需要根据不同平台的控件单独维护一套组件库，所以代码容易维护。由于组件库是同一套代码、同一个渲染引擎，所以在不同平台，组件显示外观可以做到高保真和高一致性；另外，由于不依赖原生控件，也就不会受原生布局系统的限制，这样布局系统会非常灵活。

不足：

- 动态性不足；为了保证UI绘制性能，自绘UI系统一般都会采用AOT模式编译其发布包，所以应用发布后，不能像Hybrid和RN那些使用JavaScript（JIT）作为开发语言的框架那样动态下发代码。
- 开发效率低：QT使用C++作为其开发语言，而编程效率是直接会影响APP开发效率的，C++作为一门静态语言，在UI开发方面灵活性不及JavaScript这样的动态语言，另外，C++需要开发者手动去管理内存分配，没有JavaScript及Java中垃圾回收（GC）的机制。
也许你已经猜到Flutter就属于这一类跨平台技术，没错，Flutter正是实现一套自绘引擎，并拥有一套自己的UI布局系统。不过，自绘制引擎的思路并不是什么新概念，Flutter并不是第一个尝试这么做的，在它之前有一个典型的代表，即大名鼎鼎的QT。


### Flutter出世

Flutter是Google发布的一个用于创建跨平台、高性能移动应用的框架。Flutter和QT mobile一样，都没有使用原生控件，相反都实现了一个自绘引擎，使用自身的布局、绘制系统。

- 2017 年 Google I/O 大会上，Google 首次推出了一款新的用于创建跨平台、高性能的移动应用框架——Flutter。
- 2018年2月，Flutter发布了第一个Beta版本，同年五月， 在2018年Google I/O 大会上，Flutter 更新到了 beta 3 版本。
- 2018年6月，Flutter发布了首个预览版本，这意味着 Flutter 进入了正式版（1.0）发布前的最后阶段。


# 初识Flutter

Flutter 是 Google推出并开源的移动应用开发框架，主打跨平台、高保真、高性能。开发者可以通过 Dart语言开发 App，一套代码同时运行在 iOS 和 Android平台。

**跨平台自绘引擎**

Flutter与用于构建移动应用程序的其它大多数框架不同，因为Flutter既不使用WebView，也不使用操作系统的原生控件。 相反，Flutter使用自己的高性能渲染引擎Skia来绘制widget。这样不仅可以保证在Android和iOS上UI的一致性，而且也可以避免对原生控件依赖而带来的限制及高昂的维护成本。


# 配置环境

- [下载flutter sdk](https://flutter.dev/docs/development/tools/sdk/releases?tab=macos#macos)

- 解压后配置环境`export PATH=/Users/openXu/lib/flutter/bin:$PATH`，`source ～/.bash_profile`，`echo $PATH`

- flutter doctor

- Android studio 安装Flutter、Dart插件

- 创建Flutter项目，项目报错，Terminal中执行`flutter packages get`，如果卡死，添加镜像

```xml
export PUB_HOSTED_URL=https://pub.flutter-io.cn
export FLUTTER_STORAGE_BASE_URL=https://storage.flutter-io.cn
```

- 运行项目卡死`flutter doctor --android-licenses`

- 运行项目卡死


```xml
//项目/android/build.gradle
maven { url "http://download.flutter.io" }
maven { url 'https://maven.aliyun.com/repository/google' }
maven { url 'https://maven.aliyun.com/repository/jcenter' }
maven { url 'http://maven.aliyun.com/nexus/content/groups/public' }

///Users/openXu/lib/flutter/packages/flutter_tools/gradle/flutter.gradle
buildscript {
    repositories {
        //google()
        //jcenter()
        maven { url 'https://maven.aliyun.com/repository/google' }
        maven { url 'https://maven.aliyun.com/repository/jcenter' }
        maven { url 'http://maven.aliyun.com/nexus/content/groups/public' }
    }

  //private static final String DEFAULT_MAVEN_HOST = "https://storage.googleapis.com";
    private static final String DEFAULT_MAVEN_HOST = "https://storage.flutter-io.cn/download.flutter.io";
   

void addFlutterDependencies(buildType) {
        //...
            repositories {
                maven {
                    url repository
                }
                maven { url 'https://maven.aliyun.com/repository/google' }
                maven { url 'https://maven.aliyun.com/repository/jcenter' }
                maven { url 'http://maven.aliyun.com/nexus/content/groups/public' }
            }

```







