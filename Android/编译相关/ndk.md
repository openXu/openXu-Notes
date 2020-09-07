



LLDB:是c/c++的调试器，可以用来做NDK开发的调试。


Android Studio支持两种NDK开发模式，ndk-build和CMake，ndk-build是在之前使用Eclipse(ADT)开发时使用的，环境配置和开发流程比较繁琐，对开发者来说不太友好。Android Studio后面支持的CMake(跨平台的安装（编译）工具)可以让开发人员更加方便的开发C++，环境配置非常简单。


# Cmake

通过在gradle中添加`CMakeLists.txt`文件路径将C++工程连接到当前工程参与编译。

可以通过`File -> Link C++ Project with Gradle`选择需要参与构建的native工程，设置Project Path为CMakeLists.txt路径。

或者直接在build.gradle中配置：

```xml
android {
    //在android模块下gradle文件中配置CMakeLists.txt路径
    externalNativeBuild {
        cmake {
            path file('../jni-lib/CMakeLists.txt')
        }
    }
}
```

## Android Studio编译C++可执行程序

新建工程默认是让C++源码编译成动态库的，如果我们要编译成可执行文件(带main函数)，需要修改CMakefile.txt：

```xml

add_library(
            native-lib
            SHARED  
            src/main/cpp/CTest.cpp)
//修改为：


```





