---
layout: post
title:  "Android系统广播(Broadcast)注册，发送，接收流程解析"
author: "陈宇瀚"
date:   2021-01-27 18:23:00 +0800
header-img: "img/img-head/img-head-boot-process.jpg"
categories: article
tags:
  - Android
  - System
  - Broadcast
---
# Android系统广播(Broadcast)机制
*以下广播简称Broadcast*
## Broadcast介绍
&ensp;&ensp; <font color='OrangeRed'>Broadcast(广播)机制</font>是Android四大组件之一，在四大组件的另外两个组件<font color='OrangeRed'>Activity</font>和<font color='OrangeRed'>Service</font>拥有发送和接收广播的能力。Android<font color='DodgerBlue'>广播机制</font>是在<font color='OrangeRed'>Binder</font>进程间通信机制的基础上实现的，内部基于消息发布和订阅的事件驱动模型，广播发送者负责发送消息，广播接收者需要先订阅消息，然后才能收到消息。<font color='OrangeRed'>Binder</font>进程间通信与<font color='DodgerBlue'>广播机制</font>的区别在于：
- <font color='OrangeRed'>Binder</font>中客户端需要先知道服务端的存在，并获取到它的一个代理对象；
- <font color='DodgerBlue'>广播机制</font>中，广播发送者实现不需要知道广播接收者的存在，这样可以降低发送者和接收者的<font color='DodgerBlue'>耦合度</font>。提高系统的<font color='DodgerBlue'>可扩展性和可维护性</font>。

&ensp;&ensp; <font color='OrangeRed'>Broadcast</font>有三种类型
- <font color='DodgerBlue'>普通广播</font>：最常用的广播，不在意顺序；
- <font color='DodgerBlue'>有序广播</font>：发送的广播被接收者有序的接收，根据接收对象的优先级（Priority属性的值决定，值越大，优先级越高；Priority属性相同时，动态注册的广播优先于静态注册的广播）来决定接受顺序。有序广播可以对广播进行拦截，这样之后的接收者就接受不到广播了，也可以对广播内容进行修改；
- <font color='DodgerBlue'>粘性广播</font>：粘性广播一般用来确保重要的状态改变后的信息被持久保存，当下一个注册粘性广播的接收者注册成功后可以获得对应类型的广播之前返回的数据状态；(*android 5.0之后将其设置为deprecated,不再推荐应用使用*)
- <font color='DodgerBlue'>系统广播</font>：系统内部的广播，如开机，网络状态变化等情况会发出相应的广播，应用监听时部分系统广播可能需要系统权限；
- <font color='DodgerBlue'>应用内广播</font>：用于应用内的广播机制，相对于普通广播，安全性和效率都更高，一般使用<font color='OrangeRed'>LocalBroadcastManager</font>来注册和发送

&ensp;&ensp; <font color='OrangeRed'>Broadcast</font>存在一个注册中心，也可以说是一个调度中心，即<font color='OrangeRed'>AMS(ActivityManagerService)</font>。广播接收者将自己注册到<font color='OrangeRed'>AMS</font>中，并指定要接收的广播类型；广播发送者发送广播时，发送的广播首先会发送到<font color='OrangeRed'>AMS</font>，<font color='OrangeRed'>AMS</font>根据广播的类型找到对应的<font color='DodgerBlue'>广播接收者</font>，找到后边将广播发送给其处理。

## Broadcast的使用
&ensp;&ensp; 这里以普通广播为例子，<font color='OrangeRed'>Broadcast</font>接收者有两种注册方式，一种是<font color='DodgerBlue'>静态注册</font>，一种是<font color='DodgerBlue'>动态注册</font>：
- 是<font color='DodgerBlue'>静态注册</font>：将想要接收的广播类型，广播配置写入Android应用的配置文件<font color='DodgerBlue'>AndroidManifest.xml</font>中：
```xml
 AndroidManifest.xml
 //这里注册监听系统开机广播android.intent.action.BOOT_COMPLETED
 <receiver android:name=".BootCompletedReceiver">
            <intent-filter android:priority="999">
                <action android:name="android.intent.action.BOOT_COMPLETED" />
            </intent-filter>
 </receiver>
 //在BootCompletedReceiver内进行处理
 public class BootCompletedReceiver extends BroadcastReceiver {
    @Override
    public void onReceive(Context context, Intent intent) {
       if (intent.getAction().equals(Intent.ACTION_BOOT_COMPLETED)) {
            //接收到开机广播
        }
    }
}
```
- <font color='DodgerBlue'>动态注册</font>：在代码中手动调用<font color='OrangeRed'>Context.registerReceiver</font>将<font color='OrangeRed'>Broadcast Receiver</font>组件注册到<font color='OrangeRed'>AMS</font>中，<font color='OrangeRed'>Activity</font>和<font color='OrangeRed'>Service</font>内部都实现了<font color='OrangeRed'>Context</font>接口，所以都可以进行<font color='DodgerBlue'>动态注册</font>：
```java
public class MainActivity extends Activity {
    TestReceiver testReceiver;
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        //设置广播接收类型
        IntentFilter filter = new IntentFilter();
        filter.addAction("com.cyh.test");
        //设置接收优先级[-1000,1000]，默认为0
        filter.setPriority(1000)
        //实例化广播接收者
        testReceiver = new TestReceiver();
        registerReceiver(testReceiver,filter);
    }
    class TestReceiver extends BroadcastReceiver{
        @Override
        public void onReceive(Context context, Intent intent) {
           if (intent.getAction().equals("com.cyh.test")) {
                //接收到自定动态广播
            }
        }
    }
    
}
```
*（广播的发送分为<font color='DodgerBlue'>有序无序</font>两种，这里针对有序的广播）静态注册中的android:priority=""和动态注册中的IntentFilter.setPriority(int)可以用来设置广播接收者的优先级，默认都是0 , 范围是[-1000, 1000]，值越大优先级越高，优先级越高越早收到。*

&ensp;&ensp; 在相同优先级接收同个类型广播时，<font color='DodgerBlue'>动态注册</font>的广播接收器比<font color='DodgerBlue'>静态注册</font>的广播接收者更快的接收到对应的广播，这个之后会进行分析。

## Broadcast的流程分析
*&ensp;&ensp; 注：以下源码基于rk3399_industry Android7.1.2*

&ensp;&ensp; <font color='OrangeRed'>Broadcast</font>的流程可分为<font color='DodgerBlue'>注册</font>，<font color='DodgerBlue'>发送</font>和<font color='DodgerBlue'>接收</font>三个部分，这里依次分析下
### Broadcast接收者的注册过程
&ensp;&ensp; 在Android系统的<font color='OrangeRed'>Broadcast</font>机制中，前面提到，<font color='OrangeRed'>AMS</font>作为一个注册和调度中心负责注册和转发<font color='OrangeRed'>Broadcast</font>。所以<font color='OrangeRed'>BroadcastReceiver</font>的注册过程就是把它注册到<font color='OrangeRed'>AMS</font>的过程。

&ensp;&ensp; 这里我们分析<font color='DodgerBlue'>动态注册</font>广播的过程，<font color='OrangeRed'>Activity</font>和<font color='OrangeRed'>Service</font>有一个共同的父类<font color='OrangeRed'>ContextWrapper</font>，所以它们对应的注册过程其实是调用<font color='OrangeRed'>ContextWrapper.registerReceiver</font>，接下来我们按照流程逐步分析调用流程的源码。

