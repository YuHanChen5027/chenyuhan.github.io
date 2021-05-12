---
layout: post
title:  "Android SystemUI-StatusBar启动过程简单分析"
author: "陈宇瀚"
date:   2021-04-27 15:34:00 +0800
header-img: "img/img-head/img-head-boot-process.jpg"
categories: article
tags:
  - Android
  - SystemUI
  - StatusBar
---
# SystemUI-StatusBar启动过程简单分析
## SystemUI的启动
SystemUI本质上是一个app，在系统中对应的源码路径为：
**frameworks/base/packages/SystemUI**

在源码中编译后会生成
**out/target/product/product_name/system/priv-app/SystemUI/SystemUI.apk**。

在**system/priv-app**下意味着**SystemUI**开机就会被安装，SystemUI并不是自己启动的，它是在**SystemServer进程**创建过程中被开启的，**SystemServer进程**启动时会启动各种系统服务（如AMS、PMS等）SystemUI的启动就在这一流程中，下面以源码中流程代码来分析这一启动过程：

*以下源码基于MT2712 MR2004 Android9.0版本源码，流程代码中省略了一些内容*

**platform/frameworks/base/services/java/com/android/server/SystemServer.java**
```java
public final class SystemServer {
    ....
    // SystemServer进程启动入口
    public static void main(String[] args) {
        new SystemServer().run();
    }
    private void run() {
        ....
        // 启动各种系统服务
        try {
            startBootstrapServices();
            startCoreServices();
            // SystemUI的启动在其他服务的启动过程中
            startOtherServices();
            SystemServerInitThreadPool.shutdown();
        } catch (Throwable ex) {
           ....
        } finally {
            traceEnd();
        }
        ....
        // Loop forever.
        Looper.loop();
    }
    ....
    private void startOtherServices() {
        ....
        final WindowManagerService windowManagerF1 = wm;
        traceBeginAndSlog("StartSystemUI");
        try {
            // 启动SystemUI入口
            startSystemUi(context, windowManagerF1);
        } catch (Throwable e) {
            reportWtf("starting System UI", e);
        }
        ....
    }
    
    static final void startSystemUi(Context context, WindowManagerService windowManager) {
        Intent intent = new Intent();
        intent.setComponent(new ComponentName("com.android.systemui",
                    "com.android.systemui.SystemUIService"));
        intent.addFlags(Intent.FLAG_DEBUG_TRIAGED_MISSING);
        context.startServiceAsUser(intent, UserHandle.SYSTEM);
        windowManager.onSystemUiStarted();
    }
}
```
通过最后调用到的**startSystemUi**可以看出，最后通过创建**Intent**，启动**SystemUIService**。

## SystemUI的加载
这里进入**frameworks/basepackages/SystemUI/src/com/android/systemui/SystemUIService.java**源码中查看启动后执行了什么
```java
public class SystemUIService extends Service {
    @Override
    public void onCreate() {
        super.onCreate();
        ((SystemUIApplication) getApplication()).startServicesIfNeeded();
        ....
    }
}
```
在**onCreate**中获取了**SystemUIApplication**并调用了其**startServicesIfNeeded**方法

**platform/frameworks/base/packages/SystemUI/src/com/android/systemui/SystemUIApplication.java**
```java
public class SystemUIApplication extends Application implements SysUiServiceProvider {
    ....
    public void startServicesIfNeeded() {
        String[] names = getResources().getStringArray(R.array.config_systemUIServiceComponents);
        startServicesIfNeeded(names);
    }
    
    private void startServicesIfNeeded(String[] services) {
        if (mServicesStarted) {
            return;
        }
        mServices = new SystemUI[services.length];
        ....
        final int N = services.length;
        for (int i = 0; i < N; i++) {
            String clsName = services[i];
            ....
            Class cls;
            try {
                // 通过反射调组件类启动
                cls = Class.forName(clsName);
                mServices[i] = (SystemUI) cls.newInstance();
            } catch(ClassNotFoundException ex){
                throw new RuntimeException(ex);
            } catch (IllegalAccessException ex) {
                throw new RuntimeException(ex);
            } catch (InstantiationException ex) {
                throw new RuntimeException(ex);
            }

            mServices[i].mContext = this;
            mServices[i].mComponents = mComponents;
            mServices[i].start();
            ....
            if (mBootCompleted) {
                mServices[i].onBootCompleted();
            }
        }
        ....
        mServicesStarted = true;
    }


}
```
在方法内创建了一个字符数组**names**，读取**res/values/config.xml**文件中定义的数组**config_systemUIServiceComponents**里面包含了各种**SystemUI**的各个组件和服务类的完整路径，之后通过循环和反射调用各个组件的**start**方法，并将它们放入一个**SystemUI**数组管理；
```xml
    <string-array name="config_systemUIServiceComponents" translatable="false">
        <item>com.android.systemui.Dependency</item>
        <item>com.android.systemui.Dependency</item>
        <item>com.android.systemui.util.NotificationChannels</item>
        <item>com.android.systemui.statusbar.CommandQueue$CommandQueueStart</item>
        <item>com.android.systemui.keyguard.KeyguardViewMediator</item>
        <item>com.android.systemui.SystemBars</item>
        <item>com.android.systemui.volume.VolumeUI</item>
        <item>com.android.systemui.usb.StorageNotification</item>
        <item>com.android.systemui.power.PowerUI</item>
        <item>com.android.systemui.media.RingtonePlayer</item>
        <item>com.android.systemui.keyboard.KeyboardUI</item>
        <item>com.android.systemui.pip.PipUI</item>
        <item>com.android.systemui.shortcut.ShortcutKeyDispatcher</item>
        <item>@string/config_systemUIVendorServiceComponent</item>
        <item>com.android.systemui.util.leak.GarbageMonitor$Service</item>
        <item>com.android.systemui.LatencyTester</item>
        <item>com.android.systemui.globalactions.GlobalActionsComponent</item>
        <item>com.android.systemui.ScreenDecorations</item>
        <item>com.android.systemui.fingerprint.FingerprintDialogImpl</item>
        <item>com.android.systemui.SliceBroadcastRelayHandler</item>
    </string-array>
```
这里分析以**com.android.systemui.SystemBars**为主。

