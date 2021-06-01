---
layout: post
title:  "SystemUI StatusBar移动信号图标更新流程"
author: "陈宇瀚"
date:   2021-05-31 18:00:00 +0800
header-img: "img/img-head/img-head-boot-process.jpg"
categories: article
tags:
  - Android
  - SystemUI
---
# SystemUI StatusBar 手机信号相关图标的显示和更新流程分析

*以下源码基于AC8015版 android 9.0*
## StatusBar的图标控制器
**SystemUI**中**StatusBar**的图标控制器实现类为**StatusBarIconControllerImpl**，其继承**StatusBarIconController**接口，用于跟踪所有图标的状态，并将对应的状态发送给注册的图标管理器（**IconManagers**）；
### StatusBarIconControllerImpl的创建和获取
**StatusBarIconController**在**StatusBr**的**start**方法一步步调用到**makeStatusBarView**之后从**Dependency**中获取，而**StatusBr**的**start**方法则是随着**SystemUI**的启动而启动，具体是在**SystemBars**中通过反射实例化，并调用其**start**方法。有关**start**方法的源码如下：
**/SystemUI/src/com/android/systemui/statusbar/phone/StatusBar.java**
```java
public class StatusBar extends SystemUI implements DemoMode,
        DragDownHelper.DragDownCallback, ActivityStarter, OnUnlockMethodChangedListener,
        OnHeadsUpChangedListener, CommandQueue.Callbacks, ZenModeController.Callback,
        ColorExtractor.OnColorsChangedListener, ConfigurationListener, NotificationPresenter {
        ....
        private NetworkController mNetworkController;
        protected StatusBarIconController mIconController;
        private PhoneStatusBarPolicy mIconPolicy;
        private StatusBarSignalPolicy mSignalPolicy;
        ....
        @Override
        public void start() {
            ....
            // 网络控制器，之后移动信号图标显示和更新流程相关
            mNetworkController = Dependency.get(NetworkController.class);
            ....
            // 内部会构造mIconController
            createAndAddWindows();
            ....
            // 最后，调用图标策略以安装/更新所有图标。
            mIconPolicy = new PhoneStatusBarPolicy(mContext, mIconController);
            mSignalPolicy = new StatusBarSignalPolicy(mContext, mIconController);
            ....
        }    
        ....
    public void createAndAddWindows() {
        addStatusBarWindow();
    }
    private void addStatusBarWindow() {
        makeStatusBarView();
        ....
    }
    protected void makeStatusBarView() {
        final Context context = mContext;
        ....
        // StatusBarIconControllerImpl获取
        mIconController = Dependency.get(StatusBarIconController.class);
        ....
        // 快速设置相关内容加载
        View container = mStatusBarWindow.findViewById(R.id.qs_frame);
            if (container != null) {
                FragmentHostManager fragmentHostManager = FragmentHostManager.get(container);
                ExtensionFragmentListener.attachExtensonToFragment(container, QS.TAG, R.id.qs_frame,
                        Dependency.get(ExtensionController.class)
                        ....
                        .build());
                // 传入mIconController创造一个快速设置Host
                final QSTileHost qsh = SystemUIFactory.getInstance().createQSTileHost(mContext, this,
                        mIconController);
                ....
                fragmentHostManager.addTagListener(QS.TAG, (tag, f) -> {
                    QS qs = (QS) f;
                    if (qs instanceof QSFragment) {
                        ((QSFragment) qs).setHost(qsh);
                        mQSPanel = ((QSFragment) qs).getQsPanel();
                        mQSPanel.setBrightnessMirror(mBrightnessMirrorController);
                        mKeyguardStatusBar.setQSPanel(mQSPanel);
                    }
                });
            }
    }
    ....
}
```
**Dependency**通过懒加载提前初始化**SystemUI**整个生命周期都需要存在，同时适用于**SystemUI**大部分地方的类，其在**SystemUIApplication**（**SystemBars**启动前）启动时进行初始化，本质也是通过反射实例化，并调用其**start**方法。其中与**StatusBarIconControllerImpl**相关源码如下：
**/SystemUI/src/com/android/systemui/Dependency.java**
```java
public class Dependency extends SystemUI {
    private final ArrayMap<Object, DependencyProvider> mProviders = new ArrayMap<>();
    @Override
    public void start() {
        ....
        mProviders.put(StatusBarIconController.class, () ->
                new StatusBarIconControllerImpl(mContext));
                
        mProviders.put(NetworkController.class, () ->
                new NetworkControllerImpl(mContext, getDependency(BG_LOOPER),
                        getDependency(DeviceProvisionedController.class)));
    }
    
    ....
    public static <T> T get(Class<T> cls) {
        if(sDependency == null) {
            Log.d("Dependency","sDependency is null");
            return null;
        }
        return sDependency.getDependency(cls);
    }
}
```
### StatusBarIconControllerImpl的构造方法
**/SystemUI/src/com/android/systemui/statusbar/phone/StatusBarIconControllerImpl**
```java
public class StatusBarIconControllerImpl extends StatusBarIconList implements Tunable,
        ConfigurationListener, Dumpable, CommandQueue.Callbacks, StatusBarIconController {
        private final ArrayList<IconManager> mIconGroups = new ArrayList<>();
        private Context mContext;
        ....
        public StatusBarIconControllerImpl(Context context) {
            // 这里根据系统配置的config_statusBarIcons添加对应的Icon槽，用于之后Icon的加载
            super(context.getResources().getStringArray(
                    com.android.internal.R.array.config_statusBarIcons));
            Dependency.get(ConfigurationController.class).addCallback(this);

            mContext = context;
            ....
            // 设置CommandQueue监听图标相关回调
            SysUiServiceProvider.getComponent(context, CommandQueue.class)
                    .addCallbacks(this);
            Dependency.get(TunerService.class).addTunable(this, ICON_BLACKLIST);
        }
            
}
```
**StatusBarIconControllerImpl**继承了*StatusBarIconController**接口,当我们在**StatusBar**中获取到它的实例后，还会将它传给**PhoneStatusBarPolicy**和**StatusBarSignalPolicy**对象。**PhoneStatusBarPolicy**控制启动时装载哪些图标（蓝牙，定位等），而**StatusBarSignalPolicy**控制网络信号图标（移动网络，WiFi，以太网）的变化。