**frameworks/base/core/java/android/content/ContextWrapper.java**
```java
public class ContextWrapper extends Context {
    Context mBase;
    ....
    @Override
    public Intent registerReceiver(
        BroadcastReceiver receiver, IntentFilter filter) {
        return mBase.registerReceiver(receiver, filter);
    }
    ....
    
}
```
&ensp;&ensp; 在之前的[Android应用程序启动入口ActivityThread.main流程分析](https://yuhanchen5027.github.io/article/2021/01/16/Android-application-startup-activitythread-main-analysis/)分析过，在我们启动**Activity**时会创建一个<font color='OrangeRed'>ContextImpl</font>对象，然后通过<font color='OrangeRed'>Activity.attach</font>传给我们启动的<font color='OrangeRed'>Activity</font>，其内部就会将该对象赋值给<font color='OrangeRed'>mBase</font>；<font color='OrangeRed'>Service</font>的<font color='OrangeRed'>mBase</font>方法也是类似的赋值流程，这里放个简易的源码应该更好理解
```java
/**
 *  frameworks/base/core/java/android/app/ActivityThread.java
 */
public final class ActivityThread {
    ....
    //将Activity启动起来
    private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
        ....
         if (activity != null) {
                //创建一个ContextImpl对象，是创建的Activity的上下文环境
                Context appContext = createBaseContextForActivity(r, activity);
                ....
                //传入参数，调用attach进行Activity的初始化
                activity.attach(appContext, ....);
                ....
                }
            }
    ....
    //启动Service
    private void handleCreateService(CreateServiceData data) {
        ....
            //创建Service的一个上下文环境对象实例
            ContextImpl context = ContextImpl.createAppContext(this, packageInfo);
            ....
            //通过上面传入和创建的参数初始化service
            service.attach(context, this, data.info.name, data.token, app,
                    ActivityManagerNative.getDefault());
            ....
    }
    ....
}
/**
 *  frameworks/base/core/java/android/app/Activity.java
 */
public class Activity extends ContextThemeWrapper
        implements .... {
            ....
            final void attach(Context context, ....) {
                ....
                attachBaseContext(context);
                ....
            }
            ....
        }
        
/**
 *  frameworks/base/core/java/android/view/ContextThemeWrapper.java
 */
@Override
public class ContextThemeWrapper extends ContextWrapper {
    ....
    protected void attachBaseContext(Context newBase) {
        super.attachBaseContext(newBase);
    }
    ....
}

/**
 *  frameworks/base/core/java/android/app/Service.java
 */
public abstract class Service extends ContextWrapper implements ComponentCallbacks2 {
    ....
    public final void attach(
            Context context,
            ....) {
        attachBaseContext(context);
        ....
    }
    ....
}
 
/**
 *  frameworks/base/core/java/android/content/ContextWrapper.java
 */
public class ContextWrapper extends Context {
    Context mBase;
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
&ensp;&ensp; 可以看到最后都会将生成的<font color='OrangeRed'>ContextImpl</font>对象赋值给对应的<font color='OrangeRed'>mBase</font>
对象。接下来继续分析<font color='OrangeRed'>Broadcast</font>的注册过程，<font color='OrangeRed'>mBase.registerReceiver</font>即<font color='OrangeRed'>ContextImpl.registerReceiver</font>函数。
#### ContextImpl.registerReceiver
**/frameworks/base/core/java/android/app/ContextImpl.java**
```java
class ContextImpl extends Context {
    final LoadedApk mPackageInfo;
    ....
    @Override
    public Intent registerReceiver(BroadcastReceiver receiver, IntentFilter filter) {
        return registerReceiver(receiver, filter, null, null);
    }
    
    @Override
    public Intent registerReceiver(BroadcastReceiver receiver, IntentFilter filter,
            String broadcastPermission, Handler scheduler) {
        //getOuterContext()指向类内部的一个mOuterContext对象
        //指向我们调用注册广播的Activity或者Service
        return registerReceiverInternal(receiver, getUserId(),
                filter, broadcastPermission, scheduler, getOuterContext());
    }
    
    @Override
    public Intent registerReceiverAsUser(BroadcastReceiver receiver, UserHandle user,
            IntentFilter filter, String broadcastPermission, Handler scheduler) {
        //getOuterContext()
        return registerReceiverInternal(receiver, user.getIdentifier(),
                filter, broadcastPermission, scheduler, getOuterContext());
    }
    
    private Intent registerReceiverInternal(BroadcastReceiver receiver, int userId,
            IntentFilter filter, String broadcastPermission,
            Handler scheduler, Context context) {
        IIntentReceiver rd = null;
        if (receiver != null) {
            if (mPackageInfo != null && context != null) {
                if (scheduler == null) {
                    //获得mMainThread用来描述当前应用程序进程，其getHandler()返回一个Handler对象
                    //该Handler可以向当前应用程序进程的主线程的消息对列发送消息
                    scheduler = mMainThread.getHandler();
                }
                //将广播接收者receiver封装成一个IIntentReceiver接口的Binder本地对象rd
                rd = mPackageInfo.getReceiverDispatcher(
                    receiver, context, scheduler,
                    mMainThread.getInstrumentation(), true);
            } else {
               ....
            }
        }
        try {
            //调用AMS代理对象的registerReceiver函数将获得的Binder本地对象rd
            //以及对应的IntentFilter对象filter发送给AMS，以便于AMS可以将rd对象注册在其内部，
            //并能根据filter来将对应的广播发送给他处理
            final Intent intent =
            ActivityManagerNative.getDefault().registerReceiver(
                    mMainThread.getApplicationThread(), mBasePackageName,
                    rd, filter, broadcastPermission, userId);
            if (intent != null) {
                //设置类加载器
                intent.setExtrasClassLoader(getClassLoader());
                //准备进入流程
                intent.prepareToEnterProcess();
            }
            return intent;
        } catch (RemoteException e) {
            throw e.rethrowFromSystemServer();
        }
    }
}
```
##### LoadedApk.getReceiverDispatcher
&ensp;&ensp; 这里我们首先看下如何将广播接收者<font color='OrangeRed'>receiver</font>封装成一个<font color='OrangeRed'>IIntentReceiver</font>接口的<font color='OrangeRed'>Binder</font>本地对象<font color='OrangeRed'>rd</font>
**/frameworks/base/core/java/android/app/LoadedApk.java**
```java
public final class LoadedApk {
    // 用来保存Activity/Service和他们所关联的ReceiverDispatcher对象Map。
    private final ArrayMap<Context, ArrayMap<BroadcastReceiver, ReceiverDispatcher>> mReceivers
        = new ArrayMap<Context, ArrayMap<BroadcastReceiver, LoadedApk.ReceiverDispatcher>>();
    ....
    public IIntentReceiver getReceiverDispatcher(BroadcastReceiver r,
            Context context, Handler handler,
            Instrumentation instrumentation, boolean registered) {
        synchronized (mReceivers) {
            // 该对象负责将被注册的广播接收者与注册它的Activity/Service组件关联起来。
            LoadedApk.ReceiverDispatcher rd = null;
            ArrayMap<BroadcastReceiver, LoadedApk.ReceiverDispatcher> map = null;
            if (registered) {
                //根据Activity/Service的Context获得之前生成的广播接收者和ReceiverDispatcher对应的map对象。
                map = mReceivers.get(context);
                 //判断是否存在map对象
                if (map != null) {
                    //获得注册的广播对应的ReceiverDispatcher对象
                    rd = map.get(r);
                }
            }
            if (rd == null) {
                //如果该广播接收者未注册过，生成一个对应的ReceiverDispatcher对象
                rd = new ReceiverDispatcher(r, context, handler,
                        instrumentation, registered);
                if (registered) {
                    if (map == null) {
                        //如果对应的Activity/Service没有注册过广播，初始化对应的map
                        map = new ArrayMap<BroadcastReceiver, LoadedApk.ReceiverDispatcher>();
                        //在将map保存在LoadedApk的mReceivers对象中
                        mReceivers.put(context, map);
                    }
                    //将广播接收者和对应的ReceiverDispatcher对象rd放入map对象中
                    map.put(r, rd);
                }
            } else {
                rd.validate(context, handler);
            }
            rd.mForgotten = false;
            //获得对应ReceiverDispatcher对象rd的IIntentreceiver接口的Binder本地对象
            return rd.getIIntentReceiver();
        }
    }
    ....
    static final class ReceiverDispatcher {
        
        final static class InnerReceiver extends IIntentReceiver.Stub {
            final WeakReference<LoadedApk.ReceiverDispatcher> mDispatcher;
            final LoadedApk.ReceiverDispatcher mStrongRef;

            InnerReceiver(LoadedApk.ReceiverDispatcher rd, boolean strong) {
                mDispatcher = new WeakReference<LoadedApk.ReceiverDispatcher>(rd);
                mStrongRef = strong ? rd : null;
            }
        }
        ....
        //该对象就是的IIntentreceiver接口的Binder本地对象
        final IIntentReceiver.Stub mIIntentReceiver;
        //指向了一个广播接收者
        final BroadcastReceiver mReceiver;
        //指向Activity/Service组件
        final Context mContext;
        //指向Activity组件相关联的Handler对象
        final Handler mActivityThread;
        ....

        ReceiverDispatcher(BroadcastReceiver receiver, Context context,
                Handler activityThread, Instrumentation instrumentation,
                boolean registered) {
            if (activityThread == null) {
                throw new NullPointerException("Handler must not be null");
            }
            mIIntentReceiver = new InnerReceiver(this, !registered);
            mReceiver = receiver;
            mContext = context;
            mActivityThread = activityThread;
            mInstrumentation = instrumentation;
            ....
        }
        ....

        IIntentReceiver getIIntentReceiver() {
            return mIIntentReceiver;
        }
        ....

    }
}
```
&ensp;&ensp; 每一个注册过广播接收者的<font color='OrangeRed'> Activity </font>或<font color='OrangeRed'> Service </font>组件在<font color='Crimson'> LoadedApk </font>类中都有个对应的<font color='OrangeRed'>ReceiverDispatcher</font>对象，该对象负责将<font color='DodgerBlue'>被注册的广播接收者</font>与<font color='DodgerBlue'>注册它的Activity/Service</font>组件关联起来。这些对象，以关联的<font color='DodgerBlue'>广播接收者</font>作为关键字保存在一个<font color='OrangeRed'>ArrayMap</font>中。之后对应的<font color='OrangeRed'>ArrayMap</font>又以<font color='DodgerBlue'>注册它的Activity/Service </font>的<font color='OrangeRed'>context</font>作为关键字保存在<font color='OrangeRed'>LoadedApk</font>的成员变量<font color='OrangeRed'>mReceivers</font>对象中。最后通过<font color='OrangeRed'>ReceiverDispatcher对象rd </font>对应的<font color='OrangeRed'>getIIntentReceiver</font>方法获得其<font color='OrangeRed'>IIntentreceiver</font>接口的<font color='OrangeRed'>Binder</font>本地对象。之后再回到<font color='OrangeRed'/>ContextImpl.registerReceiver</font>注册方法内，将<font color='OrangeRed'>IntentReceiver</font>对象发给<font color='OrangeRed'>AMS</font>进行注册。
#### ActivityManagerProxy.registerReceiver
**/frameworks/base/core/java/android/app/ActivityManagerNative.java**
```java
public abstract class ActivityManagerNative extends Binder implements IActivityManager{
    ....
    class ActivityManagerProxy implements IActivityManager{
        ....
            public Intent registerReceiver(IApplicationThread caller, String packageName,
            IIntentReceiver receiver,
            IntentFilter filter, String perm, int userId) throws RemoteException{
        //将传入的参数封装进Parcel对象data
        Parcel data = Parcel.obtain();
        Parcel reply = Parcel.obtain();
        data.writeInterfaceToken(IActivityManager.descriptor);
        data.writeStrongBinder(caller != null ? caller.asBinder() : null);
        data.writeString(packageName);
        data.writeStrongBinder(receiver != null ? receiver.asBinder() : null);
        filter.writeToParcel(data, 0);
        data.writeString(perm);
        data.writeInt(userId);
        //调用Binder代理对象mRemote向AMS发送一个类型为REGISTER_RECEIVER_TRANSACTION的进程间通信请求
        mRemote.transact(REGISTER_RECEIVER_TRANSACTION, data, reply, 0);
        reply.readException();
        Intent intent = null;
        int haveIntent = reply.readInt();
        if (haveIntent != 0) {
            intent = Intent.CREATOR.createFromParcel(reply);
        }
        reply.recycle();
        data.recycle();
        return intent;
    }
    ....
    }
}
```
#### ActivityManagerService.registerReceiver
**/frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java**
```java
public final class ActivityManagerService extends ActivityManagerNative
        implements Watchdog.Monitor, BatteryStatsImpl.BatteryCallback {
    ....
    // 保存已注册用于广播的所有IIntentReceiver。
    final HashMap<IBinder, ReceiverList> mRegisteredReceivers = new HashMap<>();
    public Intent registerReceiver(IApplicationThread caller, String callerPackage,
            IIntentReceiver receiver, IntentFilter filter, String permission, int userId) {
        ....
        ProcessRecord callerApp = null;
        int callingUid;
        int callingPid;
        synchronized(this) {
            if (caller != null) {
                // 获得注册广播的Activity所在的应用程序进程
                callerApp = getRecordForAppLocked(caller);
                ....
                callingUid = callerApp.info.uid;
                callingPid = callerApp.pid;
            } else {
                ....
            }
            
            ....
            // 参数filter中有一系列ACTION，用来描述一系列不同类型的广播
            // 这里获取注册广播的ACTION列表
            Iterator<String> actions = filter.actionsIterator();
            if (actions == null) {
                ArrayList<String> noAction = new ArrayList<String>(1);
                noAction.add(null);
                actions = noAction.iterator();
            }
            
            // 一个粘性(sticky)广播被发送到AMS后就会一直保存在AMS中
            // 直到AMS再接收到另一个同类型的粘性(sticky)广播为止
            // 一个Activity组件再向AMS中注册接收某一个类型的广播时
            // 如果AMS内已经保存有这种类型的广播，则直接将该粘性(sticky)广播返回给Activity
            // 以便他可以知道系统上一次发送的对应类型的广播的内容。
            // 在Activity和Service中可以调用他们父类ContextWrapper.sebdStickyBroadcast像AMS发送一个粘性广播
            // 收集粘性(sticky)广播
            int[] userIds = { UserHandle.USER_ALL, UserHandle.getUserId(callingUid) };
            while (actions.hasNext()) {
                String action = actions.next();
                for (int id : userIds) {
                    ArrayMap<String, ArrayList<Intent>> stickies = mStickyBroadcasts.get(id);
                    if (stickies != null) {
                        ArrayList<Intent> intents = stickies.get(action);
                        if (intents != null) {
                            if (stickyIntents == null) {
                                stickyIntents = new ArrayList<Intent>();
                            }
                            // 将匹配的intent存入到stickyIntents中
                            stickyIntents.addAll(intents);
                        }
                    }
                }
            }
        }
        //
        ArrayList<Intent> allSticky = null;
        if (stickyIntents != null) {
            final ContentResolver resolver = mContext.getContentResolver();
            // 查找任何匹配的粘性广播
            for (int i = 0, N = stickyIntents.size(); i < N; i++) {
                Intent intent = stickyIntents.get(i);
                if (filter.match(resolver, intent, true, TAG) >= 0) {
                    if (allSticky == null) {
                        allSticky = new ArrayList<Intent>();
                    }
                    allSticky.add(intent);
                }
            }
        }

        // 判断列表allSticky是否等于null，不等于null则代表存在对应类型的粘性广播
        // 此时AMS就会将allSticky列表中的第一个粘性广播赋值给sticky
        Intent sticky = allSticky != null ? allSticky.get(0) : null;
        if (receiver == null) {
            // 如果receiver==null，代表没有要注册的广播接收者
            return sticky;
        }
        synchronized (this) {
            if (callerApp != null && (callerApp.thread == null
                    || callerApp.thread.asBinder() != caller.asBinder())) {
                // 注册广播的应用程序进程已经死亡
                return null;
            }
            // 在同一个应用程序中，不同的Activity可能会使用同一个InnerReceiver对象来注册不同的广播接收者
            // 所以，AMS中就会使用一个ReceiverList列表来保存这些使用了相同InnerReceiver对象来注册的广播接收者
            // 并且以他们所使用的InnerReceiver对象作为关键字保存在mRegisteredReceivers中
            // 从AMS的mRegisteredReceivers成员对象中找到receiver对应的ReceiverList
            // ReceiverList 是一个广播接收者列表
            ReceiverList rl = mRegisteredReceivers.get(receiver.asBinder());
            if (rl == null) {
                // 如果没有对应的ReceiverList，这里创建并添加进AMS成员对象mRegisteredReceivers
                rl = new ReceiverList(this, callerApp, callingPid, callingUid,
                        userId, receiver);
                ....
                mRegisteredReceivers.put(receiver.asBinder(), rl);
            } 
            ....
            // 每一个动态广播接收者在AMS都是使用一个BroadcastFilter来描述的
            // BroadcastFilter对象将广播接收者列表列表rl和广播接收类型filter关联起来
            BroadcastFilter bf = new BroadcastFilter(filter, rl, callerPackage,
                    permission, callingUid, userId);
            // 将bf添加进之前所获得的ReceiverList对象rl
            rl.add(bf);
            // 将bf添加进AMS的成员变量mReceiverResolver中，
            mReceiverResolver.addFilter(bf);
            //之后AMS接收到广播之后就可以在mReceiverResolver中找到对应的广播接收者了
            ....
            // 将之前得到的粘性广播返回给正在注册对应类型广播的接收者
            // 这样他就可以知道系统上一次发送该粘性广播时发送了什么内容的数据
            return sticky;
        }
    }
    
}
```
&ensp;&ensp; 在的<font color='OrangeRed'> Activity </font>或<font color='OrangeRed'> Service </font> 注册一个<font color='DodgerBlue'>广播接收者</font>时，并不是将其注册到<font color='OrangeRed'>AMS</font>中，而是将与它关联的<font color='OrangeRed'>InnerReceiver</font>对象注册到<font color='OrangeRed'>AMS</font>中，当<font color='OrangeRed'>AMS</font>接收到广播时，会根据<font color='DodgerBlue'>广播类型</font>在内部找到对应的<font color='OrangeRed'>InnerReceiver</font>对象，然后在通过这个对象将这个广播发送给对应的<font color='DodgerBlue'>广播接收者</font>处理。

&ensp;&ensp; 注册过程这边画了一个简单的流程图:
!![image](/img/in_post/post_broadcase_registered.png)
### Broadcast的发送过程
&ensp;&ensp; <font color='OrangeRed'>Broadcast</font>的发送过程可简单描述为以下几个过程：
1. 一个的<font color='OrangeRed'> Activity </font>或<font color='OrangeRed'> Service </font>调用<font color='OrangeRed'>sendBroadcast</font>将指定类型的广播发送给<font color='OrangeRed'>AMS</font>；
2. <font color='OrangeRed'>AMS</font>接收到发送的广播后，找到该<font color='DodgerBlue'>广播类型</font>对应的<font color='DodgerBlue'>广播接收者</font>，将他们添加到一个<font color='DodgerBlue'>广播调度队列</font>中，最后向
<font color='OrangeRed'>AMS</font>发送进程间通信请求。
3. <font color='OrangeRed'>AMS</font>接收到将对应的进程间通信请求后，就会从<font color='DodgerBlue'>广播调度队列</font>中找到需要接受广播的<font color='DodgerBlue'>广播接收者</font>，然后将对应的广播发送给它们所在的应用程序进程
4. <font color='DodgerBlue'>广播接收者</font>所在应用程序进程接收<font color='OrangeRed'>AMS</font>所转发过来的对应的广播后，将广播封装成一个<font color='DodgerBlue'>消息</font>，然后将<font color='DodgerBlue'>消息</font>发送到主线程的<font color='DodgerBlue'>消息队列</font>中等待处理，轮到该消息进行处理时才会将对应的广播发送给<font color='DodgerBlue'>广播接收者</font>进行处理
&ensp;&ensp; 接下来根据调用流程和源码来分析这一过程
#### ContextWrapper.sendBroadcast
**frameworks/base/core/java/android/content/ContextWrapper.java**
```java
public class ContextWrapper extends Context {
    Context mBase;
    ....
    @Override
    public void sendBroadcast(Intent intent) {
        mBase.sendBroadcast(intent);
    }
    ....
    
}
```
#### ContextImpl.registerReceiver
**/frameworks/base/core/java/android/app/ContextImpl.java**
```java
class ContextImpl extends Context {
    ....
        @Override
    public void sendBroadcast(Intent intent) {
        warnIfCallingFromSystemProcess();
        String resolvedType = intent.resolveTypeIfNeeded(getContentResolver());
        try {
            intent.prepareToLeaveProcess(this);
            //调用AMS代理对象的broadcastIntent函数将参数intent对应的广播发送给AMS
            ActivityManagerNative.getDefault().broadcastIntent(
                    mMainThread.getApplicationThread(), intent, resolvedType, null,
                    Activity.RESULT_OK, null, null, null, AppOpsManager.OP_NONE, null, false, false,
                    getUserId());
        } catch (RemoteException e) {
            throw e.rethrowFromSystemServer();
        }
    }
    ....
}
```
#### ActivityManagerProxy.registerReceiver
**/frameworks/base/core/java/android/app/ActivityManagerNative.java**
```java
public abstract class ActivityManagerNative extends Binder implements IActivityManager{
    ....
        public int broadcastIntent(IApplicationThread caller,
            Intent intent, String resolvedType, IIntentReceiver resultTo,
            int resultCode, String resultData, Bundle map,
            String[] requiredPermissions, int appOp, Bundle options, boolean serialized,
            boolean sticky, int userId) throws RemoteException{
        //将数据封进Parcel对象data
        Parcel data = Parcel.obtain();
        Parcel reply = Parcel.obtain();
        data.writeInterfaceToken(IActivityManager.descriptor);
        data.writeStrongBinder(caller != null ? caller.asBinder() : null);
        intent.writeToParcel(data, 0);
        data.writeString(resolvedType);
        data.writeStrongBinder(resultTo != null ? resultTo.asBinder() : null);
        data.writeInt(resultCode);
        data.writeString(resultData);
        data.writeBundle(map);
        data.writeStringArray(requiredPermissions);
        data.writeInt(appOp);
        data.writeBundle(options);
        data.writeInt(serialized ? 1 : 0);
        data.writeInt(sticky ? 1 : 0);
        data.writeInt(userId);
        //调用Binder代理对象mRemote向AMS发送一个类型为BROADCAST_INTENT_TRANSACTION的进程间通信请求
        mRemote.transact(BROADCAST_INTENT_TRANSACTION, data, reply, 0);
        reply.readException();
        int res = reply.readInt();
        reply.recycle();
        data.recycle();
        return res;
    }
    .... 
}
```
#### ActivityManagerService.broadcastIntent
**/frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java**
```java
public final class ActivityManagerService extends ActivityManagerNative
        implements Watchdog.Monitor, BatteryStatsImpl.BatteryCallback {
    ....
        public final int broadcastIntent(IApplicationThread caller,
            Intent intent, String resolvedType, IIntentReceiver resultTo,
            int resultCode, String resultData, Bundle resultExtras,
            String[] requiredPermissions, int appOp, Bundle bOptions,
            boolean serialized, boolean sticky, int userId) {
        enforceNotIsolatedCaller("broadcastIntent");
        synchronized(this) {
            // 验证intent所描述的广播是否合法
            intent = verifyBroadcastLocked(intent);
            // 获得发送广播的应用程序进程
            final ProcessRecord callerApp = getRecordForAppLocked(caller);
            final int callingPid = Binder.getCallingPid();
            final int callingUid = Binder.getCallingUid();
            final long origId = Binder.clearCallingIdentity();
            // 调用broadcastIntentLocked进行接下来的处理
            int res = broadcastIntentLocked(callerApp,
                    callerApp != null ? callerApp.info.packageName : null,
                    intent, resolvedType, resultTo, resultCode, resultData, resultExtras,
                    requiredPermissions, appOp, bOptions, serialized, sticky,
                    callingPid, callingUid, userId);
            Binder.restoreCallingIdentity(origId);
            return res;
        }
    }
    
    final int broadcastIntentLocked(ProcessRecord callerApp,
            String callerPackage, Intent intent, String resolvedType,
            IIntentReceiver resultTo, int resultCode, String resultData,
            Bundle resultExtras, String[] requiredPermissions, int appOp, Bundle bOptions,
            boolean ordered, boolean sticky, int callingPid, int callingUid, int userId) {
        intent = new Intent(intent);

        // 默认情况下，广播不会发送到已停止的应用程序。
        intent.addFlags(Intent.FLAG_EXCLUDE_STOPPED_PACKAGES);
        ....
        
        // 判断发送的广播是否是受保护的广播，如一些开机广播之类的，应用是不能发的
        ....

        // 阻止非系统应用发送受保护的广播。
        ....
        // 处理一些应用安装，删除，清除数据等类型的广播
        ....        
        
        // 这之后就是我们的针对我们自定义广播发送的处理
        // 如果有需要，将其添加到粘性列表中，当然根据上面的流程知道这里的sticky是false，所以可以直接跳过下面的if内部的解析。
        if (sticky) {
            //检查是否有发送粘性广播的权限
            if (checkPermission(android.Manifest.permission.BROADCAST_STICKY,
                    callingPid, callingUid)
                    != PackageManager.PERMISSION_GRANTED) {
                ....
            }
            ....
            // UserHandle.USER_ALL有一组单独的粘性广播列表
            if (userId != UserHandle.USER_ALL) {
                // 如果是针对userId的广播，确保不要与USER_ALL的广播冲突
                ArrayMap<String, ArrayList<Intent>> stickies = mStickyBroadcasts.get(
                        UserHandle.USER_ALL);
                if (stickies != null) {
                    ArrayList<Intent> list = stickies.get(intent.getAction());
                    if (list != null) {
                        int N = list.size();
                        int i;
                        for (i=0; i<N; i++) {
                            if (intent.filterEquals(list.get(i))) {
                                throw new IllegalArgumentException(
                                        "Sticky broadcast " + intent + " for user "
                                        + userId + " conflicts with existing global broadcast");
                            }
                        }
                    }
                }
            }
            // 查找AMS中是否已经存在userId对应的粘性广播列表
            ArrayMap<String, ArrayList<Intent>> stickies = mStickyBroadcasts.get(userId);
            if (stickies == null) {
                //不存在就创建，并以userId为关键字存入AMS的mStickyBroadcasts成员变量中
                stickies = new ArrayMap<>();
                mStickyBroadcasts.put(userId, stickies);
            }
            // 查找userId对应粘性广播列表的中是否已经存在intent对应Action名称的粘性广播列表
            ArrayList<Intent> list = stickies.get(intent.getAction());
            if (list == null) {
                //不存在就创建，并以对应Action名称为关键字存入粘性广播列表中
                list = new ArrayList<>();
                stickies.put(intent.getAction(), list);
            }
            final int stickiesCount = list.size();
            int i;
            //循环检查粘性广播列表中是否存在一个与intent一致的广播
            for (i = 0; i < stickiesCount; i++) {
                if (intent.filterEquals(list.get(i))) {
                    // 存在就替换
                    list.set(i, new Intent(intent));
                    break;
                }
            }
            if (i >= stickiesCount) {
                //不存在就添加进list中
                list.add(new Intent(intent));
            }
        }
        // 不是粘性广播
        ....
        // 如果指定了广播接收者的名称
        
        // receivers用来保存所有静态广播接收者，生成后是一个List<ResolveInfo>对象
        List receivers = null;
        // registeredReceivers用来保存所有的动态广播接收者
        // 根据注册时候可知BroadcastFilter是广播接收者列表和广播类型的关联类
        List<BroadcastFilter> registeredReceivers = null;
        // 这里判断是否需要将广播发送给静态注册的广播接收者，如果以下判断==0为false
        // 就代表只需要发送给动态注册广播接收者
        // FLAG_RECEIVER_REGISTERED_ONLY代表只能发给动态注册的广播
        if ((intent.getFlags()&Intent.FLAG_RECEIVER_REGISTERED_ONLY)
                 == 0) {
            // collectReceiverComponents获得intent对应的所有静态注册广播接收者
            receivers = collectReceiverComponents(intent, resolvedType, callingUid, users);
        }
        if (intent.getComponent() == null) {
            // 这里是代表广播是要发送给所有注册的广播接收者的
            if (userId == UserHandle.USER_ALL && callingUid == Process.SHELL_UID) {
                // 一次查询一个目标用户，不包括受shell限制的用户
                // 找到所有动态注册的广播接收者保存在registeredReceivers中
                    if (mUserController.hasUserRestriction(
                            UserManager.DISALLOW_DEBUGGING_FEATURES, users[i])) {
                        continue;
                    }
                    List<BroadcastFilter> registeredReceiversForUser =
                            mReceiverResolver.queryIntent(intent,
                                    resolvedType, false, users[i]);
                    if (registeredReceivers == null) {
                        registeredReceivers = registeredReceiversForUser;
                    } else if (registeredReceiversForUser != null) {
                        registeredReceivers.addAll(registeredReceiversForUser);
                    }
                }
            } else {
                // 找到特定的广播接收者赋值给registeredReceivers
                registeredReceivers = mReceiverResolver.queryIntent(intent,
                        resolvedType, false, userId);
            }
        }
        
        // 到这里就得到了所有的广播接收者，静态的保存在receivers，动态的保存在registeredReceivers
        
        // 有的时候连续发了多次广播，但是之前的广播还没有转发给广播接收者
        // 当FLAG_RECEIVER_REPLACE_PENDING!=0的时候就代表这种情况
        // 此时AMS会用新的广播代替之前没来得及发出去的广播
        final boolean replacePending =
                (intent.getFlags()&Intent.FLAG_RECEIVER_REPLACE_PENDING) != 0;
        // NR是动态注册的接收者的数量
        int NR = registeredReceivers != null ? registeredReceivers.size() : 0;
        // ordered用来描述当前发送的广播是否是一个有序广播
        if (!ordered && NR > 0) {
            // 如果是一个无序广播，且存在动态注册的广播接收者
            // 将发送的广播转发给动态注册的广播接收者
            // 所以之前也说同等情况下动态广播比静态广播更收到无序广播
            ....
            final BroadcastQueue queue = broadcastQueueForIntent(intent);
            // 将intent所描述的广播和动态注册的目标广播接收者封装成BroadcastRecord类型对象
            // 用来描述一个AMS要执行的广播转发任务
            BroadcastRecord r = new BroadcastRecord(queue, intent, callerApp,
                    callerPackage, callingPid, callingUid, resolvedType, requiredPermissions,
                    appOp, brOptions, registeredReceivers, resultTo, resultCode, resultData,
                    resultExtras, ordered, sticky, false, userId);
            
            // replaceParallelBroadcastLocked会判断AMS的BroadcastQueue中的
            // 无序广播调度队列mParallelBroadcasts中是否存在对应的广播
            // 存在的话就替换掉原本存在的广播转发任务
            final boolean replaced = replacePending && queue.replaceParallelBroadcastLocked(r);
            // 如果替换了就不需要重复调用广播转发任务
            if (!replaced) {
                // 如果不是替换，就会在无序广播调度队列queue中添加这个广播转发任务
                queue.enqueueParallelBroadcastLocked(r);
                // 该方法会调用queue进行广播转发任务
                queue.scheduleBroadcastsLocked();
            }
            registeredReceivers = null;
            NR = 0;
        }
        // 如果是一个有序广播，将静态广播接收者列表receivers和动态广播接收者列表registeredReceivers合并
        int ir = 0;
        if (receivers != null) {
            ....
            // NT是静态注册的接收者的数量
            int NT = receivers != null ? receivers.size() : 0;
            int it = 0;
            ResolveInfo curt = null;
            BroadcastFilter curr = null;
            // 一个循环判断优先级合并的过程，这里采用双指针的方式合并
            while (it < NT && ir < NR) {
                if (curt == null) {
                    curt = (ResolveInfo)receivers.get(it);
                }
                if (curr == null) {
                    curr = registeredReceivers.get(ir);
                }
                // 对比接受接收优先级
                if (curr.getPriority() >= curt.priority) {
                    // 最后合并在receivers列表中
                    receivers.add(it, curr);
                    ir++;
                    curr = null;
                    it++;
                    NT++;
                } else {
                    // Skip to the next ResolveInfo in the final list.
                    it++;
                    curt = null;
                }
            }
        }
        // 动态数组中还有剩余未被添加的广播接收者
        while (ir < NR) {
            if (receivers == null) {
                receivers = new ArrayList();
            }
            receivers.add(registeredReceivers.get(ir));
            ir++;
        }
        ....
        //最后receivers就是按照优先级排序的静态和动态注册的广播接收者的集合
        if ((receivers != null && receivers.size() > 0)
                || resultTo != null) {
            BroadcastQueue queue = broadcastQueueForIntent(intent);
            //将intent所描述的广播和所有广播接收者封装成BroadcastRecord类型对象
            BroadcastRecord r = new BroadcastRecord(queue, intent, callerApp,
                    callerPackage, callingPid, callingUid, resolvedType,
                    requiredPermissions, appOp, brOptions, receivers, resultTo, resultCode,
                    resultData, resultExtras, ordered, sticky, false, userId);
            // replaceOrderedBroadcastLocked会判断AMS的BroadcastQueue中的
            // 有序广播调度队列mOrderedBroadcasts中是否存在对应的广播
            // 存在的话就替换掉原本存在的广播
            boolean replaced = replacePending && queue.replaceOrderedBroadcastLocked(r);
            
            // 如果替换了就不需要重复调用广播转发任务。
            if (!replaced) {
                // 如果不是替换，就会将r所描述的广播转发任务添加到AMS的有序广播调度队列中
                queue.enqueueOrderedBroadcastLocked(r);
                // 该方法会调用queue进行广播转发任务
                queue.scheduleBroadcastsLocked();
            }
        } else {
            ....
        }
        
        return ActivityManager.BROADCAST_SUCCESS;
    }
        
}
```
&ensp;&ensp; 这一步找到了所有的<font color='DodgerBlue'>广播接收者</font>，并根据情况将他们放在<font color='DodgerBlue'>无序广播调度队列**mParallelBroadcasts**</font>中和<font color='DodgerBlue'>有序广播调度队列**mOrderedBroadcasts**</font>中，在<font color='DodgerBlue'>非替换状态</font>下代表是新加入的广播，此时就会调用<font color='OrangeRed'>BroadcastQueue.scheduleBroadcastsLocked</font>将参数<font color='DodgerBlue'>intent所描述的广播</font>转发给对应的<font color='DodgerBlue'>广播接收者</font>接收。<font color='DodgerBlue'>替换状态</font>下就不需要调用该方法。

&ensp;&ensp; 下面简单看下上面针对<font color='DodgerBlue'>无序</font>/<font color='OrangeRed'>有序</font>广播的替换广播转发任务r的<font color='DodgerBlue'>queue.replaceParallelBroadcastLocked(r)</font>/<font color='OrangeRed'>queue.replaceOrderedBroadcastLocked(r)</font>函数和添加广播转发任务r的<font color='DodgerBlue'>queue.enqueueParallelBroadcastLocked(r)</font>/<font color='OrangeRed'>queue.enqueueOrderedBroadcastLocked(r)</font>函数对应的代码。
##### BroadcastQueue替换和添加广播转发任务相关操作
**frameworks/base/services/core/java/com/android/server/am/BroadcastQueue.java**
```java
public final class BroadcastQueue {
    ....
    // 无序广播列表
    final ArrayList<BroadcastRecord> mParallelBroadcasts = new ArrayList<>();
    // 有序广播列表
    final ArrayList<BroadcastRecord> mOrderedBroadcasts = new ArrayList<>();
    // 无序广播转发任务替换
    public final boolean replaceParallelBroadcastLocked(BroadcastRecord r) {
        final Intent intent = r.intent;
        for (int i = mParallelBroadcasts.size() - 1; i >= 0; i--) {
            final Intent curIntent = mParallelBroadcasts.get(i).intent;
            if (intent.filterEquals(curIntent)) {
                mParallelBroadcasts.set(i, r);
                return true;
            }
        }
        return false;
    }
    
