八、so 热修复方案
8.1 对加载过程进行封装，替换 System.loadLibrary
在加载 so 库的时候，系统提供了两个接口

System.loadLibrary(String libName)：用来加载已经安装的 apk 中的 so
System.load(String pathName)：可以加载自定义路径下的 so
通过上面两个方法，我们可以想到，如果有补丁 so 下发，我们就调用 System.load 去加载，如果没有补丁 so 没有下发，那么还是调用 System.loadLibrary 去加载系统目录下的 so，原理比较简单，但是我们需要再上面进行一层封装，并对调用 System.loadLibrary 的地方都进行替换。

8.2 反射注入补丁 so 路径
还记得上面 dex 插桩的原理么？在 DexPathList 中有 dexElements 变量，代表着所有 dex 文件，其实 DexPathList 中还有另一个变量就是 Element[] nativeLibraryPathElements，代表的是 so 的路径，在加载 so 的时候也会遍历 nativeLibraryPathElements 进行加载，代码如下：

public String findLibrary(String libraryName) {
        String fileName = System.mapLibraryName(libraryName);
        // 遍历 nativeLibraryPathElements 
        for (Element element : nativeLibraryPathElements) {
            String path = element.findNativeLibrary(fileName);

            if (path != null) {
                return path;
            }
        }
        return null;
    }
看到这里我们就知道如何去做了吧，就像 dex 插桩一样的方法，将 so 的路径插入到 nativeLibraryPathElements 之前即可。

# 怎么更新 so 文件？

在 Android 项目中使用 native 函数前需要先调用 System.loadLibrary(libName)。

当 lib 文件需要更新或者有 bug 时候怎么办？首先想到的是在代码中把加载 so 文件的代码改成System.load(libFilePath)，让系统加载自己指定的 libFilePath 文件。然而这样的改动需要

在源代码中修改或者使用工具在编译期把 loadLibrary 接口改为 load
patch 库把 so 文件从 patch 文件中复制到特定目录
这样在运行期才有可能加载更新后的 so 文件。

通过分析系统加载 so 文件的方式后，我们使用了更简单的处理方法。查找 lib 文件是通过调用 PathClassLoader 的 findLibrary，最终调用到 DexPathList 的 findLibrary。DexPathList 会在自己维护的列表目录中查找对应的 lib 文件是否存在。所以我们在发现 patch 文件中有 so 文件变更的时候，会在 PathClassLoader 的 nativeLibraryDirectories（Android6.0以下）或者nativeLibraryPathElements （Android 6.0及以上）的最前面插入自定义的lib文件目录。这样 ClassLoader 在 findLibrary 的时候会先在自定义的 lib 目录中查找，优先加载变更过的 so 文件。

## 原理

**热修复or冷启动修复**

so库的实时生效涉及到Dalvik虚拟机注册native方法，如果是动态注册的native方法需要实时生效，必须对so文件改名，但也不能确保一定是注册的补丁包中的native方法，需要看补丁库和原库在hashtable中的位置谁靠前。而so库静态注册的native方法要实时生效，需要先解除之前注册的原so库的native方法，这也是很困难的，因为不知道那个方法发生了修复需要被解除。实时生效还涉及到其他问题，感兴趣的可以参考一下资料中的《深入探索Android热修复技术原理》，这是阿里手机淘宝技术团队写的，内容偏向于原理理论。

总之so库修复要实现实时生效是很困难的，而且有各种兼容性问题，所以通常采用冷启动生效的方案。

**hook点探索**


Java API提供了两个接口加载so库：

- System.loadLibrary(String libName)：参数为so库名称，用来加载已经安装的apk中(apk压缩文件libs目录下)的so

- System.load(String pathName)：参数为so库在磁盘中完整路径，可以加载自定义路径下的so

上面两中方式最后都调用了nativeLoad这个native方法加载so库，

```Java
//java.lang.Runtime

 private synchronized void loadLibrary0(ClassLoader loader, Class<?> callerClass, String libname) {
        if (libname.indexOf((int)File.separatorChar) != -1) {
            throw new UnsatisfiedLinkError(
    "Directory separator should not appear in library name: " + libname);
        }
        String libraryName = libname;
        if (loader != null && !(loader instanceof BootClassLoader)) {
            //★ 调用DexPathList类加载器的方法获取so库路径
            String filename = loader.findLibrary(libraryName);
            if (filename == null) {
                throw new UnsatisfiedLinkError(loader + " couldn't find \"" +
                                               System.mapLibraryName(libraryName) + "\"");
            }
            String error = nativeLoad(filename, loader);
            if (error != null) {
                throw new UnsatisfiedLinkError(error);
            }
            return;
        }
        getLibPaths();
        String filename = System.mapLibraryName(libraryName);
        String error = nativeLoad(filename, loader, callerClass);
        if (error != null) {
            throw new UnsatisfiedLinkError(error);
        }
    }

private static native String nativeLoad(String filename, ClassLoader loader, Class<?> caller);
```



通过上面两个方法，我们可以想到，如果有补丁 so 下发，我们就调用 System.load 去加载，如果没有补丁 so 没有下发，那么还是调用 System.loadLibrary 去加载系统目录下的 so，原理比较简单，但是我们需要再上面进行一层封装，并对调用 System.loadLibrary 的地方都进行替换。

## 接口调用替换方案实现






