到这里就完成了状态栏图标控制器**StatusBarIconControllerImpl**的构造。之后要看下具体的**Icon**显示和更新是如下传入到这里控制的。这里只分析移动信号图标的显示和更新流程，其他图标可以该流程参考。

## StatusBar的手机信号处理类
**Icon**的信号处理控制逻辑基于**NetworkControllerImpl**，其继承自**BroadcastReceiver**，类内部监听网络状态变化，SIM卡状态变化等广播，执行相应的操作。
### NetworkControllerImpl的创建
**NetworkControllerImpl**该类在**Dependency**的**start**方法中构造，**StatusBar**的**start**方法中获取并赋值给其内部的**mNetworkController**变量。过程与**StatusBarIconControllerImpl**类似，这里就略过了。

### NetworkControllerImpl构造方法
**/SystemUI/src/com/android/systemui/statusbar/policy/NetworkControllerImpl.java**
```java
public class NetworkControllerImpl extends BroadcastReceiver
        implements NetworkController, DemoMode, DataUsageController.NetworkNameProvider,
        ConfigurationChangedReceiver, Dumpable {
    ....
    // WiFi信号控制器
    /*final */WifiSignalController mWifiSignalController = null;

    // 以太网信号控制器
    @VisibleForTesting
    final EthernetSignalController mEthernetSignalController;

    // 移动网络信号控制器列表（每个SIM卡对应一个）
    @VisibleForTesting
    final SparseArray<MobileSignalController> mMobileSignalControllers = new SparseArray<>();
    
    // 移动网络，WiFi网络，网络连接状态管理器
    private final TelephonyManager mPhone;
    private final WifiManager mWifiManager;
    private final ConnectivityManager mConnectivityManager;
    // 每一张SIM卡都对应一个Subscription，用谁家的SIM卡就相当于订阅(Subscription)谁家的业务。
    // 而每一个Subscription对应一个MobileSignalController
    private List<SubscriptionInfo> mCurrentSubscriptions = new ArrayList<>();
    // 处理程序，用于接收所有广播。
    private final Handler mReceiverHandler;
    // 处理程序，所有回调都在其上进行。
    private final CallbackHandler mCallbackHandler;
    
    public NetworkControllerImpl(Context context, Looper bgLooper,
            DeviceProvisionedController deviceProvisionedController) {
        // 1.进入构造
        this(context, (ConnectivityManager) context.getSystemService(Context.CONNECTIVITY_SERVICE),
                (TelephonyManager) context.getSystemService(Context.TELEPHONY_SERVICE),
                (WifiManager) context.getSystemService(Context.WIFI_SERVICE),
                SubscriptionManager.from(context), Config.readConfig(context), bgLooper,
                new CallbackHandler(),
                new AccessPointControllerImpl(context),
                new DataUsageController(context),
                new SubscriptionDefaults(),
                deviceProvisionedController);
        // 3. 设置广播监听
        mReceiverHandler.post(mRegisterListeners);
    }
    // 2.构造控制器，管理器和设置连接状态监听
    @VisibleForTesting
    NetworkControllerImpl(Context context, ConnectivityManager connectivityManager,
            TelephonyManager telephonyManager, WifiManager wifiManager,
            SubscriptionManager subManager, Config config, Looper bgLooper,
            CallbackHandler callbackHandler,
            AccessPointControllerImpl accessPointController,
            DataUsageController dataUsageController,
            SubscriptionDefaults defaultsHandler,
            DeviceProvisionedController deviceProvisionedController) {
                mContext = context;
        // 配置各种变量
        ....
        // 内部处理所有广播监听
        mReceiverHandler = new Handler(bgLooper);
        // 内部处理所有回调监听，通过addCallback向其内部的监听列表添加监听对象。
        mCallbackHandler = callbackHandler;
        ....
        mConnectivityManager = connectivityManager;
        // telephony
        mPhone = telephonyManager;

        // wifi
        mWifiManager = wifiManager;     
        ....
        if (mWifiManager != null) {
            mWifiSignalController = new WifiSignalController(mContext, mHasMobileDataFeature,
                    mCallbackHandler, this, mWifiManager);
        }
        mEthernetSignalController = new EthernetSignalController(mContext, mCallbackHandler, this);
        ....
        // 用于设置连接状态的监听
        ConnectivityManager.NetworkCallback callback = new ConnectivityManager.NetworkCallback(){
            private Network mLastNetwork;
            private NetworkCapabilities mLastNetworkCapabilities;

            @Override
            public void onCapabilitiesChanged(
                Network network, NetworkCapabilities networkCapabilities) {
                boolean lastValidated = (mLastNetworkCapabilities != null) &&
                    mLastNetworkCapabilities.hasCapability(NET_CAPABILITY_VALIDATED);
                boolean validated =
                    networkCapabilities.hasCapability(NET_CAPABILITY_VALIDATED);

                // 这个回调会被调用很多次(例如当信号强度发生变化时)，
                // 这里可以避免在连接状态保持不变的情况下更新图标。
                if (network.equals(mLastNetwork) &&
                    networkCapabilities.equalsTransportTypes(mLastNetworkCapabilities) &&
                    validated == lastValidated) {
                    return;
                }
                mLastNetwork = network;
                mLastNetworkCapabilities = networkCapabilities;
                updateConnectivity();
            }
        };
        mConnectivityManager.registerDefaultNetworkCallback(callback, mReceiverHandler);
    }
    
    ....
    private final Runnable mRegisterListeners = new Runnable() {
        @Override
        public void run() {
            registerListeners();
        }
    };
    // 3.设置广播监听
    private void registerListeners() {
        ....
        IntentFilter filter = new IntentFilter();
        // wifi相关
        // wifi信号强度广播
        filter.addAction(WifiManager.RSSI_CHANGED_ACTION);
        // wifi状态变化广播
        filter.addAction(WifiManager.WIFI_STATE_CHANGED_ACTION);
        // wifi连接状态改变
        filter.addAction(WifiManager.NETWORK_STATE_CHANGED_ACTION);
        
        // 移动网络相关
        // SIM卡状态改变
        filter.addAction(TelephonyIntents.ACTION_SIM_STATE_CHANGED);
        // 数据语音订阅修改
        filter.addAction(TelephonyIntents.ACTION_DEFAULT_DATA_SUBSCRIPTION_CHANGED);
        filter.addAction(TelephonyIntents.ACTION_DEFAULT_VOICE_SUBSCRIPTION_CHANGED);
        filter.addAction(TelephonyIntents.ACTION_SERVICE_STATE_CHANGED);
        filter.addAction(TelephonyIntents.SPN_STRINGS_UPDATED_ACTION);
        // 连接状态相关
        // 网络连接状态发生变化
        filter.addAction(ConnectivityManager.CONNECTIVITY_ACTION);
        // 网络连接可能不好
        filter.addAction(ConnectivityManager.INET_CONDITION_ACTION);
        // 切换飞行模式时
        filter.addAction(Intent.ACTION_AIRPLANE_MODE_CHANGED);
        filter.addAction(CarrierConfigManager.ACTION_CARRIER_CONFIG_CHANGED);
        mContext.registerReceiver(this, filter, null, mReceiverHandler);
        mListening = true;

        // 4.更新移动网络控制器
        updateMobileControllers();
    }
    
    private void updateMobileControllers() {
        if (!mListening) {
            return;
        }
        doUpdateMobileControllers();
    }
    @VisibleForTesting
    void doUpdateMobileControllers() {
        // 获得当前插入的SIM卡的列表
        List<SubscriptionInfo> subscriptions = mSubscriptionManager.getActiveSubscriptionInfoList();
        ....
        // 设置当前插入SIM卡的Subscriptions（订阅信息）
        setCurrentSubscriptions(subscriptions);
        ....
    }
    @VisibleForTesting
    void setCurrentSubscriptions(List<SubscriptionInfo> subscriptions) {
        mCurrentSubscriptions = subscriptions;
        
        // 建立一个缓存列表存放之前的的MobileSignalController列表
        SparseArray<MobileSignalController> cachedControllers =
                new SparseArray<MobileSignalController>();
        for (int i = 0; i < mMobileSignalControllers.size(); i++) {
            cachedControllers.put(mMobileSignalControllers.keyAt(i),
                    mMobileSignalControllers.valueAt(i));
        }
        // 清空当前的MobileSignalController列表
        mMobileSignalControllers.clear();
        
        // 获得当前SIM卡数量
        final int num = subscriptions.size();
        
        // 每个卡有一个对应的MobileSignalController
        for (int i = 0; i < num; i++) {
            int subId = subscriptions.get(i).getSubscriptionId();
            // 如果之前缓存列表存在这个SIM卡的控制器直接复用，否则新建
            if (cachedControllers.indexOfKey(subId) >= 0) {
                mMobileSignalControllers.put(subId, cachedControllers.get(subId));
                cachedControllers.remove(subId);
            } else {
                MobileSignalController controller = new MobileSignalController(mContext, mConfig,
                        mHasMobileDataFeature, mPhone, mCallbackHandler,
                        this, subscriptions.get(i), mSubDefaults, mReceiverHandler.getLooper());
                controller.setUserSetupComplete(mUserSetup);
                mMobileSignalControllers.put(subId, controller);
                // 设置第0个插槽的SIM卡的控制器为默认的控制器
                if (subscriptions.get(i).getSimSlotIndex() == 0) {
                    mDefaultSignalController = controller;
                }
                if (mListening) {
                    // 调用MobileSignalController的registerListener注册监听
                    controller.registerListener();
                }
            }
            ....
            // 强制更新SignalClusters和NetworkSignalChangedCallbacks上的所有回调。
            notifyAllListeners();
            ....
        }
    }
}
```
看一下**MobileSignalController**的**registerListener**方法，该方法内设置了一些**telephony**状态，数据变化相关的监听。有变化就会调用**updateTelephony**进行状态更新。