    // 有序广播转发任务替换
    public final boolean replaceOrderedBroadcastLocked(BroadcastRecord r) {
        final Intent intent = r.intent;
        for (int i = mOrderedBroadcasts.size() - 1; i > 0; i--) {
            if (intent.filterEquals(mOrderedBroadcasts.get(i).intent)) {
                mOrderedBroadcasts.set(i, r);
                return true;
            }
        }
        return false;
    }
    
    // 无序广播转发任务添加
    public void enqueueParallelBroadcastLocked(BroadcastRecord r) {
        mParallelBroadcasts.add(r);
        r.enqueueClockTime = System.currentTimeMillis();
    }
    // 有序广播转发任务添加
    public void enqueueOrderedBroadcastLocked(BroadcastRecord r) {
        mOrderedBroadcasts.add(r);
        r.enqueueClockTime = System.currentTimeMillis();
    }
}
```
&ensp;&ensp; 这部分代码比较简单，就不做讲解了，之后我们来看关于<font color='OrangeRed'>BroadcastQueue.scheduleBroadcastsLocked</font>这个调度广播队列的广播转发任务这个操作
#### BroadcastQueue.scheduleBroadcastsLocked
**frameworks/base/services/core/java/com/android/server/am/BroadcastQueue.java**
```java
public final class BroadcastQueue {
    static final int BROADCAST_INTENT_MSG = ActivityManagerService.FIRST_BROADCAST_QUEUE_MSG;
    ....
    public void scheduleBroadcastsLocked() {
        ....
        // mBroadcastsScheduled 用来描述AMS是否已经向它所在的线程的消息队列发送一个
        // 类型为BROADCAST_INTENT_MSG的消息。
        // AMS就是通过BROADCAST_INTENT_MSG消息来调度保存在无序广播调度队列
        // mParallelBroadcasts和有序广播调度队列mOrderedBroadcasts
        // 中的广播转发任务。
        
        // mBroadcastsScheduled == true代表AMS所在线程的消息队列已经存在一个类型为
        // BROADCAST_INTENT_MSG的消息
        if (mBroadcastsScheduled) {
            return;
        }
        // 向线程的消息队列发送一个类型为BROADCAST_INTENT_MSG的消息
        mHandler.sendMessage(mHandler.obtainMessage(BROADCAST_INTENT_MSG, this));
        mBroadcastsScheduled = true;
    }
    ....
    
