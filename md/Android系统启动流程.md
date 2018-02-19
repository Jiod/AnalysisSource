# Android系统启动流程

标签（空格分隔）： 源码分析

------
第一阶段：Android设备上电后，首先会从处理器片上ROM的启动引导代码开始执行，片上ROM会寻找Bootloader代码，并加载到内存。***（这一步由“芯片厂商”负责设计和实现）***

第二阶段：Bootloader开始执行，首先负责完成硬件的初始化，然后找到Linux内核代码，并加载到内存。***（这一步由“设备厂商”负责设计和实现）***

第三阶段：Linux内核开始启动，初始化各种软硬件环境，加载驱动程序，挂载根文件系统，并执行init程序，由此开启Android的世界。***（这一步则是Android内核开发过程中需要涉及的地方）***

![Android系统启动流程](http://s3.51cto.com/wyfs02/M01/6E/1C/wKioL1V0NLqz5WkHAAFZtLWnJvI406.jpg)

init进程通过解析init.rc文件(system/core/rootdir/Init.rc)启动其他的系统服务进程，包括我们常见的Zygote、SystemServer等等。

一、启动service_manager服务
```
#启动service_manager服务
#源码文件所在位置：frameworks/base/cmds/servicemanager/service_manager.c
#详情解析：http://blog.csdn.net/w2865673691/article/details/26724701
service servicemanager /system/bin/servicemanager
    user system
    critical
    onrestart restart zygote
    onrestart restart media
```

二、启动Zygote进程
```
# zygote进程
# 源代码是/frameworks/base/cmds/app_process/app_main.cpp
# 注意后面的参数，里面的代码会用到
service zygote /system/bin/app_process -Xzygote /system/bin --zygote --start-system-server
    socket zygote stream 666
    onrestart write /sys/android_power/request_state wake
    onrestart write /sys/power/state on
    onrestart restart media
    onrestart restart netd
```

```
// 启动参数：-Xzygote /system/bin --zygote --start-system-server
int main(int argc, const char* const argv[])
{
    /*** 省略一些代码 ***/
    // Next arg is startup classname or "--zygote"
    if (i < argc) {
        arg = argv[i++];
        if (0 == strcmp("--zygote", arg)) {
            bool startSystemServer = (i < argc) ? strcmp(argv[i], "--start-system-server") == 0 : false;
            setArgv0(argv0, "zygote");
            set_process_name("zygote");
            // 加载ZyogteInit，并执行ZygoteInit的main方法，这里的startSystemServer=true
            runtime.start("com.android.internal.os.ZygoteInit", startSystemServer);
        } else {
            set_process_name(argv0);

            runtime.mClassName = arg;

            // Remainder of args get passed to startup class main()
            runtime.mArgC = argc-i;
            runtime.mArgV = argv+i;

            LOGV("App process is starting with pid=%d, class=%s.\n", getpid(), runtime.getClassName());
            runtime.start();
        }
    } else {
        LOG_ALWAYS_FATAL("app_process: no class name or --zygote supplied.");
        fprintf(stderr, "Error: no class name or --zygote supplied.\n");
        app_usage();
        return 10;
    }

}
```
上面启动了zygote进程，并加载执行ZygoteInit#main方法，这应该是系统启动执行的第一个Java文件
```
public static void main(String argv[]) {
    try {
        VMRuntime.getRuntime().setMinimumHeapSize(5 * 1024 * 1024);

        // Start profiling the zygote initialization.
        SamplingProfilerIntegration.start();

        // 创建socket，用来与ActivityManagerService进行通信
        registerZygoteSocket();
        EventLog.writeEvent(LOG_BOOT_PROGRESS_PRELOAD_START, SystemClock.uptimeMillis());
        // 预加载类和资源
        preloadClasses();
        //cacheRegisterMaps();
        preloadResources();
        EventLog.writeEvent(LOG_BOOT_PROGRESS_PRELOAD_END, SystemClock.uptimeMillis());

        // Finish profiling the zygote initialization.
        SamplingProfilerIntegration.writeZygoteSnapshot();

        // Do an initial gc to clean up after startup
        gc();

        // If requested, start system server directly from Zygote
        if (argv.length != 2) {
            throw new RuntimeException(argv[0] + USAGE_STRING);
        }

        if (argv[1].equals("true")) {
            // 启动system_server进程，其实就是调用SystemServer类的main方法
            startSystemServer();
        } else if (!argv[1].equals("false")) {
            throw new RuntimeException(argv[0] + USAGE_STRING);
        }

        Log.i(TAG, "Accepting command socket connections");

        if (ZYGOTE_FORK_MODE) {
            runForkMode();
        } else {
            // 开启一个循环，处理ActivityManagerService创建应用进程的请求
            runSelectLoopMode();
        }
        // 关闭Socket
        closeServerSocket();
    } catch (MethodAndArgsCaller caller) {
        caller.run();
    } catch (RuntimeException ex) {
        Log.e(TAG, "Zygote died with exception", ex);
        closeServerSocket();
        throw ex;
    }
}
```
ZyogteInit主要是干了这样几件事情：
> * 启动Socket用于和ActivityManagerServer通讯
> * 预加载类和资源
> * 启动SystemServer
> * 开启循环，处理ActivityManagerService发送过来的请求

重点看看这个启动SystemServer
```
private static boolean startSystemServer()
        throws MethodAndArgsCaller, RuntimeException {
    /* Hardcoded command line to start the system server */
    String args[] = {
        "--setuid=1000",
        "--setgid=1000",
        "--setgroups=1001,1002,1003,1004,1005,1006,1007,1008,1009,1010,1018,3001,3002,3003",
        "--capabilities=130104352,130104352",
        "--runtime-init",
        "--nice-name=system_server",
        "com.android.server.SystemServer",
    };
    ZygoteConnection.Arguments parsedArgs = null;

    int pid;

    try {
        parsedArgs = new ZygoteConnection.Arguments(args);

        /*
         * Enable debugging of the system process if *either* the command line flags
         * indicate it should be debuggable or the ro.debuggable system property
         * is set to "1"
         */
        int debugFlags = parsedArgs.debugFlags;
        if ("1".equals(SystemProperties.get("ro.debuggable")))
            debugFlags |= Zygote.DEBUG_ENABLE_DEBUGGER;

        /* 创建system_server进程 */
        pid = Zygote.forkSystemServer(
                parsedArgs.uid, parsedArgs.gid,
                parsedArgs.gids, debugFlags, null,
                parsedArgs.permittedCapabilities,
                parsedArgs.effectiveCapabilities);
    } catch (IllegalArgumentException ex) {
        throw new RuntimeException(ex);
    }

    /* For child process */
    if (pid == 0) {
        handleSystemServerProcess(parsedArgs);
    }

    return true;
}
```
代码中fork了一个system_server进程。加载执行了SystemServer.java文件
```
// zygote进程fork出，并执行这个方法，可以看下ZygoteInit的startSystemServer方法
public static void main(String[] args) {
    if (System.currentTimeMillis() < EARLIEST_SUPPORTED_TIME) {
        // If a device's clock is before 1970 (before 0), a lot of
        // APIs crash dealing with negative numbers, notably
        // java.io.File#setLastModified, so instead we fake it and
        // hope that time from cell towers or NTP fixes it
        // shortly.
        Slog.w(TAG, "System clock is before 1970; setting to 1970.");
        SystemClock.setCurrentTimeMillis(EARLIEST_SUPPORTED_TIME);
    }

    if (SamplingProfilerIntegration.isEnabled()) {
        SamplingProfilerIntegration.start();
        timer = new Timer();
        timer.schedule(new TimerTask() {
            @Override
            public void run() {
                SamplingProfilerIntegration.writeSnapshot("system_server");
            }
        }, SNAPSHOT_INTERVAL, SNAPSHOT_INTERVAL);
    }

    // The system server has to run all of the time, so it needs to be
    // as efficient as possible with its memory usage.
    VMRuntime.getRuntime().setTargetHeapUtilization(0.8f);

    System.loadLibrary("android_servers");
    //  这个方法是一个native方法，init1方法又会转回来调用init2方法
    init1(args);
    
    public static final void init2() {
        Slog.i(TAG, "Entered the Android system server!");
        // 创建一个实例，注册各种binder，ServiceManager.addService
        Thread thr = new ServerThread();
        thr.setName("android.server.ServerThread");
        thr.start();
    }
}
```
代码中调用init2方法，创建了ServerThread实例并且启动，ServerThread继承自Thread，因此会执行ServerThread的run方法。
在ServerThread的run方法中会创建各种Server实例，并添加到ServerManager中；
```
((ActivityManagerService)ActivityManagerNative.getDefault())
            .systemReady(new Runnable() {
        public void run() {
            //各种service.systemRead();
        }
    });
```
service注册完成后，会调用ActivityManagerService.systemReady()方法，在这个方法中会调用ActivityStack的resumeTopActivityLocked方法。
```
final boolean resumeTopActivityLocked(ActivityRecord prev) {
    // Find the first activity that is not finishing.
    ActivityRecord next = topRunningActivityLocked(null);

    // Remember how we'll process this pause/resume situation, and ensure
    // that the state is reset however we wind up proceeding.
    final boolean userLeaving = mUserLeaving;
    mUserLeaving = false;

    if (next == null) {
        // There are no more activities!  Let's just start up the
        // Launcher...
        if (mMainStack) {
            return mService.startHomeActivityLocked();
        }
    }
}
```
继续调用ActivityManagerServie.startHomeActivityLocked()方法，这就是要启动launcher界面啦。

------
作者 [@fishpan][1]     
2018 年 02月 10日    

[1]: https://github.com/Jiod





