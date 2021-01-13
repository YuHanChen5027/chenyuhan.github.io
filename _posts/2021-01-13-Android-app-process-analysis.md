---
layout: post
title:  "Android应用程序进程的启动过程"
author: "陈宇瀚"
date:   2021-01-13 18:06:00 +0800
categories: article
header-img: "img/img-head/img-head-boot-process.jpg"
tags:
  - Android
  - 应用程序进程
  - 开机流程
---
&ensp;&ensp; **ActivityManagerService**在启动一个应用程序组件时，如果发现该组件所需的应用程序进程还为创建，则会请求**Zygote**进程将该应用程序进程启动起来。根据**Android开机流程分析那一部分的文章得知**，**Zygote**是通过**fork**也就是复制自身的方式来创建一个新的应用程序进程，所以新的应用程序进程自然包含了**Zygote**内部的虚拟机实例的拷贝，**Binder**线程池以及一个消息循环；虚拟机拷贝可以将使用**Java语言**所开发的应用程序组件运行起来。同时也有了自己的消息处理机制，以及可自定实现的**Binder**进程间通信等功能。下面根据源码详细分析下这个过程

&ensp;&ensp; *以下源码基于rk3399_industry Android7.1.2*
## 应用程序进程的创建
&ensp;&ensp; 当我们点击**Launcher**上的应用图标时会进入到**Launcher.java**(这里以系统自带的**/packages/apps/Launcher3/**为例)然后一系列调用下进入**LauncherAppsCompatV16**中的**startActivityForProfile**方法，内部实现如下:
```java
  public void startActivityForProfile(ComponentName component, UserHandleCompat user,
            Rect sourceBounds, Bundle opts) {
        Intent launchIntent = new Intent(Intent.ACTION_MAIN);
        launchIntent.addCategory(Intent.CATEGORY_LAUNCHER);
        launchIntent.setComponent(component);
        launchIntent.setSourceBounds(sourceBounds);
        launchIntent.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
        mContext.startActivity(launchIntent, opts);
    }
    
```
&ensp;&ensp; 这里可以看出从**Launcher**这样点击一个icon启动一个应用其实是启动对应的**Activity**，由于**Launcher.java**继承自**Activity**，所以之后会进入到**Activity**的**startActivity**方法，接下来分析该方法
### Activity.startActivity
```java
 public void startActivity(Intent intent, @Nullable Bundle options) {
        if (options != null) {
            startActivityForResult(intent, -1, options);
        } else {
            startActivityForResult(intent, -1);
        }
    }
 //Instrumentation是用来监控应用程序和系统之间的交互操作
 private Instrumentation mInstrumentation;
 public void startActivityForResult(@RequiresPermission Intent intent, int requestCode,
            @Nullable Bundle options) {
        if (mParent == null) {
            options = transferSpringboardActivityOptions(options);
            //启动Activity是应用程序和系统之间的交互操作，
            //调用execStartActivity来代为执行启动Activityde组件的操作，以便它可以监控这个过程
            Instrumentation.ActivityResult ar =
                mInstrumentation.execStartActivity(
                    this, mMainThread.getApplicationThread(), mToken, this,
                    intent, requestCode, options);
            ....
        } else {
           ....
        }
    }
```
&ensp;&ensp; 这里介绍下**mInstrumentation.execStartActivity**传入的两个重要参数
- **mMainThread**是**ActivityThread**类型，用于描述一个应用程序进程，每当系统启动一个应用程序进程时，内部都会加载一个**ActivityThread**类实例，并保存在该进程中启动的Activity的成员变量**mMainThread**中；**mMainThread.getApplicationThread()**用来获取其内部的一个**ApplicationThread**类型的**Binder**本地对象；
- **mToken**为**IBinder**类型，是一个**Binder**代理对象，指向了**AMS**中一个类型为**ActivityRecord**的**Binder**本地对象，每一个启动的**Activity**在**AMS**中都有一个对应的**ActivityRecord**对象，用来维护对应的**Activity**的运行状态。这里是**Launcher**的成员变量。

#### Instrumentation.execStartActivity
**framework/base/core/java/android/app/Instrumentation.java**
```java
public class Instrumentation{
    ....

    public ActivityResult execStartActivity(
            Context who, IBinder contextThread, IBinder token, Activity target,
            Intent intent, int requestCode, Bundle options) {
        ....
        try {
            intent.migrateExtraStreamToClipData();
            intent.prepareToLeaveProcess(who);
            //调用ActivityManagerNative.getDefault()获得AMS的代理对象
            //之后调用startActivity通知AMS将一个Activity启动起来
            int result = ActivityManagerNative.getDefault()
                .startActivity(whoThread, who.getBasePackageName(), intent,
                        intent.resolveTypeIfNeeded(who.getContentResolver()),
                        token, target != null ? target.mEmbeddedID : null,
                        requestCode, 0, null, options);
            checkStartActivityResult(result, intent);
        } catch (RemoteException e) {
            throw new RuntimeException("Failure from system", e);
        }
        return null;
    }
    
}

public abstract class ActivityManagerNative extends Binder implements IActivityManager{
    static public IActivityManager getDefault() {
        return gDefault.get();
    }
    
     private static final Singleton<IActivityManager> gDefault = new Singleton<IActivityManager>() {
        protected IActivityManager create() {
            IBinder b = ServiceManager.getService("activity");
            if (false) {
                Log.v("ActivityManager", "default service binder = " + b);
            }
            IActivityManager am = asInterface(b);
            if (false) {
                Log.v("ActivityManager", "default service = " + am);
            }
            return am;
        }
    };
    
    class ActivityManagerProxy implements IActivityManager{
        ....
        public int startActivity(IApplicationThread caller, String    callingPackage, Intent intent,
                String resolvedType, IBinder resultTo, String resultWho, int     requestCode,
                int startFlags, ProfilerInfo profilerInfo, Bundle options) throws     RemoteException {
            //将传进来的参数写入Parcel对象data中
            Parcel data = Parcel.obtain();
            Parcel reply = Parcel.obtain();
            data.writeInterfaceToken(IActivityManager.descriptor);
            data.writeStrongBinder(caller != null ? caller.asBinder() : null);
            data.writeString(callingPackage);
            intent.writeToParcel(data, 0);
            data.writeString(resolvedType);
            data.writeStrongBinder(resultTo);
            data.writeString(resultWho);
            data.writeInt(requestCode);
            data.writeInt(startFlags);
            if (profilerInfo != null) {
                data.writeInt(1);
                profilerInfo.writeToParcel(data,     Parcelable.PARCELABLE_WRITE_RETURN_VALUE);
            } else {
                data.writeInt(0);
            }
            if (options != null) {
                data.writeInt(1);
                options.writeToParcel(data, 0);
            } else {
                data.writeInt(0);
            }
            //通过Binder代理对象mRemote向AMS发送类型为
            //START_ACTIVITY_TRANSACTION的进程间通信请求。
            mRemote.transact(START_ACTIVITY_TRANSACTION, data, reply, 0);
            reply.readException();
            int result = reply.readInt();
            reply.recycle();
            data.recycle();
            return result;
        }
    ....
            
    }
}
```
&ensp;&ensp; **gDefault**是一个单例的应用了**AMS**的代理对象。**ServiceManager.getService("activity")**就是获取了**AMS**代理对象；回到**Instrumentation.execStartActivity**，接下来就调用了**ActivityManagerProxy**类的成员函数**startActivity**来通知**AMS**启动一个**Activity**组件了。其中传入的参数多达十个，我们需要重点关注的是以下几个:
- **caller**:指向启动的**Launcher**组件所运行在的应用程序进程的**ApplicationThread**对象；
- **intent**:包含了即将要启动的**Activity**的组件信息；
- **resultTo**:即是**AMS**中的**ActivityRecord**对象，保存了**Launcher**组建的详细信息；

&ensp;&ensp; 之后就进入了**AMS**的**startActivity**中来进行后续的启动操作
### ActivityManagerService.startActivity
```java
public final class ActivityManagerService extends ActivityManagerNative
        implements Watchdog.Monitor, BatteryStatsImpl.BatteryCallback {
        
    public final int startActivity(IApplicationThread caller, String callingPackage,
            Intent intent, String resolvedType, IBinder resultTo, String resultWho, int requestCode,
            int startFlags, ProfilerInfo profilerInfo, Bundle bOptions) {
        return startActivityAsUser(caller, callingPackage, intent, resolvedType, resultTo,
                resultWho, requestCode, startFlags, profilerInfo, bOptions,
                UserHandle.getCallingUserId());
    }
    
    public final int startActivityAsUser(IApplicationThread caller, String callingPackage,
            Intent intent, String resolvedType, IBinder resultTo, String resultWho, int requestCode,
            int startFlags, ProfilerInfo profilerInfo, Bundle bOptions, int userId) {
        enforceNotIsolatedCaller("startActivity");
        userId = mUserController.handleIncomingUser(Binder.getCallingPid(), Binder.getCallingUid(),
                userId, false, ALLOW_FULL_ONLY, "startActivity", null);
        //mActivityStarter用来描述一个类型为ActivityStarter,封装了一个activity启动的过程（包括上下文环境)      
        return mActivityStarter.startActivityMayWait(caller, -1, callingPackage, intent,
                resolvedType, null, null, resultTo, resultWho, requestCode, startFlags,
                profilerInfo, null, null, bOptions, false, userId, null, null);
    }
}

```
&ensp;&ensp; 之后的流程比较繁琐，这里不在详细介绍，这里以网图记录下，毕竟还是主要讲应用程序进程的启动；
![image](/img/in_post/app_process_reference.png)
引用自:[Android 7.1.2(Android N) Activity启动流程分析](https://blog.csdn.net/qingdaohaishanhu/article/details/87548414)
根据图上流程发现，启动过程中没，会进入到**AMS**的**startProcessLocked**的方法。
### ActivityManagerService.startProcessLocked
```java
   private final void startProcessLocked(ProcessRecord app, String hostingType,
            String hostingNameStr, String abiOverride, String entryPoint, String[] entryPointArgs) {
        ....
        try {
            //获得用户要创建的应用程序进程的用户id和用户组id
            int uid = app.uid;
            int[] gids = null;
            int mountExternal = Zygote.MOUNT_EXTERNAL_NONE;
            ....

            // 启动进程。成功并返回包含新进程PID的结果
            boolean isActivityProcess = (entryPoint == null);
            if (entryPoint == null) entryPoint = "android.app.ActivityThread";
            ....
            //调用Process.start创建应用程序进程，同时指定应用程序入口为
            //android.app.ActivityThread的main函数
            Process.ProcessStartResult startResult = Process.start(entryPoint,
                    app.processName, uid, uid, gids, debugFlags, mountExternal,
                    app.info.targetSdkVersion, app.info.seinfo, requiredAbi, instructionSet,
                    app.info.dataDir, entryPointArgs);
            ....
        } catch (RuntimeException e) {
           ....
        }
    }
    
public class Process{
    public static final ProcessStartResult start(final String processClass,
                                  final String niceName,
                                  int uid, int gid, int[] gids,
                                  int debugFlags, int mountExternal,
                                  int targetSdkVersion,
                                  String seInfo,
                                  String abi,
                                  String instructionSet,
                                  String appDataDir,
                                  String[] zygoteArgs) {
        try {
            return startViaZygote(processClass, niceName, uid, gid, gids,
                    debugFlags, mountExternal, targetSdkVersion, seInfo,
                    abi, instructionSet, appDataDir, zygoteArgs);
        } catch (ZygoteStartFailedEx ex) {
           ....
        }
    }

    private static ProcessStartResult startViaZygote(final String processClass,
                                  final String niceName,
                                  final int uid, final int gid,
                                  final int[] gids,
                                  int debugFlags, int mountExternal,
                                  int targetSdkVersion,
                                  String seInfo,
                                  String abi,
                                  String instructionSet,
                                  String appDataDir,
                                  String[] extraArgs)
                                  throws ZygoteStartFailedEx {
        synchronized(Process.class) {
            //将要创建的应用程序进程的启动参数保存在argsForZygote中
            ArrayList<String> argsForZygote = new ArrayList<String>();
            // --runtime-args, --setuid=, --setgid=,
            // and --setgroups= must go first
            // --runtime-args表示要在新创建的应用程序进程中初始化运行时库，
            // 以及启动一个Binder线程池
            argsForZygote.add("--runtime-args");
            argsForZygote.add("--setuid=" + uid);
            argsForZygote.add("--setgid=" + gid);
            ....
            argsForZygote.add("--target-sdk-version=" + targetSdkVersion);
            
            // --setgroups is a comma-separated list
            if (gids != null && gids.length > 0) {
                StringBuilder sb = new StringBuilder();
                sb.append("--setgroups=");

                int sz = gids.length;
                for (int i = 0; i < sz; i++) {
                    if (i != 0) {
                        sb.append(',');
                    }
                    sb.append(gids[i]);
                }

                argsForZygote.add(sb.toString());
            }

            if (niceName != null) {
                argsForZygote.add("--nice-name=" + niceName);
            }

            if (seInfo != null) {
                argsForZygote.add("--seinfo=" + seInfo);
            }

            if (instructionSet != null) {
                argsForZygote.add("--instruction-set=" + instructionSet);
            }

            if (appDataDir != null) {
                argsForZygote.add("--app-data-dir=" + appDataDir);
            }

            argsForZygote.add(processClass);

            if (extraArgs != null) {
                for (String arg : extraArgs) {
                    argsForZygote.add(arg);
                }
            }
            //调用zygoteSendArgsAndGetResult请求Zygote创建应用程序进程
            //openZygoteSocketIfNeeded(abi)是用来创建一个连接到Zygote进程的ZygoteState对象
            return zygoteSendArgsAndGetResult(openZygoteSocketIfNeeded(abi), argsForZygote);
        }
    }
 //与Zygote的连接状态
 static ZygoteState primaryZygoteState;
 
 //这里代表的其实是64位和32位的zygote，之前分析Zygote进程启动时有说过
 //Android系统启动时时常会启动两种zyote，我的这个系统是以64位为主，32为辅的
 //不同的Zygote在启动是会创建名称不同的socket,以下两个就是对应的sock名称
 public static final String ZYGOTE_SOCKET = "zygote";
 public static final String SECONDARY_ZYGOTE_SOCKET = "zygote_secondary";
 
 private static ZygoteState openZygoteSocketIfNeeded(String abi) throws ZygoteStartFailedEx {
        if (primaryZygoteState == null || primaryZygoteState.isClosed()) {
            try {
                //尝试连接64位Zygote进程
                primaryZygoteState = ZygoteState.connect(ZYGOTE_SOCKET);
            } catch (IOException ioe) {
                ....
            }
        }

        if (primaryZygoteState.matches(abi)) {
            return primaryZygoteState;
        }

        // 尝试连接32位Zygote进程
        if (secondaryZygoteState == null || secondaryZygoteState.isClosed()) {
            try {
            secondaryZygoteState = ZygoteState.connect(SECONDARY_ZYGOTE_SOCKET);
            } catch (IOException ioe) {
                throw new ZygoteStartFailedEx("Error connecting to secondary zygote", ioe);
            }
        }

        if (secondaryZygoteState.matches(abi)) {
            return secondaryZygoteState;
        }

        throw new ZygoteStartFailedEx("Unsupported zygote ABI: " + abi);
    }
    
 private static ProcessStartResult zygoteSendArgsAndGetResult(
            ZygoteState zygoteState, ArrayList<String> args)
            throws ZygoteStartFailedEx {
        try {
            ....
            //将要创建的应用程序进程的参数写入这个zygoteState
            //Zygote进程收到后就会创建对应的应用程序进程
            final BufferedWriter writer = zygoteState.writer;
            final DataInputStream inputStream = zygoteState.inputStream;

            writer.write(Integer.toString(args.size()));
            writer.newLine();

            for (int i = 0; i < sz; i++) {
                String arg = args.get(i);
                writer.write(arg);
                writer.newLine();
            }

            writer.flush();

            // Should there be a timeout on this?
            ProcessStartResult result = new ProcessStartResult();

            result.pid = inputStream.readInt();
            result.usingWrapper = inputStream.readBoolean();

            if (result.pid < 0) {
                throw new ZygoteStartFailedEx("fork() failed");
            }
            return result;
            } catch (IOException ex) {
            zygoteState.close();
            throw new ZygoteStartFailedEx(ex);
        }
    }
}
```
&ensp;&ensp; 这部分会将要创建的应用程序进程的信息通过**socket**发送给**Zygote**进程，根据之前
[Android开机启动流程分析一(Zygote进程)](https://yuhanchen5027.github.io/article/2021/01/03/Android-boot-process-analysis-1(Zygote-Process)/)
这一章节的介绍，**Zygote**进程启动完成后会在名为"zygote"的**socket**上等待**AMS**发送创建新的应用程序进程的请求，因此这一部分之后跟着执行的部分就回到了**Zygote**类中

### ZygoteInit.runSelectLoop
**frameworks/base/core/java/com/android/internal/os/ZygoteInit.java**
```java
private static void runSelectLoop(String abiList) throws MethodAndArgsCaller {
        ArrayList<FileDescriptor> fds = new ArrayList<FileDescriptor>();
        ArrayList<ZygoteConnection> peers = new ArrayList<ZygoteConnection>();

        fds.add(sServerSocket.getFileDescriptor());
        peers.add(null);

        while (true) {
            StructPollfd[] pollFds = new StructPollfd[fds.size()];
            for (int i = 0; i < pollFds.length; ++i) {
                pollFds[i] = new StructPollfd();
                pollFds[i].fd = fds.get(i);
                pollFds[i].events = (short) POLLIN;
            }
            try {
                Os.poll(pollFds, -1);
            } catch (ErrnoException ex) {
                throw new RuntimeException("poll failed", ex);
            }
            for (int i = pollFds.length - 1; i >= 0; --i) {
                if ((pollFds[i].revents & POLLIN) == 0) {
                    continue;
                }
                if (i == 0) {
                    ZygoteConnection newPeer = acceptCommandPeer(abiList);
                    peers.add(newPeer);
                    fds.add(newPeer.getFileDesciptor());
                } else {
                    //当有AMS发送过来的创建一个新的应用程序请求时执行
                    boolean done = peers.get(i).runOnce();
                    if (done) {
                        peers.remove(i);
                        fds.remove(i);
                    }
                }
            }
        }
    }
```
#### ZygoteConnection.runOnce()
**frameworks/base/core/java/com/android/internal/os/ZygoteConnection.java**
```java
boolean runOnce() throws ZygoteInit.MethodAndArgsCaller {

        String args[];
        Arguments parsedArgs = null;
        FileDescriptor[] descriptors;

        try {
            //获取要创建的应用程序进程启动参数
            args = readArgumentList();
            descriptors = mSocket.getAncillaryFileDescriptors();
        } catch (IOException ex) {
            closeSocket();
            return true;
        }

        ....

        int pid = -1;
        FileDescriptor childPipeFd = null;
        FileDescriptor serverPipeFd = null;
      try {
            //将启动参数args转成Arguments类型的parsedArgs
            parsedArgs = new Arguments(args);

            ....
            //调用Zygote的forkAndSpecialize静态函数来创建新的应用程序进程
            pid = Zygote.forkAndSpecialize(parsedArgs.uid, parsedArgs.gid, parsedArgs.gids,
                    parsedArgs.debugFlags, rlimits, parsedArgs.mountExternal, parsedArgs.seInfo,
                    parsedArgs.niceName, fdsToClose, parsedArgs.instructionSet,
                    parsedArgs.appDataDir);
        } catch (ErrnoException ex) {
           ....
        } catch (IllegalArgumentException ex) {
           ....
        } catch (ZygoteSecurityException ex) {
           ....
        }
        try {
            if (pid == 0) {
                // 在新创建的进程中
                IoUtils.closeQuietly(serverPipeFd);
                serverPipeFd = null;
                handleChildProc(parsedArgs, descriptors, childPipeFd, newStderr);
                
                return true;
            } else {
                ....
            }
        } finally {
           ....
        }
    }

  private void handleChildProc(Arguments parsedArgs,
            FileDescriptor[] descriptors, FileDescriptor pipeFd, PrintStream newStderr)
            throws ZygoteInit.MethodAndArgsCaller {
        ....
        //关闭复制zygote进程所拥有的用来接收AMS创建新应用程序进程的socket
        closeSocket();
        ZygoteInit.closeServerSocket();

        ....
        if (parsedArgs.invokeWith != null) {
            ....
        } else {
            RuntimeInit.zygoteInit(parsedArgs.targetSdkVersion,
                    parsedArgs.remainingArgs, null /* classLoader */);
        }
    }

