---
layout: post
title:  "Android应用程序启动入口ActivityThread.main流程分析"
author: "陈宇瀚"
date:   2021-01-16 12:04:06 +0800
header-img: "img/img-head/img-head-boot-process.jpg"
categories: article
tags:
  - Android
  - System
  - 应用程序启动流程
  - ActivityThread.main
---
&ensp;&ensp; 之前[Android应用程序进程的启动过程](https://yuhanchen5027.github.io/article/2021/01/13/Android-app-process-analysis//)文章内有说过，当一个Android应用程序进程启动后，应用程序进程的入口就是**ActivityThread**类的**main**函数，**ActivityThread**的作用管理应用的**主进程**的执行，并根据**AMS**的要求，通过**IApplicationThread**接口负责调度和执行**Activities**和**Broadcasts**和其他操作，接下来这里会从**main**方法开始分析应用的启动流程;

&ensp;&ensp; *以下源码基于rk3399_industry Android7.1.2*
## ActivityThread.main
&ensp;&ensp; 总的来说这里主要是要向**AMS**发送一个进程启动完成通知，然后接收到回复在进行之后的步骤。
```java
public final class ActivityThread {
    final ApplicationThread mAppThread = new ApplicationThread();
    public static void main(String[] args) {
        Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "ActivityThreadMain");
        ....
        //创建主线程的Looper对象
        Looper.prepareMainLooper();
    
        //创建ActivityThread实例，创建会同时在它内部创建一个ApplicationThread对象mAppThread
        //mAppThread是一个Binder本地对象
        //AMS就是通过该对象来和应用程序进程通信的
        ActivityThread thread = new ActivityThread();
        //调用attch函数向AMS发送一个启动完成的通知
        thread.attach(false);

        if (sMainThreadHandler == null) {
            sMainThreadHandler = thread.getHandler();
        }

        Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
        //进入消息循环
        Looper.loop();
        ....
    }

    private void attach(boolean system) {
        //此时传入的system为false
        sCurrentActivityThread = this;
        mSystemThread = system;
        if (!system) {
            //开启虚拟机的jit即时编译功能
            ViewRootImpl.addFirstDrawHandler(new Runnable() {
                @Override
                public void run() {
                    ensureJitEnabled();
                }
            });
            android.ddm.DdmHandleAppName.setAppName("<pre-initialized>",
                                                    UserHandle.myUserId());
            RuntimeInit.setApplicationObject(mAppThread.asBinder());
            //获得一个AMS的代理对象ActivityManagerProxy
            final IActivityManager mgr = ActivityManagerNative.getDefault();
            try {
                //向AMS发送一个进程间通信请求
                mgr.attachApplication(mAppThread);
            } catch (RemoteException ex) {
                ....
            }
            // 观察是否快接近heap的上限值
            BinderInternal.addGcWatcher(new Runnable() {
                @Override public void run() {
                    if (!mSomeActivitiesChanged) {
                        return;
                    }
                    Runtime runtime = Runtime.getRuntime();
                    long dalvikMax = runtime.maxMemory();
                    long dalvikUsed = runtime.totalMemory() - runtime.freeMemory();
                    //判断已用内存是否超过最大内存的3/4
                    if (dalvikUsed > ((3*dalvikMax)/4)) {
                        if (DEBUG_MEMORY_TRIM) Slog.d(TAG, "Dalvik max=" + (dalvikMax/1024)
                                + " total=" + (runtime.totalMemory()/1024)
                                + " used=" + (dalvikUsed/1024));
                        mSomeActivitiesChanged = false;
                        try {
                            //当已用内存超过最大内存的3/4,则请求释放内存空间
                            mgr.releaseSomeActivities(mAppThread);
                        } catch (RemoteException e) {
                            throw e.rethrowFromSystemServer();
                        }
                    }
                }
            });
        } else {
            ....
        }

        //添加dropbox日志到libcore
        DropBox.setReporter(new DropBoxReporter());
        ....
    }
}
```
&ensp;&ensp; 接下来我们就会进入到**AMS**的代理对象**ActivityManagerProxy**的**attachApplication**函数向**AMS**发送一个进程间通信请求;

## ActivityManagerProxy.attachApplication
**frameworks/base/core/java/android/app/ActivityManagerNative.java**
```java
public abstract class ActivityManagerNative extends Binder implements IActivityManager{

    ....
    class ActivityManagerProxy implements IActivityManager{
        ....
        public void attachApplication(IApplicationThread app) throws RemoteException{
            //将传入的参数写入data
            Parcel data = Parcel.obtain();
            Parcel reply = Parcel.obtain();
            data.writeInterfaceToken(IActivityManager.descriptor);
            data.writeStrongBinder(app.asBinder());
            //通过Binder代理对象mRemote向AMS发送类型为ATTACH_APPLICATION_TRANSACTION的进程间通信请求
            //通知应用程序进程启动完成
            mRemote.transact(ATTACH_APPLICATION_TRANSACTION, data, reply, 0);
            reply.readException();
            data.recycle();
            reply.recycle();
        }
        
    }
}
```
## ActivityManagerService.attachApplication
**frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java**
```java
public final class ActivityManagerService extends ActivityManagerNative
        implements Watchdog.Monitor, BatteryStatsImpl.BatteryCallback {
    ....
    public final void attachApplication(IApplicationThread thread) {
        synchronized (this) {
            //AMS接收到应用程序进程启动完成
            int callingPid = Binder.getCallingPid();
            final long origId = Binder.clearCallingIdentity();
            //执行启动应用的主Activity的操作
            attachApplicationLocked(thread, callingPid);
            Binder.restoreCallingIdentity(origId);
        }
    }
    //保存进程pid的的对象，在应用程序进程创建完毕后会以pid为关键字，将对应的ProcessRecord对象存入进去
    final SparseArray<ProcessRecord> mPidsSelfLocked = new SparseArray<ProcessRecord>();
   private final boolean attachApplicationLocked(IApplicationThread thread,
            int pid) {
        //参数pid为应用程序进程的pid
        ProcessRecord app;
        //获得当前时间戳
        long startTime = SystemClock.uptimeMillis();
        if (pid != MY_PID && pid >= 0) {
            synchronized (mPidsSelfLocked) {
                //获得对应pid所对应的ProcessRecord对象赋值给app
                app = mPidsSelfLocked.get(pid);
            }
        } else {
            app = null;
        }
        ....
        final String processName = app.processName;
        ....
        //对app对象进行初始化
        //这个调用内会将app.thread = thread，即指向传入的ApplicationThread的代理对象
        //这样AMS就可以通过这个thread和新创建的应用程序进程进行通信了
        app.makeActive(thread, mProcessStats);
        app.curAdj = app.setAdj = app.verifiedAdj = ProcessList.INVALID_ADJ;
        app.curSchedGroup = app.setSchedGroup = ProcessList.SCHED_GROUP_DEFAULT;
        app.forcingToForeground = null;
        updateProcessForegroundLocked(app, false, false);
        app.hasShownUi = false;
        app.debugging = false;
        app.cached = false;
        app.killedByAm = false;
        app.killed = false;
        ....
        //这个PROC_START_TIMEOUT_MSG是一个判断是否启动超时的消息，在创建应用程序进程之前发送
        //这里是删除该消息，因为应用程序已经在规定时间内启动起来了
        mHandler.removeMessages(PROC_START_TIMEOUT_MSG, app);

        boolean normalMode = mProcessesReady || isAllowedWhileBooting(app.info);
        ....
        try {
            ....
            //完成前面的大量准备工作后，重点是这个方法，通知要启动的Application组件
            thread.bindApplication(processName, appInfo, providers, app.instrumentationClass,
                    profilerInfo, app.instrumentationArguments, app.instrumentationWatcher,
                    app.instrumentationUiAutomationConnection, testMode,
                    mBinderTransactionTrackingEnabled, enableTrackAllocation,
                    isRestrictedBackupMode || !normalMode, app.persistent,
                    new Configuration(mConfiguration), app.compat,
                    getCommonServicesLocked(app.isolated),
                                   mCoreSettingsObserver.getCoreSettingsLocked());
            ....
        } catch (Exception e) {
            ....
        }
        ....
        if (normalMode) {
            try {
                //在Application启动完毕后这里启动对应APP的主Activity
                if (mStackSupervisor.attachApplicationLocked(app)) {
                    didSomething = true;
                }
            } catch (Exception e) {
                Slog.wtf(TAG, "Exception thrown launching activities in " + app, e);
                badApp = true;
            }
        }
        
        //当Activity启动完成后启动所有运行在此进程的service
        if (!badApp) {
            try {
                didSomething |= mServices.attachApplicationLocked(app, processName);
            } catch (Exception e) {
                ....
                badApp = true;
            }
        }
        ....
    }
```
**thread**是一个**ApplicationThreadProxy**的**Binder**代理对象，所以这里最后会调用到**ApplicationThreadProxy.bindApplication**函数来向之前的应用程序进程发送一个进程间通信请求。

## Application的启动
### ApplicationThreadProxy.bindApplication
**frameworks/base/core/java/android/app/ApplicationThreadNative.java**
```java
public abstract class ApplicationThreadNative extends Binder
        implements IApplicationThread {
    ....
    class ApplicationThreadProxy implements IApplicationThread {
        ....
        public final void bindApplication(String packageName, ApplicationInfo info,
            List<ProviderInfo> providers, ComponentName testName, ProfilerInfo profilerInfo,
            Bundle testArgs, IInstrumentationWatcher testWatcher,
            IUiAutomationConnection uiAutomationConnection, int debugMode,
            boolean enableBinderTracking, boolean trackAllocation, boolean restrictedBackupMode,
            boolean persistent, Configuration config, CompatibilityInfo compatInfo,
            Map<String, IBinder> services, Bundle coreSettings) throws RemoteException {
            Parcel data = Parcel.obtain();
            data.writeInterfaceToken(IApplicationThread.descriptor);
            data.writeString(packageName);
            info.writeToParcel(data, 0);
            data.writeTypedList(providers);
            if (testName == null) {
                data.writeInt(0);
            } else {
                data.writeInt(1);
                testName.writeToParcel(data, 0);
            }
            if (profilerInfo != null) {
                data.writeInt(1);
                profilerInfo.writeToParcel(data,     Parcelable.PARCELABLE_WRITE_RETURN_VALUE);
            } else {
                data.writeInt(0);
            }
            data.writeBundle(testArgs);
            data.writeStrongInterface(testWatcher);
            data.writeStrongInterface(uiAutomationConnection);
            data.writeInt(debugMode);
            data.writeInt(enableBinderTracking ? 1 : 0);
            data.writeInt(trackAllocation ? 1 : 0);
            data.writeInt(restrictedBackupMode ? 1 : 0);
            data.writeInt(persistent ? 1 : 0);
            config.writeToParcel(data, 0);
            compatInfo.writeToParcel(data, 0);
            data.writeMap(services);
            data.writeBundle(coreSettings);
            mRemote.transact(BIND_APPLICATION_TRANSACTION, data, null,
                    IBinder.FLAG_ONEWAY);
            data.recycle();
        }
        
    }
}
```
&ensp;&ensp; 这部分就是将传入的参数写入**Parcel**对象**data**，然后通过**Binder**代理对象**mRemote**向前面的应用程序进程发送一个类行为**BIND_APPLICATION_TRANSACTION**的进程间通信请求。之后就又回到**ApplicationThread.bindApplication**函数。
### ApplicationThread.bindApplication
**frameworks/base/core/java/android/app/ActivityThread.java**
```java
public final class ActivityThread {
    ....
    private class ApplicationThread extends ApplicationThreadNative {
        ....
         public final void bindApplication(String processName, ApplicationInfo appInfo,
                List<ProviderInfo> providers, ComponentName instrumentationName,
                ProfilerInfo profilerInfo, Bundle instrumentationArgs,
                IInstrumentationWatcher instrumentationWatcher,
                IUiAutomationConnection instrumentationUiConnection, int debugMode,
                boolean enableBinderTracking, boolean trackAllocation,
                boolean isRestrictedBackupMode, boolean persistent, Configuration config,
                CompatibilityInfo compatInfo, Map<String, IBinder> services, Bundle coreSettings) {

            if (services != null) {
                // 在服务管理器中设置服务缓存
                ServiceManager.initServiceCache(services);
            }

            setCoreSettings(coreSettings);
            //将传入的应用参数封装成data
            AppBindData data = new AppBindData();
            data.processName = processName;
            data.appInfo = appInfo;
            data.providers = providers;
            data.instrumentationName = instrumentationName;
            data.instrumentationArgs = instrumentationArgs;
            data.instrumentationWatcher = instrumentationWatcher;
            data.instrumentationUiAutomationConnection = instrumentationUiConnection;
            data.debugMode = debugMode;
            data.enableBinderTracking = enableBinderTracking;
            data.trackAllocation = trackAllocation;
            data.restrictedBackupMode = isRestrictedBackupMode;
            data.persistent = persistent;
            data.config = config;
            data.compatInfo = compatInfo;
            data.initProfilerInfo = profilerInfo;
            //发送了一个BIND_APPLICATION消息，之后来看消息处理
            sendMessage(H.BIND_APPLICATION, data);
        }
        ....
         
    }
    ....
    public void handleMessage(Message msg) {
        switch (msg.what) {
            ....
            case BIND_APPLICATION:
                    Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "bindApplication");
                    AppBindData data = (AppBindData)msg.obj;
                    //在跳转到handleBindApplication方法
                    handleBindApplication(data);
                    break;
            ....
            
        }
    }
    
    //handleBindApplication函数非常的长，这里取重点部分讲解
    private void handleBindApplication(AppBindData data) {
        ....

        mBoundApplication = data;
        ....
        /初始化一个ContextImpl对象appContext
        final ContextImpl appContext = ContextImpl.createAppContext(this, data.info);
        //创建一个Application对象
        Application app = data.info.makeApplication(data.restrictedBackupMode, null);
        mInitialApplication = app;
    
        try{
            //调用callApplicationOnCreate，方法内部其实就是调用app.onCreate方法
            //之后就进入了Application的生命周期的onCreate()方法了
            //mInstrumentation是一个代理层，最终Apllication的创建，Activity的创建，以及生命周期都会经过这个对象去执行
            mInstrumentation.callApplicationOnCreate(app);
            ....
        } catch (Exception e) {
            ....
        }
    }
}
```
&ensp;&ensp; 这部分**callApplicationOnCreate**比较简单，所以还是关注下如何创建的**Application**对象，**data**是一个类型为**AppBindData**的对象，其**info**变量的类行为**LoadedApk**，**LoadedApk**对象可以说是**APK**文件在内存中的表示，所以我们要来简单看下**LoadedApk.makeApplication**函数；
#### LoadedApk.makeApplication
**frameworks/base/core/java/android/app/LoadedApk.java**
```java
public final class LoadedApk {
    ....
    public Application makeApplication(boolean forceDefaultAppClass,
            Instrumentation instrumentation) {
        if (mApplication != null) {
            return mApplication;
        }
        Application app = null;
        //获得Application的类名
        String appClass = mApplicationInfo.className;
        if (forceDefaultAppClass || (appClass == null)) {
            appClass = "android.app.Application";
        }

        try {
            java.lang.ClassLoader cl = getClassLoader();
            if (!mPackageName.equals("android")) {
                initializeJavaContextClassLoader();
            }
            //创建上下文ContextImpl
            ContextImpl appContext = ContextImpl.createAppContext(mActivityThread, this);
            //又回到了当前应用程秀进程的Instrumentation对象中
            app = mActivityThread.mInstrumentation.newApplication(
                    cl, appClass, appContext);
            appContext.setOuterContext(app);
        } catch (Exception e) {
            ....
        }
        mActivityThread.mAllApplications.add(app);
        mApplication = app;
        ....
        return app;
    }
    
}
```
#### Instrumentation.newApplication
**frameworks/base/core/java/android/app/Instrumentation.java**
```java
public class Instrumentation {
    ....
    public Application newApplication(ClassLoader cl, String className, Context context)
            throws InstantiationException, IllegalAccessException,
            ClassNotFoundException {
        return newApplication(cl.loadClass(className), context);
    }
    
    static public Application newApplication(Class<?> clazz, Context context)
            throws InstantiationException, IllegalAccessException,
            ClassNotFoundException {
        //通过反射创建一个Application对象
        Application app = (Application)clazz.newInstance();
        //调用其attach方法，将app与context绑定
        app.attach(context);
        return app;
    }
    ....
    
}
```

#### Application.attach & ContextWrapper.attachBaseContext
**frameworks/base/core/java/android/app/Application.java**
**frameworks/base/core/java/android/content/ContextWrapper.java**
```java
public class Application extends ContextWrapper implements ComponentCallbacks2 {
    ....
    final void attach(Context context) {
        attachBaseContext(context);
        mLoadedApk = ContextImpl.getImpl(context).mPackageInfo;
    }
    ....
}
public class ContextWrapper extends Context {
    ....
    protected void attachBaseContext(Context base) {
        if (mBase != null) {
            throw new IllegalStateException("Base context already set");
        }
        mBase = base;
    }   
    ....
}
```
&ensp;&ensp; 这样就将**Application**和之前创建的上下文绑定起来了。

## Activity和Service的启动
&ensp;&ensp; **Application**启动完成后，回到**AMS**的**attachApplication**函数内，之后会调用**mStackSupervisor.attachApplicationLocked(app)**和**mServices.attachApplicationLocked(app,processName)**分别启动**Activity**和**Service**;
### Activity启动：ActivityStackSupervisor.attachApplicationLocked 
**frameworks/base/services/core/java/com/android/server/am/ActivityStackSupervisor.java**
```java
public final class ActivityStackSupervisor implements DisplayListener {
    ....
    boolean attachApplicationLocked(ProcessRecord app) throws RemoteException {   
        //获得进程名赋值给processName
        final String processName = app.processName;
        boolean didSomething = false;
        for (int displayNdx = mActivityDisplays.size() - 1; displayNdx >= 0; --displayNdx) {
            ArrayList<ActivityStack> stacks = mActivityDisplays.valueAt(displayNdx).mStacks;
            for (int stackNdx = stacks.size() - 1; stackNdx >= 0; --stackNdx) {
                final ActivityStack stack = stacks.get(stackNdx);
                if (!isFocusedStack(stack)) {
                    continue;
                }
                //获得位于Activity栈顶端的ActivityRecord对象hr，与它对应的Activity组件就是要启动的Activity组件
                ActivityRecord hr = stack.topRunningActivityLocked();
                if (hr != null) {
                    //判断要启动的Activity的用户I(UID)和进程名是否与传入的ProcessRecord对象app的一致
                    if (hr.app == null && app.uid == hr.info.applicationInfo.uid
                            && processName.equals(hr.processName)) {
                        try {
                            //一致，代表Activity是在app中启动的，调用realStartActivityLocked启动Activity
                            if (realStartActivityLocked(hr, app, true, true)) {
                                didSomething = true;
                            }
                        } catch (RemoteException e) {
                            Slog.w(TAG, "Exception in new application when starting activity "
                                  + hr.intent.getComponent().flattenToShortString(), e);
                            throw e;
                        }
                    }
                }
            }
        }
        if (!didSomething) {
            ensureActivitiesVisibleLocked(null, 0, !PRESERVE_WINDOWS);
        }
        return didSomething;
    }
    ....
    
final boolean realStartActivityLocked(ActivityRecord r, ProcessRecord app,
            boolean andResume, boolean checkConfig) throws RemoteException {
        ....
        //将传入的将要启动的Activity的ActivityRecord对象r的app设置为我们应用程序进程的app对象
        r.app = app;
        ....
        //idx<0代表r不在app的Activity组建列表中
        int idx = app.activities.indexOf(r);
        if (idx < 0) {
            //将Activity添加进app的Activity组建列表activities中
            app.activities.add(r);
        }
        ....
        try {
            ....
            //一些准备工作
            List<ResultInfo> results = null;
            List<ReferrerIntent> newIntents = null;
            if (andResume) {
                results = r.results;
                newIntents = r.newIntents;
            }
            ....
            if (r.isHomeActivity()) {
                // Home进程是任务的根进程.
                mService.mHomeProcess = task.mActivities.get(0).app;
            }
            ....
            //通知前面创建的应用程序进程启动Activity组件(即r所描述的Activity)
            app.thread.scheduleLaunchActivity(new Intent(r.intent), r.appToken,
                    System.identityHashCode(r), r.info, new Configuration(mService.mConfiguration),
                    new Configuration(task.mOverrideConfig), r.compat, r.launchedFromPackage,
                    task.voiceInteractor, app.repProcState, r.icicle, r.persistentState, results,
                    newIntents, !andResume, mService.isNextTransitionForward(), profilerInfo,r.task.taskId);
            ....
       } catch (RemoteException e) {
           ....
       }
         ....
        return true;
    }
}
```
&ensp;&ensp; 在做了一些赋值和一些准备工作之后，就会开始通知前面创建的应用程序进程启动要创建的**Activity**，由源码可知**app**是**ProcessRecord**对象，而**app.thread**对应位**ApplicationThreadProxy**代理类，之后看对应的代码；
### ApplicationThreadProxy.scheduleLaunchActivity
**frameworks/base/core/java/android/app/ApplicationThreadNative.java**
```java
public abstract class ApplicationThreadNative extends Binder
        implements IApplicationThread {
    ....
    class ApplicationThreadProxy implements IApplicationThread {
        ....
        public final void scheduleLaunchActivity(Intent intent, IBinder token, int ident,
            ActivityInfo info, Configuration curConfig, Configuration overrideConfig,
            CompatibilityInfo compatInfo, String referrer, IVoiceInteractor voiceInteractor,
            int procState, Bundle state, PersistableBundle persistentState,
            List<ResultInfo> pendingResults, List<ReferrerIntent> pendingNewIntents,
            boolean notResumed, boolean isForward, ProfilerInfo profilerInfo,int taskId) throws RemoteException {
        Parcel data = Parcel.obtain();
        data.writeInterfaceToken(IApplicationThread.descriptor);
        intent.writeToParcel(data, 0);
        data.writeStrongBinder(token);
        data.writeInt(ident);
        info.writeToParcel(data, 0);
        curConfig.writeToParcel(data, 0);
        if (overrideConfig != null) {
            data.writeInt(1);
            overrideConfig.writeToParcel(data, 0);
        } else {
            data.writeInt(0);
        }
        compatInfo.writeToParcel(data, 0);
        data.writeString(referrer);
        data.writeStrongBinder(voiceInteractor != null ? voiceInteractor.asBinder() : null);
        data.writeInt(procState);
        data.writeBundle(state);
        data.writePersistableBundle(persistentState);
        data.writeTypedList(pendingResults);
        data.writeTypedList(pendingNewIntents);
        data.writeInt(notResumed ? 1 : 0);
        data.writeInt(isForward ? 1 : 0);
        if (profilerInfo != null) {
            data.writeInt(1);
            profilerInfo.writeToParcel(data, Parcelable.PARCELABLE_WRITE_RETURN_VALUE);
        } else {
            data.writeInt(0);
        }
        data.writeInt(taskId);
        //调用Binder代理对象mRemote发送一个SCHEDULE_LAUNCH_ACTIVITY_TRANSACTION的进程间通信请求
        mRemote.transact(SCHEDULE_LAUNCH_ACTIVITY_TRANSACTION, data, null,
                IBinder.FLAG_ONEWAY);
        data.recycle();
    }        
        ....
    }
    ....
}
```
### ApplicationThread.scheduleLaunchActivity
**frameworks/base/core/java/android/app/ActivityThread.java**
```java
public final class ActivityThread {
    ....
    private class ApplicationThread extends ApplicationThreadNative {
         ....
         public final void scheduleLaunchActivity(Intent intent, IBinder token, int ident,
                ActivityInfo info, Configuration curConfig, Configuration overrideConfig,
                CompatibilityInfo compatInfo, String referrer, IVoiceInteractor voiceInteractor,
                int procState, Bundle state, PersistableBundle persistentState,
                List<ResultInfo> pendingResults, List<ReferrerIntent> pendingNewIntents,
                boolean notResumed, boolean isForward, ProfilerInfo profilerInfo,int taskId_param) {

            updateProcessState(procState, false);
            //将要启动的Activity封装成ActivityClientRecord对象r
            ActivityClientRecord r = new ActivityClientRecord();

            r.token = token;
            r.ident = ident;
            r.intent = intent;
            r.referrer = referrer;
            r.voiceInteractor = voiceInteractor;
            r.activityInfo = info;
            r.compatInfo = compatInfo;
            r.state = state;
            r.persistentState = persistentState;

            r.pendingResults = pendingResults;
            r.pendingIntents = pendingNewIntents;

            r.startsNotResumed = notResumed;
            r.isForward = isForward;

            r.profilerInfo = profilerInfo;

            r.overrideConfig = overrideConfig;
            r.taskId = taskId_param;
            updatePendingConfiguration(curConfig);
            //发送类型为LAUNCH_ACTIVITY，带着r的消息给创建的应用程序金层呢的消息队列处理
            sendMessage(H.LAUNCH_ACTIVITY, r);
        }
        ....
        
    }
    
    ....
    public void handleMessage(Message msg) {
         switch (msg.what) {
            ....
            case LAUNCH_ACTIVITY: {
                    //将发送来的obj转成ActivityClientRecord对象r
                    final ActivityClientRecord r = (ActivityClientRecord) msg.obj;
                    //调用getPackageInfoNoCheck获得一个LoadedApk对象保存在r.的成员函数packageInfo中
                    r.packageInfo = getPackageInfoNoCheck(
                            r.activityInfo.applicationInfo, r.compatInfo);
                    //进入函数handleLaunchActivity来启动Activity组件
                    handleLaunchActivity(r, null, "LAUNCH_ACTIVITY");
                } break;
            ....
         }
    }
    ....
    
private void handleLaunchActivity(ActivityClientRecord r, Intent customIntent, String reason) {
        ....

        // 确保我们使用最新的配置运行。
        handleConfigurationChanged(null, null);

        // 创建Activity前进行初始化
        WindowManagerGlobal.initialize();
        // performLaunchActivity函数会将Activity启动起来
        Activity a = performLaunchActivity(r, customIntent);

        if (a != null) {
            r.createdConfig = new Configuration(mConfiguration);
            reportSizeConfigurations(r);
            Bundle oldState = r.state;
            //将Activity组件的状态设置为Resumed，代表它是系统当前显示的Activity
            handleResumeActivity(r.token, false, r.isForward,
                    !r.activity.mFinished && !r.startsNotResumed, r.lastProcessedSeq, reason);

            if (!r.activity.mFinished && r.startsNotResumed) {
                //如果actvity没有finish，同时并不在resume状态，让Activity进入Pause状态，调用onPause生命方法。下方进入这个方法
                performPauseActivityIfNeeded(r, reason);
                ....
            }
        } else {
           ....
        }   
    }   
    ....
//将Activity启动起来
private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
        ....
        //获得要启动的Activity组件的包名和类名赋值给component
        ComponentName component = r.intent.getComponent();
        ....

        Activity activity = null;
        try {
            //通过类加载器将component加载到内存中
            java.lang.ClassLoader cl = r.packageInfo.getClassLoader();
            //构造出一个对应Activity的实例activity
            activity = mInstrumentation.newActivity(
                    cl, component.getClassName(), r.intent);
            ....
        } catch (Exception e) {
            ....
        }
        
        try {
            Application app = r.packageInfo.makeApplication(false, mInstrumentation);

            if (activity != null) {
                //创建一个ContextImpl对象，是创建的Activity的上下文环境
                Context appContext = createBaseContextForActivity(r, activity);
                ....
                //传入参数，调用attach进行Activity的初始化
                activity.attach(appContext, this, getInstrumentation(), r.token,
                        r.ident, app, r.intent, r.activityInfo, title, r.parent,
                        r.embeddedID, r.lastNonConfigurationInstances, config,
                        r.referrer, r.voiceInteractor, window);

                ....
                //通知Activity启动起来，callActivityOnCreate内部会调用Activity的生命周期onCreate()
                mInstrumentation.callActivityOnCreate(activity, r.state);
                //设置一些r的变量
                ....
                }
            }
            ....
            //将r的token对象保存在ActivityThread的成员变量mActivities中
            mActivities.put(r.token, r);
            //r.token是一个Binder代理对象，只想了AMS内部的一个ActivityRecord对象
            //这个ActivityRecord和对象r一样，都是用来描述启动的Activity的，只不过前者用于AMS，后者用于应用程序进程中。

        } catch (SuperNotCalledException e) {
          ....

        } catch (Exception e) {
          ....
        }
        return activity;
    }
    ....
//将Activity组件的状态设置为Resumed，代表它是系统当前显示的Activity
final void handleResumeActivity(IBinder token,
            boolean clearHide, boolean isForward, boolean reallyResume, int seq, String reason) {
        //获得启动的Activity的ActivityClientRecord对象，并赋值给r
        ActivityClientRecord r = mActivities.get(token);
        
        //最后会调用到mInstrumentation.callActivityOnResume，之后Activity的onResume生命周期就会被调用了
        r = performResumeActivity(token, clearHide, reason);
        if (r != null) {
            final Activity a = r.activity;
            ....

           if (r.window == null && !a.mFinished && willBeVisible) {
                r.window = r.activity.getWindow();
                View decor = r.window.getDecorView();
                decor.setVisibility(View.INVISIBLE);
                ViewManager wm = a.getWindowManager();
                WindowManager.LayoutParams l = r.window.getAttributes();
                a.mDecor = decor;
                l.type = WindowManager.LayoutParams.TYPE_BASE_APPLICATION;
                l.softInputMode |= forwardBit;
                if (r.mPreserveWindow) {
                    a.mWindowAdded = true;
                    r.mPreserveWindow = false;
                    ViewRootImpl impl = decor.getViewRootImpl();
                    if (impl != null) {
                        impl.notifyChildRebuilt();
                    }
                }
                if (a.mVisibleFromClient && !a.mWindowAdded) {
                    a.mWindowAdded = true;
                    wm.addView(decor, l);
                }

            } else if (!willBeVisible) {
               ....
            }
               ....
            }

            // 通知AMS activity已经进入resume状态
            if (reallyResume) {
                try {
                    ActivityManagerNative.getDefault().activityResumed(token);
                } catch (RemoteException ex) {
                   ....
                }
            }

        } else {
          ....
        }
    }
    ....
private void performPauseActivityIfNeeded(ActivityClientRecord r, String reason) {
        if (r.paused) {
            // 已经处于pause状态
            return;
        }

        try {
            r.activity.mCalled = false;
            //调用Activity的onPause方法，函数内是调用Activity.performPause方法，最后会调用onPause生命周期
            mInstrumentation.callActivityOnPause(r.activity);
                    r.activity.getComponentName().getClassName(), reason);
            ....
        } catch (SuperNotCalledException e) {
            ....
        } catch (Exception e) {
            ....
        }
        r.paused = true;
    }
    ....
}
```
&ensp;&ensp; 代码中为何需要获得一个**LoadedApk**对象，原因在于每一个Android应用程序都是打包在一个**apk**文件中，**apk**文件中包含了一个Android应用程序的所有资源，所以才需要在启动**Activity**时加载，为了可以访问到内部的资源文件。在**ActivityThread**类中就通过**LoadedApk**独享来描述加载的**apk**文件。

&ensp;&ensp; 当我们加载完**apk**之后，便会进行初始化配置，创建要启动的**Activity**的实例，以此调用其**attach**，**onCreate**，**onReume**方法和生命周期，中间夹杂着一些上下文创建，参数配置等操作没有细说。

### Service启动：ActiveServices.attachApplicationLocked 
**frameworks/base/services/core/java/com/android/server/am/ActiveServices.java**
```java
public final class ActiveServices {
    ....
        boolean attachApplicationLocked(ProcessRecord proc, String processName)
            throws RemoteException {
        boolean didSomething = false;
        // 收集等待在此进程启动的所有服务。
        if (mPendingServices.size() > 0) {
            ServiceRecord sr = null;
            try {
                for (int i=0; i<mPendingServices.size(); i++) {
                    sr = mPendingServices.get(i);
                    //检查在mPendingServices中的Service组件是否需要在新创建的应用程序中启动
                    if (proc != sr.isolatedProc && (proc.uid != sr.appInfo.uid
                            || !processName.equals(sr.processName))) {
                        continue;
                    }
                    //若需要启动Service，则先从mPendingServices把该成员删除
                    mPendingServices.remove(i);
                    i--;
                    
                    proc.addPackage(sr.appInfo.packageName, sr.appInfo.versionCode,
                            mAm.mProcessStats);
                    //将要启动的Service启动起来
                    realStartServiceLocked(sr, proc, sr.createdFromFg);
                    ....
                }
            } catch (RemoteException e) {
               ....
            }
        }
        ....
        return didSomething;
    }
    ....
// 启动Service
private final void realStartServiceLocked(ServiceRecord r,
            ProcessRecord app, boolean execInFg) throws RemoteException {
        ....
        //将ServiceRecord对象r的app设置为传入的app,这样就将r和app绑定起来。
        //代表r所描述的Service是在app所描述的应用程序进程中启动的
        r.app = app;
        r.restartTime = r.lastActivity = SystemClock.uptimeMillis();
        
        //将要创建的Service信息添加进app的services列表中
        final boolean newService = app.services.add(r);
        ....
        
        boolean created = false;
        try {
            ....
            //app.threa是类型为ApplicationThreadProxy对象，是一个Binder代理对象，指向应用程序进程ApplicationThread
            //这里调用scheduleCreateService请求应用程序进程创建该service
            app.thread.scheduleCreateService(r, r.serviceInfo,
                    mAm.compatibilityInfoForPackageLocked(r.serviceInfo.applicationInfo),
                    app.repProcState);
            r.postNotification();
            created = true;
        } catch (DeadObjectException e) {
           ....
        } finally {
            ....
        }
        ....
    }
}
```
### ApplicationThreadProxy.scheduleCreateService
**frameworks/base/core/java/android/app/ApplicationThreadNative.java**
```java
public abstract class ApplicationThreadNative extends Binder
        implements IApplicationThread {
    ....
    class ApplicationThreadProxy implements IApplicationThread {
        ....
        public final void scheduleCreateService(IBinder token, ServiceInfo info,
            CompatibilityInfo compatInfo, int processState) throws RemoteException {
        Parcel data = Parcel.obtain();
        data.writeInterfaceToken(IApplicationThread.descriptor);
        data.writeStrongBinder(token);
        info.writeToParcel(data, 0);
        compatInfo.writeToParcel(data, 0);
        data.writeInt(processState);
        try {
            //发送一个类型为SCHEDULE_CREATE_SERVICE_TRANSACTION的进程间通信请求
            mRemote.transact(SCHEDULE_CREATE_SERVICE_TRANSACTION, data, null,
                    IBinder.FLAG_ONEWAY);
        } catch (TransactionTooLargeException e) {
            ....
        }
        data.recycle();
    }
        ....
    }
    ....
}
```
### ApplicationThread.scheduleCreateService
**frameworks/base/core/java/android/app/ActivityThread.java**
```java
public final class ActivityThread {
    ....
    public final void scheduleCreateService(IBinder token,
                ServiceInfo info, CompatibilityInfo compatInfo, int processState) {
            updateProcessState(processState, false);
            //将要启动的Service的信息封装成一个CreateServiceData对象s
            CreateServiceData s = new CreateServiceData();
            s.token = token;
            s.info = info;
            s.compatInfo = compatInfo;
            //发送了一个CREATE_SERVICE消息，之后来看消息处理
            sendMessage(H.CREATE_SERVICE, s);
        }
    ....
        ....
    public void handleMessage(Message msg) {
         switch (msg.what) {
            ....
           case CREATE_SERVICE:
                    handleCreateService((CreateServiceData)msg.obj);
                    break;
            ....
         }
    }
    ....
    private void handleCreateService(CreateServiceData data) {
        ....
        //获得将要启动的Service组件所在应用程序的LoadedApk对象
        LoadedApk packageInfo = getPackageInfoNoCheck(
                data.info.applicationInfo, data.compatInfo);
        Service service = null;
        try {
            //获得一个类加载器
            java.lang.ClassLoader cl = packageInfo.getClassLoader();
            //将要启动的Service加载到内存中，并创建其对应的一个实例service
            service = (Service) cl.loadClass(data.info.name).newInstance();
        } catch (Exception e) {
            ....
        }

        try {
            ....
            //创建Service的一个上下文环境对象实例
            ContextImpl context = ContextImpl.createAppContext(this, packageInfo);
            context.setOuterContext(service);
            //创建/获得一个应用程序对象app
            Application app = packageInfo.makeApplication(false, mInstrumentation);
            //通过上面传入和创建的参数初始化service
            service.attach(context, this, data.info.name, data.token, app,
                    ActivityManagerNative.getDefault());
            //初始化完成后调用Service的生命周期onCreate
            service.onCreate();
            //以CreateServiceData的token关键字，将service保存在ActivityThread类的成员变量mServices中
            //与之前启动Activity类似，这里的token也是一个Binder代理对象
            //指向了AMS内部的一个ServiceRecord对象，用来描述启动的Service
            mServices.put(data.token, service);
            ....
        } catch (Exception e) {
           ....
        }
    }
    ....
}
```
&ensp;&ensp; 至此要启动的**Service**的**onCreate**就被调用了，启动完成

这里简单写了一个流程图如下：!![image](/img/in_post/activitythread_analysis.png)