    final BroadcastHandler mHandler;
    
    // mHandler和AMS的处于同一个线程。
    // 所以这里与AMS中共用一个Looper
    // Looper：用于轮询消息队列，一个线程只有一个Looper
    BroadcastQueue(ActivityManagerService service, Handler handler,
            String name, long timeoutPeriod, boolean allowDelayBehindServices) {
        mService = service;
        // 传入AMS的Looper实例化BroadcastHandler
        mHandler = new BroadcastHandler(handler.getLooper());
        mQueueName = name;
        mTimeoutPeriod = timeoutPeriod;
        mDelayBehindServices = allowDelayBehindServices;
    }
    
    private final class BroadcastHandler extends Handler {
        public BroadcastHandler(Looper looper) {
            super(looper, null, true);
        }

        @Override
        public void handleMessage(Message msg) {
            switch (msg.what) {
                case BROADCAST_INTENT_MSG: {
                    // 进入processNextBroadcast函数进行广播转发
                    processNextBroadcast(true);
                } break;
                case BROADCAST_TIMEOUT_MSG: {
                    // 处理广播转发任务执行超时的情况
                    synchronized (mService) {
                        broadcastTimeoutLocked(true);
                    }
                } break;
                ....
            }
        }
    }
    
    final void processNextBroadcast(boolean fromMsg) {
        synchronized(mService) {
            // BroadcastRecord类型对象r是拥有所描述的广播和其对应的所有广播接收者的对象
            // 可以说就是广播转发任务
            BroadcastRecord r;
            ....
            // fromMsg调用时传入的是true，这里代表处理BROADCAST_INTENT_MSG的消息的
            if (fromMsg) { 
                // 将mBroadcastsScheduled设置为true，代表处理了BROADCAST_INTENT_MSG消息
                // 在我们前面流程scheduleBroadcastsLocked函数内有个判断就是防止连续多个
                // BROADCAST_INTENT_MSG消息导致多次处理，只有到了这里才能接受下一次BROADCAST_INTENT_MSG消息
                mBroadcastsScheduled = false;
            }

            // mParallelBroadcasts是无序广播调度队列，无序广播也是并行的广播任务
            // 不像有序广播任务需要等待前一个接收者处理完毕才能继续发送
            // 这里的循环就是处理无序广播转发任务
            // 即将mParallelBroadcasts内保存的广播发送给它的目标广播接收者处理
            while (mParallelBroadcasts.size() > 0) {
                r = mParallelBroadcasts.remove(0);
                r.dispatchTime = SystemClock.uptimeMillis();
                r.dispatchClockTime = System.currentTimeMillis();
                ....
                // 获得广播接收者的数量
                final int N = r.receivers.size();
                // 将无序广播发送给每一个广播接收者
                for (int i=0; i<N; i++) {
                    Object target = r.receivers.get(i);
                    // 这里就是发送的部分，之后会讲到
                    deliverToRegisteredReceiverLocked(r, (BroadcastFilter)target, false, i);
                }
                ....
            }
            ....
            // 之后是调度有序广播调度队列中的广播转发任务
            // 对于有序或静态广播而言，需要依次向每个广播接收者发送广播
            // 并在前一个接收者处理完毕后才能发送将广播发送给下一个接收者
            // 根据之前的代码知道，静态注册的广播接收者不会进入无序广播调度队列
            // 静态注册的目标广播接收者可能在广播发送出来时还未启动
           
            // 这里是用来处理静态注册但还未启动的广播接收者
            // mPendingBroadcast代表一个正在等待静态注册目标广播接收者
            // 启动起来的的广播转发任务
            if (mPendingBroadcast != null) {
                // 如果存在这样的广播
                boolean isDead;
                // 检查这个静态注册的广播接收者所运行的应用程序进程是否已经启动
                synchronized (mService.mPidsSelfLocked) {
                    ProcessRecord proc = mService.mPidsSelfLocked.get(mPendingBroadcast.curApp.pid);
                    isDead = proc == null || proc.crashing;
                }
                if (!isDead) {
                    // 如果正在启动，AMS就继续等待
                    return;
                } else {
                    // 已经启动了，准备向它发送一个广播
                    mPendingBroadcast.state = BroadcastRecord.IDLE;
                    mPendingBroadcast.nextReceiver = mPendingBroadcastRecvIndex;
                    mPendingBroadcast = null;
                }
            }
            ....
            // 循环从有序广播队列mOrderedBroadcasts中取出第一个还未处理的广播转发任务
            do {
                if (mOrderedBroadcasts.size() == 0) {
                    // 这里代表没有广播转发任务需要处理了
                    ....
                    // AMS还会释放一些因为要处理静态广播启动的不再需要的进程
                    return;
                }
                // 取出第一个队列中下一个广播转发任务
                r = mOrderedBroadcasts.get(0);
                boolean forceReceive = false;
            
                // 获得这个广播转发任务的广播接收者数量
                int numReceivers = (r.receivers != null) ? r.receivers.size() : 0;
                // 这里检查该广播是否处理超时
                // 单个广播任务的处理时间为2*mTimeoutPeriod*numReceivers
                // 也就是2*广播接收者数量*mTimeoutPeriod(单个接收者处理时间)
                // mTimeoutPeriod是AMS实例化BroadcastQueue时传入的广播超时时间
                // 其中 前台广播超时时间为10s 后台广播超时时间为60s
                if (mService.mProcessesReady && r.dispatchTime > 0) {
                    long now = SystemClock.uptimeMillis();
                    // 判断执行时间是否超过
                    if ((numReceivers > 0) &&
                            (now > r.dispatchTime + (2*mTimeoutPeriod*numReceivers))) {
                        ....
                        //未完成的话就强制结束这个广播转发任务
                        broadcastTimeoutLocked(false); 
                        forceReceive = true;
                        // 这里表示继续处理下一个广播转发任务
                        r.state = BroadcastRecord.IDLE;
                    }
                }
                // 这里判断的广播转发任务r的广播接收者是否还在处理中，如果是就等待
                if (r.state != BroadcastRecord.IDLE) {
                    ....
                    return;
                }
                // 前一个目标广播接收者已经处理完毕，之后转发给下一个广播接收者 
                
                // 检查广播转发任务r是否已经完成
                if (r.receivers == null || r.nextReceiver >= numReceivers
                        || r.resultAbort || forceReceive) {
                    // 没有其他的广播接收者了
                    ....
                    //删除之前发送的BROADCAST_TIMEOUT_MSG消息
                    //代表广播转发任务r已经完成
                    cancelBroadcastTimeoutLocked();

                    ....
                    // 将完成的广播转发任务从有序广播调度队列mOrderedBroadcasts中删除
                    mOrderedBroadcasts.remove(0);
                    // 将r设置为null，然后就可以继续执行下一个广播转发任务
                    r = null;
                    looped = true;
                    continue;
                }
            }while (r == null);
    
            // 上面的流程是找到有序广播队列中第一个还未处理的广播任务
            // 广播转发任务r的广播接收者保存在它的成员变量receiver中
            // 下面处理其中的receiver
            // nextReceiver就是r的下一个广播接收者
            // recIdx就是下个广播接收者在广播接收者列表中的位置
            int recIdx = r.nextReceiver++;

            // 更新广播转发任务的启动时间，以便于在超时时杀死它
            r.receiverTime = SystemClock.uptimeMillis();
            if (recIdx == 0) {
                // 代表刚开始处理广播转发任务r,将当前时间记录在r.dispatchTime对象中
                r.dispatchTime = r.receiverTime;
                r.dispatchClockTime = System.currentTimeMillis();
            }
            // 检查是否已经向AMS所在线程发送了BROADCAST_TIMEOUT_MSG消息
            // 用于广播转发任务超时判断
            if (! mPendingBroadcastTimeoutMessage) {
                // 如果还没有发，就发送一个并在mTimeoutPeriod时间后处理
                long timeoutTime = r.receiverTime + mTimeoutPeriod;
                setBroadcastTimeoutLocked(timeoutTime);
                // 如果mTimeoutPeriod没有处理完成就代表处理超时了
            }
            
            // 获得下一个广播接收者
            final Object nextReceiver = r.receivers.get(recIdx);
            
            // BroadcastFilter代表动态注册的广播接收者
            if (nextReceiver instanceof BroadcastFilter) {
                BroadcastFilter filter = (BroadcastFilter)nextReceiver;
                // 调用deliverToRegisteredReceiverLocked向它发送一个广播
                // 因为动态注册的广播肯定是已经启动起来了
                deliverToRegisteredReceiverLocked(r, filter, r.ordered, recIdx);
                
                if (r.receiver == null || !r.ordered) {
                    // 这里r.receiver == null 代表广播接收者所在进程被kill
                    // r.ordered代表广播是一个无序广播
                    // 如果满足以上任意提哦啊蹇将r.state设置为BroadcastRecord.IDLE
                    // 代表不需要等待上一个目标广播接收者处理完成一个广播
                    // 就可以将该广播继续发送给下一个广播接收者
                    r.state = BroadcastRecord.IDLE;
                    // 调用scheduleBroadcastsLocked触发下一轮处理过程
                    scheduleBroadcastsLocked();
                } 
                ....
                return;
            
            }
            // 如果不是动态注册，即是静态注册的广播接收者
            // ResolveInfo是静态广播接收者的类型
            ResolveInfo info =
                (ResolveInfo)nextReceiver;
            // 获得静态注册的广播接收者所在应用程序的信息
            ComponentName component = new ComponentName(
                    info.activityInfo.applicationInfo.packageName,
                    info.activityInfo.name);

            ....
            
            // 检查权限操作
            ....
            // 获得静态广播接收者所在的应用程序进程名
            String targetProcess = info.activityInfo.processName;
            ProcessRecord app = mService.getProcessRecordLocked(targetProcess,
                    info.activityInfo.applicationInfo.uid, false);
            ....
            // 如果上面的一些权限啥的不通过，skip就会被赋值为true
            if (skip) {
                r.delivery[recIdx] = BroadcastRecord.DELIVERY_SKIPPED;
                r.receiver = null;
                r.curFilter = null;
                r.state = BroadcastRecord.IDLE;
                // 调用scheduleBroadcastsLocked触发下一轮处理过程
                scheduleBroadcastsLocked();
                return;
            }
            ....
            // 判断对应的应用程序进程是否已经启动
            if (app != null && app.thread != null) {
                //应用程序已经启动
                try {
                    app.addPackage(info.activityInfo.packageName,
                            info.activityInfo.applicationInfo.versionCode, mService.mProcessStats);
                    // 将广播发送给这个应用程序中的广播接收者处理
                    processCurBroadcastLocked(r, app);
                    return;
                    }catch (RemoteException e) {
                          ....
                    }catch (RuntimeException e) {
                          ....                    
                    }
            
            } 
            // 如果应用程序进程未启动
            // 调用AMS.startProcessLocked启动应用程序进程
            if ((r.curApp=mService.startProcessLocked(targetProcess,
                    info.activityInfo.applicationInfo, true,
                    r.intent.getFlags() | Intent.FLAG_FROM_BACKGROUND,
                    "broadcast", r.curComponent,
                    (r.intent.getFlags()&Intent.FLAG_RECEIVER_BOOT_UPGRADE) != 0, false, false))
                            == null) {
                // 应用进程启动失败时
                ....
                // 结束广播转发任务r的转发处理，方法内部会的r的一些参数置null
                finishReceiverLocked(r, r.resultCode, r.resultData,
                        r.resultExtras, r.resultAbort, false);
                // 调用scheduleBroadcastsLocked触发下一轮处理过程
                scheduleBroadcastsLocked();
                r.state = BroadcastRecord.IDLE;
                return;
            }
            // 启动成功后创建的ProcessRecord类型进程对象就会被赋值给r.curApp变量
            // 之后再将r赋值给mPendingBroadcast，表示待处理的广播的下一个广播接收者所在的应用程序进程已经启动起来了
            mPendingBroadcast = r;
            mPendingBroadcastRecvIndex = recIdx;
        }
    }
}
```
&ensp;&ensp; <font color='OrangeRed'>BroadcastQueue</font>内部有一个专门的用来处理<font color='OrangeRed'>BROADCAST_INTENT_MSG</font>消息的<font color='OrangeRed'>mHandler</font>，之后会调用内部的<font color='OrangeRed'>processNextBroadcast</font>方法，该方法内部针对<font color='DodgerBlue'>无序和有序广播</font>进行了不同的处理
- 首先会处理<font color='DodgerBlue'>无序广播</font>，循环的将无序广播调度队列<font color='OrangeRed'>mParallelBroadcasts</font>的广播转发任务取出，并调用<font color='OrangeRed'>deliverToRegisteredReceiverLocked</font>函数将广播任务转发给所有对应的<font color='DodgerBlue'>广播接收者</font>。

- 之后会处理<font color='DodgerBlue'>有序广播</font>
- - 首先判断是否存在静态注册但应用程序还在启动的广播接收者，有就等待他启动完成。
- - 循环从有序广播队列<font color='OrangeRed'>mOrderedBroadcasts</font>中取出第一个还未处理的广播转发任务，内部如果发现有还在处理的广播转发任务就等待它处理完成，如果是处理超时的就强制结束这个广播转发任务并找下一个未处理的广播转发任务，处理完成也找下一个还未处理的广播。
- - 获得了下一个还未处理的广播转发任务，得到它下一个广播接收者类型，如果是<font color='DodgerBlue'>动态注册</font>的广播接收者就会调用<font color='OrangeRed'>deliverToRegisteredReceiverLocked</font>继续进行处理。
- - 如果是<font color='DodgerBlue'>静态注册</font>的广播接收者，判断其所在的应用程序进程是否启动了，如果启动了调用用<font color='OrangeRed'>processCurBroadcastLocked</font>继续进行处理
；如果未启动，则先启动应用程序进程。启动失败就触发下一轮广播任务处理，否则在应用程序创建成功后函数内部再触发。

&ensp;&ensp; 接下来来看下对应两个流程部分的代码
#### BroadcastQueue.deliverToRegisteredReceiverLocked & BroadcastQueue.processCurBroadcastLocked
**frameworks/base/services/core/java/com/android/server/am/BroadcastQueue.java**
```java
public final class BroadcastQueue {
    ....
        private void deliverToRegisteredReceiverLocked(BroadcastRecord r,
            BroadcastFilter filter, boolean ordered, int index) {
        boolean skip = false;
        // 进行广播发送者权限检查
        if (filter.requiredPermission != null) {
            // AMS需要检查广播发送者的权限
            int perm = mService.checkComponentPermission(filter.requiredPermission,
                    r.callingPid, r.callingUid, -1, true);
            if (perm != PackageManager.PERMISSION_GRANTED) {
                // 没有权限
                skip = true;
            } else {
                // 有权限
                // 再进行AppOpsManager进行权限检测
                final int opCode = 
                AppOpsManager.permissionToOpCode(filter.requiredPermission);
                if (opCode != AppOpsManager.OP_NONE
                        && mService.mAppOpsService.noteOperation(opCode, r.callingUid,
                                r.callerPackage) != AppOpsManager.MODE_ALLOWED) {
                    // 代表AppOpsManager权限检测没通过
                    skip = true;
                }
            }
        }
        // 判断广播接收者存在需要检查的权限
        if (!skip && r.requiredPermissions != null && r.requiredPermissions.length > 0) {
            // AMS需要检查广播接收者的权限
            for (int i = 0; i < r.requiredPermissions.length; i++) {
                String requiredPermission = r.requiredPermissions[i];
                int perm = mService.checkComponentPermission(requiredPermission,
                        filter.receiverList.pid, filter.receiverList.uid, -1, true);
                if (perm != PackageManager.PERMISSION_GRANTED) {
                    // 没有权限
                    skip = true;
                    break;
                }
                // 有权限
                // 再进行AppOpsManager进行权限检测
                int appOp = AppOpsManager.permissionToOpCode(requiredPermission);
                if (appOp != AppOpsManager.OP_NONE && appOp != r.appOp
                        && mService.mAppOpsService.noteOperation(appOp,
                        filter.receiverList.uid, filter.packageName)
                        != AppOpsManager.MODE_ALLOWED) {
                    skip = true;
                    break;
                }
            }
        }
        ....   
        //省略了许多权限检查相关的内容
        
        try {
            if (filter.receiverList.app != null && filter.receiverList.app.inFullBackup) {
                // Skip delivery if full backup in progress
                // If it's an ordered broadcast, we need to continue to the next receiver.
                if (ordered) {
                    skipReceiverLocked(r);
                }
            } else {
                // 到这里代表权限检查全部通过，调用performReceiveLocked
                // 将广播转发任务r所描述的广播转发给filter所描述的目标广播接收者处理
                performReceiveLocked(filter.receiverList.app, filter.receiverList.receiver,
                        new Intent(r.intent), r.resultCode, r.resultData,
                        r.resultExtras, r.ordered, r.initialSticky, r.userId);
            }
            if (ordered) {
                r.state = BroadcastRecord.CALL_DONE_RECEIVE;
            }
        } catch (RemoteException e) {
                ....
        }
    }
    
