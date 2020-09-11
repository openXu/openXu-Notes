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