**/SystemUI/src/com/android/systemui/statusbar/policy/MobileSignalController.java**
```java
public class MobileSignalController extends SignalController<
        MobileSignalController.MobileState, MobileSignalController.MobileIconGroup> {
    ....
    private final TelephonyManager mPhone;
    public void registerListener() {
        // 通过TelephonyManager注册监听，当telephony状态发生变化时
        // 会调用updateTelephony()方法进行状态更新
        mPhone.listen(mPhoneStateListener,
                PhoneStateListener.LISTEN_SERVICE_STATE
                        | PhoneStateListener.LISTEN_SIGNAL_STRENGTHS
                        | PhoneStateListener.LISTEN_CALL_STATE
                        | PhoneStateListener.LISTEN_DATA_CONNECTION_STATE
                        | PhoneStateListener.LISTEN_DATA_ACTIVITY
                        | PhoneStateListener.LISTEN_CARRIER_NETWORK_CHANGE
                        | PhoneStateListener.LISTEN_PRECISE_DATA_CONNECTION_STATE);
        ....
    }
    //基于mServiceState、mSignalStrength、mDataNetType、mDataState和mSimState更新当前状态。
    // 当其中一个状态更新时，就会调用。这将在必要时调用监听。
    private final void updateTelephony() {
        ....
        // 最后调用到notifyListeners通知所有的SignalCallback
        notifyListenersIfNecessary();
    }
    @Override
    public void notifyListeners(SignalCallback callback) {
        MobileIconGroup icons = getIcons();
        // 根据当前网络状态配置各项参数
        ....
        // 这里的callback就是NetworkControllerImpl中的mCallbackHandler
        // 调用其setMobileDataIndicators方法进行信号状态更新
        callback.setMobileDataIndicators(statusIcon, qsIcon, typeIcon, networkIcon, volteIcon,
                qsTypeIcon,activityIn, activityOut, dataContentDescription, description,
                 icons.mIsWide, mSubscriptionInfo.getSubscriptionId(), mCurrentState.roaming,
                 false);
    }
}
```
至此就完成了**WIFI**，**以太网**，**移动网络**三个网络信号控制器的初始化,之后每个控制器会根据对应的变化显示和更新图标。
## 移动网络Icon添加和更新流程
以网络状态变化为例，此时在**NetworkControllerImpl**中收到**CONNECTIVITY_ACTION**广播，然后执行如下操作：
**/SystemUI/src/com/android/systemui/statusbar/policy/NetworkControllerImpl.java**
```java
public class NetworkControllerImpl extends BroadcastReceiver
        implements NetworkController, DemoMode, DataUsageController.NetworkNameProvider,
        ConfigurationChangedReceiver, Dumpable {
        ....
    @Override
    public void onReceive(Context context, Intent intent) {
        final String action = intent.getAction();
        switch (action) {
            ....
            case ConnectivityManager.CONNECTIVITY_ACTION:
            case ConnectivityManager.INET_CONDITION_ACTION:
                updateConnectivity();
                break;
            ....
        }
    }
    private void updateConnectivity() {
        ....
        pushConnectivityToSignals();
    }
    // 将当前连接状态推送到所有SignalController。
    private void pushConnectivityToSignals() {
        // 更新移动网络Icon图标
        for (int i = 0; i < mMobileSignalControllers.size(); i++) {
            MobileSignalController mobileSignalController = mMobileSignalControllers.valueAt(i);
            mobileSignalController.updateConnectivity(mConnectedTransports, mValidatedTransports);
        }
        // 更新WiFi网络Icon图标
        if (mWifiSignalController != null)
            mWifiSignalController.updateConnectivity(mConnectedTransports, mValidatedTransports);
        // 更新以太网Icon图标
        mEthernetSignalController.updateConnectivity(mConnectedTransports, mValidatedTransports);
    }
```
这里主要看移动网络信号**Icon**的更新，进入**MobileSignalController.updateConnectivity**方法
**/SystemUI/src/com/android/systemui/statusbar/policy/MobileSignalController.java**
```java
public class MobileSignalController extends SignalController<
        MobileSignalController.MobileState, MobileSignalController.MobileIconGroup> {
    ....
    @Override
    public void updateConnectivity(BitSet connectedTransports, BitSet validatedTransports) {
        ....
        notifyListenersIfNecessary();
    }    
    public void notifyListenersIfNecessary() {
        if (isDirty()) {
            saveLastState();
            // 这部分就回到上面分析的通知所有的SignalCallback
            notifyListeners();
        }
    }
     @Override
    public void notifyListeners(SignalCallback callback) {
        MobileIconGroup icons = getIcons();
        // 根据当前网络状态配置各项参数
        ....
        // 这里的callback就是NetworkControllerImpl中的mCallbackHandler
        // 调用其setMobileDataIndicators方法进行信号状态更新
        callback.setMobileDataIndicators(statusIcon, qsIcon, typeIcon, networkIcon, volteIcon,
                qsTypeIcon,activityIn, activityOut, dataContentDescription, description,
                 icons.mIsWide, mSubscriptionInfo.getSubscriptionId(), mCurrentState.roaming,
                 false);
    }
}
```
**/SystemUI/src/com/android/systemui/statusbar/policy/CallbackHandler.java**
```java
public class CallbackHandler extends Handler implements EmergencyListener, SignalCallback {
    ....
    @Override
    public void setMobileDataIndicators(final IconState statusIcon, final IconState qsIcon,
            final int statusType, final int networkIcon, final int volteType,
            final int qsType,final boolean activityIn,
            final boolean activityOut, final String typeContentDescription,
            final String description, final boolean isWide, final int subId, boolean roaming,
            boolean isDefaultData) {
        post(new Runnable() {
            @Override
            public void run() {
                for (SignalCallback signalCluster : mSignalCallbacks) {                    signalCluster.setMobileDataIndicators(statusIcon, qsIcon, statusType,
                            networkIcon, volteType, qsType, activityIn, activityOut,
                            typeContentDescription, description, isWide, subId, roaming,
                            isDefaultData);
                }
            }
        });
    }
    @Override
    @SuppressWarnings("unchecked")
    public void handleMessage(Message msg) {
        switch (msg.what) {
            ....
            case MSG_ADD_REMOVE_SIGNAL:
                if (msg.arg1 != 0) {
                    mSignalCallbacks.add((SignalCallback) msg.obj);
                } else {
                    mSignalCallbacks.remove((SignalCallback) msg.obj);
                }
                break;
            ....
        }
    }
    ....
    public void setListening(SignalCallback listener, boolean listening) {
        obtainMessage(MSG_ADD_REMOVE_SIGNAL, listening ? 1 : 0, 0, listener).sendToTarget();
    }
}
```
这里就会调用**mSignalCallbacks**列表中所有**SignalCallback**对象的**setMobileDataIndicators**方法，**mSignalCallbacks**内部的**SignalCallback**对象通过**setListening**方法添加，这里的**SignalCallback**对象是在**StatusBarSignalPolicy**中进行添加的，前面提到**StatusBar.start**方法内初始化了其内部一个**StatusBarSignalPolicy**对象，这里看下其内部的构造方法：
**/SystemUI/src/com/android/systemui/statusbar/phone/StatusBarSignalPolicy.java**
```java
public class StatusBarSignalPolicy implements NetworkControllerImpl.SignalCallback,
        SecurityController.SecurityControllerCallback, Tunable {
    private final StatusBarIconController mIconController;
    private final NetworkController mNetworkController;
    
    public StatusBarSignalPolicy(Context context, StatusBarIconController iconController) {
        mContext = context;

        ....

        mIconController = iconController;
        mNetworkController = Dependency.get(NetworkController.class);
        mSecurityController = Dependency.get(SecurityController.class);
        
        // 调用NetworkControllerImpl.addCallback添加到
        // CallbackHandler的信号回调列表mSignalCallbacks中
        mNetworkController.addCallback(this);
        mSecurityController.addCallback(this);
    }
    ....
   @Override
    public void setMobileDataIndicators(IconState statusIcon, IconState qsIcon, int statusType,
            int networkType, int volteIcon, int qsType, boolean activityIn, boolean activityOut,
            String typeContentDescription, String description, boolean isWide, int subId,
            boolean roaming, boolean isDefaultData) {
        MobileIconState state = getState(subId);
        ....
        // 更新当前网络信号状态

        state.visible = statusIcon.visible && !mBlockMobile;
        state.strengthId = statusIcon.icon;
        state.typeId = statusType;
        state.contentDescription = statusIcon.contentDescription;
        state.typeContentDescription = typeContentDescription;
        state.roaming = roaming;
        state.activityIn = activityIn && mActivityEnabled;
        state.activityOut = activityOut && mActivityEnabled;
        state.networkIcon = networkType;
        state.volteIcon = volteIcon;

        ....
        mIconController.setMobileIcons(mSlotMobile, MobileIconState.copyStates(mMobileStates));
        ....
    }

}
```
最后调用到**mIconController**来更新移动网络图标，**mIconController**即是**StatusBar**的**StatusBarIconControllerImpl**对象**mIconController**，**StatusBarIconControllerImpl.setMobileIcons**方法源码如下：
**/SystemUI/src/com/android/systemui/statusbar/phone/StatusBarIconControllerImpl.java**
```java
public class StatusBarIconControllerImpl extends StatusBarIconList implements Tunable,
        ConfigurationListener, Dumpable, CommandQueue.Callbacks, StatusBarIconController {
    ....
    private final ArrayList<IconManager> mIconGroups = new ArrayList<>()
    @Override
    public void setMobileIcons(String slot, List<MobileIconState> iconStates) {
        Slot mobileSlot = getSlot(slot);
        int slotIndex = getSlotIndex(slot);

        for (MobileIconState state : iconStates) {
            StatusBarIconHolder holder = mobileSlot.getHolderForTag(state.subId);
            if (holder == null) {
                holder = StatusBarIconHolder.fromMobileIconState(state);
                setIcon(slotIndex, holder);
            } else {
                holder.setMobileState(state);
                handleSet(slotIndex, holder);
            }
        }
    }
    
    @Override
    public void setIcon(int index, @NonNull StatusBarIconHolder holder) {
        boolean isNew = getIcon(index, holder.getTag()) == null;
        super.setIcon(index, holder);
        if (isNew) {
            addSystemIcon(index, holder);
        } else {
            handleSet(index, holder);
        }
    }
    private void addSystemIcon(int index, StatusBarIconHolder holder) {
        String slot = getSlotName(index);
        int viewIndex = getViewIndex(index, holder.getTag());
        boolean blocked = mIconBlacklist.contains(slot);

        mIconLogger.onIconVisibility(getSlotName(index), holder.isVisible());
        mIconGroups.forEach(l -> l.onIconAdded(viewIndex, slot, blocked, holder));
    }
    
    private void handleSet(int index, StatusBarIconHolder holder) {
        int viewIndex = getViewIndex(index, holder.getTag());
        mIconLogger.onIconVisibility(getSlotName(index), holder.isVisible());
        mIconGroups.forEach(l -> l.onSetIconHolder(viewIndex, holder));
    }
    
    @Override
    public void addIconGroup(IconManager group) {
        mIconGroups.add(group);
        List<Slot> allSlots = getSlots();
        for (int i = 0; i < allSlots.size(); i++) {
            Slot slot = allSlots.get(i);
            List<StatusBarIconHolder> holders = slot.getHolderListInViewOrder();
            boolean blocked = mIconBlacklist.contains(slot.getName());

            for (StatusBarIconHolder holder : holders) {
                int tag = holder.getTag();
                int viewIndex = getViewIndex(getSlotIndex(slot.getName()), holder.getTag());
                group.onIconAdded(viewIndex, slot.getName(), blocked, holder);
            }
        }
    }
}
```
这里根据是否是新增的**Icon**,分别进入到**addSystemIcon**和**handleSet**两个方法，最后遍历**mIconGroups**分别执行**onIconAdded**和**onSetIconHolder**回调。历**mIconGroups**是一个**IconManager**列表，通过**addIconGroup**向其内部添加**IconManager**。