    void performReceiveLocked(ProcessRecord app, IIntentReceiver receiver,
            Intent intent, int resultCode, String data, Bundle extras,
            boolean ordered, boolean sticky, int sendingUser) throws RemoteException {
        // app 用来描述广播接收者所运行的应用程序进程
        // receiver指向实现了IIntentReceiver接口的Binder代理对象，用来描述广播接收者
        // intent 用来描述要发送的广播
        if (app != null) {
            if (app.thread != null) {
                try {
                    // 如果目标广播接收者需要通过它运行在应用程序进程间接接收一个广播
                    // 这里就通过app的ApplicationThread代理对象thread的
                    // scheduleRegisteredReceiver函数来向它发送广播
                    // 之后就进入了ApplicationThreadProxy.scheduleRegisteredReceiver函数，之后分析
                    app.thread.scheduleRegisteredReceiver(receiver, intent, resultCode,
                            data, extras, ordered, sticky, sendingUser, app.repProcState);
                } catch (RemoteException ex) {
                    ...
                }
            } else {
                // 应用程序进程已经死了。广播接收者不存在
                throw new RemoteException("app.thread must not be null");
            }
        } else {
            // 不需要间接接收广播，调用IntentReceiver.performReceive处理
            receiver.performReceive(intent, resultCode, data, extras, ordered,
                    sticky, sendingUser);
        }
    }
    ....
    private final void processCurBroadcastLocked(BroadcastRecord r,
            ProcessRecord app) throws RemoteException {
        ....

        r.receiver = app.thread.asBinder();
        r.curApp = app;
        app.curReceiver = r;
        app.forceProcessStateUpTo(ActivityManager.PROCESS_STATE_RECEIVER);
        mService.updateLruProcessLocked(app, false, null);
        mService.updateOomAdjLocked();

        // 告诉应用程序启动此接收器。
        r.intent.setComponent(r.curComponent);

        boolean started = false;
        try {
            mService.notifyPackageUse(r.intent.getComponent().getPackageName(),
            PackageManager.NOTIFY_PACKAGE_USE_BROADCAST_RECEIVER);
            // 这里通过app的ApplicationThread代理对象thread的
            // scheduleReceiver
            // 之后就进入了ApplicationThreadProxy.scheduleReceiver，之后分析
            app.thread.scheduleReceiver(new Intent(r.intent), r.curReceiver,
                    mService.compatibilityInfoForPackageLocked(r.curReceiver.applicationInfo),
                    r.resultCode, r.resultData, r.resultExtras, r.ordered, r.userId,
                    app.repProcState);
            started = true;
        } finally {
            if (!started) {
                r.receiver = null;
                r.curApp = null;
                app.curReceiver = null;
            }
        }
    }
    ....
}
```
#### ApplicationThreadProxy.scheduleRegisteredReceiver & ApplicationThreadProxy.scheduleReceiver
**/framworks/base/core/java/android/app/ApplicationThreadNative.java**
```java
public abstract class ApplicationThreadNative extends Binder
        implements IApplicationThread {
        ....
        class ApplicationThreadProxy implements IApplicationThread {
            public void scheduleRegisteredReceiver(IIntentReceiver receiver, Intent intent,
               int resultCode, String dataStr, Bundle extras, boolean ordered,
               boolean sticky, int sendingUser, int processState) throws RemoteException {
            //将传入的参数写入Parcel类型对象data中
            Parcel data = Parcel.obtain();
            data.writeInterfaceToken(IApplicationThread.descriptor);
            data.writeStrongBinder(receiver.asBinder());
            intent.writeToParcel(data, 0);
            data.writeInt(resultCode);
            data.writeString(dataStr);
            data.writeBundle(extras);
            data.writeInt(ordered ? 1 : 0);
            data.writeInt(sticky ? 1 : 0);
            data.writeInt(sendingUser);
            data.writeInt(processState);
            // 通过Binder代理对象mRemote向应用程序发送一个
            // 类型为SCHEDULE_REGISTERED_RECEIVER_TRANSACTION的进程间通信请求
            mRemote.transact(SCHEDULE_REGISTERED_RECEIVER_TRANSACTION, data, null,
                   IBinder.FLAG_ONEWAY);
            data.recycle();
            }
            ....
            
            public final void scheduleReceiver(Intent intent, ActivityInfo info,
               CompatibilityInfo compatInfo, int resultCode, String resultData,
               Bundle map, boolean sync, int sendingUser, int processState) throws RemoteException {
            // 将传入的参数写入Parcel类型对象data中
            Parcel data = Parcel.obtain();
            data.writeInterfaceToken(IApplicationThread.descriptor);
            intent.writeToParcel(data, 0);
            info.writeToParcel(data, 0);
            compatInfo.writeToParcel(data, 0);
            data.writeInt(resultCode);
            data.writeString(resultData);
            data.writeBundle(map);
            data.writeInt(sync ? 1 : 0);
            data.writeInt(sendingUser);
            data.writeInt(processState);
            // 通过Binder代理对象mRemote向应用程序发送一个
            // 类型为SCHEDULE_RECEIVER_TRANSACTION的进程间通信请求
            mRemote.transact(SCHEDULE_RECEIVER_TRANSACTION, data, null,
                   IBinder.FLAG_ONEWAY);
            data.recycle();
    }
        }
        ....
}
```
&ensp;&ensp; 之后就进入<font color='OrangeRed'>AMS</font>中继续处理类型为<font color='OrangeRed'>SCHEDULE_REGISTERED_RECEIVER_TRANSACTION</font>和<font color='OrangeRed'>SCHEDULE_RECEIVER_TRANSACTION</font>进程间通信请求。
#### ApplicationThread.scheduleRegisteredReceiver & ApplicationThread.scheduleReceiver
**frameworks/base/core/java/android/app/ActivityThread.java**
```java
public final class ActivityThread {
    ....
    private class ApplicationThread extends ApplicationThreadNative {
        ....
        public void scheduleRegisteredReceiver(IIntentReceiver receiver, Intent intent,
                int resultCode, String dataStr, Bundle extras, boolean ordered,
                boolean sticky, int sendingUser, int processState) throws RemoteException {
            ....
            // 这里就进入到InnerReceiver对象receiver的performReceive方法
            // 每一个InnerReceiver对象内部都封装了一个广播接收者
            // 并代替它所封装的广播接收者注册到了AMS中
            // AMS发送一个广播时，实际上是将广播发送给这个与目标接收者相关联的InnerReceiver对象
            receiver.performReceive(intent, resultCode, dataStr, extras, ordered,
                    sticky, sendingUser);
        }
        ....
        public final void scheduleReceiver(Intent intent, ActivityInfo info,
                CompatibilityInfo compatInfo, int resultCode, String data, Bundle extras,
                boolean sync, int sendingUser, int processState) {
            ....
            // 将广播的信息封装成ReceiverData对象r
            ReceiverData r = new ReceiverData(intent, resultCode, data, extras,
                    sync, false, mAppThread.asBinder(), sendingUser);
            r.info = info;
            r.compatInfo = compatInfo;
            // 调用ActivityThread内部类H类型对象mH发送类型为RECEIVER的消息
            sendMessage(H.RECEIVER, r);
        }
        ....
    }
    private void sendMessage(int what, Object obj) {
        sendMessage(what, obj, 0, 0, false);
    }
    private void sendMessage(int what, Object obj, int arg1, int arg2, boolean async) {
        Message msg = Message.obtain();
        msg.what = what;
        msg.obj = obj;
        msg.arg1 = arg1;
        msg.arg2 = arg2;
        if (async) {
            msg.setAsynchronous(true);
        }
        mH.sendMessage(msg);
    }
    ....
    final H mH = new H();
    private class H extends Handler {
        ....
        public void handleMessage(Message msg) {
            switch (msg.what) {
                ....
                case RECEIVER:
                    handleReceiver((ReceiverData)msg.obj);
                    ....
                    break; 
                ....
            }
        ....
    }
    private void handleReceiver(ReceiverData data) {
        ....
        String component = data.intent.getComponent().getClassName();
    
        LoadedApk packageInfo = getPackageInfoNoCheck(
                data.info.applicationInfo, data.compatInfo);

        IActivityManager mgr = ActivityManagerNative.getDefault();

        BroadcastReceiver receiver;
        try {
            java.lang.ClassLoader cl = packageInfo.getClassLoader();
            data.intent.setExtrasClassLoader(cl);
            data.intent.prepareToEnterProcess();
            data.setExtrasClassLoader(cl);
            // 创建广播接收者的实例receiver
            receiver = (BroadcastReceiver)cl.loadClass(component).newInstance();
        } catch (Exception e) {
            ....
        }

        try {
            Application app = packageInfo.makeApplication(false, mInstrumentation);

            ContextImpl context = (ContextImpl)app.getBaseContext();
            sCurrentBroadcastIntent.set(data.intent);
            receiver.setPendingResult(data);
            // 调用广播接收者receiver的onReceive方法处理广播
            receiver.onReceive(context.getReceiverRestrictedContext(),
                    data.intent);
        } catch (Exception e) {
            ....
        } finally {
            sCurrentBroadcastIntent.set(null);
        }

        if (receiver.getPendingResult() != null) {
            data.finish();
        }
    }
    ....
}
```
&ensp;&ensp; <font color='DodgerBlue'>静态广播接收者</font>的处理到这里就结束了，<font color='OrangeRed'>ApplicationThread</font>会将广播信息通过<font color='OrangeRed'>ActivityThread</font>的<font color='OrangeRed'>mH</font>对象发送，并在其<font color='OrangeRed'>handleMessage</font>内调用<font color='OrangeRed'>handleReceiver</font>进行处理，<font color='OrangeRed'>handleReceiver</font>函数会创建<font color='DodgerBlue'>广播接收者</font>的实例<font color='OrangeRed'>receiver</font>，并获得其所在应用程序进程的<font color='DodgerBlue'>上下文环境</font>，最后调用<font color='OrangeRed'>receiver.onReceive</font>将对应的广播发送过去。

&ensp;&ensp; <font color='DodgerBlue'>有序和无序的动态广播接收者</font>的处理就进入到了<font color='OrangeRed'>InnerReceiver.performReceive</font>进行处理。
##### InnerReceiver.performReceive
**frameworks/base/core/java/android/app/LoadedApk.java**
```java
public final class LoadedApk {
    ....
    static final class ReceiverDispatcher {
        final static class InnerReceiver extends IIntentReceiver.Stub {
            final WeakReference<LoadedApk.ReceiverDispatcher> mDispatcher;
            final LoadedApk.ReceiverDispatcher mStrongRef;
            ....
            @Override
            public void performReceive(Intent intent, int resultCode, String data,
                    Bundle extras, boolean ordered, boolean sticky, int sendingUser) {
                final LoadedApk.ReceiverDispatcher rd;
                if (intent == null) {
                    Log.wtf(TAG, "Null intent received");
                    rd = null;
                } else {
                    rd = mDispatcher.get();
                }
                if (rd != null) {
                    // 调用ReceiverDispatcher的performReceive接收intent所描述的广播
                    rd.performReceive(intent, resultCode, data, extras,
                            ordered, sticky, sendingUser);
                } else {
                    ....
                }
            }
        }
        ....
        final Handler mActivityThread;
        public void performReceive(Intent intent, int resultCode, String data,
                Bundle extras, boolean ordered, boolean sticky, int sendingUser) {
            // Args是ReceiverDispatcher的内部类，继承了Runnable接口
            final Args args = new Args(intent, resultCode, data, extras, ordered,
                    sticky, sendingUser);
            ....
            // mActivityThread是一个Handler对象，指向了ActivityThread类的成员变量mH
            // 这里将intent所描述的广播封装成Args类型(继承Runnable的类型，之后分析)对象args，通mActivityThread
            // 的post方法发送到应用程序的主线程的消息队列中。
            if (intent == null || !mActivityThread.post(args)) {
                if (mRegistered && ordered) {
                    IActivityManager mgr = ActivityManagerNative.getDefault();
                    args.sendFinished(mgr);
                }
            }
        }
    }
    ....
}
// ActivityThread类的成员变量mH对象是内部类H，继承自Handler，没有重写post方法
// 所以上面调用mActivityThread.post最后是调用到Handler的post方法
public class Handler {
    ....
    private static Message getPostMessage(Runnable r) {
        Message m = Message.obtain();
        m.callback = r;
        return m;
    }
    ....
    public final boolean post(Runnable r){
       // 调用getPostMessage将Runnable封装成Message，r是其中的callback对象
       // 最后会将封装好的消息加入主线程Looper的MessageQueue对象mQueue中
       // 消息添加后，Looper中就会处理，最终会调用到Handler的dispatchMessage方法处理这个Message
       return  sendMessageDelayed(getPostMessage(r), 0);
    }
    ....
    public void dispatchMessage(Message msg) {
        //前面得知，广播被封装成Args对象并被赋值给封装后的Message对象的callback变量
        // 所以msg.callback 不为null
        if (msg.callback != null) {
            // 进入handleCallback方法
            handleCallback(msg);
        } else {
            ....
        }
    }
    ....
    private static void handleCallback(Message message) {
        // 这里调用的其实就是Args.run方法，所以我们又回到Args类分析
        message.callback.run();
    }
    ....
}
```
##### Args.run
**frameworks/base/core/java/android/app/LoadedApk.java**
```java
public final class LoadedApk {
    ....
    static final class ReceiverDispatcher {
        final BroadcastReceiver mReceiver;
        // 用来描述mReceiver所指向的广播接收者是否已经注册到AMS中
        final boolean mRegistered;
        ....
        final class Args extends BroadcastReceiver.PendingResult implements Runnable {
            // 用来描述一个广播，其目标接收者就是mReceiver指向的广播接收者
            // 上一个过程创建Args实例时赋值
            private Intent mCurIntent;
            // 用来描述mCurIntent所描述的广播是否是有序的
            private final boolean mOrdered;
            ....
            public void run() {
                // mReceiver 指向一个广播接收者
                final BroadcastReceiver receiver = mReceiver;
                final boolean ordered = mOrdered;

                final IActivityManager mgr = ActivityManagerNative.getDefault();
                final Intent intent = mCurIntent;
                ....
                
                try {
                    ClassLoader cl =  mReceiver.getClass().getClassLoader();
                    intent.setExtrasClassLoader(cl);
                    intent.prepareToEnterProcess();
                    setExtrasClassLoader(cl);
                    // 这里的设置是为了receiver.getPendingResult()!=null为true
                    // 也是代表广播发送成功
                    receiver.setPendingResult(this);
                    //调用广播接收者的onReceive处理广播
                    receiver.onReceive(mContext, intent);
                } catch (Exception e) {
                    ....
                }
                // 广播发送成功
                if (receiver.getPendingResult() != null) {
                    // 这里调用父类BroadcastReceiver.PendingResult内的finish方法，下面看一下。
                    finish();
                }
            }
            ....
        }
        ....
        
    }
    ....
}

public abstract class BroadcastReceiver {
    ....
    public static class PendingResult {
        // 表示广播是否是有序的
        final boolean mOrderedHint;
        // 广播的状态，有组件状态，已注册到AMS的状态，未注册到AMS的状态
        final int mType;
        
