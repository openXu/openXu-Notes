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