**IconManager**用于将信息从**StatusBarIconController**转换为**ViewGroup**中的**ImageViews**。

查看源码有四个地方调用添加，主要看包含折叠状态的状态栏**CollapsedStatusBarFragment**类，这个就是我们状态栏，以及扩展状态下的状态栏，其中添加的是**IconManager**的子类**DarkIconManager**:
**/SystemUI/src/com/android/systemui/statusbar/phone/CollapsedStatusBarFragment.java**
```java
public class CollapsedStatusBarFragment extends Fragment implements CommandQueue.Callbacks {
    ....
    private DarkIconManager mDarkIconManager;
    ....
    @Override
    public View onCreateView(LayoutInflater inflater, @Nullable ViewGroup container,
            Bundle savedInstanceState) {
        return inflater.inflate(R.layout.status_bar, container, false);
    }
    @Override
    public void onViewCreated(View view, @Nullable Bundle savedInstanceState) {
        ....
        // 这里可以看出status_bar布局中的statusIcons就是我们展示各种Icon的区域
        mDarkIconManager = new DarkIconManager(view.findViewById(R.id.statusIcons));
        ....
        
    }
    
}

```
看下内部的**onIconAdded**和**onSetIconHolder**方法：
**/SystemUI/src/com/android/systemui/statusbar/phone/StatusBarIconController.java**
```java
public interface StatusBarIconController {
    ....
    public static class DarkIconManager extends IconManager {
        ....
        public DarkIconManager(LinearLayout linearLayout) {
            // 将布局传入IconManager
            super(linearLayout);
            mIconHPadding = mContext.getResources().getDimensionPixelSize(
                    R.dimen.status_bar_icon_padding);
            mDarkIconDispatcher = Dependency.get(DarkIconDispatcher.class);
        }
        ....
        @Override
        protected void onIconAdded(int index, String slot, boolean blocked,
                                   StatusBarIconHolder holder) {
            // 调用到父类的addHolder方法
            StatusIconDisplayable view = addHolder(index, slot, blocked, holder);
            ....
        }
    }
    
    public static class IconManager implements DemoMode {
        ....
        protected final ViewGroup mGroup;
        protected final Context mContext;
        public IconManager(ViewGroup group) {
            mGroup = group;
            mContext = group.getContext();
            mIconSize = mContext.getResources().getDimensionPixelSize(
                    R.dimen.status_bar_height);
            ....
        }
        ....
        protected StatusIconDisplayable addHolder(int index, String slot, boolean blocked,
                                                  StatusBarIconHolder holder) {
            switch (holder.getType()) {
                case TYPE_ICON:
                    return addIcon(index, slot, blocked, holder.getIcon());
                case TYPE_WIFI:
                    return addSignalIcon(index, slot, holder.getWifiState());
                case TYPE_MOBILE:
                    return addMobileIcon(index, slot, holder.getMobileState());
            }

            return null;
        }
        @VisibleForTesting
        protected StatusBarMobileView addMobileIcon(int index, String slot, MobileIconState state) {
            // 创建一个StatusBarMobileView
            StatusBarMobileView view = onCreateStatusBarMobileView(slot);
            view.applyMobileState(state);
            // 将view 添加进ViewGroup
            mGroup.addView(view, index, onCreateLayoutParams());
            ....
            return view;
        }
        
        private StatusBarMobileView onCreateStatusBarMobileView(String slot) {
            StatusBarMobileView view = StatusBarMobileView.fromContext(mContext, slot);
            return view;
        }
    
        ....
        public void onSetIconHolder(int viewIndex, StatusBarIconHolder holder) {
            switch (holder.getType()) {
                case TYPE_ICON:
                    onSetIcon(viewIndex, holder.getIcon());
                    return;
                case TYPE_WIFI:
                    onSetSignalIcon(viewIndex, holder.getWifiState());
                    return;
                case TYPE_MOBILE:
                    onSetMobileIcon(viewIndex, holder.getMobileState());
                default:
                    break;
            }
        }
        public void onSetMobileIcon(int viewIndex, MobileIconState state) {
            StatusBarMobileView view = (StatusBarMobileView) mGroup.getChildAt(viewIndex);
            if (view != null) {
                view.applyMobileState(state);
            }
            ....
        }
        ....
    }
    
}
```
这里根据不同的**StatusBarIconHolder**类型，设置不同的网络**Icon**，上面列出了移动信号**Icon**相关的方法。
**SystemUI**状态栏图标根据源码可大体分为三种：
1. **StatusBarIconView**
2. **StatusBarWifiView**
3. **StatusBarMobileView**
 