        ....
        public final void finish() {
            if (mType == TYPE_COMPONENT) {
                final IActivityManager mgr = ActivityManagerNative.getDefault();
                if (QueuedWork.hasPendingWork()) {
                    QueuedWork.singleThreadExecutor().execute( new Runnable() {
                        @Override public void run() {
                            sendFinished(mgr);
                        }
                    });
                } else {
                    sendFinished(mgr);
                }
            } else if (mOrderedHint && mType != TYPE_UNREGISTERED) {
                // 有序广播，同时注册到AMS的广播接收者才需要通知AMS广播已经处理完成
                // 之后AMS收到后就可以将有序广播转发给下一个目标接收者处理了
                final IActivityManager mgr = ActivityManagerNative.getDefault();
                sendFinished(mgr);
            }
            
            // 未注册到AMS当然就不用调用sendFinished通知AMS广播已经处理完成
        }
        ....
        public void sendFinished(IActivityManager am) {
            synchronized (this) {
                if (mFinished) {
                    throw new IllegalStateException("广播已经完成");
                }
                mFinished = true;
                try {
                    ....
                    if (mOrderedHint) {
                        am.finishReceiver(mToken, mResultCode, mResultData, mResultExtras,
                                mAbortBroadcast, mFlags);
                    } else {
                        // 广播已经发送给了广播接收者，但广播不是有序广播
                        // 但是我们还是需要告诉AMS广播已经发送完成
                        am.finishReceiver(mToken, 0, null, null, false, mFlags);
                    }
                } catch (RemoteException ex) {
                }
            }
        }
        ....
    }
}
```
&ensp;&ensp; 这里可以知道当广播发送出去后基本都会有一个通知<font color='OrangeRed'>AMS</font>广播发送完成的动作，针对有序广播，以便于<font color='OrangeRed'>AMS</font>可以将<font color='DodgerBlue'>有序广播</font>发送给下一个<font color='DodgerBlue'>广播接收者</font>。

&ensp;&ensp; 发送过程较为复杂，这里画了个流程图，有序要可以看下
!![image](/img/in_post/post_broadcase_send.png)
### Broadcast的接收过程
&ensp;&ensp; 接收过程就是在<font color='OrangeRed'>BroadcastReceiver.onReceive</font>方法内部了，根据继承类的实现不同，处理方式就不同，这里就放下抽象类的生命源码。
**frameworks/base/core/java/android/content/BroadcastReceiver.java**
```java
public abstract class BroadcastReceiver {
    ....
     public abstract void onReceive(Context context, Intent intent);
    ....
}
```
# 参考文献
《Android系统源代码情景分析(第三版)》

[广播相关学习-processNextBroadcast逻辑](https://www.jianshu.com/p/adc4faa000b9?utm_campaign=maleskine&utm_content=note&utm_medium=seo_notes&utm_source=recommendation)


 <font color='DodgerBlue'>Broadcast(广播)机制</font>
 $\color{DodgerBlue}{Broadcast(广播)机制}$