## StatusBars的加载
**platform/frameworks/base/packages/SystemUI/src/com/android/systemui/SystemBars.java**
```java
public class SystemBars extends SystemUI {
    ....
    @Override
    public void start() {
        createStatusBarFromConfig();
    }
    .... 
    private void createStatusBarFromConfig() {
        // 读取res/values/config.xml获取config_statusBarComponent属性
        // 这里读到的时com.android.systemui.statusbar.phone.StatusBar
        final String clsName = mContext.getString(R.string.config_statusBarComponent);
        ....
        Class<?> cls = null;
        try {
            cls = mContext.getClassLoader().loadClass(clsName);
        } catch (Throwable t) {
            throw andLog("Error loading status bar component: " + clsName, t);
        }
        try {
            mStatusBar = (SystemUI) cls.newInstance();
        } catch (Throwable t) {
            throw andLog("Error creating status bar component: " + clsName, t);
        }
        mStatusBar.mContext = mContext;
        mStatusBar.mComponents = mComponents;
        mStatusBar.start();
    }

}
```
这里与上衣过程类似，也是从**config.xml**文件中读取一个完整路径，然后通过反射调用其**start**方法，这里调用到**StatusBar.java**类的**start**方法。
**platform/frameworks/base/packages/SystemUI/src/com/android/systemui/statusbar/phone/StatusBar.java**
```java

public class StatusBar extends SystemUI implements DemoMode,
        DragDownHelper.DragDownCallback, ActivityStarter, OnUnlockMethodChangedListener,
        OnHeadsUpChangedListener, CommandQueue.Callbacks, ZenModeController.Callback,
        ColorExtractor.OnColorsChangedListener, ConfigurationListener, NotificationPresenter {
        ....
        
    protected StatusBarWindowView mStatusBarWindow;
    protected StatusBarWindowManager mStatusBarWindowManager;
    
    @Override
    public void start() {
        ....

        mWindowManager = (WindowManager) mContext.getSystemService(Context.WINDOW_SERVICE);

        mDisplay = mWindowManager.getDefaultDisplay();
        updateDisplaySize();
        ....
        // 1.这里开始加载StatusBar布局并添加到界面上
        createAndAddWindows();
        ....
    }
    
    public void createAndAddWindows() {
        // 2
        addStatusBarWindow();
    }

    private void addStatusBarWindow() {
        // 3.构造SatusBarView
        makeStatusBarView();
        ....
        // 6.将mStatusBarWindow加载到系统界面上
        // getStatusBarHeight()对应的高度值在下方的updateResources()内进行加载
        mStatusBarWindowManager.add(mStatusBarWindow, getStatusBarHeight());
    }

    protected void makeStatusBarView() {
        ....
        updateResources();//包含一些状态栏高度等资源的加载
        ....
        // 4.通过inflate将布局加载给mStatusBarWindow
        inflateStatusBarWindow();
        // 5.将CollapsedStatusBarFragment加载进mStatusBarWindow，对应的就是显示在顶部的状态栏
        FragmentHostManager.get(mStatusBarWindow)
                .addTagListener(CollapsedStatusBarFragment.TAG, (tag, fragment) -> {
                    CollapsedStatusBarFragment statusBarFragment =
                            (CollapsedStatusBarFragment) fragment;
                    statusBarFragment.initNotificationIconArea(mNotificationIconAreaController);
                    mStatusBarView = (PhoneStatusBarView) fragment.getView();
                    // mStatusBarView的一些设置
                    ....

                    setAreThereNotifications();
                    checkBarModes();
                }).getFragmentManager()
                .beginTransaction()
                .replace(R.id.status_bar_container, new CollapsedStatusBarFragment(),
                        CollapsedStatusBarFragment.TAG)
                .commit();
        ....

    }
    
    protected void inflateStatusBarWindow(Context context) {
        mStatusBarWindow = (StatusBarWindowView) View.inflate(context,
                R.layout.super_status_bar, null);
    }
            
}
```
针对**StatusBar**，当我们调用**StatusBar**的**start**方法后，我们会加载状态栏的一些资源（如高度），之后通过**inflate**将**super_status_bar.xml**对应的布局加载给**mStatusBarWindow**对象；

