---
layout: post
title:  "Android开机启动流程分析二(System进程)"
author: "陈宇瀚"
date:   2021-01-08 15:23:02 +0800
header-img: "img/img-head/img-head-boot-process.jpg"
categories: article
tags:
  - Android 
  - System
  - 开机启动流程
---
&ensp;&ensp; 在[Android开机启动流程分析一Zygote进程](https://yuhanchen5027.github.io/article/2021/01/03/Android-boot-process-analysis-1(Zygote-Process)/)那一章中知道在**ZygoteInit.main**方法中，会调用**ZygoteInit.startSystemServer**进行**System进程以及相关服务**的启动，之后就会进入循环等待模式，等待**ActivityMangagerService**创建新应用进程的请求。接下来从**ZygoteInit.startSystemServer**开始通过源码分析**System**进程和服务的启动流程。

&ensp;&ensp; *以下源码基于rk3399_industry Android7.1.2*
## ZygoteInit.startSystemServer
&ensp;&ensp; **startSystemServer**:为**System**服务进程准备参数，并从**Zygote**中**fork**一个**System进程**出来。源码如下：
```java
// 这里根据不同的系统产生的参数不同 以rk3399_industry Android7.1.2为例
// adbList = ro.product.cpu.abilist64
// socketName = “zygote”
private static boolean startSystemServer(String abiList, String socketName)
            throws MethodAndArgsCaller, RuntimeException {
        long capabilities = posixCapabilitiesAsBits(
            OsConstants.CAP_IPC_LOCK,
            OsConstants.CAP_KILL,
            OsConstants.CAP_NET_ADMIN,
            OsConstants.CAP_NET_BIND_SERVICE,
            OsConstants.CAP_NET_BROADCAST,
            OsConstants.CAP_NET_RAW,
            OsConstants.CAP_SYS_MODULE,
            OsConstants.CAP_SYS_NICE,
            OsConstants.CAP_SYS_PTRACE,
            OsConstants.CAP_SYS_TIME,
            OsConstants.CAP_SYS_TTY_CONFIG
        );
        ....
        /* 保存System进程的启动参数 */
        String args[] = {
            "--setuid=1000",
            "--setgid=1000",
            "--setgroups=1001,1002,1003,1004,1005,1006,1007,1008,1009,1010,1018,1021,1032,3001,3002,3003,3006,3007,3009,3010",
            "--capabilities=" + capabilities + "," + capabilities,
            "--nice-name=system_server",
            "--runtime-args",
            "com.android.server.SystemServer",
        };
        ZygoteConnection.Arguments parsedArgs = null;

        int pid;

        try {
            parsedArgs = new ZygoteConnection.Arguments(args);
            ZygoteConnection.applyDebuggerSystemProperty(parsedArgs);
            ZygoteConnection.applyInvokeWithSystemProperty(parsedArgs);
            
            // 通过Zygote的静态成员函数forkSystemServer创建一个子进程 
            // 该进程就是Android系统的Systen进程
            // 进程用户ID和用户组ID都被设置为1000
            // 并且还具有用户组1001~1010,1018,1021,1032,3001~3003,3006,3007,3009,3010的权限
            pid = Zygote.forkSystemServer( //forkSystemServer最终通过函数fork在当前进程创建一个子进程
                    parsedArgs.uid, parsedArgs.gid,
                    parsedArgs.gids,
                    parsedArgs.debugFlags,
                    null,
                    parsedArgs.permittedCapabilities,
                    parsedArgs.effectiveCapabilities);
        } catch (IllegalArgumentException ex) {
            throw new RuntimeException(ex);
        }
        /* For child process */
        if (pid == 0) {
         // 当Zygote.forkSystemServer返回0代表目前是在创建的子进程中
            if (hasSecondZygote(abiList)) {
                if(isBox){
                    waitForSecondaryZygote(socketName);
                }
                Log.d(TAG,"--------call waitForSecondaryZygote,skip this---,abiList= "+abiList);
            }
            // 启动System进程
            handleSystemServerProcess(parsedArgs);
        }

        return true;
    }
    
    
    /**
     * fork成功后进行启动System进程和服务的剩余工作
     */
    private static void handleSystemServerProcess(
            ZygoteConnection.Arguments parsedArgs)
            throws ZygoteInit.MethodAndArgsCaller {
        //由之前分析可得,Zygote进程启动时会创建一个Server的socket，
        //用来监听AMS请求创建新的应用程序，System进程是Zygote进程fork出来的，
        //所以同样包含了这个socket，但并不需要，所以在此处关闭
        //内部其实就是将sServerSocket close并置null
        closeServerSocket();

        //将umask设置为0077，这样新文件和目录将默认为所有者权限。
        Os.umask(S_IRWXG | S_IRWXO);

        if (parsedArgs.niceName != null) {
            Process.setArgV0(parsedArgs.niceName);//设置当前进程名system_server
        }

        final String systemServerClasspath = Os.getenv("SYSTEMSERVERCLASSPATH");
        if (systemServerClasspath != null) {
            //进行dex优化
            performSystemServerDexOpt(systemServerClasspath);
        }
        //启动systemserver时invokeWith为null
        if (parsedArgs.invokeWith != null) {
            String[] args = parsedArgs.remainingArgs;
            // If we have a non-null system server class path, we'll have to duplicate the
            // existing arguments and append the classpath to it. ART will handle the classpath
            // correctly when we exec a new process.
            if (systemServerClasspath != null) {
                String[] amendedArgs = new String[args.length + 2];
                amendedArgs[0] = "-cp";
                amendedArgs[1] = systemServerClasspath;
                System.arraycopy(parsedArgs.remainingArgs, 0, amendedArgs, 2, parsedArgs.remainingArgs.length);
            }

            WrapperInit.execApplication(parsedArgs.invokeWith,
                    parsedArgs.niceName, parsedArgs.targetSdkVersion,
                    VMRuntime.getCurrentInstructionSet(), null, args);
        } else {
            ClassLoader cl = null;
            if (systemServerClasspath != null) {
                cl = createSystemServerClassLoader(systemServerClasspath,
                                                   parsedArgs.targetSdkVersion);

                Thread.currentThread().setContextClassLoader(cl);
            }

            /*
             * 将其余参数传递给SystemServer。
             */
            // parsedArgs.remainingArgs 
            // 表示之前System进程启动参数args[]中不能被Arguments解析的参数
            // 这里parsedArgs.remainingArgs = {"com.android.server.SystemServer"}
            RuntimeInit.zygoteInit(parsedArgs.targetSdkVersion, parsedArgs.remainingArgs, cl);
        }
    }
```
&ensp;&ensp; 由上面分析得出**startSystemServer**设置了一些进程的权限，名称，参数的配置，并从**zygote**进程中**fork**出一个子进程(即**SystemServer**进程)，关闭了复制出来的多余的**socket**，并将剩余参数传递给**SystemServer**,调用**RuntimeInit.zygoteInit**来进一步启动**System进程**。

### RuntimeInit.zygoteInit
**frameworks/base/core/java/com/android/internal/os/RuntimeInit.java**
```java
public static final void zygoteInit(int targetSdkVersion, String[] argv, ClassLoader classLoader)
            throws ZygoteInit.MethodAndArgsCaller {
        Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "RuntimeInit");
        //重定向log输出
        redirectLogStreams();
        //设置System进程的时区和键盘布局等一些通用信息
        commonInit();
        //native层的初始化
        nativeZygoteInit();
        //此时argv={"com.android.server.SystemServer"}，该方法内会调用SystemServer的main方法,
        applicationInit(targetSdkVersion, argv, classLoader);
    }
    
     /**
     * 重定向System.out和System.err 至Android log
     */
    public static void redirectLogStreams() {
        System.out.close();
        System.setOut(new AndroidPrintStream(Log.INFO, "System.out"));
        System.err.close();
        System.setErr(new AndroidPrintStream(Log.WARN, "System.err"));
    }
    
    //一些通用信息的设置
    private static final void commonInit() {

        /* 设置适用于VM中的所有线程的默认处理程序*/
        Thread.setDefaultUncaughtExceptionHandler(new UncaughtHandler());

        /*
         * Install a TimezoneGetter subclass for ZoneInfo.db
         */
        TimezoneGetter.setInstance(new TimezoneGetter() {
            @Override
            public String getId() {
                return SystemProperties.get("persist.sys.timezone");
            }
        });
        TimeZone.setDefault(null);

        /*
         * Sets handler for java.util.logging to use Android log facilities.
         * The odd "new instance-and-then-throw-away" is a mirror of how
         * the "java.util.logging.config.class" system property works. We
         * can't use the system property here since the logger has almost
         * certainly already been initialized.
         */
        LogManager.getLogManager().reset();
        new AndroidConfig();

        /*
         * 设置HttpURLConnection使用的默认HTTP用户代理。
         */
        String userAgent = getDefaultUserAgent();
        System.setProperty("http.agent", userAgent);

        /*
         * 设置socket的tag，用于网络流量统计
         */
        NetworkManagementSocketTagger.install();

        /*
         * If we're running in an emulator launched with "-trace", put the
         * VM into emulator trace profiling mode so that the user can hit
         * F9/F10 at any time to capture traces.  This has performance
         * consequences, so it's not something you want to do always.
         */
        String trace = SystemProperties.get("ro.kernel.android.tracing");
        if (trace.equals("1")) {
            Slog.i(TAG, "NOTE: emulator trace profiling enabled");
            Debug.enableEmulatorTraceOutput();
        }

        initialized = true;
    }
```
**zygoteInit**方法主要进行了System进程的时区和键盘布局等一些通用信息的设置，native层的初始化，之后会调用调用应用程序java层的main方法。下面来细看下这些设置和初始化方法。
#### nativeZygoteInit()
&ensp;&ensp; **nativeZygoteInit()**方法最后会调用jni方法，位于**frameworks/base/core/jni/AndroidRuntime.cpp**方法内，源码如下：
```java
static void com_android_internal_os_RuntimeInit_nativeZygoteInit(JNIEnv* env, jobject clazz)
{
    //根据 app_process的执行 那部分分析得出，gCurRuntime是一个AppRuntime对象
    //AppRuntime类继承了AndroidRuntime类，Zygote进程创建一个AppRuntime对象时
    //AndroidRuntime的构造函数被调用，gCurRuntime就会被初始化。
    //由于该进程是Zygote fork出来的，所以也有gCurRuntime。
    //onZygoteInit方法位于frameworks/base/cmds/app_process/app_main.cpp内
    gCurRuntime->onZygoteInit();
}

/*frameworks/base/cmds/app_process/app_main.cpp*/
//注：virtual为c++中的虚函数，指一个类中你希望重载的成员函数，可以理解成Java的抽象(abstract)
  virtual void onZygoteInit()
    {
        sp<ProcessState> proc = ProcessState::self();
        ALOGV("App process: starting thread pool.\n");
        //之后进入ProcessState类中
        proc->startThreadPool();
    }

/*frameworks/native/libs/binder/ProcessState.cpp*/
void ProcessState::startThreadPool()
{
    AutoMutex _l(mLock);
    if (!mThreadPoolStarted) {
        mThreadPoolStarted = true;
        spawnPooledThread(true);
    }
}
void ProcessState::spawnPooledThread(bool isMain)
{
    if (mThreadPoolStarted) {
        String8 name = makeBinderThreadName();
        ALOGV("Spawning new pooled thread, name=%s\n", name.string());
        sp<Thread> t = new PoolThread(isMain);
        t->run(name.string());
    }
}

String8 ProcessState::makeBinderThreadName() {
    int32_t s = android_atomic_add(1, &mThreadPoolSeq);
    pid_t pid = getpid();
    String8 name;
    name.appendFormat("Binder:%d_%X", pid, s);
    return name;
}
```
&ensp;&ensp; 这部分代码与**Binder**相关，总的来说**nativeZygoteInit()**会在**System**进程中启动一个**Binder**线程池，便与**Binder**通信系统建立了联系，之后**System服务**就可以使用**Binder**通信了。

#### RuntimeInit.applicationInit
**frameworks/base/core/java/com/android/internal/os/RuntimeInit.java**
```java
 private static void applicationInit(int targetSdkVersion, String[] argv, ClassLoader classLoader)
            throws ZygoteInit.MethodAndArgsCaller {
        nativeSetExitWithoutCleanup(true);

        VMRuntime.getRuntime().setTargetHeapUtilization(0.75f);
        VMRuntime.getRuntime().setTargetSdkVersion(targetSdkVersion);
        // 这里的Arguments类与上面的类不是同一个，这个Arguments是RuntimeInit中的一个内部类
        final Arguments args;
        try {
            // argv = {"com.android.server.SystemServer"}
            args = new Arguments(argv);
        } catch (IllegalArgumentException ex) {
            Slog.e(TAG, ex.getMessage());
            // let the process exit
            return;
        }
        ....
        
        // 之后进行SystemServer的启动，将剩余参数传递给SystemServer.main方法
        invokeStaticMain(args.startClass, args.startArgs, classLoader);
    }
    
    static class Arguments {
        String startClass;

        String[] startArgs;
        
        ....
        Arguments(String args[]) throws IllegalArgumentException {
            parseArgs(args);
        }

        ....
        private void parseArgs(String args[])
                throws IllegalArgumentException {
            int curArg = 0;
            for (; curArg < args.length; curArg++) {
                String arg = args[curArg];

                if (arg.equals("--")) {
                    curArg++;
                    break;
                } else if (!arg.startsWith("--")) {
                    break;
                }
            }

            if (curArg == args.length) {
                throw new IllegalArgumentException("Missing classname argument to RuntimeInit!");
            }

            startClass = args[curArg++];
            startArgs = new String[args.length - curArg];
            System.arraycopy(args, curArg, startArgs, 0, startArgs.length);
        }
    }
    
    /**
     * 调用className对应类的main方法，调用失败抛出RuntimeException异常
     *
     * @param className 完整路径的类名
     * @param argv 传递给main()方法的参数
     * @param classLoader 类加载器
     */
    private static void invokeStaticMain(String className, String[] argv, ClassLoader classLoader)
            throws ZygoteInit.MethodAndArgsCaller {
        Class<?> cl;

        try {
            //将SystemServer类加载到当前进程中
            cl = Class.forName(className, true, classLoader);
        } catch (ClassNotFoundException ex) {
            throw new RuntimeException(
                    "Missing class when invoking static main " + className,
                    ex);
        }

        Method m;
        try {
            //获得它的静态成员函数main，保存在Method对象m中
            m = cl.getMethod("main", new Class[] { String[].class });
        } catch (NoSuchMethodException ex) {
            throw new RuntimeException(
                    "Missing static main on " + className, ex);
        } catch (SecurityException ex) {
            throw new RuntimeException(
                    "Problem getting static main on " + className, ex);
        }

        int modifiers = m.getModifiers();
        if (! (Modifier.isStatic(modifiers) && Modifier.isPublic(modifiers))) {
            throw new RuntimeException(
                    "Main method is not public and static on " + className);
        }

        /**
        * 将Method对象m封装在一个MethodAndArgsCaller对象中，
        * 并将这个对象MethodAndArgsCaller对象作为一个异常对象抛出给应用程序处理
        */
        throw new ZygoteInit.MethodAndArgsCaller(m, argv);
    }
```
&ensp;&ensp; 由于新创建的**System**进程复制了**Zygote**进程的地址空间，因此，新创建的进程的调用堆栈与**Zygote**进程的一致，因此当**RuntimeInit.invokeStaticMain**抛出一个**MethodAndArgsCaller**的异常时，系统就会沿着这个调用过程往后寻找一个代码块捕获并处理。这部分的代码就在**ZygoteInit.main**函数内，我们往下看源码：
```java
public static void main(String argv[]) {
        ....
        try {
            ....
        } catch (MethodAndArgsCaller caller) {
            //重点部分
            caller.run();
        } catch (Throwable ex) {
            ....
        }
    }
```
&ensp;&ensp; 可以看到**ZygoteInit.main**中通过**try-catch**捕获了**MethodAndArgsCaller**这一异常，并调用了**MethodAndArgsCaller**对象的**run**方法;我们接下来就看一下这个**run**方法;
&ensp;&ensp; **ZygoteInit.MethodAndArgsCaller**:位于**frameworks/base/core/java/com/android/internal/os/ZygoteInit.java**文件内
```java
public static class MethodAndArgsCaller extends Exception
            implements Runnable {
        /** 要调用的方法 */
        private final Method mMethod;

        /** 参数数组 */
        private final String[] mArgs;

        public MethodAndArgsCaller(Method method, String[] args) {
            mMethod = method;
            mArgs = args;
        }

        public void run() {
            try {
                mMethod.invoke(null, new Object[] { mArgs });
            } catch (IllegalAccessException ex) {
                throw new RuntimeException(ex);
            } catch (InvocationTargetException ex) {
                Throwable cause = ex.getCause();
                if (cause instanceof RuntimeException) {
                    throw (RuntimeException) cause;
                } else if (cause instanceof Error) {
                    throw (Error) cause;
                }
                throw new RuntimeException(ex);
            }
        }
    }
```
&ensp;&ensp; **MethodAndArgsCaller**类有两个成员变量，**mMethod**指向了**SystemServer**类的静态成员函数**main**，**mArgs**指向了传入的参数；最后**run**通过反射调用**SystemServer.main**；

&ensp;&ensp; 这里绕了一大圈，为何要**RuntimeInit.applicationInit->invokeStaticMain-> throw new ZygoteInit.MethodAndArgsCaller(m, argv);**执行这一大步，不直接调用？

&ensp;&ensp; 原因是我们每创建一个新的进程，进入到对应的**main**方法前都有一个内部初始化运行时库，启动**Binder**线程池等操作，在执行到**main**时其实已经执行了许多代码了，为了使新创建的进程认为它的入口是**main**，所以才用了抛出异常的方式回到**ZygoteInit.main**来间接调用，也是利用了**Java**的异常处理机制清除之前的调用堆栈。

## SystemServer.main
**frameworks/base/services/java/com/android/server/SystemServer.java**
源码如下：
```java
// 可支持的最早时间，即1970年
private static final long EARLIEST_SUPPORTED_TIME = 86400 * 1000;
public static void main(String[] args) {
        new SystemServer().run();
    }
    
  private void run() {
        try {
            ....
            //防止设备时间早1970之前导致部分api处理出错，
            if (System.currentTimeMillis() < EARLIEST_SUPPORTED_TIME) {
                Slog.w(TAG, "System clock is before 1970; setting to 1970.");
                //强制设置时间在1970年之后
                SystemClock.setCurrentTimeMillis(EARLIEST_SUPPORTED_TIME);
            }

            ....

            // 进入Android System服务
            Slog.i(TAG, "Entered the Android system server!");
            ....

            //清除vm内存增长限制，启动过程需要较多的虚拟机内存空间
            VMRuntime.getRuntime().clearGrowthLimit();
            // 系统服务器必须一直运行，因此它需要尽可能有效地使用内存。
            // 设置内存利用率我哦0.8
            VMRuntime.getRuntime().setTargetHeapUtilization(0.8f);

            ....

            // 确保当前系统进程的binder调用，总是以前台优先级运行(foreground priority)
            BinderInternal.disableBackgroundScheduling(true);
            // 增加system_server中绑定线程的数量
            BinderInternal.setMaxThreads(sMaxBinderThreads);
            // Prepare the main looper thread (this thread).
            android.os.Process.setThreadPriority(
                android.os.Process.THREAD_PRIORITY_FOREGROUND);
            android.os.Process.setCanSelfBackground(false);
            // 主线程looper就在当前线程运行
            Looper.prepareMainLooper();

            // 初始化本地服务。
            System.loadLibrary("android_servers");

            // 检测上次关机过程是否失败，该方法可能不会返回
            performPendingShutdown();

            // 初始化系统上下文
            createSystemContext();

            // 创建系统服务管理器
            mSystemServiceManager = new SystemServiceManager(mSystemContext);
            mSystemServiceManager.setRuntimeRestarted(mRuntimeRestart);
            //将mSystemServiceManager添加到本地服务的成员sLocalServiceObjects
            //sLocalServiceObjects是一个ArrayMap对象
            LocalServices.addService(SystemServiceManager.class, mSystemServiceManager);
        } finally {
            Trace.traceEnd(Trace.TRACE_TAG_SYSTEM_SERVER);
        }
        // 启动各种系统服务
        try {
            startBootstrapServices(); //启动引导服务
            startCoreServices();    //启动核心服务
            startOtherServices();   //启动其他服务
        } catch (Throwable ex) {
            throw ex;
        } finally {
            Trace.traceEnd(Trace.TRACE_TAG_SYSTEM_SERVER);
        }

        // debug版本，会将日志输出到dropbox，用于分析
        if (StrictMode.conditionallyEnableDebugLogging()) {
            Slog.i(TAG, "Enabled StrictMode for system server main thread.");
        }

        // 一直循环执行
        Looper.loop();
        throw new RuntimeException("Main thread loop unexpectedly exited");
    }
    
    // 初始化系统上下文
    private void createSystemContext() {
        //创建ActivityThread对象和ContextImpl，创建Application，并调用其onCreate()方法
        ActivityThread activityThread = ActivityThread.systemMain();
        //LoadedApk对象
        mSystemContext = activityThread.getSystemContext();
        //设置主题
        mSystemContext.setTheme(DEFAULT_SYSTEM_THEME);
    }
    
    /* frameworks/base/core/java/android/app/ActivityThread.java */
    //ActivityThread.systemMain();
    public static ActivityThread systemMain() {
        // 判断应用内存，对于低内存设备，禁止应用加速
        if (!ActivityManager.isHighEndGfx()) {
            ThreadedRenderer.disable(true);
        } else {
            ThreadedRenderer.enableForegroundTrimming();
        }
        //创建ActivityThread
        ActivityThread thread = new ActivityThread();
        //创建Application以及调用其onCreate()方法
        thread.attach(true);
        return thread;
    }
    
    private void attach(boolean system) {
        sCurrentActivityThread = this;
        mSystemThread = system;
        if (!system) {
           ....
        }  else {
            //system=true system进程
            android.ddm.DdmHandleAppName.setAppName("system_process",
                    UserHandle.myUserId());
            try {
                mInstrumentation = new Instrumentation();
                // 创建应用上下文，这里也会调用一次getSystemContext()
                // 该方法内会创建LoadedApk
                // getSystemContext().mPackageInfo其实就是“android”
                ContextImpl context = ContextImpl.createAppContext(
                        this, getSystemContext().mPackageInfo);
                //创建Application
                mInitialApplication = context.mPackageInfo.makeApplication(true, null);
                //调用Application.onCreate方法
                mInitialApplication.onCreate();
            } catch (Exception e) {
                throw new RuntimeException(
                        "Unable to instantiate Application():" + e.toString(), e);
            }
        }

        //设置一些视图回调方法
        ViewRootImpl.addConfigCallback(new ComponentCallbacks2() {
           .....
        });
    }
    
  //ActivityThread.getSystemContext
  public ContextImpl getSystemContext() {
        synchronized (this) {
            if (mSystemContext == null) {
                mSystemContext = ContextImpl.createSystemContext(this);
            }
            return mSystemContext;
        }
    }
    /* frameworks/base/core/java/android/app/ContextImpl.java*/
    //ContextImpl.createSystemContext(this);
    static ContextImpl createSystemContext(ActivityThread mainThread) {
        // 创建LoadedApk对象
        LoadedApk packageInfo = new LoadedApk(mainThread);
        // 创建ContextImpl对象
        ContextImpl context = new ContextImpl(null, mainThread,
                packageInfo, null, null, 0, null, null, Display.INVALID_DISPLAY);
        context.mResources.updateConfiguration(context.mResourcesManager.getConfiguration(),
                context.mResourcesManager.getDisplayMetrics());
        return context;
    }
    /***    
    LoadedApk(ActivityThread activityThread) {
    mActivityThread = activityThread;
    mApplicationInfo = new ApplicationInfo();
    mApplicationInfo.packageName = "android";
    mPackageName = "android";
    mAppDir = null;
    mResDir = null;
    mSplitAppDirs = null;
    mSplitResDirs = null;
    mOverlayDirs = null;
    mSharedLibraries = null;
    mDataDir = null;
    mDataDirFile = null;
    mDeviceProtectedDataDirFile = null;
    mCredentialProtectedDataDirFile = null;
    mLibDir = null;
    mBaseClassLoader = null;
    mSecurityViolation = false;
    mIncludeCode = true;
    mRegisterPackage = false;
    mClassLoader = ClassLoader.getSystemClassLoader();
    mResources = Resources.getSystem();
    }   
    ***/
    /* frameworks/base/core/java/android/app/LoadedApk.java */
    //LoadedApk.createAppContext();
     public Application makeApplication(boolean forceDefaultAppClass,
            Instrumentation instrumentation) {
        if (mApplication != null) {
            return mApplication;
        }

        Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "makeApplication");

        Application app = null;

        String appClass = mApplicationInfo.className;
        if (forceDefaultAppClass || (appClass == null)) {
            appClass = "android.app.Application";
        }

        try {
            java.lang.ClassLoader cl = getClassLoader();
            if (!mPackageName.equals("android")) {
                Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER,
                        "initializeJavaContextClassLoader");
                initializeJavaContextClassLoader();
                Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
            }
            ContextImpl appContext = ContextImpl.createAppContext(mActivityThread, this);
            // 创建Application
            app = mActivityThread.mInstrumentation.newApplication(
                    cl, appClass, appContext);
            appContext.setOuterContext(app);
        } catch (Exception e) {
           ....
        }
        //将创建的app添加到应用列表。
        mActivityThread.mAllApplications.add(app);
        mApplication = app;
        ....
        return app;
    }

```
&ensp;&ensp; 这部分是**SystemServer**的一些基础准备，之后会进行一系列系统服务的启动。
### 系统服务的启动
分为三部分
- startBootstrapServices()：引导服务
- startCoreServices()：核心服务
- startOtherServices()：其他服务

#### startBootstrapServices()：引导服务
```java
private void startBootstrapServices() {
        // 等待Install完成启动，它会创建具有合适权限的关键目录，如/data/user
        // 需要在初始化其他服务前完成
        Installer installer = mSystemServiceManager.startService(Installer.class);

        // 启动ActivityManagerService
        mActivityManagerService = mSystemServiceManager.startService(
                ActivityManagerService.Lifecycle.class).getService();
        mActivityManagerService.setSystemServiceManager(mSystemServiceManager);
        mActivityManagerService.setInstaller(installer);

        // 启动PowerManagerService
        mPowerManagerService = mSystemServiceManager.startService(PowerManagerService.class);

        // PowerManagerService已经启动，通过ActivityManagerService初始化PowerManagement
        mActivityManagerService.initPowerManagement();

        // 管理led和显示背光，需要它来控制显示。
        mSystemServiceManager.startService(LightsService.class);

        // 启动DisplayManagerService
        mDisplayManagerService = mSystemServiceManager.startService(DisplayManagerService.class);

        // 设置默认显示，在初始化PackageManagerService之前
        mSystemServiceManager.startBootPhase(SystemService.PHASE_WAIT_FOR_DEFAULT_DISPLAY);

        // 当设备进行加密时，只运行核心应用
        String cryptState = SystemProperties.get("vold.decrypt");
        if (ENCRYPTING_STATE.equals(cryptState)) {
            Slog.w(TAG, "Detected encryption in progress - only parsing core apps");
            mOnlyCore = true;
        } else if (ENCRYPTED_STATE.equals(cryptState)) {
            Slog.w(TAG, "Device encrypted - only parsing core apps");
            mOnlyCore = true;
        }
        //启动PackageManagerService
        mPackageManagerService = PackageManagerService.main(mSystemContext, installer,
                mFactoryTestMode != FactoryTest.FACTORY_TEST_OFF, mOnlyCore);
        mFirstBoot = mPackageManagerService.isFirstBoot();
        mPackageManager = mSystemContext.getPackageManager();

        //管理A/B OTA dexting? 不太理解 这里还得再研究下
        if (!mOnlyCore) {
            boolean disableOtaDexopt = SystemProperties.getBoolean("config.disable_otadexopt",
                    false);
            if (!disableOtaDexopt) {
                traceBeginAndSlog("StartOtaDexOptService");
                try {
                    OtaDexoptService.main(mSystemContext, mPackageManagerService);
                } catch (Throwable e) {
                    reportWtf("starting OtaDexOptService", e);
                } finally {
                    Trace.traceEnd(Trace.TRACE_TAG_SYSTEM_SERVER);
                }
            }
        }

        //启动UserManagerService
        mSystemServiceManager.startService(UserManagerService.LifeCycle.class);

        //初始化用于从包中缓存资源的属性缓存。不太理解？求大神解答
        AttributeCache.init(mSystemContext);

        // 为系统进程设置Application实例并启动。
        mActivityManagerService.setSystemProcess();

        //启动传感器服务
        startSensorService();
    }
    //传感器服务时一个jni方法，这部分放到之后再分析，先放一放
    private static native void startSensorService();
```
&ensp;&ensp; 总的来说，引导服务包括：ActivityManagerService，PowerManagerService，LightsService，DisplayManagerService，PackageManagerService，UserManagerService，传感器服务。

#### startCoreServices()：核心服务
```java
/**
 * Starts some essential services that are not tangled up in the bootstrap process.
 */
 private void startCoreServices() {
    // Tracks the battery level.  Requires LightService.
    mSystemServiceManager.startService(BatteryService.class);

    // Tracks application usage stats.
    mSystemServiceManager.startService(UsageStatsService.class);
    mActivityManagerService.setUsageStatsManager(
            LocalServices.getService(UsageStatsManagerInternal.class));

    // Tracks whether the updatable WebView is in a ready state and watches for update installs.
    mWebViewUpdateService = mSystemServiceManager.startService(WebViewUpdateService.class);
 }
这部分是Android的核心服务，包含了电池管理服务，应用使用管理服务以及WebView更新服务。
```
#### startOtherServices()：其他服务
&ensp;&ensp; **startOtherServices()**这部分源码超多，估计得有个1000来行吧，为了让展示更容易看，去掉了一些简单的启动服务代码，去掉了其中的try-catch结构，内部的try-catch大部分都是抛出之后打印信息。
```java
private void startOtherServices() {
        final Context context = mSystemContext;
        //声明各种服务，但没有实例化
        VibratorService vibrator = null;
        IMountService mountService = null;
        ....
        Usb usbOtg = null;
        //读取配置信息
        boolean disableStorage = SystemProperties.getBoolean("config.disable_storage", false);
        boolean disableBluetooth = SystemProperties.getBoolean("config.disable_bluetooth", false);
        ....
        //该值代表是否处于模拟器中
        boolean isEmulator = SystemProperties.get("ro.kernel.qemu").equals("1");

        SystemConfig.getInstance();

        //启动StartSchedulingPolicyService、TelecomLoaderService、TelephonyRegistry、TelephonyRegistry，启动代码省略
        ....     

        mEntropyMixer = new EntropyMixer(context);
        mContentResolver = context.getContentResolver();

        if (!disableCameraService) {
            //启动CameraService
            mSystemServiceManager.startService(CameraService.class);
        }

        //AccountManagerService再启动ContentService，启动代码省略
        ....        
        //InstallSystemProviders
        mActivityManagerService.installSystemProviders();
        //启动VibratorService，震动服务
        vibrator = new VibratorService(context);
        ServiceManager.addService("vibrator", vibrator);

        if (!disableConsumerIr) {
            //启动ConsumerIrService,远程控制服务，如红外遥控
            consumerIr = new ConsumerIrService(context);
            ServiceManager.addService(Context.CONSUMER_IR_SERVICE, consumerIr);
        }
        //启动AlarmManagerService
        mSystemServiceManager.startService(AlarmManagerService.class);

        //初始化看门狗
        final Watchdog watchdog = Watchdog.getInstance();
        watchdog.init(context, mActivityManagerService);

        //启动InputManagerService
        inputManager = new InputManagerService(context);

        //启动WindowManagerService
        wm = WindowManagerService.main(context, inputManager,
                mFactoryTestMode != FactoryTest.FACTORY_TEST_LOW_LEVEL,
                !mFirstBoot, mOnlyCore);
        ServiceManager.addService(Context.WINDOW_SERVICE, wm);
        ServiceManager.addService(Context.INPUT_SERVICE, inputManager);

        if (!disableVrManager) {
            //启动VrManagerService
            mSystemServiceManager.startService(VrManagerService.class);
        }
        mActivityManagerService.setWindowManager(wm);

        inputManager.setWindowManagerCallbacks(wm.getInputMonitor());
        inputManager.start();

        mDisplayManagerService.windowManagerAndInputReady();

        if (isEmulator) {
            //模拟器情况跳过蓝牙服务
        } else if (mFactoryTestMode == FactoryTest.FACTORY_TEST_LOW_LEVEL) {
            Slog.i(TAG, "没有蓝牙服务(出厂测试)");
        } else if (!context.getPackageManager().hasSystemFeature
                   (PackageManager.FEATURE_BLUETOOTH)) {
            Slog.i(TAG, "没有蓝牙服务(蓝牙硬件不存在)");
        } else if (disableBluetooth) {
            Slog.i(TAG, "蓝牙服务被配置禁用");
        } else {
            //启动BluetoothService
            mSystemServiceManager.startService(BluetoothService.class);
        }
        
        //启动MetricsLoggerService，IpConnectivityMetrics，PinnerService
        ....


        // UI显示所需的服务。
        if (mFactoryTestMode != FactoryTest.FACTORY_TEST_LOW_LEVEL) {
            //在FACTORY_TEST_LOW_LEVEL模式下，有些服务不需要启动，此处是非FACTORY_TEST_LOW_LEVEL情况下
            //启动InputMethodManagerService
            mSystemServiceManager.startService(InputMethodManagerService.Lifecycle.class);
            //启动AccessibilityManagerService
            ServiceManager.addService(Context.ACCESSIBILITY_SERVICE,
                        new AccessibilityManagerService(context));
        }
        //进行一些尺寸初始化
        wm.displayReady();

        if (mFactoryTestMode != FactoryTest.FACTORY_TEST_LOW_LEVEL) {
            if (!disableStorage &&
                !"0".equals(SystemProperties.get("system_init.startmountservice"))) {
                //NotificationManagerService依赖于MountService(用于媒体/ usb通知)，所以我们必须首先启动MountService。
                mSystemServiceManager.startService(MOUNT_SERVICE_CLASS);
                mountService = IMountService.Stub.asInterface(
                            ServiceManager.getService("mount"));
            }
        }

        //启动UiModeManagerService
        mSystemServiceManager.startService(UiModeManagerService.class);
        //mOnlyCore 用于判断是否只扫描系统的目录
        if (!mOnlyCore) {
            //检查system app是否需要更新，需要则更新
            mPackageManagerService.updatePackagesIfNeeded();
        }

        //检查system app是否需要fs裁剪
        mPackageManagerService.performFstrimIfNeeded();

        if (mFactoryTestMode != FactoryTest.FACTORY_TEST_LOW_LEVEL) {
            if (!disableNonCoreServices) {
                //启动LockSettingsService
                mSystemServiceManager.startService(LOCK_SETTINGS_SERVICE_CLASS);
                lockSettings = ILockSettings.Stub.asInterface(
                            ServiceManager.getService("lock_settings"));

                if (!SystemProperties.get(PERSISTENT_DATA_BLOCK_PROP).equals("")) {
                    //启动PersistentDataBlockService
                    mSystemServiceManager.startService(PersistentDataBlockService.class);
                }

                //启动DeviceIdleController
                mSystemServiceManager.startService(DeviceIdleController.class);

                //启动DevicePolicyManagerService
                mSystemServiceManager.startService(DevicePolicyManagerService.Lifecycle.class);
            }
            //disableSystemUI表示是否禁用系统界面
            if (!disableSystemUI) {
                //启动StatusBarManagerService:状态栏服务
                statusBar = new StatusBarManagerService(context, wm);
                ServiceManager.addService(Context.STATUS_BAR_SERVICE, statusBar);
            }
            //disableNonCoreServices代表是否关闭非核心服务
            if (!disableNonCoreServices) {
                //启动ClipboardService
                ServiceManager.addService(Context.CLIPBOARD_SERVICE,
                        new ClipboardService(context));
            }
            //disableNetwork代表是否禁用网络
            if (!disableNetwork) {
                //启动NetworkManagementService
                networkManagement = NetworkManagementService.create(context);
                ServiceManager.addService(Context.NETWORKMANAGEMENT_SERVICE, networkManagement);
            }
            //disableTextServices表示是否禁用文字服务
            if (!disableNonCoreServices && !disableTextServices) {
                //启动TextServicesManagerService
                mSystemServiceManager.startService(TextServicesManagerService.Lifecycle.class);
            }

            if (!disableNetwork) {
                //启动NetworkScoreService
                networkScore = new NetworkScoreService(context);
                ServiceManager.addService(Context.NETWORK_SCORE_SERVICE, networkScore);

                //启动NetworkStatsService
                networkStats = NetworkStatsService.create(context, networkManagement);
                ServiceManager.addService(Context.NETWORK_STATS_SERVICE, networkStats);

                //启动NetworkPolicyManagerService
                networkPolicy = new NetworkPolicyManagerService(context,
                        mActivityManagerService, networkStats, networkManagement);
                ServiceManager.addService(Context.NETWORK_POLICY_SERVICE, networkPolicy);

                if (context.getPackageManager().hasSystemFeature(PackageManager.FEATURE_WIFI_NAN)) {
                    /启动WifiNanServic
                    mSystemServiceManager.startService(WIFI_NAN_SERVICE_CLASS);
                } else {
                    Slog.i(TAG, "无Wi-Fi NAN服务(不支持NAN)");
                }
                //启动WifiP2pService、WifiService和WifiScanningService
                mSystemServiceManager.startService(WIFI_P2P_SERVICE_CLASS);
                mSystemServiceManager.startService(WIFI_SERVICE_CLASS);
                mSystemServiceManager.startService(
                 "com.android.server.wifi.scanner.WifiScanningService");

                if (!disableRtt) {
                    //启动RttService
                    mSystemServiceManager.startService("com.android.server.wifi.RttService");
                }
                //查看是否包含以太网或者USB HOST模式
                if (mPackageManager.hasSystemFeature(PackageManager.FEATURE_ETHERNET) ||
                    mPackageManager.hasSystemFeature(PackageManager.FEATURE_USB_HOST)) {
                    //启动EthernetService
                    mSystemServiceManager.startService(ETHERNET_SERVICE_CLASS);
                    if (mPackageManager.hasSystemFeature(PackageManager.FEATURE_PPPOE)) {
                        //启动PppoeService mSystemServiceManager.startService(PPPOE_SERVICE_CLASS);
                    }
                }

                //启动ConnectivityService
                connectivity = new ConnectivityService(
                        context, networkManagement, networkStats, networkPolicy);
                ServiceManager.addService(Context.CONNECTIVITY_SERVICE, connectivity);
                networkStats.bindConnectivityManager(connectivity);
                networkPolicy.bindConnectivityManager(connectivity);

                //启动NsdService
                serviceDiscovery = NsdService.create(context);
                ServiceManager.addService(
                            Context.NSD_SERVICE, serviceDiscovery);
            }

            if (!disableNonCoreServices) {
                //启动UpdateLockService
                ServiceManager.addService(Context.UPDATE_LOCK_SERVICE,
                            new UpdateLockService(context));
            }

            if (!disableNonCoreServices) {
                //启动RecoverySystemService
                mSystemServiceManager.startService(RecoverySystemService.class);
            }

            ....
            //启动NotificationManagerService
            mSystemServiceManager.startService(NotificationManagerService.class);
            notification = INotificationManager.Stub.asInterface(
                    ServiceManager.getService(Context.NOTIFICATION_SERVICE));
            networkPolicy.bindNotificationManager(notification);
            
            //启动DeviceStorageMonitorService
            mSystemServiceManager.startService(DeviceStorageMonitorService.class);   if (!disableLocation) {
                //启动LocationManagerService
                location = new LocationManagerService(context);
                ServiceManager.addService(Context.LOCATION_SERVICE, location);

                //启动CountryDetectorService
                countryDetector = new CountryDetectorService(context);
                ServiceManager.addService(Context.COUNTRY_DETECTOR, countryDetector);
            }

            if (!disableNonCoreServices && !disableSearchManager) {
                //启动SearchManagerService
                mSystemServiceManager.startService(SEARCH_MANAGER_SERVICE_CLASS);
            }
            //启动DropBoxManagerService
            mSystemServiceManager.startService(DropBoxManagerService.class);

            if (!disableNonCoreServices && context.getResources().getBoolean(
                        R.bool.config_enableWallpaperService)) {
                //启动WallpaperManagerService
                mSystemServiceManager.startService(WALLPAPER_SERVICE_CLASS);
            }
            //启动AudioService
            mSystemServiceManager.startService(AudioService.Lifecycle.class);

            if (!disableNonCoreServices && !isBox ) {
                //启动DockObserver
                mSystemServiceManager.startService(DockObserver.class);

                if (context.getPackageManager().hasSystemFeature(PackageManager.FEATURE_WATCH)) {
                    mSystemServiceManager.startService(THERMAL_OBSERVER_CLASS);
                }
            }

            //启动WiredAccessoryManager
            // 监听有线耳机的变化
            inputManager.setWiredAccessoryCallbacks(
                    new WiredAccessoryManager(context, inputManager));

            if (!disableNonCoreServices) {
                if (mPackageManager.hasSystemFeature(PackageManager.FEATURE_MIDI)) {
                    // 启动 MIDI Manager service
                    mSystemServiceManager.startService(MIDI_SERVICE_CLASS);
                }

                if (mPackageManager.hasSystemFeature(PackageManager.FEATURE_USB_HOST)
                        || mPackageManager.hasSystemFeature(
                                PackageManager.FEATURE_USB_ACCESSORY)) {
                    // 管理USB主机和设备支持，启动UsbService
                    mSystemServiceManager.startService(USB_SERVICE_CLASS);
                }

                if (!disableSerial) {
                    //启动SerialService，支持串行端口
                    serial = new SerialService(context);
                    ServiceManager.addService(Context.SERIAL_SERVICE, serial);
                }

                //启动HardwarePropertiesManagerService
                hardwarePropertiesService = new HardwarePropertiesManagerService(context);
                ServiceManager.addService(Context.HARDWARE_PROPERTIES_SERVICE,hardwarePropertiesService);
            }
            //启动TwilightService
            mSystemServiceManager.startService(TwilightService.class);

            if (NightDisplayController.isAvailable(context)) {
                //启动NightDisplayService
                mSystemServiceManager.startService(NightDisplayService.class);
            }
            //启动JobSchedulerService
            mSystemServiceManager.startService(JobSchedulerService.class);
            //启动SoundTriggerService
            mSystemServiceManager.startService(SoundTriggerService.class);
              if (!disableNonCoreServices) {
                if (mPackageManager.hasSystemFeature(PackageManager.FEATURE_BACKUP)) {
                    //启动BackupManagerService
                    mSystemServiceManager.startService(BACKUP_MANAGER_SERVICE_CLASS);
                }

                if (mPackageManager.hasSystemFeature(PackageManager.FEATURE_APP_WIDGETS)
                    || context.getResources().getBoolean(R.bool.config_enableAppWidgetService)) {
                    //启动AppWidgetService
                    mSystemServiceManager.startService(APPWIDGET_SERVICE_CLASS);
                }

                if (mPackageManager.hasSystemFeature(PackageManager.FEATURE_VOICE_RECOGNIZERS)) {
                    //启动VoiceInteractionManagerService
                    mSystemServiceManager.startService(VOICE_RECOGNITION_MANAGER_SERVICE_CLASS);
                }

                if (GestureLauncherService.isGestureLauncherEnabled(context.getResources())) {
                    Slog.i(TAG, "手势启动服务");
                    //启动GestureLauncherService
                    mSystemServiceManager.startService(GestureLauncherService.class);
                }
                usbOtg = new Usb(context);
                ServiceManager.addService("usb_otg", usbOtg);
                usbOtg.start();
                //启动SensorNotificationService和ContextHubSystemService
                mSystemServiceManager.startService(SensorNotificationService.class);
                mSystemServiceManager.startService(ContextHubSystemService.class);
            }
        
            //启动DiskStatsService
            ServiceManager.addService("diskstats", new DiskStatsService(context));
           if (!disableSamplingProfiler) {
                //启动SamplingProfilerService
                ServiceManager.addService("samplingprofiler",
                                new SamplingProfilerService(context));
            }

            if (!disableNetwork && !disableNetworkTime) {
                //启动NetworkTimeUpdateService
                networkTimeUpdater = new NetworkTimeUpdateService(context);
                ServiceManager.addService("network_time_update_service", networkTimeUpdater);
            }
            //启动CommonTimeManagementService
            commonTimeMgmtService = new CommonTimeManagementService(context);
            ServiceManager.addService("commontime_management", commonTimeMgmtService);
            
            if (!disableNetwork) {
                CertBlacklister blacklister = new CertBlacklister(context);
            }

            if (!disableNetwork && !disableNonCoreServices && EmergencyAffordanceManager.ENABLED) {
                //启动EmergencyAffordanceService
                mSystemServiceManager.startService(EmergencyAffordanceService.class);
            }

            if (!disableNonCoreServices) {
                //启动DreamManagerService
                mSystemServiceManager.startService(DreamManagerService.class);
            }
            if (!disableNonCoreServices && ZygoteInit.PRELOAD_RESOURCES) {
                //启动AssetAtlasService
                atlas = new AssetAtlasService(context);
                ServiceManager.addService(AssetAtlasService.ASSET_ATLAS_SERVICE, atlas);
            }

            if (!disableNonCoreServices) {
                //启动GraphicsStatsService
                ServiceManager.addService(GraphicsStatsService.GRAPHICS_STATS_SERVICE,
                        new GraphicsStatsService(context));
            }

            if (mPackageManager.hasSystemFeature(PackageManager.FEATURE_PRINTING)) {
                    //启动PrintManagerService
                    mSystemServiceManager.startService(PRINT_MANAGER_SERVICE_CLASS);
            }
            //启动RestrictionsManagerService
            mSystemServiceManager.startService(RestrictionsManagerService.class);
            //启动MediaSessionService
            mSystemServiceManager.startService(MediaSessionService.class);

            if (mPackageManager.hasSystemFeature(PackageManager.FEATURE_HDMI_CEC)) {
                //启动HdmiControlService
                mSystemServiceManager.startService(HdmiControlService.class);
            }

            if (mPackageManager.hasSystemFeature(PackageManager.FEATURE_LIVE_TV)) {
                //启动TvInputManagerService
                mSystemServiceManager.startService(TvInputManagerService.class);
            }

            if (mPackageManager.hasSystemFeature(PackageManager.FEATURE_PICTURE_IN_PICTURE)) {
                //启动MediaResourceMonitorService
                mSystemServiceManager.startService(MediaResourceMonitorService.class);
            }

            if (mPackageManager.hasSystemFeature(PackageManager.FEATURE_LEANBACK)) {
                //启动TvRemoteService
                mSystemServiceManager.startService(TvRemoteService.class);
            }
            if (!disableNonCoreServices) {
                //启动MediaRouterService
                mediaRouter = new MediaRouterService(context);
                ServiceManager.addService(Context.MEDIA_ROUTER_SERVICE, mediaRouter);

                if (!disableTrustManager) {
                    //启动TrustManagerService
                    mSystemServiceManager.startService(TrustManagerService.class);
                }

                if (mPackageManager.hasSystemFeature(PackageManager.FEATURE_FINGERPRINT)) {
                    //启动FingerprintService
                    mSystemServiceManager.startService(FingerprintService.class);
                }
                //启动BackgroundDexOptService
                BackgroundDexOptService.schedule(context);
            }
            // 启动LauncherAppsService和ShortcutService，LauncherAppsService会使用到ShortcutService
            mSystemServiceManager.startService(ShortcutService.Lifecycle.class);
            mSystemServiceManager.startService(LauncherAppsService.class);
        }
      if (!disableNonCoreServices && !disableMediaProjection) {
            //启动MediaProjectionManagerService
            mSystemServiceManager.startService(MediaProjectionManagerService.class);
        }

	//判断是否是手表之类的穿戴设备，if内部是一些穿戴设备服务
        if (context.getPackageManager().hasSystemFeature(PackageManager.FEATURE_WATCH)) {
            //启动WearBluetoothService、WearWifiMediatorService
            mSystemServiceManager.startService(WEAR_BLUETOOTH_SERVICE_CLASS);
            mSystemServiceManager.startService(WEAR_WIFI_MEDIATOR_SERVICE_CLASS);
            if (SystemProperties.getBoolean("config.enable_cellmediator", false)) {
                //启动WearCellularMediatorService
                mSystemServiceManager.startService(WEAR_CELLULAR_MEDIATOR_SERVICE_CLASS);
            }
          if (!disableNonCoreServices) {
              //启动WearTimeService
              mSystemServiceManager.startService(WEAR_TIME_SERVICE_CLASS);
          }
        }

        // 为system_server进程启用JIT
        VMRuntime.getRuntime().startJitCompilation();

        // 启动MmsServiceBroker:彩信服务
        mmsService = mSystemServiceManager.startService(MmsServiceBroker.class);

        if (Settings.Global.getInt(mContentResolver, Settings.Global.DEVICE_PROVISIONED, 0) == 0 ||
                UserManager.isDeviceInDemoMode(mSystemContext)) {
            mSystemServiceManager.startService(RetailDemoModeService.class);
        }

        // 现在开始启动应用程序进程

        //震动服务准备完成
        vibrator.systemReady();
        //锁设置服务准备完成
        if (lockSettings != null) {
            lockSettings.systemReady();
        }
        // DevicePolicyManager需要初始化
        mSystemServiceManager.startBootPhase(SystemService.PHASE_LOCK_SETTINGS_READY);
        mSystemServiceManager.startBootPhase(SystemService.PHASE_SYSTEM_SERVICES_READY);
        //WindowManagerService准备完成
        wm.systemReady();
        if (safeMode) {
            mActivityManagerService.showSafeModeOverlay();
        }
        //我们在WindowManagerService准备完成前就已经使用到了config，这里手动更新此上下文配置
        Configuration config = wm.computeNewConfiguration();
        DisplayMetrics metrics = new DisplayMetrics();
        WindowManager w = (WindowManager)context.getSystemService(Context.WINDOW_SERVICE);
        w.getDefaultDisplay().getMetrics(metrics);
        context.getResources().updateConfiguration(config, metrics);

        // 系统上下文的主题可能与配置有关。
        final Theme systemTheme = context.getTheme();
        if (systemTheme.getChangingConfigurations() != 0) {
            systemTheme.rebase();
        }
        //PowerManagerService准备好了
        mPowerManagerService.systemReady(mActivityManagerService.getAppOpsService());
        //PackageManagerService准备好了
        mPackageManagerService.systemReady();
        //DisplayManagerService准备好了
        mDisplayManagerService.systemReady(safeMode, mOnlyCore);

        ....

        mActivityManagerService.systemReady(new Runnable() {
            @Override
            public void run() {
                mSystemServiceManager.startBootPhase(
                        SystemService.PHASE_ACTIVITY_MANAGER_READY);

                mActivityManagerService.startObservingNativeCrashes();
                if (!mOnlyCore) {
                    mWebViewUpdateService.prepareWebViewInSystemServer();
                }
                //启动SystemUI
                startSystemUi(context);
                //一系列systemReady方法
                if (networkScoreF != null) networkScoreF.systemReady();
                if (networkManagementF != null) networkManagementF.systemReady();
                if (networkStatsF != null) networkStatsF.systemReady();
                if (networkPolicyF != null) networkPolicyF.systemReady();
                if (connectivityF != null) connectivityF.systemReady();
                Watchdog.getInstance().start();//看门狗开始工作

                // 现在可以让各种系统服务启动它们的第三方代码了…
                mSystemServiceManager.startBootPhase(
                        SystemService.PHASE_THIRD_PARTY_APPS_CAN_START);

                if (locationF != null) locationF.systemRunning();
                if (countryDetectorF != null) countryDetectorF.systemRunning();
                if (networkTimeUpdaterF != null) networkTimeUpdaterF.systemRunning();
                if (commonTimeMgmtServiceF != null) {
                        commonTimeMgmtServiceF.systemRunning();
                    }
                if (atlasF != null) atlasF.systemRunning();
                if (inputManagerF != null) inputManagerF.systemRunning();
                if (telephonyRegistryF != null) telephonyRegistryF.systemRunning();
                if (mediaRouterF != null) mediaRouterF.systemRunning();

                if (mmsServiceF != null) mmsServiceF.systemRunning();
                if (networkScoreF != null) networkScoreF.systemRunning();
            }
        });
    }
```
&ensp;&ensp; 这部分启动了大量的服务。之后**System**进程就启动完成了，并进入**Looper.loop()**状态，等待消息的到来；

这里画了一个简单的流程图，可以简单看下：
![image](/img/in_post/system_process_img.png)


