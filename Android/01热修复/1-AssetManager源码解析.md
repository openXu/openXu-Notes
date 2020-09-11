https://www.androidos.net.cn/sourcecode

本文主线源码基于[Android8.0(API 26)](https://www.androidos.net.cn/sourcecode)

查看Framework Java层隐藏代码前往[android-hidden-api](https://github.com/anggrayudi/android-hidden-api)下载某一个版本的android.jar，替换`sdk\platforms\android-xxx`下对应版本的android.jar后rebuild。

# AssetManager创建分析

https://blog.csdn.net/luoshengyang/article/details/8791064

## 相关类之间的关系

android程序中我们使用`context.getResources().getString(R.string.app_name)`等方式获取资源，Context的真实实现类ContextImpl中持有`Resources`的实例mResources，Resources中持有`ResourcesImpl`实例mResourcesImpl，而ResourcesImpl中保存了`AssetManager`的实例`mAssets`，这个mAssets就是我们用来获取资源的对象。

> `ResourcesImpl`是资源访问的实现，在Android7.0(API 24)之前`mAssets`是直接放在Resources类中的，7.0及以后版本中Resources仅作为一个资源包装类，它不再维护`mAssets`，而是由`ResourcesImpl`维护的。

> 应用程序除了要访问自己安装包中的资源之外，还需要访问系统资源，Resources中有一个static的成员变量`static Resources mSystem`，它用来访问系统的资源，系统的资源打包在/system/framework/framework-res.apk文件中。同样在AssetManager中也有一个`static AssetManager sSystem`。

Android工程中的资源文件在编译时通过aapt将res目录下的资源文件编译成二进制格式，并通过链接生成R文件和resources.arsc资源表。在Android程序中，实际上最终都是通过`AssetManager`获取的。

AssetManager有Java层和C++层，Java层的功能都是通过C++层的AssetManager类实现的。Java层中每个`AssetManager`对象都有一个long类型的`mObject`，它保存了C++层AssetManager对象的地址，通过这个指针就能实现Java层与C++层的AssetManager对象关联。

## 源码跟踪

关键源码位置：
java层：
- android.content.res.Resources
- android.content.res.ResourcesImpl
- android.content.res.AssetManager
jni层：
- frameworks/base/core/jni/android_util_AssetManager.cpp
C++层：
- frameworks/base/libs/androidfw/include/androidfw/AssetManager.h
- frameworks/base/libs/androidfw/AssetManager.cpp 


下面以Activity启动过程中，为ContextImpl创建AssetManager对象为例，源码追踪从ActivityThread创建Activity开始，直到AssetManager对象创建完（注释中带★的表示下一步要继续追踪）：

**1. android.app.ActivityThread.performLaunchActivity()**

```Java
private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
      //...★创建一个ContextImpl对象
      ContextImpl appContext = createBaseContextForActivity(r);
      Activity activity = null;
      try {
          java.lang.ClassLoader cl = appContext.getClassLoader();
          activity = mInstrumentation.newActivity(
                  cl, component.getClassName(), r.intent);
      }//...
      try {
          //... 将创建的ContextImpl对象赋值给ContextWrapper的mBase成员变量
          activity.attach(appContext,...);
          //...
      }//...
      return activity;
  }
```

> 该方法创建一个Activity，调用了`createBaseContextForActivity(r)`，进而调用`ContextImpl.createActivityContext()`来为Activity创建ContextImpl对象

**2. android.app.ContextImpl.createActivityContext()**

```Java
static ContextImpl createActivityContext(ActivityThread mainThread,
            LoadedApk packageInfo, ActivityInfo activityInfo, IBinder activityToken, int displayId,
          Configuration overrideConfiguration) {
      
      //LoadedApk描述当前启动activity所属apk，里面就包含了apk路径等信息
      String[] splitDirs = packageInfo.getSplitResDirs();
      ClassLoader classLoader = packageInfo.getClassLoader();
      //...new 一个ContextImpl对象
      ContextImpl context = new ContextImpl(null, mainThread, packageInfo, activityInfo.splitName,
              activityToken, null, 0, classLoader);
      //创建Resources对象并设置给ContextImpl
      final ResourcesManager resourcesManager = ResourcesManager.getInstance();
      context.setResources(//★
        resourcesManager.createBaseActivityResources(activityToken,
              packageInfo.getResDir(),
              splitDirs,
              ...));
      return context;
  }
```

> 调用`ContextImpl`的静态方法`createActivityContext()`创建ContextImpl实例，该方法首先new一个ContextImpl对象，然后调用单例ResourcesManager的`createBaseActivityResources()`方法为ContextImpl创建Resources对象并设置给ContextImpl

**3. android.app.ResourcesManager.createBaseActivityResources()**

```Java
public @Nullable Resources createBaseActivityResources(...) {
    //根据应用的一些信息（apk路径、显示情况、分辨率等）生成一个key
    final ResourcesKey key = new ResourcesKey(resDir,  splitResDirs, overlayDirs, libDirs, displayId,...);
    //...
    synchronized (this) {
        // Force the creation of an ActivityResourcesStruct.
        getOrCreateActivityResourcesStructLocked(activityToken);
    }
    // Update any existing Activity Resources references.
    updateResourcesForActivity(activityToken, overrideConfig, displayId, false /* movedToDifferentDisplay */);
    // ★根据key创建一个Resources对象
    return getOrCreateResources(activityToken, key, classLoader);
}
```

> 该方法首先根据应用的一些信息生成一个ResourcesKey（包含资源路径），然后调用`getOrCreateResources()`根据key请求一个Resources对象

**4. android.app.ResourcesManager.getOrCreateResources()**

```Java
private @Nullable Resources getOrCreateResources(IBinder activityToken, ResourcesKey key,ClassLoader classLoader) {
    synchronized (this) {//...
        if (activityToken != null) {
            //...从ArrayMap中根据key查找ResourcesImpl
            ResourcesImpl resourcesImpl = findResourcesImplForKeyLocked(key);
            if (resourcesImpl != null) {
              //如果找到了就使用该ResourcesImpl对象包裹上Resources返回
                return getOrCreateResourcesForActivityLocked(activityToken, classLoader,
                        resourcesImpl, key.mCompatInfo);
            }
        } else {//...}
    }

    //★如果没有找到，就创建一个ResourcesImpl对象
    ResourcesImpl resourcesImpl = createResourcesImpl(key);
    synchronized (this) {
        ResourcesImpl existingResourcesImpl = findResourcesImplForKeyLocked(key);
        if (existingResourcesImpl != null) {
            resourcesImpl.getAssets().close();
            resourcesImpl = existingResourcesImpl;
        } else {
            // 将ResourcesImpl缓存起来
            mResourceImpls.put(key, new WeakReference<>(resourcesImpl));
        }
        //创建Resources，并将ResourcesImpl对象设置给它后返回
        final Resources resources;
        if (activityToken != null) {
            resources = getOrCreateResourcesForActivityLocked(activityToken, classLoader,
                    resourcesImpl, key.mCompatInfo);
        } else {
            resources = getOrCreateResourcesLocked(classLoader, resourcesImpl, key.mCompatInfo);
        }
        return resources;
    }
}
```

> 首先从`ArrayMap<ResourcesKey, WeakReference<ResourcesImpl>> mResourceImpls`中根据key查找ResourcesImpl对象，这个ArrayMap维护当前应用进程中加载的每个apk文件(apk路径生成的key)及对应的ResourcesImpl对象，通常情况下，同一个应用不同Activity的ResourcesKey是相同的，也就是说使用的是同一个ResourcesImpl对象。如果没找到就创建，找到了就使用这个对象，然后根据ResourcesImpl对象创建Resources。这里假设没有找到对应的ResourcesImpl对象，继续调用`createResourcesImpl()`。

**5. android.app.ResourcesManager.createResourcesImpl()**

```java
private @Nullable ResourcesImpl createResourcesImpl(@NonNull ResourcesKey key) {
    //...★创建java层的AssetManager对象
    final AssetManager assets = createAssetManager(key);

    final DisplayMetrics dm = getDisplayMetrics(key.mDisplayId, daj);
    final Configuration config = generateConfig(key, dm);
    //创建ResourcesImpl对象，并将AssetManager对象设置给ResourcesImpl的mAssets
    //★ResourcesImpl构造方法中调用了updateConfiguration()方法，这个方法后面会继续跟踪
    final ResourcesImpl impl = new ResourcesImpl(assets, dm, config, daj);
    return impl;
}
```

> 该方法中调用了`createAssetManager(key)`创建一个AssetManager对象，然后通过构造方法new一个ResourcesImpl对象，下面看看创建AssetManager做了那些事

**6. android.app.ResourcesManager.createAssetManager(key)**

```Java
 protected @Nullable AssetManager createAssetManager(@NonNull final ResourcesKey key) {
    //★new一个AssetManager对象，构造方法中调用了init()C++方法
    AssetManager assets = new AssetManager();
    if (key.mResDir != null) {
        //★调用addAssetPath()方法添加资源路径
        if (assets.addAssetPath(key.mResDir) == 0) {
            return null;
        }
    }
    //...
    return assets;
}
```

> 该方法首先通过构造函数创建AssetManager对象，然后调用`addAssetPath()`添加资源路径，下面我们看看Java层的AssetManager类

**7. android.content.res.AssetManager**

```Java
public final class AssetManager implements AutoCloseable {
    private long mObject;  //保存C++层AssetManager对象地址
    public AssetManager() {
        synchronized (this) {
            //...调用framework native层C++方法，目的就是创建C++层的AssetManager对象
            init(false);
            ensureSystemAssets();
        }
    }
    private native final void init(boolean isSystem);
    
    public final int addAssetPath(String path) {
      //该方法实际调用addAssetPathNative()
        return  addAssetPathInternal(path, false);
    }
    //是一个JNI方法，对应android_content_AssetManager_addAssetPath
    private native final int addAssetPathNative(String path, boolean appAsLib);
}
```

> AssetManager的构造方法中调用`init()`方法，该方法是framework native层C++方法，目的就是创建C++层的AssetManager对象。第6步紧接着调用了`addAssetPath()`添加资源路径，这个方法也是C++的。下面我们开始追踪jni层的C++代码。

需要注意的是Android8.0（API26-27）和Android9.0（API28）关于底层资源管理做了比较大的改动，9.0开始引入了AssetManager2，底层结构发生了比较大的变化[简单参考](https://blog.csdn.net/dayong198866/article/details/103575296)，相关资料文档比较少，但是核心的AssetManager功能还是差不多。跟踪这部分代码我们主要搞清楚底层是怎样管理资源的，只需要关注关键点即可，这里我们还是继续跟踪API26的代码。

> C++不了解的同学可以自行搜索学习一下基本语法，最起码要认识代码，比如通过[C++菜鸟教程](https://www.runoob.com/cplusplus/cpp-tutorial.html)

**8. android_content_AssetManager_init()**

```C++
//源码位置frameworks/base/core/jni/android_util_AssetManager.cpp
//AssetManager类定义在frameworks/base/libs/androidfw/include/androidfw/AssetManager.h
static void android_content_AssetManager_init(JNIEnv* env, jobject clazz, jboolean isSystem)
{
    //...首先创建一个C++层的AssetManager对象
    AssetManager* am = new AssetManager();
    //...★添加默认的资源路径
    am->addDefaultAssets();
    //...将C++层AssetManager对象的地址保存在clazz描述的Java层AssetManager对象成员变量mObject中
    env->SetLongField(clazz, gAssetManagerOffsets.mObject, reinterpret_cast<jlong>(am));
}
```

**9. AssetManager::addDefaultAssets()**

```C++
//源码位置frameworks/base/libs/androidfw/AssetManager.cpp

static const char* kSystemAssets = "framework/framework-res.apk";

bool AssetManager::addDefaultAssets()
{
    //通过环境变量ANDROID_ROOT获得Android系统路径（通常为/system）
    const char* root = getenv("ANDROID_ROOT");
    LOG_ALWAYS_FATAL_IF(root == NULL, "ANDROID_ROOT not set");
    //根据root创建一个类型为String8的path对象
    String8 path(root);
    //调用path的appendPath()方法将kSystemAssets追加到root后面，得到系统资源文件绝对路径/system/framework/framework-res.apk
    path.appendPath(kSystemAssets);
    //★将系统资源路径添加到当前AssetManager对象中去
    return addAssetPath(path, NULL, false /* appAsLib */, true /* isSystemAsset */);
}
```

**10. AssetManager::addAssetPath()**

```C++
tatic const char* kAppZipName = NULL; //"classes.jar";
bool AssetManager::addAssetPath(const String8& path, int32_t* cookie, bool appAsLib, bool isSystemAsset) {
    AutoMutex _l(mLock);
    asset_path ap;
    //资源路径校验
    String8 realPath(path);
    // 如果不为null，表示资源文件是保存在path目录下的classes.jar文件中
    if (kAppZipName) {  
        realPath.appendPath(kAppZipName);
    }
    //得到path是表示一个文件或者一个目录，判断该文件或者目录是否存在
    ap.type = ::getFileType(realPath.string());
    if (ap.type == kFileTypeRegular) {
        ap.path = realPath;
    } else {
        ap.path = path;
        ap.type = ::getFileType(path.string());
        if (ap.type != kFileTypeDirectory && ap.type != kFileTypeRegular) {
            return false;
        }
    }
    //检查path描述的apk路径是否添加过，如果添加过就跳过不再处理
    for (size_t i=0; i<mAssetPaths.size(); i++) {
        if (mAssetPaths[i].path == ap.path) {
            if (cookie) {
                *cookie = static_cast<int32_t>(i+1);
            }
            return true;
        }
    }
    ap.isSystemAsset = isSystemAsset;
    //将path路径添加到mAssetPaths(一个vector容器)末尾
    mAssetPaths.add(ap);
    //mAssetPaths的大小保存在cookie中
    if (cookie) {
        *cookie = static_cast<int32_t>(mAssetPaths.size());
    }

#ifdef __ANDROID__
    /*检查刚刚添加的Apk文件路径是否是保存在/system/framework/目录下面的。如果是的话，那么就会在/vendor/overlay/framework/目录下找到一个同名的Apk文件，并且也会将该Apk文件添加到成员变量mAssetPaths所描述的一个Vector中去。这是一种资源覆盖机制，手机厂商可以利用它来自定义的系统资源，即用自定义的系统资源来覆盖系统默认的系统资源，以达到个性化系统界面的目的。关于Android系统的资源覆盖（Overlay）机制，可以参考frameworks/base/libs/utils目录下的READ文件。
    */
    asset_path oap;
    for (size_t idx = 0; mZipSet.getOverlay(ap.path, idx, &oap); idx++) {
        oap.isSystemAsset = isSystemAsset;
        mAssetPaths.add(oap);
    }
#endif

    if (mResources != NULL) {
        //☆添加资源到资源表，将资源文件下的资源添加到资源表中，只有第一次获取资源后mResources才不为空，所以一般情况下不会执行这里，后面讲获取资源源码时会看到mResources什么时候被初始化
        appendPathToResTable(ap, appAsLib);
    }

    return true;
}
```

> addAssetPath()方法就是将path描述的资源路径保存到mAssetPaths动态数组（存储asset_path结构体对象）中，asset_path是一个结构体，它封装了资源路径以及文件类型(是文件还是文件夹)等信息，到此C++层的AssetManager对象就算创建完成，上面第6步又调用`assets.addAssetPath(key.mResDir)`将应用程序的资源路径也添加进去。addAssetPath()方法最后会调用`appendPathToResTable()`去加载路径下的resources.arsc并解析，将资源索引添加到ResTable中保存。其实到这里AssetManager创建过程算是跟踪完了，但是关于资源还有一个重要的事情，就是设置设备信息，感兴趣可以继续往下看

**11. android_content_AssetManager_addAssetPath()**

```C++
//源码位置frameworks/base/core/jni/android_util_AssetManager.cpp
static jint android_content_AssetManager_addAssetPath(JNIEnv* env, jobject clazz,
                                                       jstring path, jboolean appAsLib)
{
    //...
    //根据clazz指向的Java层AssetManager对象中的mObject保存的地址得到C++层AssetManager指针
    AssetManager* am = assetManagerForJavaObject(env, clazz);

    int32_t cookie;
    //继续调用第10步的AssetManager::addAssetPath()添加应用程序资源路径
    bool res = am->addAssetPath(String8(path8.c_str()), &cookie, appAsLib);
    return (res) ? static_cast<jint>(cookie) : 0;
}
```

> 在第5步时，我们创建了AssetManager对象，并通过构造方法创建ResourcesImpl，ResourcesImpl的构造方法中调用了`updateConfiguration()`方法，下面我们看看它做了什么事情

**12. android.content.res.ResourcesImpl.updateConfiguration()**

```java
private final Configuration mConfiguration = new Configuration();

public void updateConfiguration(Configuration config, DisplayMetrics metrics,
                                    CompatibilityInfo compat) {
  try {
      synchronized (mAccessLock) {
          //...
          final @Config int configChanges = calcConfigChanges(config);
          //区域设置
          LocaleList locales = mConfiguration.getLocales();
          if (locales.isEmpty()) {
              locales = LocaleList.getDefault();
              mConfiguration.setLocales(locales);
          }
          //...屏幕密度Dpi
          if (mConfiguration.densityDpi != Configuration.DENSITY_DPI_UNDEFINED) {
              mMetrics.densityDpi = mConfiguration.densityDpi;
              mMetrics.density =
                      mConfiguration.densityDpi * DisplayMetrics.DENSITY_DEFAULT_SCALE;
          }
          //字体缩放
          mMetrics.scaledDensity = mMetrics.density *
                  (mConfiguration.fontScale != 0 ? mConfiguration.fontScale : 1.0f);
          //...★调用jni方法
          mAssets.setConfiguration(mConfiguration.mcc, mConfiguration.mnc,
                  adjustLanguageTag(mConfiguration.getLocales().get(0).toLanguageTag()),
                  mConfiguration.orientation,
                  mConfiguration.touchscreen,
                  mConfiguration.densityDpi, ...);
          //...
      }
  }
}

```

> updateConfiguration()主要用来更新ResourcesImpl成员变量mConfiguration的信息（18个资源维度），Configuration描述了设备的配置信息（国家、地区、语言、屏幕密度等），在android项目中，同一张图片可能有多种分辨率放到不同的资源目录下，比如mipmap-hdpi、mipmap-xhdpi，但是它们的资源名和id是相同的，当一台设备上需要显示一张图片时应该加载那个目录下的图片呢？这就需要根据Configuration中记录的屏幕密度做判断了。方法中最后调用了`mAssets.setConfiguration()`，这是一个jni方法


**13. android_content_AssetManager_setConfiguration()**

```C++
//源码位置frameworks/base/core/jni/android_util_AssetManager.cpp
static void android_content_AssetManager_setConfiguration(JNIEnv* env, jobject clazz,
                                                          jint mcc, jint mnc,
                                                          jstring locale, jint orientation,
                                                          ...)
{
    //得到C++层AssetManager指针
    AssetManager* am = assetManagerForJavaObject(env, clazz);
    //创建一个ResTable_config对象，用来描述设备的当前配置信息
    ResTable_config config;  
    memset(&config, 0, sizeof(config));

    const char* locale8 = locale != NULL ? env->GetStringUTFChars(locale, NULL) : NULL;

    static const jint kScreenLayoutRoundMask = 0x300;
    static const jint kScreenLayoutRoundShift = 8;

    config.mcc = (uint16_t)mcc;
    config.mnc = (uint16_t)mnc;
    config.orientation = (uint8_t)orientation;
    config.touchscreen = (uint8_t)touchscreen;
    config.density = (uint16_t)density;
    //...
    //★最后调用AssetManager的setConfiguration()      
    am->setConfiguration(config, locale8);

    if (locale != NULL) env->ReleaseStringUTFChars(locale, locale8);
}
```

**14. AssetManager::setConfiguration()**

```C++
//源码位置frameworks/base/libs/androidfw/AssetManager.cpp
void AssetManager::setConfiguration(const ResTable_config& config, const char* locale)
{
    AutoMutex _l(mLock);
    *mConfig = config;
    if (locale) {
        //如果参数local(设备的国家、地区和语言信息)的值不等于NULL，设置给AssetManager的mLocale成员变量，然后调用updateResourceParamsLocked更新资源表中的设备配置信息
        setLocaleLocked(locale);
    } else if (config.language[0] != 0) {
        //如果local为null，但是参数config中保存的语言信息不为空，则跟上面一样设置AssetManager
        char spec[RESTABLE_MAX_LOCALE_LEN];
        config.getBcp47Locale(spec);
        setLocaleLocked(spec);
    } else {
        //如果参数中没有设备的国家、地区和语言信息，则说明这些信息不需要更新，调用updateResourceParamsLocked更新资源表中的设备配置信息
        updateResourceParamsLocked();
    }
}
```

**15. AssetManager::updateResourceParamsLocked()**

```C++
//mutable ResTable* mResources;
void AssetManager::updateResourceParamsLocked() const
{
    ATRACE_CALL();
    //成员变量mResources是一个ResTable类型的指针，ResTable描述的是一个资源索引表，对应apk包中的resources.arsc二进制解析，其实就是将arsc文件解析后的对象表示
    ResTable* res = mResources;
    //...
    if (mLocale) {
        mConfig->setBcp47Locale(mLocale);
    } else {
        mConfig->clearLocale();
    }
    //调用ResTable的setParameters设置配置参数，关于ResTable的结构这里就不再深入，后面有机会再说
    res->setParameters(mConfig);
}
```

## 总结

源码的跟踪可粗可细，我们要有目标的去跟踪，而不是死扣代码细节，只需要跟着主线，关键部位的代码解析即可。上面我们跟踪AssetManager创建过程主要弄清楚以下几点：

- Context提供的功能都是由ContextImpl实现的，每个ContextImpl对象中都维护了Resources类型成员变量mResources
- 真正实现资源操作的是AssetManager类，这个类的实例在不同android版本中有所不同。在为ContextImpl创建Resources是会为它创建一个AssetManager的实例mAssets

> 在Android7.0(API 24)之前，Resources中直接持有AssetManager的实例mAssets；7.0及以后版本中Resources仅作为一个资源包装类，它不再维护mAssets，而是持有`ResourcesImpl`的成员变量mResourcesImpl，ResourcesImpl持有mAssets，ResourcesImpl是一个隐藏类

- AssetManager对应有Java层和JNI C++层，Java层的AssetManager最终都是通过调用C++层AssetManager实现的，Java层AssetManager的`long mObject`中保存了C++层AssetManager对象的地址，从而保持联系

- C++层AssetManager对象维护了一个ResTable类型的对象mResources，它是对apk包中的resources.arsc二进制解析封装，资源的获取都是通过这个表来进行搜素的

---------------------

# ResTable简要分析

关键源码位置：
- frameworks/base/libs/androidfw/include/androidfw/ResourceTypes.h
- frameworks/base/libs/androidfw/ResourceTypes.cpp

0x7f0b0000
资源ID = Package ID + Type ID + Entry ID
   ID
0x7f0b0000  ic_launcher res/mipmap-mdpi-v4/ic_launcher.png  res/mipmap-xhdpi-v4/ic_launcher.png


1.ResTable(一个ResTable可以加载多个arsc，比如系统资源和应用程序资源)

## 相关源码解析

ResTable是一个资源索引表，它主要是通过加载resources.arsc文件并解析后用ResTable对象表示，C++层AssetManager中有一个ResTable类型成员变量mResources，通过它可以根据资源ID获取到对应的资源值。下面我们看看ResTable中是什么样的结构，里面包含那些内容分别代表什么。

```C++
//源码位置 frameworks/base/libs/androidfw/include/androidfw/ResourceTypes.h

/*
 * ResTable类定义在ResourceTypes.h文件中，它有一系列的结构体成员变量
 * AssetManager.addAssetPath(path)方法可以添加一个apk路径，然后加载其resources.arsc文件解析成一个ResTable对象；
 * AssetManager中有一个ResTable类型成员变量mResources，它是所有资源表的集合，addAssetPath()方法解析的ResTable对象会通过调用mResources—>add(sharedRes)添加到总索引表中
 */
class ResTable  
{ 
  private:
    struct Header;
    struct Type;
    struct Entry;
    struct Package;
    struct PackageGroup;
    typedef Vector<Type*> TypeList;
}

//源码位置 frameworks/base/libs/androidfw/ResourceTypes.cpp

struct ResTable::Header
{
    explicit Header(ResTable* _owner) : owner(_owner), ownedData(NULL), header(NULL),
        resourceIDMap(NULL), resourceIDMapSize(0) { }

    ~Header()
    {
        free(resourceIDMap);
    }

    const ResTable* const           owner;
    void*                           ownedData;
    const ResTable_header*          header;
    size_t                          size;
    const uint8_t*                  dataEnd;
    size_t                          index;
    int32_t                         cookie;

    ResStringPool                   values;
    uint32_t*                       resourceIDMap;
    size_t                          resourceIDMapSize;
};

// A group of objects describing a particular resource package.
// The first in 'package' is always the root object (from the resource
// table that defined the package); the ones after are skins on top of it.
struct ResTable::PackageGroup
{
    PackageGroup(
            ResTable* _owner, const String16& _name, uint32_t _id,
            bool appAsLib, bool _isSystemAsset)
        : owner(_owner)
        , name(_name)
        , id(_id)
        , largestTypeId(0)
        , dynamicRefTable(static_cast<uint8_t>(_id), appAsLib)
        , isSystemAsset(_isSystemAsset)
    { }
}
struct ResTable::Package
{
    Package(ResTable* _owner, const Header* _header, const ResTable_package* _package)
        : owner(_owner), header(_header), package(_package), typeIdOffset(0) {
        if (dtohs(package->header.headerSize) == sizeof(*package)) {
            // The package structure is the same size as the definition.
            // This means it contains the typeIdOffset field.
            typeIdOffset = package->typeIdOffset;
        }
    }

    const ResTable* const           owner;
    const Header* const             header;
    const ResTable_package* const   package;

    ResStringPool                   typeStrings;
    ResStringPool                   keyStrings;

    size_t                          typeIdOffset;
};


struct ResTable::Entry {
    ResTable_config config;
    const ResTable_entry* entry;
    const ResTable_type* type;
    uint32_t specFlags;
    const Package* package;

    StringPoolRef typeStr;
    StringPoolRef keyStr;
};

struct ResTable::Type
{
    Type(const Header* _header, const Package* _package, size_t count)
        : header(_header), package(_package), entryCount(count),
          typeSpec(NULL), typeSpecFlags(NULL) { }
    const Header* const             header;
    const Package* const            package;
    const size_t                    entryCount;
    const ResTable_typeSpec*        typeSpec;
    const uint32_t*                 typeSpecFlags;
    IdmapEntries                    idmapEntries;
    Vector<const ResTable_type*>    configs;
};




```



# AssetManager获取资源分析

```Java
setContentView(R.layout.activity_main);  //resource.getLayout()
getResources().getLayout(R.layout.activity_main);  //mAssets.openXmlBlockAsset(assetCookie, file);
getResources().getString(R.string.app_name);  //mResourcesImpl.getAssets().getResourceText(id);
getAssets().open("assetFileName");
getAssets().list("path");
```

我们通常使用Resources对象根据资源id获取对应的资源，使用AssetManager获取assets目录下的文件。不管获取什么资源，最终都是通过AssetManager对象获取的。接下来我们通过加载布局layout文件跟踪整个资源获取过程。

## 源码解析

**1. setContentView(R.layout.activity_main)**

```Java
//为activity设置布局id
setContentView(R.layout.activity_main);

//android.app.Activity
private Window mWindow;   //mWindow是PhoneWindow对象，用于管理顶层窗口的外观和行为
public void setContentView(@LayoutRes int layoutResID) {
    getWindow().setContentView(layoutResID);  //2. ★调用PhoneWindow的方法
    initWindowDecorActionBar();
}
```

**2. PhoneWindow.setContentView()**

```Java
//com.android.internal.policy.PhoneWindow
ViewGroup mContentParent;
private LayoutInflater mLayoutInflater;
@Override
public void setContentView(int layoutResID) {
    if (mContentParent == null) {
        installDecor();
    } else if (!hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
        mContentParent.removeAllViews();
    }
    if (hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
        final Scene newScene = Scene.getSceneForLayout(mContentParent, layoutResID, getContext());
        transitionTo(newScene);
    } else {
        //★调用布局填充器的inflate方法将对应布局id的layout填充到activity根视图中
        mLayoutInflater.inflate(layoutResID, mContentParent);
    }
    //...
}
```

**3. LayoutInflater.inflate()**

```Java
//android.view.LayoutInflater
public View inflate(@LayoutRes int resource, @Nullable ViewGroup root, boolean attachToRoot) {
    //getContext()拿到的就是Activity对象
    final Resources res = getContext().getResources();
    //4. ★利用Resources通过layoutID得到一个xml解析器
    final XmlResourceParser parser = res.getLayout(resource);
    try {
        return inflate(parser, root, attachToRoot);
    }
}
```


**4. Resources.getLayout()**

```Java
//android.content.res.Resources
public XmlResourceParser getLayout(@LayoutRes int id) throws NotFoundException {
    return loadXmlResourceParser(id, "layout");
}
XmlResourceParser loadXmlResourceParser(@AnyRes int id, @NonNull String type)
        throws NotFoundException {
    /*
     * TypedValue中可以存放资源类型，资源值等信息，不同类型资源的资源值不同，比如strings.xml中定义的类型对应资源值就是字符串本身，而layout对应的资源值是xml文件名称
     * 这里仅仅是获取一个资源类型数据值的容器，下面getValue()获取id对应的资源相关信息后存储在这个容器中
     */
    final TypedValue value = obtainTempTypedValue();
    try {
        final ResourcesImpl impl = mResourcesImpl;
        //★ 获得该资源ID所对应的资源信息（资源类型，资源值等）存储在TypedValue中
        impl.getValue(id, value, true);
        if (value.type == TypedValue.TYPE_STRING) {
            //★ 
            return impl.loadXmlResourceParser(value.string.toString(), id,
                    value.assetCookie, type);
        }
        throw new NotFoundException("Resource ID #0x" + Integer.toHexString(id)
                + " type #0x" + Integer.toHexString(value.type) + " is not valid");
    } finally {
        releaseTempTypedValue(value);
    }
}
```

> loadXmlResourceParser()方法的作用是根据id返回一个layout布局的xml解析器，首先调用`getValue()`根据id获取layout对应的资源值(布局文件名称)，然后调用`loadXmlResourceParser()`加载布局文件返回xml解析器，下面我们对这两个方法分别追踪

**5. ResourcesImpl.getValue()**

```Java
void getValue(@AnyRes int id, TypedValue outValue, boolean resolveRefs) throws NotFoundException {
    //★通过成员变量mAssets（AssetManager对象）获取id对应资源值
    boolean found = mAssets.getResourceValue(id, 0, outValue, resolveRefs);
    if (found) 
        return;
    throw new NotFoundException("Resource ID #0x" + Integer.toHexString(id));
}
```

**6. AssetManager.getResourceValue()**

```Java
final boolean getResourceValue(@AnyRes int resId, int densityDpi, @NonNull TypedValue outValue, boolean resolveRefs) {
      synchronized (this) {
          //★加载资源id对应的资源，将资源数据存放到outValue中，该方法是jni方法
          final int block = loadResourceValue(resId, (short) densityDpi, outValue, resolveRefs);
          if (block < 0) 
              return false;
          outValue.changingConfigurations = ActivityInfo.activityInfoConfigNativeToJava(
                  outValue.changingConfigurations);

          if (outValue.type == TypedValue.TYPE_STRING) {
            //StringBlock数组元素描述的是当前应用程序使用的每一个资源索引表的资源项值字符串资源池
              outValue.string = mStringBlocks[block].get(outValue.data);
          }
          return true;
      }
}

private native final int loadResourceValue(int ident, short density, TypedValue outValue, boolean resolve);
```

**7. android_content_AssetManager_loadResourceValue()**

```Java
//frameworks/base/core/jni/android_util_AssetManager.cpp
static jint android_content_AssetManager_loadResourceValue(JNIEnv* env, jobject clazz,jint ident,...){
    //...得到C++层AssetManager对象指针
    AssetManager* am = assetManagerForJavaObject(env, clazz);
    //...★调用其getResources()方法获得ResTable对象，ResTable描述的是一个资源表（通过加载解析resources.arsc）
    const ResTable& res(am->getResources());

    Res_value value;         //资源项值
    ResTable_config config;  //配置信息
    uint32_t typeSpecFlags;
    //★ 通过ResTable获得资源id对应的资源项值和配置信息
    ssize_t block = res.getResource(ident, &value, false, density, &typeSpecFlags, &config);
    //...
    uint32_t ref = ident;
    if (resolve) {
        //★ 解析前面所得到的资源项值
        block = res.resolveReference(&value, block, &ref, &typeSpecFlags, &config);
        //...
    }
    if (block >= 0) {
        //将上述得到的资源项值及其配置信息拷贝到参数outValue所描述的一个Java层的TypedValue对象中去
        return copyValue(env, outValue, &res, value, ref, block, typeSpecFlags, &config);
    }

    return static_cast<jint>(block);
}
```
> 该方法首先调用getResources()获得Restable资源索引表，然后调用索引表的resolveReference()方法获取对应id资源值，最后将资源值信息拷贝到Java层的TypedValue对象中。下面我们先看看资源索引表的获取

**8. AssetManager::getResTable()**

```Java
const ResTable* AssetManager::getResTable(bool required) const
{
    //获取AssetManager对象的成员变量mResources，如果不为空则直接返回
    ResTable* rt = mResources;
    if (rt)   //第一次根据id获取资源mResources是null，
        return rt;
    //互斥锁
    AutoMutex _l(mLock);
    //再次判断避免其它线程已经解析初始化了mResources
    if (mResources != NULL) 
        return mResources;
    if (required) {
        LOG_FATAL_IF(mAssetPaths.size() == 0, "No assets added to AssetManager");
    }
    //★★★★★★创建ResTable对象为mResources总索引表赋值
    mResources = new ResTable();
    updateResourceParamsLocked();

    bool onlyEmptyResources = true;
    const size_t N = mAssetPaths.size();
    //迭代所有资产包，从每个包中收集资源，为什么会有多个资源包呢？因为默认会有一个系统资源包，还有应用自身的资源包，从AssetManager创建过程我们可以知道
    for (size_t i=0; i<N; i++) {
        //★appendPathToResTable()方法加载指定路径下的resources.arsc文件
        bool empty = appendPathToResTable(mAssetPaths.itemAt(i));
        onlyEmptyResources = onlyEmptyResources && empty;
    }
    //...
    return mResources;
}
```

> getResTable()方法首先判断mResources有么有被加载过，如果不为空则说明已经加载直接返回，否则遍历资源路径数组，挨个加载resources.arsc文件

**9. AssetManager.appendPathToResTable()**

```Java
bool AssetManager::appendPathToResTable(const asset_path& ap, bool appAsLib) const {
    // 跳过与系统覆盖相对应的路径
    if (ap.isSystemOverlay) {
        return true;
    }

    Asset* ass = NULL;
    ResTable* sharedRes = NULL;
    bool shared = true;
    bool onlyEmptyResources = true;
    ATRACE_NAME(ap.path.string());
    Asset* idmap = openIdmapLocked(ap);
    size_t nextEntryIdx = mResources->getTableCount();
    if (ap.type != kFileTypeDirectory) {
        if (nextEntryIdx == 0) {
            //第一项通常是framwork框架资源，我们希望避免每次解析它
            sharedRes = const_cast<AssetManager*>(this)->
                mZipSet.getZipResourceTable(ap.path);
            if (sharedRes != NULL) {
                //跳过预装的系统覆盖包的数量
                nextEntryIdx = sharedRes->getTableCount();
            }
        }
        if (sharedRes == NULL) {
           //调用成员变量mZipSet的getZipResourceTableAsset()方法获得一个对应的Asset对象
            ass = const_cast<AssetManager*>(this)->
                mZipSet.getZipResourceTableAsset(ap.path);
            if (ass == NULL) {
                ALOGV("loading resource table %s\n", ap.path.string());
                //如果resources.arsc没有提取出来，则提取resources.arsc
                ass = const_cast<AssetManager*>(this)->
                    openNonAssetInPathLocked("resources.arsc",
                                             Asset::ACCESS_BUFFER,
                                             ap);
                if (ass != NULL && ass != kExcludedAsset) {
                    ass = const_cast<AssetManager*>(this)->
                        mZipSet.setZipResourceTableAsset(ap.path, ass);
                }
            }
            if (nextEntryIdx == 0 && ass != NULL) {
                sharedRes = new ResTable();
                //调用ResTable的add方法将提取resources.arsc得到的Asset对象添加到资源表中
                sharedRes->add(ass, idmap, nextEntryIdx + 1, false);
#ifdef __ANDROID__
                const char* data = getenv("ANDROID_DATA");
                LOG_ALWAYS_FATAL_IF(data == NULL, "ANDROID_DATA not set");
                String8 overlaysListPath(data);
                overlaysListPath.appendPath(kResourceCache);
                overlaysListPath.appendPath("overlays.list");
                addSystemOverlays(overlaysListPath.string(), ap.path, sharedRes, nextEntryIdx);
#endif
                sharedRes = const_cast<AssetManager*>(this)->
                    mZipSet.setZipResourceTable(ap.path, sharedRes);
            }
        }
    } else {
        ALOGV("loading resource table %s\n", ap.path.string());
        ass = const_cast<AssetManager*>(this)->
            openNonAssetInPathLocked("resources.arsc",
                                     Asset::ACCESS_BUFFER,
                                     ap);
        shared = false;
    }

    if ((ass != NULL || sharedRes != NULL) && ass != kExcludedAsset) {
        ALOGV("Installing resource asset %p in to table %p\n", ass, mResources);
        //将刚刚加载resources.arsc得到的数据添加到总的资源索引表中
        if (sharedRes != NULL) {
            ALOGV("Copying existing resources for %s", ap.path.string());
            mResources->add(sharedRes, ap.isSystemAsset);
        } else {
            ALOGV("Parsing resources for %s", ap.path.string());
            mResources->add(ass, idmap, nextEntryIdx + 1, !shared, appAsLib, ap.isSystemAsset);
        }
        onlyEmptyResources = false;

        if (!shared) {
            delete ass;
        }
    } else {
        mResources->addEmpty(nextEntryIdx + 1);
    }

    if (idmap != NULL) {
        delete idmap;
    }
    return onlyEmptyResources;
}
```

> 这个方法主要逻辑是加载解析一个resources.arsc文件，将解析得到的数据用C++的一个对象表示，然后添加到AssetManager成员变量mResources总索引表中，关于resources.arsc文件到底是怎样加载的，可继续跟踪`frameworks/base/libs/androidfw/Asset.cpp`中的相关方法。下面我们回到第7步第二个跟踪点`res.getResource()`获得资源id对应的资源项值和配置信息

**10. ResTable.getResource()**

```Java
//源码位置 frameworks/base/libs/androidfw/ResourceTypes.cpp
ssize_t ResTable::getResource(uint32_t resID, Res_value* outValue, bool mayBeBag, uint16_t density, uint32_t* outSpecFlags, ResTable_config* outConfig) const
{
    //0x7f0b0000 资源ID = Package ID + Type ID + Entry ID
    //根据资源id分别获得它的Pakcage ID、Type ID、Entry ID，保存在p、t、e中
    const ssize_t p = getResourcePackageIndex(resID);
    const int t = Res_GETTYPE(resID);
    const int e = Res_GETENTRY(resID);
    //...
    //根据Package ID获得对应的PackageGroup
    const PackageGroup* const grp = mPackageGroups[p];
    //...

    // Allow overriding density
    ResTable_config desiredConfig = mParams;
    if (density > 0) {
        desiredConfig.density = density;
    }

    Entry entry;
    //获取具体资源的封装对象
    status_t err = getEntry(grp, t, e, &desiredConfig, &entry);
    //...
    const Res_value* value = reinterpret_cast<const Res_value*>(
            reinterpret_cast<const uint8_t*>(entry.entry) + entry.entry->size);

    outValue->size = dtohs(value->size);
    outValue->res0 = value->res0;
    outValue->dataType = value->dataType;
    outValue->data = dtohl(value->data);

    // The reference may be pointing to a resource in a shared library. These
    // references have build-time generated package IDs. These ids may not match
    // the actual package IDs of the corresponding packages in this ResTable.
    // We need to fix the package ID based on a mapping.
    if (grp->dynamicRefTable.lookupResourceValue(outValue) != NO_ERROR) {
        ALOGW("Failed to resolve referenced package: 0x%08x", outValue->data);
        return BAD_VALUE;
    }

    if (kDebugTableNoisy) {
        size_t len;
        printf("Found value: pkg=%zu, type=%d, str=%s, int=%d\n",
                entry.package->header->index,
                outValue->dataType,
                outValue->dataType == Res_value::TYPE_STRING ?
                    String8(entry.package->header->values.stringAt(outValue->data, &len)).string() :
                    "",
                outValue->data);
    }

    if (outSpecFlags != NULL) {
        *outSpecFlags = entry.specFlags;
    }

    if (outConfig != NULL) {
        *outConfig = entry.config;
    }

    return entry.package->header->index;
}
```

**11. ResTable.getEntry()**

```Java
status_t ResTable::getEntry(
        const PackageGroup* packageGroup, int typeIndex, int entryIndex,
        const ResTable_config* config,
        Entry* outEntry) const
{
    const TypeList& typeList = packageGroup->types[typeIndex];
    if (typeList.isEmpty()) {
        ALOGV("Skipping entry type index 0x%02x because type is NULL!\n", typeIndex);
        return BAD_TYPE;
    }

    const ResTable_type* bestType = NULL;
    uint32_t bestOffset = ResTable_type::NO_ENTRY;
    const Package* bestPackage = NULL;
    uint32_t specFlags = 0;
    uint8_t actualTypeIndex = typeIndex;
    //
    ResTable_config bestConfig;
    memset(&bestConfig, 0, sizeof(bestConfig));

    // Iterate over the Types of each package.
    const size_t typeCount = typeList.size();
    for (size_t i = 0; i < typeCount; i++) {
        const Type* const typeSpec = typeList[i];

        int realEntryIndex = entryIndex;
        int realTypeIndex = typeIndex;
        bool currentTypeIsOverlay = false;

        // Runtime overlay packages provide a mapping of app resource
        // ID to package resource ID.
        if (typeSpec->idmapEntries.hasEntries()) {
            uint16_t overlayEntryIndex;
            if (typeSpec->idmapEntries.lookup(entryIndex, &overlayEntryIndex) != NO_ERROR) {
                // No such mapping exists
                continue;
            }
            realEntryIndex = overlayEntryIndex;
            realTypeIndex = typeSpec->idmapEntries.overlayTypeId() - 1;
            currentTypeIsOverlay = true;
        }

        // Check that the entry idx is within range of the declared entry count (ResTable_typeSpec).
        // Particular types (ResTable_type) may be encoded with sparse entries, and so their
        // entryCount do not need to match.
        if (static_cast<size_t>(realEntryIndex) >= typeSpec->entryCount) {
            ALOGW("For resource 0x%08x, entry index(%d) is beyond type entryCount(%d)",
                    Res_MAKEID(packageGroup->id - 1, typeIndex, entryIndex),
                    entryIndex, static_cast<int>(typeSpec->entryCount));
            // We should normally abort here, but some legacy apps declare
            // resources in the 'android' package (old bug in AAPT).
            continue;
        }

        // Aggregate all the flags for each package that defines this entry.
        if (typeSpec->typeSpecFlags != NULL) {
            specFlags |= dtohl(typeSpec->typeSpecFlags[realEntryIndex]);
        } else {
            specFlags = -1;
        }

        const Vector<const ResTable_type*>* candidateConfigs = &typeSpec->configs;

        std::shared_ptr<Vector<const ResTable_type*>> filteredConfigs;
        if (config && memcmp(&mParams, config, sizeof(mParams)) == 0) {
            // Grab the lock first so we can safely get the current filtered list.
            AutoMutex _lock(mFilteredConfigLock);

            // This configuration is equal to the one we have previously cached for,
            // so use the filtered configs.

            const TypeCacheEntry& cacheEntry = packageGroup->typeCacheEntries[typeIndex];
            if (i < cacheEntry.filteredConfigs.size()) {
                if (cacheEntry.filteredConfigs[i]) {
                    // Grab a reference to the shared_ptr so it doesn't get destroyed while
                    // going through this list.
                    filteredConfigs = cacheEntry.filteredConfigs[i];

                    // Use this filtered list.
                    candidateConfigs = filteredConfigs.get();
                }
            }
        }

        const size_t numConfigs = candidateConfigs->size();
        for (size_t c = 0; c < numConfigs; c++) {
            const ResTable_type* const thisType = candidateConfigs->itemAt(c);
            if (thisType == NULL) {
                continue;
            }

            ResTable_config thisConfig;
            thisConfig.copyFromDtoH(thisType->config);

            // Check to make sure this one is valid for the current parameters.
            if (config != NULL && !thisConfig.match(*config)) {
                continue;
            }

            const uint32_t* const eindex = reinterpret_cast<const uint32_t*>(
                    reinterpret_cast<const uint8_t*>(thisType) + dtohs(thisType->header.headerSize));

            uint32_t thisOffset;

            // Check if there is the desired entry in this type.
            if (thisType->flags & ResTable_type::FLAG_SPARSE) {
                // This is encoded as a sparse map, so perform a binary search.
                const ResTable_sparseTypeEntry* sparseIndices =
                        reinterpret_cast<const ResTable_sparseTypeEntry*>(eindex);
                const ResTable_sparseTypeEntry* result = std::lower_bound(
                        sparseIndices, sparseIndices + dtohl(thisType->entryCount), realEntryIndex,
                        keyCompare);
                if (result == sparseIndices + dtohl(thisType->entryCount)
                        || dtohs(result->idx) != realEntryIndex) {
                    // No entry found.
                    continue;
                }

                // Extract the offset from the entry. Each offset must be a multiple of 4
                // so we store it as the real offset divided by 4.
                thisOffset = dtohs(result->offset) * 4u;
            } else {
                if (static_cast<uint32_t>(realEntryIndex) >= dtohl(thisType->entryCount)) {
                    // Entry does not exist.
                    continue;
                }

                thisOffset = dtohl(eindex[realEntryIndex]);
            }

            if (thisOffset == ResTable_type::NO_ENTRY) {
                // There is no entry for this index and configuration.
                continue;
            }

            if (bestType != NULL) {
                // Check if this one is less specific than the last found.  If so,
                // we will skip it.  We check starting with things we most care
                // about to those we least care about.
                if (!thisConfig.isBetterThan(bestConfig, config)) {
                    if (!currentTypeIsOverlay || thisConfig.compare(bestConfig) != 0) {
                        continue;
                    }
                }
            }

            bestType = thisType;
            bestOffset = thisOffset;
            bestConfig = thisConfig;
            bestPackage = typeSpec->package;
            actualTypeIndex = realTypeIndex;

            // If no config was specified, any type will do, so skip
            if (config == NULL) {
                break;
            }
        }
    }

    if (bestType == NULL) {
        return BAD_INDEX;
    }

    bestOffset += dtohl(bestType->entriesStart);

    if (bestOffset > (dtohl(bestType->header.size)-sizeof(ResTable_entry))) {
        ALOGW("ResTable_entry at 0x%x is beyond type chunk data 0x%x",
                bestOffset, dtohl(bestType->header.size));
        return BAD_TYPE;
    }
    if ((bestOffset & 0x3) != 0) {
        ALOGW("ResTable_entry at 0x%x is not on an integer boundary", bestOffset);
        return BAD_TYPE;
    }

    const ResTable_entry* const entry = reinterpret_cast<const ResTable_entry*>(
            reinterpret_cast<const uint8_t*>(bestType) + bestOffset);
    if (dtohs(entry->size) < sizeof(*entry)) {
        ALOGW("ResTable_entry size 0x%x is too small", dtohs(entry->size));
        return BAD_TYPE;
    }

    if (outEntry != NULL) {
        outEntry->entry = entry;
        outEntry->config = bestConfig;
        outEntry->type = bestType;
        outEntry->specFlags = specFlags;
        outEntry->package = bestPackage;
        outEntry->typeStr = StringPoolRef(&bestPackage->typeStrings, actualTypeIndex - bestPackage->typeIdOffset);
        outEntry->keyStr = StringPoolRef(&bestPackage->keyStrings, dtohl(entry->key.index));
    }
    return NO_ERROR;
}
```


**7. AssetManager.getResourceValue()**

```Java

```


**7. AssetManager.getResourceValue()**

```Java

```


**7. AssetManager.getResourceValue()**

```Java

```


**7. AssetManager.getResourceValue()**

```Java

```


**7. AssetManager.getResourceValue()**

```Java

```


**7. AssetManager.getResourceValue()**

```Java

```


**7. AssetManager.getResourceValue()**

```Java

```

































