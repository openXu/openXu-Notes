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

下面以Activity启动过程中，为ContextImpl创建AssetManager对象为例，源码追踪从ActivityThread创建Activity开始，直到AssetManager对象创建完（注释中带★的表示下一步要继续追踪）：

1. android.app.ActivityThread.performLaunchActivity()

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

2. android.app.ContextImpl.createActivityContext()

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

3. android.app.ResourcesManager.createBaseActivityResources()

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

4. android.app.ResourcesManager.getOrCreateResources()

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

5. android.app.ResourcesManager.createResourcesImpl()

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

6. android.app.ResourcesManager.createAssetManager(key)

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

7. android.content.res.AssetManager

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

8. android_content_AssetManager_init()

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

9. AssetManager::addDefaultAssets()

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

10. AssetManager::addAssetPath()

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
        //☆添加资源到资源表，将资源文件下的资源添加到资源表中
        appendPathToResTable(ap, appAsLib);
    }

    return true;
}
```

> addAssetPath()方法就是将path描述的资源路径保存到mAssetPaths动态数组中，到此C++层的AssetManager对象就算创建完成，上面第6步又调用`assets.addAssetPath(key.mResDir)`将应用程序的资源路径也添加进去。addAssetPath()方法最后会调用`appendPathToResTable()`去加载路径下的resources.arsc并解析，将资源索引添加到ResTable中保存。其实到这里AssetManager创建过程算是跟踪完了，但是关于资源还有一个重要的事情，就是设置设备信息，感兴趣可以继续往下看

11. android_content_AssetManager_addAssetPath()

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

12. android.content.res.ResourcesImpl.updateConfiguration()

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


13. android_content_AssetManager_setConfiguration()

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

14. AssetManager::setConfiguration()

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

15. AssetManager::updateResourceParamsLocked()

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


---------------------


# AssetManager获取资源分析

```Java
setContentView(R.layout.activity_main);  //resource.getLayout()
getResources().getLayout(R.layout.activity_main);  //mAssets.openXmlBlockAsset(assetCookie, file);
getResources().getString(R.string.app_name);  //mResourcesImpl.getAssets().getResourceText(id);
getAssets().open("assetFileName");
getAssets().list("path");
```

我们通常使用Resources对象根据资源id获取对应的资源，使用AssetManager获取assets目录下的文件。不管获取什么资源，最终都是通过AssetManager对象获取的。接下来我们通过加载布局layout文件跟踪整个资源获取过程。

```Java
//为activity设置布局id
setContentView(R.layout.activity_main);

//android.app.Activity
private Window mWindow;   //PhoneWindow对象
public void setContentView(@LayoutRes int layoutResID) {
    getWindow().setContentView(layoutResID);  //★
    initWindowDecorActionBar();
}

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
        final Scene newScene = Scene.getSceneForLayout(mContentParent, layoutResID,
                getContext());
        transitionTo(newScene);
    } else {
        //★调用布局填充器的inflate方法将对应布局id的layout填充到activity根视图中
        mLayoutInflater.inflate(layoutResID, mContentParent);
    }
    //...
}

//android.view.LayoutInflater
public View inflate(@LayoutRes int resource, @Nullable ViewGroup root) {
    return inflate(resource, root, root != null);
}
public View inflate(@LayoutRes int resource, @Nullable ViewGroup root, boolean attachToRoot) {
    final Resources res = getContext().getResources();
    //★利用Resources通过layoutID得到一个xml解析器
    final XmlResourceParser parser = res.getLayout(resource);
    try {
        return inflate(parser, root, attachToRoot);
    } finally {
        parser.close();
    }
}

//android.content.res.Resources
public XmlResourceParser getLayout(@LayoutRes int id) throws NotFoundException {
    return loadXmlResourceParser(id, "layout");
}
XmlResourceParser loadXmlResourceParser(@AnyRes int id, @NonNull String type)
        throws NotFoundException {
    final TypedValue value = obtainTempTypedValue();
    try {
        final ResourcesImpl impl = mResourcesImpl;
        impl.getValue(id, value, true);
        if (value.type == TypedValue.TYPE_STRING) {
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






