这里主要以移动网络相关图标（**StatusBarMobileView**）进行分析，添加**Icon**时首先会创建一个
**StatusBarMobileView**，然后调用**StatusBarMobileView**的**applyMobileState**更新其显示状态，最后将其加入到**CollapsedStatusBarFragment**中放置**Icon**的**ViewGroup**中，这样就完成了添加过程；

而更新过程则是直接调用**StatusBarMobileView**的**applyMobileState**更新其显示状态，更新部分代码如下：
**/SystemUI/src/com/android/systemui/statusbar/StatusBarMobileView.java**
```java
public class StatusBarMobileView extends FrameLayout implements DarkReceiver,
        StatusIconDisplayable {
    ....
    public void applyMobileState(MobileIconState state) {
        if (state == null) {
            setVisibility(View.GONE);
            mState = null;
            return;
        }

        if (mState == null) {
            // 添加View时
            mState = state.copy();
            // 初始化移动网络图标状态
            initViewState();
            return;
        }
        if (!mState.equals(state)) {
            // 状态变化了,更新移动网络图标显示状态
            updateState(state.copy());
        }
    }
    
    private void updateState(MobileIconState state) {
        setContentDescription(state.contentDescription);
        if (mState.visible != state.visible) {
            // 可见状态变化
            mMobileGroup.setVisibility(state.visible ? View.VISIBLE : View.GONE);.
            requestLayout();
        }
        if (mState.strengthId != state.strengthId) {
            // 信号强度变化
            mNewMobile.setImageResource(state.strengthId);
        }

        if (mState.typeId != state.typeId) {
            if (state.typeId != 0) {
                mMobileType.setContentDescription(state.typeContentDescription);
                mMobileType.setImageResource(state.typeId);
                // 网络类型图标G/3G/4G的显示
                mMobileType.setVisibility(View.VISIBLE /*View.VISIBLE*/);
            } else {
                mMobileType.setVisibility(View.GONE);
            }
        }
        
        // 移动漫游Icon
        mMobileRoaming.setVisibility(state.roaming ? View.VISIBLE : View.GONE);
        // 数据输入Icon
        mIn.setVisibility(state.activityIn ? View.VISIBLE : View.GONE);
        // 数据输出Icon
        mOut.setVisibility(state.activityIn ? View.VISIBLE : View.GONE);
        mInoutContainer.setVisibility((state.activityIn || state.activityOut)
                ? View.VISIBLE : View.GONE);

        mState = state;
    }
            
}
```
至此就完成了移动网络图标的添加和更新。

## 流程图
![image](/img/in_post/systemui_statusbar_mobile_signal_icon_update_process.png)

