

**基于[Android 10](https://www.androidos.net.cn/android/10.0.0_r6/xref)源码分析**

当按下手机电源键，bootloader程序被启动，它的作用是初始化硬件，将Linux Kernel镜像拷贝到内存中，并完成硬件平台相关初始化工作，比如挂载文件系统，初始化驱动模块，最后启动第一个用户进程init（进程号为1）。

## 一. Zygote启动

Android系统中所有的应用程序进程，以及用来运行系统关键服务的SystemService进程都是由Zygote(受精卵)进程负责创建的，Zygote称之为进程孵化器。它是通过复制自身的方式来创建SystemService进程和应用程序进程的，由于Zygote进程在启动时会在内部创建一个虚拟机实例，因此通过复制Zygote进程得到的SystemService进程和应用程序进程可以快速的在内部获得一个虚拟机实例拷贝。

```xml
//查看进程 windows使用findstr，linux使用grep
C:\Users\admin>adb shell ps|findstr zygote
root           485     1 2227192  33092 0                   0 S zygote64
root           486     1 1707120  25940 0                   0 S zygote
webview_zygote 1656    1 1465452  13916 0                   0 S webview_zygote32

//查看所以进程
C:\Users\admin>adb shell ps
root             1     0   22196   1020 0                   0 S init
root           485     1 2227192  33092 0                   0 S zygote64
//自己开发的应用进程，其父pid为485，说明是由zygote64进程启动的
u0_a645      24419   485 2995984 104748 0                   0 S syberos.sdisclient

```

Zygote进程启动后会马上将SystemService进程启动起来，以便它启动系统关键服务，包括Activity管理服务ActivityManagerService，Window管理服务WindowManagerService，Package管理服务PackageManagerService等服务。

### 1. 启动脚本

Zygote进程是由Android系统第一个进程init启动起来的，init进程在内核加载完成后就启动了，它在启动过程中会读取根目录下的init.rc脚本文件（rc是run command缩写，是一种脚本文件后缀，这些脚本通常在程序启动阶段被调用），以便可以将其他需要开机启动的进程一起启动起来，其中就包括zygote进程。init.rc脚本内容如下：

**[system/core/rootdir/init.rc](https://www.androidos.net.cn/android/10.0.0_r6/xref/system/core/rootdir/init.rc)**

```xml
//引入zygote启动脚本，${ro.zygote}由各个厂家使用，现在主流厂家基本使用zygote64_32，在rootdir目录下有init.zygote64_32.rc脚本
import /init.${ro.zygote}.rc   

    # 启动必要的重要服务
    start servicemanager   //Binder服务管家
    start hwservicemanager
    start vndservicemanager

    trigger zygote-start  //触发zygote启动
```

**[system/core/rootdir/init.zygote64_32.rc](https://www.androidos.net.cn/android/10.0.0_r6/xref/system/core/rootdir/init.zygote64_32.rc)**

```xml
//以service的形式启动zygote
service zygote /system/bin/app_process64 -Xzygote /system/bin --zygote --start-system-server --socket-name=zygote
    class main
    priority -20
    user root
    group root readproc reserved_disk
    //创建一个名为zygote的socket
    socket zygote stream 660 root system
    socket usap_pool_primary stream 660 root system
    onrestart write /sys/android_power/request_state wake
    //onrestart指当进程重启时执行后面的命令
    onrestart write /sys/power/state on
    onrestart restart audioserver
    onrestart restart cameraserver
    onrestart restart media
    onrestart restart netd
    onrestart restart wificond
    writepid /dev/cpuset/foreground/tasks

service zygote_secondary /system/bin/app_process32 -Xzygote /system/bin --zygote --socket-name=zygote_secondary --enable-lazy-preload
    class main
    priority -20
    user root
    group root readproc reserved_disk
    socket zygote_secondary stream 660 root system
    socket usap_pool_secondary stream 660 root system
    onrestart restart zygote
    writepid /dev/cpuset/foreground/tasks
```

zygote启动脚本中指定zygote以服务的形式启动，它对应的应用程序文件为`/system/bin/app_process64`和`/system/bin/app_process32`。`--start-system-server`表示zygote启动后马上将System进程启动起来。

后面创建一个名为zygote的socket，用于执行进程间通信，访问权限为666(所有用户都可以对它进行读写)，可以在/dev/socket 中看到一个zygote的socket。System进程中的ActivityManagerService就是通过这个socket来请求zygote进程创建新的应用程序进程的。

从脚本中可以看到zygote进程加载的应用程序文件为`/system/bin/app_process64`和`/system/bin/app_process32`，接下来我们从这个应用程序文件的入口函数main开始分析zygote进程的启动过程。

### 2. app_process.main

[frameworks/base/cmds/app_process/app_main.cpp](https://www.androidos.net.cn/android/10.0.0_r6/xref/frameworks/base/cmds/app_process/app_main.cpp)

```C++
static const char ZYGOTE_NICE_NAME[] = "zygote64";
static const char ZYGOTE_NICE_NAME[] = "zygote";
int main(int argc, char* const argv[])
{
	...
	// 创建一个AppRuntime对象runtime，接下来的zygote进程就是通过它来进一步启动的
    AppRuntime runtime(argv[0], computeArgBlockSize(argc, argv));
    ...
    bool zygote = false;
    bool startSystemServer = false;
    ...
    while (i < argc) {
        const char* arg = argv[i++];
        //遍历argv参数，如果app_process的启动参数中包含--zygote，则说明这个应用程序app_process是zygote
        if (strcmp(arg, "--zygote") == 0) {
            zygote = true;
            //64位系统nice_name为zygote64; 32位系统为zygote
            niceName = ZYGOTE_NICE_NAME;
        } else if (strcmp(arg, "--start-system-server") == 0) {
        	//启动参数中包含--start-system-server
            startSystemServer = true;
        } ...
    }
    ...
     if (startSystemServer) {
     		//启动参数中包含--start-system-server，将参数添加到zygote启动参数中，表示zygote进程在启动完成后，需要将System进程启动起来
            args.add(String8("start-system-server"));
        }
    ...
    if (!niceName.isEmpty()) {
    	//设置进程名称为zygote\zygote64，之前的名称是app_process
        runtime.setArgv0(niceName.string(), true /* setProcName */);
    }
    if (zygote) {
    	// ★ 调用AppRuntime对象runtime的成员函数start进一步启动zygote进程（Java层）
        runtime.start("com.android.internal.os.ZygoteInit", args, zygote);
    } else if (className) {
        runtime.start("com.android.internal.os.RuntimeInit", args, zygote);
    } else {
        fprintf(stderr, "Error: no class name or --zygote supplied.\n");
        app_usage();
        LOG_ALWAYS_FATAL("app_process: no class name or --zygote supplied.");
    }
}
```

main函数中首先创建一个AppRuntime对象runtime，然后解析app_process的启动参数是否包含"--zygote"，如果包含则说明这个应用程序app_process是在zygote进程中启动，如果包含"--start-system-server"参数，则添加到runtime的start函数参数中，表示zygote进程启动完成后需要将System进程启动起来。最后调用runtime的start函数进一步启动zygote进程。

AppRuntime类的成员函数start是从其父类AndroidRuntime继承下来的，下面继续分析AndroidRuntime的start实现。

### 3. AndroidRuntime.start

[frameworks/base/core/jni/AndroidRuntime.cpp](https://www.androidos.net.cn/android/10.0.0_r6/xref/frameworks/base/core/jni/AndroidRuntime.cpp)

```C++
/*
 * 启动android运行时，涉及到虚拟机的启动并调用参数中className对应类中的main方法。从前面调用过程中可以得知className为com.android.internal.os.ZygoteInit，参数中包含start-system-server
 */
void AndroidRuntime::start(const char* className, const Vector<String8>& options, bool zygote)
{
    ...
    JNIEnv* env;
    // ★ 调用成员函数startVm在zygote进程中创建一个虚拟机实例
    if (startVm(&mJavaVM, &env, zygote) != 0) {
        return;
    }
    onVmCreated(env);

    // ★ startReg函数在这个虚拟机实例中注册一系列JNI方法
    if (startReg(env) < 0) {
        ALOGE("Unable to register all android natives\n");
        return;
    }
    ...

    /*启动虚拟机，将当前线程作为虚拟机的主线程*/

	//将"com.android.internal.os.ZygoteInit"转换为"com/android/internal/os/ZygoteInit"
    char* slashClassName = toSlashClassName(className != NULL ? className : "");
    //获取ZygoteInit类
    jclass startClass = env->FindClass(slashClassName);
    if (startClass == NULL) {
        ALOGE("JavaVM unable to locate class '%s'\n", slashClassName);
        /* keep going */
    } else {
    	//获取类中static的main方法的Method ID
        jmethodID startMeth = env->GetStaticMethodID(startClass, "main",
            "([Ljava/lang/String;)V");
        if (startMeth == NULL) {
            ALOGE("JavaVM unable to find main() in '%s'\n", className);
            /* keep going */
        } else {
        	// ★ 调用ZygoteInit.main()
            env->CallStaticVoidMethod(startClass, startMeth, strArray);

#if 0
            if (env->ExceptionCheck())
                threadExitUncaughtException(env);
#endif
        }
    }
    free(slashClassName);
}
```

AndroidRuntime.start函数首先调用startVm在zygote进程中创建一个虚拟机实例，然后通过startReg函数在虚拟机实例中注册一系列JNI方法，最后调用ZygoteInit类的main()函数进一步启动zygote。

### 3. ZygoteInit.main

[frameworks/base/core/java/com/android/internal/os/ZygoteInit.java](https://www.androidos.net.cn/android/10.0.0_r6/xref/frameworks/base/core/java/com/android/internal/os/ZygoteInit.java)

```Java
public static void main(String argv[]) {
    ZygoteServer zygoteServer = null;

    // Mark zygote start. This ensures that thread creation will throw
    // an error.
    ZygoteHooks.startZygoteNoThreadCreation();

    // Zygote goes into its own process group.
    try {
        Os.setpgid(0, 0);
    } catch (ErrnoException ex) {
        throw new RuntimeException("Failed to setpgid(0,0)", ex);
    }

    Runnable caller;
    try {
        // Report Zygote start time to tron unless it is a runtime restart
        if (!"1".equals(SystemProperties.get("sys.boot_completed"))) {
            MetricsLogger.histogram(null, "boot_zygote_init",
                    (int) SystemClock.elapsedRealtime());
        }

        String bootTimeTag = Process.is64Bit() ? "Zygote64Timing" : "Zygote32Timing";
        TimingsTraceLog bootTimingsTraceLog = new TimingsTraceLog(bootTimeTag,
                Trace.TRACE_TAG_DALVIK);
        bootTimingsTraceLog.traceBegin("ZygoteInit");
        RuntimeInit.enableDdms();

        boolean startSystemServer = false;
        String zygoteSocketName = "zygote";
        String abiList = null;
        boolean enableLazyPreload = false;
        for (int i = 1; i < argv.length; i++) {
            if ("start-system-server".equals(argv[i])) {
                startSystemServer = true;
            } else if ("--enable-lazy-preload".equals(argv[i])) {
                enableLazyPreload = true;
            } else if (argv[i].startsWith(ABI_LIST_ARG)) {
                abiList = argv[i].substring(ABI_LIST_ARG.length());
            } else if (argv[i].startsWith(SOCKET_NAME_ARG)) {
                zygoteSocketName = argv[i].substring(SOCKET_NAME_ARG.length());
            } else {
                throw new RuntimeException("Unknown command line argument: " + argv[i]);
            }
        }

        final boolean isPrimaryZygote = zygoteSocketName.equals(Zygote.PRIMARY_SOCKET_NAME);

        if (abiList == null) {
            throw new RuntimeException("No ABI list supplied.");
        }

        // In some configurations, we avoid preloading resources and classes eagerly.
        // In such cases, we will preload things prior to our first fork.
        if (!enableLazyPreload) {
            bootTimingsTraceLog.traceBegin("ZygotePreload");
            EventLog.writeEvent(LOG_BOOT_PROGRESS_PRELOAD_START,
                    SystemClock.uptimeMillis());
            preload(bootTimingsTraceLog);
            EventLog.writeEvent(LOG_BOOT_PROGRESS_PRELOAD_END,
                    SystemClock.uptimeMillis());
            bootTimingsTraceLog.traceEnd(); // ZygotePreload
        } else {
            Zygote.resetNicePriority();
        }

        // Do an initial gc to clean up after startup
        bootTimingsTraceLog.traceBegin("PostZygoteInitGC");
        gcAndFinalize();
        bootTimingsTraceLog.traceEnd(); // PostZygoteInitGC

        bootTimingsTraceLog.traceEnd(); // ZygoteInit
        // Disable tracing so that forked processes do not inherit stale tracing tags from
        // Zygote.
        Trace.setTracingEnabled(false, 0);


        Zygote.initNativeState(isPrimaryZygote);

        ZygoteHooks.stopZygoteNoThreadCreation();

        zygoteServer = new ZygoteServer(isPrimaryZygote);

        if (startSystemServer) {
            Runnable r = forkSystemServer(abiList, zygoteSocketName, zygoteServer);

            // {@code r == null} in the parent (zygote) process, and {@code r != null} in the
            // child (system_server) process.
            if (r != null) {
                r.run();
                return;
            }
        }

        Log.i(TAG, "Accepting command socket connections");

        // The select loop returns early in the child process after a fork and
        // loops forever in the zygote.
        caller = zygoteServer.runSelectLoop(abiList);
    } catch (Throwable ex) {
        Log.e(TAG, "System zygote died with exception", ex);
        throw ex;
    } finally {
        if (zygoteServer != null) {
            zygoteServer.closeServerSocket();
        }
    }

    // We're in the child process and have exited the select loop. Proceed to execute the
    // command.
    if (caller != null) {
        caller.run();
    }
}
```



https://blog.csdn.net/yiranfeng/article/details/103549872

https://www.cnblogs.com/herenzhiming/articles/4998045.html