在**super_status_bar.xml**布局文件中有一个**status_bar_container**的**FrameLayout**，之后会被用来放置**CollapsedStatusBarFragment**,这里存放了众多的状态栏**View**。这一步完成后就会将将**mStatusBarWindow**加载到系统界面上

## StatusBar加载到系统界面
**StatusBar**通过**StatusBarWindowManager.add**加载到系统界面上，具体的源码如下：
**platform/frameworks/base/packages/SystemUI/src/com/android/systemui/statusbar/phone/StatusBarWindowManager.java**
```java
 /**
     * 将状态栏视图添加到WindowMangager。
     *
     * @param statusBarView 
     * @param barHeight 折叠状态下状态栏高度
     */
    public void add(View statusBarView, int barHeight) {
        
        mWindowManager.getDefaultDisplay().getSize(mPoint);
        int navigationbarWidth = getNavigationBarWidth();
        mLp = new WindowManager.LayoutParams(
                mPoint.x - navigationbarWidth,
                barHeight,
                navigationbarWidth,
                0,
                WindowManager.LayoutParams.TYPE_STATUS_BAR,
                WindowManager.LayoutParams.FLAG_NOT_FOCUSABLE
                        | WindowManager.LayoutParams.FLAG_TOUCHABLE_WHEN_WAKING
                        | WindowManager.LayoutParams.FLAG_SPLIT_TOUCH
                        | WindowManager.LayoutParams.FLAG_WATCH_OUTSIDE_TOUCH
                        | WindowManager.LayoutParams.FLAG_DRAWS_SYSTEM_BAR_BACKGROUNDS,
                PixelFormat.TRANSLUCENT);
        mLp.token = new Binder();
        mLp.gravity = Gravity.TOP;
        mLp.softInputMode = WindowManager.LayoutParams.SOFT_INPUT_ADJUST_RESIZE;
        mLp.setTitle("StatusBar");
        mLp.packageName = mContext.getPackageName();
        mLp.layoutInDisplayCutoutMode = LAYOUT_IN_DISPLAY_CUTOUT_MODE_ALWAYS;
        mStatusBarView = statusBarView;
        mBarHeight = barHeight;
        mWindowManager.addView(mStatusBarView, mLp);
        mLpChanged = new WindowManager.LayoutParams();
        mLpChanged.copyFrom(mLp);
    }
```
这里传入的**mStatusBarWindow**就通过**WindowManager**加载到相应的位置上了。

## CollapsedStatusBarFragment扩展学习记录
这里展示**CollapsedStatusBarFragment**的两个生命周期。这里包含了布局和显示内容的加载
**frameworks/base、packages/SystemUI/src/com/android/systemui/statusbar/phone/CollapsedStatusBarFragment.java**
```java
public class CollapsedStatusBarFragment extends Fragment implements CommandQueue.Callbacks {
    ....
    private PhoneStatusBarView mStatusBar;
    private KeyguardMonitor mKeyguardMonitor;
    private NetworkController mNetworkController;
    private StatusBar mStatusBarComponent;
    ....

    @Override
    public View onCreateView(LayoutInflater inflater, @Nullable ViewGroup container,
            Bundle savedInstanceState) {
        // 加载布局status_bar.xml
        return inflater.inflate(R.layout.status_bar, container, false);
    }

    @Override
    public void onViewCreated(View view, @Nullable Bundle savedInstanceState) {
        super.onViewCreated(view, savedInstanceState);
        mStatusBar = (PhoneStatusBarView) view;
        if (savedInstanceState != null && savedInstanceState.containsKey(EXTRA_PANEL_STATE)) {
            mStatusBar.go(savedInstanceState.getInt(EXTRA_PANEL_STATE));
        }
        mDarkIconManager = new DarkIconManager(view.findViewById(R.id.statusIcons));
        mDarkIconManager.setShouldLog(true);
        Dependency.get(StatusBarIconController.class).addIconGroup(mDarkIconManager);
        // 用来显示状态栏图标的区域（(蓝牙、wifi、VPN、网卡、SIM卡信号、飞行模式等) 电池）
        mSystemIconArea = mStatusBar.findViewById(R.id.system_icon_area);
        mClockView = mStatusBar.findViewById(R.id.clock);
        showSystemIconArea(false);
        showClock(false);
        initEmergencyCryptkeeperText();
        initOperatorName();
    }
    
}
```
**CollapsedStatusBarFragment**内还有一些**Notification**相关的内容，之后有时间再继续分析。