```
&ensp;&ensp; **runOnce**内通过**Zygote.forkAndSpecialize**创建一个新的应用程序进程，当它的返回值为0时，证明创建成功并成功到了新创建的应用程序进程中。之后调用**handleChildProc**进行一些关闭**socket**和调用**RuntimeInit.zygoteInit*进行应用程序初始化运行时库，以及启动一个**Binder**线程池*。

#### RuntimeInit.zygoteInit
**frameworks/base/core/java/com/android/internal/os/RuntimeInit.java**
```java
  public static final void zygoteInit(int targetSdkVersion, String[] argv, ClassLoader classLoader)
            throws ZygoteInit.MethodAndArgsCaller {
        //重定向log输出
        redirectLogStreams();
        //设置System进程的时区和键盘布局等一些通用信息
        commonInit();
        //native层的初始化
        nativeZygoteInit();
        //调用应用程序java层的main方法
        applicationInit(targetSdkVersion, argv, classLoader);
    }
    
  private static void applicationInit(int targetSdkVersion, String[] argv, ClassLoader classLoader)
            throws ZygoteInit.MethodAndArgsCaller {
        nativeSetExitWithoutCleanup(true);

        VMRuntime.getRuntime().setTargetHeapUtilization(0.75f);
        VMRuntime.getRuntime().setTargetSdkVersion(targetSdkVersion);

        final Arguments args;
        try {
            //argv内有一个参数是com.android.server.SystemServer
            args = new Arguments(argv);
        } catch (IllegalArgumentException ex) {
            Slog.e(TAG, ex.getMessage());
            // let the process exit
            return;
        }

        // 之后进行SystemServer的启动，将剩余参数传递给SystemServer.main方法
        invokeStaticMain(args.startClass, args.startArgs, classLoader);
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
这部分在[Android开机启动流程分析二(System进程)](https://yuhanchen5027.github.io/article/2021/01/08/Android-boot-process-analysis-2(System-Process)/)
中有做过分析，这里就不在详细介绍。

在**ActivityManagerService.startProcessLocked**这一部分我们得知启动一个新的应用程序进程的入口函数为**ActivityThread**的**main**函数,所以最后调用**applicationInit**会反射执行到**ActivityThread.main**中，到这里，一个新的应用程序进程就启动完成了。

以下是一个简易的启动过程流程图，凑活看下：
![image](/img/in_post/app_process_img.png)
