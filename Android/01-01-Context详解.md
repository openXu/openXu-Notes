> 版权声明：本文为openXu原创文章[【openXu的博客】](http://blog.csdn.net/xmxkf)，未经博主允许不得以任何形式转载

@[TOC](目录)

Android程序和一个Java程序最大的区别是Android的系统组件不能像普通Java程序通过new创建对象，而是通过系统提供的特殊方式，这是因为new出来的组件没有携带上下文环境Context。为什么要设计Context？其实说起来跟Android的沙盒机制有一定关系，一个Android程序只能访问自己的东西，比如资源文件、数据库及文件存储、开启页面等等，而这些功能都是由Context提供的，Context由系统创建而不能new，所以上面说的资源文件路径、存储路径、页面等就只能限制在自己应用程序里面了，当然可以通过暴力反射修改这些内容，比如下面修改SharedPreferences存储路径:

```Java
private static SharedPreferences getSharedPreferences(boolean def, Context context, String fileName) {
    if(!def){
        try {
            // 获取ContextWrapper对象中的mBase对象，mBase是ContextImpl的实例
            Field field = ContextWrapper.class.getDeclaredField("mBase");
            field.setAccessible(true);
            // 获取mBase变量
            Object mBase = field.get(context);
            // 获取ContextImpl.mPreferencesDir变量，该File对象就是数据文件的保存路径
            field = mBase.getClass().getDeclaredField("mPreferencesDir");
            field.setAccessible(true);
            // 创建自定义路径(可以保存在SD卡，这样卸载应用之后重新安装也可以读取配置)
            File file = new File(FILE_PATH);
            // 修改mPreferencesDir变量的值
            field.set(mBase, file);
            // 返回修改路径以后的SharedPreferences :%FILE_PATH%/%fileName%.xml
            return context.getSharedPreferences(fileName, Activity.MODE_PRIVATE);
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
    // 返回默认路径下的 SharedPreferences : /data/data/%package_name%/shared_prefs/%fileName%.xml
    return context.getSharedPreferences(fileName, Context.MODE_PRIVATE);
}
```

## 1. Context的作用

Context有什么用呢？那就直接看看它有哪些方法，下面铺出部分源码：

```Java
public abstract class Context {
	public abstract Context getApplicationContext();
	/**1. 操作资源文件*/
	public abstract AssetManager getAssets();
    public abstract Resources getResources();
    /**2. 获取应用程序信息*/
    public abstract PackageManager getPackageManager();
    public abstract ApplicationInfo getApplicationInfo();
    public abstract String getPackageName();
    /**3. 获取主线程Looper*/
    public abstract Looper getMainLooper();
    /**4. SharedPreferences存储、Sqlite数据库相关*/
    public abstract SQLiteDatabase openOrCreateDatabase(String name,int mode, CursorFactory factory);
    public abstract boolean deleteDatabase(String name);
    public abstract SharedPreferences getSharedPreferences(String name, int mode);
    /**5. 操作文件系统*/
    public abstract File getFilesDir();
    public abstract File getExternalFilesDir(@Nullable String type);
    public abstract File getCacheDir();
    /**6. 检查权限*/
    public abstract int checkPermission(String permission, int pid, int uid);
	/**7. 开启Activity*/
    public abstract void startActivity(Intent intent);
    /**8. 发送、注册广播*/
    public abstract void sendBroadcast(Intent intent);
    public abstract Intent registerReceiver(BroadcastReceiver receiver, IntentFilter filter);
    public abstract void unregisterReceiver(BroadcastReceiver receiver);
    /**9. 启动、停止Service*/
    public abstract ComponentName startService(Intent service);
    public abstract boolean stopService(Intent name);

    ...
}
```

从源码可看出我们在Android开发中基本都是在围着Context转，比如跳转ctivity、启动Service、发送广播、操作资源文件、操作数据存储等功能，都是由Context提供的。

## 2. Context类的实现

上下文被封装成名为`Context`的抽象类，它的继承关系如下：

```Java
               Context
                  |
        --------------------
        |                  |
   ContextImpl<-------ContextWrapper(拥有ContextImpl的实例对象mBase，代其实现Context功能)
                           |
               --------------------------
               |           |            |
         Application    Service   ContextThemeWrapper
                                        |
                                    Activity
```

Context有两个直系子类ContextImpl、ContextWrapper，故名思意ContextImpl是上下文功能实现类，ContextWrapper是上下文功能的封装类。ContextWrapper又有三个直接子类，其中ContextThemeWrapper是带主题的上下文封装类，它的直接子类就是Activity。由此可见Context一共有三种常用类型，分别是Application、Service、Activity。

Context中都是抽象方法，那谁来实现他们呢？总不会是Activity、Service、Applicaiton这些末级子类吧？我们先看看它的子类`ContextWrapper`：

```Java
public class ContextWrapper extends Context {
	/**ContextWrapper对Context的实现都委托给mBase对象的*/
    Context mBase;

    public ContextWrapper(Context base) {
        mBase = base;
    }
    protected void attachBaseContext(Context base) {
        if (mBase != null) {
            throw new IllegalStateException("Base context already set");
        }
        mBase = base;
    }
    public Context getBaseContext() {
        return mBase;
    }
    @Override
    public Resources getResources(){
        return mBase.getResources();
    }
    @Override
    public Context getApplicationContext() {
        return mBase.getApplicationContext();
    }
    @Override
    public void startActivity(Intent intent) {
        mBase.startActivity(intent);
    }
    @Override
    public void sendBroadcast(Intent intent) {
        mBase.sendBroadcast(intent);
    }
    @Override
    public Intent registerReceiver(
        BroadcastReceiver receiver, IntentFilter filter) {
        return mBase.registerReceiver(receiver, filter);
    }
    @Override
    public ComponentName startService(Intent service) {
        return mBase.startService(service);
    }
    ...
}
```

乍一看Context的抽象方法都是由`ContextWrapper`实现的，但是`ContextWrapper`并没有真正实现，而是全权委托给了名为`mBase`的对象，这个对象是`ContextImpl`类的实例对象。很多人可能比较奇怪`ContextImpl`类为什么看不了？因为这是一个隐藏类，被打了`@hide`注解，我们没办法看，为什么我知道mBase是`ContextImpl`类的实例呢？因为在Activity中打印`getBaseContext()`的返回值结果如下：

```xml
com.openxu.along V/openxu: getBaseContext()=android.app.ContextImpl@f4b677b
```

`mBase`对象是系统创建的，然后调用`attachBaseContext(Context base)`将mBase对象传递给`ContextWrapper`，所以说`ContextImpl`才是真正的上下文功能实现类。

## 3. ContextWrapper与ContextImpl关联

ContextWrapper中的mBase是什么时候被赋值的呢？也就是说`attachBaseContext(Context base)`是什么时候被调用的呢？ContextWrapper的子类有Activity、Service、Applicaiton，所以得分情况讨论，但不外乎一点都是在他们被创建的之后关联上ContextImpl的。

这里涉及到查看Android被隐藏的源码，github上有人去除了android.jar中的@hide注解，可以在[android-hidden-api](https://github.com/anggrayudi/android-hidden-api)中下载某一个版本的android.jar，将自己sdk目录下（比如`F:\IDE\sdk\platforms\android-28`）的替换，然后修改`compileSdkVersion`和`targetSdkVersion`，rebuild即可

### 3.1 Application

```Java
//android.app.LoadedApk.java
public Application makeApplication(boolean forceDefaultAppClass,
                                   Instrumentation instrumentation) {
    Application app = null;
    // 创建ContextImpl实例
    ContextImpl appContext = ContextImpl.createAppContext(mActivityThread, this);
    //创建Application实例，并将mBase指向appContext
    //★newApplication()方法中调用了app.attach(context)-->attachBaseContext(context)
    app = mActivityThread.mInstrumentation.newApplication(cl, appClass, appContext);
    appContext.setOuterContext(app);
    return app;
}
```

### 3.2 Service

```Java
//android.app.ActivityThread.java
private void handleCreateService(ActivityThread.CreateServiceData data) {
    //获取LoadedApk对象
    LoadedApk packageInfo = getPackageInfoNoCheck(
            data.info.applicationInfo, data.compatInfo);
    Service service = null;
    try {
        java.lang.ClassLoader cl = packageInfo.getClassLoader();
        //创建service
        service = packageInfo.getAppFactory()
                .instantiateService(cl, data.info.name, data.intent);
    } catch (Exception e) {
    }

    try {
        //创建ContextImpl实例
        ContextImpl context = ContextImpl.createAppContext(this, packageInfo);
        context.setOuterContext(service);
        Application app = packageInfo.makeApplication(false, mInstrumentation);
        //★service的attach方法中调用了attachBaseContext(context)初始化mBase
        service.attach(context, this, data.info.name, data.token, app,
                ActivityManager.getService());
        //调用service生命周期方法
        service.onCreate();
    } catch (Exception e) {
    }
}
```

### 3.3 Activity

```Java
//android.app.ActivityThread.java
private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
    //...
    //创建ContextImpl实例
    ContextImpl appContext = createBaseContextForActivity(r);
    Activity activity = null;
    //创建Activity对象
    activity = mInstrumentation.newActivity(
            cl, component.getClassName(), r.intent);
    Application app = r.packageInfo.makeApplication(false, mInstrumentation);
    if (activity != null) {
        //...
        appContext.setOuterContext(activity);
        //★Activity的attach()方法中调用了attachBaseContext(context)初始化mBase
        activity.attach(appContext, this, getInstrumentation(), r.token,
                r.ident, app, r.intent, r.activityInfo, title, r.parent,
                r.embeddedID, r.lastNonConfigurationInstances, config,
                r.referrer, r.voiceInteractor, window, r.configCallback);

    }
    return activity;
}
```

## 4. Context数量

弄懂了上面Context的继承及实现关系，如果要问一个Android应用程序究竟有多少个Context实例对象被创建？答案是`Context数量=(Activity数量+Service数量+1个Application)*2`。

Activity、Service、Application都是ContextWrapper的子类，他们的实例对象就是Context类型，这个不难理解，但是为什么要乘以2呢？因为ContextWrapper中的mBase是ContextImpl类型，它也属于Context，每创建一个Activity、Service、Application的对象将伴随一个ContextImpl对象的创建并赋值给mBase。

我们都知道Android有四大组件，除了Activity、Service还有BroadcastReceiver、ContentProvider，为什么后两个组件不是Context的子类呢？因为这两个持有的Context都是其他地方传过去的，也就是说虽然在它们里面能使用上下文，但是这个上下文是别人的，别人给他用的。


## 5. Application Context

每个应用程序都有一个Application，如果我们没有继承Application并在清单文件注册，应用程序启动时会创建一个默认的Application实例。

```Java
Context
    |
    |--getApplicationContext()

Service
    |
    |--getApplication() 

Activity
    |
    |--getApplication() 
```

Context中有一个抽象的`getApplicationContext()`方法，Activity、Service中各自有`getApplication()`方法，这两个方法返回的都是当前应用程序的`Application`实例对象。

```xml
openxu: getApplication()=com.openxu.single.MyApplication@f18630a
openxu: getApplicationContext()=com.openxu.single.MyApplication@f18630a
```

既然这两个方法返回的是同一个对象，那为什么还要设计两个方法呢？原因就是这两个方法他们处于不同的类中，如果在Activity、Service中我们需要使用Application对象，这两个方法都能够调用。但是如果我们想在任何拥有Context的的场景中获取Application呢？比如BroadcastReceiver、ContentProvider或者Dialog中，就可以调用`context.getApplicationContext()`。


## 6. Context中的Resources

Context中getResources()方法，ContextImpl实现了该方法并返回mResources成员变量，mResources是在ContextImpl被创建后调用`setResources()`初始化的。Applicaion和Service创建时都调用`ContextImpl.createAppContext()`来创建ContextImpl对象，Activity创建时调用`ContextImpl.createActivityContext()`来创建ContextImpl对象。这两个方法中都最终调用了`ResourcesManager.getOrCreateResources(IBinder activityToken, ResourcesKey key, ClassLoader classLoader)`获取Resources实例。

```Java
private @Nullable
Resources getOrCreateResources(@Nullable IBinder activityToken,
                               @NonNull ResourcesKey key, @NonNull ClassLoader classLoader) {
    //activityToken：是否为Activity获取Resources对象
    //key：ResourcesKey记录着资源文件的路径、Configuration等信息
    synchronized (this) {
        if (activityToken != null) {
            ResourcesImpl resourcesImpl = findResourcesImplForKeyLocked(key);
            if (resourcesImpl != null) {
                //activityToken不为空，是为Activity获取Resources对象
                return getOrCreateResourcesForActivityLocked(activityToken, classLoader,
                        resourcesImpl, key.mCompatInfo);
            }
        } else {
            ResourcesImpl resourcesImpl = findResourcesImplForKeyLocked(key);
            if (resourcesImpl != null) {
                //Application & Service获取Resources对象
                return getOrCreateResourcesLocked(classLoader, resourcesImpl, key.mCompatInfo);
            }
        }
        //如果没有获取到ResourcesImpl，则创建一个新的实例
        ResourcesImpl resourcesImpl = createResourcesImpl(key);
        // 根据key将ResourcesImpl缓存到map中
        mResourceImpls.put(key, new WeakReference<>(resourcesImpl));
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

Resources和ResourcesImpl是两个不同的类，他们没有继承关系，Resources中持有ResourcesImpl的引用，Resources中的方法都是在调用ResourcesImpl的方法，所以可以将ResourcesImpl看作是Resources的实现类。不管是Applicaiton、Service、Activity，他们虽然持有不同的的Resources对象，但是ResourcesImpl实例是同一个，整个应用程序就只有这一个ResourcesImpl。至于为什么Activity的Resources和Applicaiton、Service的不是同一种类型，其实他们的Resources都是Resources或者其子类，Activity是页面相关的，带有主题Theme的内容，所以在Resources基础上封装了Theme相关的方法比如`getTheme()`。


## 7. Context的使用

Android开发中Context的使用无处不在，但是刚刚说了这么多中Context，怎么确定什么场景下使用那种Context？其实在真实开发中，我们不必太多讲究，在要使用Context的时候把能拿到的Context拿过来用就行了。但是得注意下面几点：

- 开启一个Activity，那应该也是在Activity或者Fragment中调用startActivity()，其实就是使用了Activity的Context。但是如果非得在Service中开启一个Activity，只需要加入FLAG_ACTIVITY_NEW_TASK标记即可。

- Dialog只能在Activity中显示，也就是说在创建Dialog时，构造方法中只能传递Activity类型的Context，否则会报错

- 不要在生命周期长于Activity的对象中持有Activity的引用（比如静态static对象），可能造成内存泄漏，下面这种情况应该使用Application的Context
```Java
public class Singleton {
    private static Singleton instance;
    private Context mContext;
    private Singleton(Context context) {
        this.mContext = context;
    }
    public static Singleton getInstance(Context context) {
        if (instance == null) {
            instance = new Singleton(context);
        }
        return instance;
    }
}

```

- 在没有Context的情况下如果要使用Context该怎么办？可以在Application中提供方法获取全局的Application实例

```Java

public class MyApplication extends Application {
	private static MyApplication app;
	public static MyApplication getInstance() {
		return app;
	}
	@Override
	public void onCreate() {
		super.onCreate();
		app = this;
	}
}
```
















