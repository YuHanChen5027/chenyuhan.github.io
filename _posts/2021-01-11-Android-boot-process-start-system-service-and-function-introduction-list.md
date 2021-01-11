---
layout: post
title:  "Android开机流程启动的系统服务以及功能介绍列表"
author: "陈宇瀚"
date:   2021-01-11 17:59:00 +0800
categories: article
tags:
  - Android
  - Service
  - 系统服务
  - 开机流程
---
这里按照启动**System**进程时系统服务在三种启动函数的位置分类分为三部分
即**startBootstrapServices(),startCoreServices(),startOtherServices()**
- startBootstrapServices():启动引导服务
- startCoreServices():启动核心服务
- startOtherServices():启动其他服务

在启动之前还会启动一个系统服务管理类**SystemServiceManager**：主要用于开机时的各种系统服务的初始化和配置工作。

*列表服务顺序按照源码中的启动调用顺序(基于rk3399_industry Android 7.1.2代码)*
## startBootstrapServices()

服务名 | 功能简介
---|---
Installer | 创建具有合适权限的关键目录，如/data/user
ActivityManagerService | 简称AMS，管理着Android四大组建的生命周期，同时也管理着各个应用程序进程
PowerManagerService | 电源管理服务
LightsService | 管理设备各种led灯和显示背光等
DisplayManagerService | 管理设备显示的生命周期，会根据当前连接的物理显示设备控制其逻辑显示，当状态变化时，会向系统和应用程序发送通知
PackageManagerService | 简称PMS，用于APK的权限验证、安装、删除等操作
UserManagerService | 主要功能是创建和删除用户，以及查询用户信息

这部分是Android引导服务的启动，内部包含了Android Framework层如**AMS，PMS**这种非常重要，且经常用到的服务，除了这些服务，在该函数内部还会进行一些传感器服务的启动，但这部分并非直接调用java类，而是调用了jni方法，这里就不再细分。

## startCoreServices()

服务名 | 功能简介
---|---
BatteryService | 对设备电池状态进行监控的服务，当电池，充电状态，温度等信息发生变化时，会以广播的形式通知其他相关的进程和服务。
UsageStatsService | 收集用户对每一个APP的使用频率、使用时常等信息；
WebViewUpdateService | 用于WebView的更新

这部分是Android的核心服务，包含了电池管理服务，应用使用管理服务以及WebView更新服务。

## startOtherService()

服务名 | 功能简介
---|---
CameraService | 管理设备相机功能
VibratorService | 管理设备震动功能
ConsumerIrService | 远程控制服务，如红外遥控
AlarmManagerService | Android定时服务，提供闹铃和定时器等功能
InputManagerService | 事件传递分发服务
WindowManagerService | 对系统中的所有窗口进行管理窗口的管理服务
VrManagerService | VR相关功能服务
BluetoothService | 蓝牙服务
InputMethodManagerService | 输入法服务
AccessibilityManagerService | 辅助管理程序截获所有的用户输入，并根据这些输入给用户一些额外的反馈，起到辅助的效果，View的点击、焦点等事件分发管理服。
MountService | 用于媒体/ usb通知 / 外接设备的插入拔出等事件的监听
UiModeManagerService | 管理设备UI模式，汽车模式、电视模式、手表模式、黑夜模式等
LockSettingsService | 管理用户的锁屏功能，锁屏密码等
PersistentDataBlockService | 管理数据块的服务？之后有空详细看下
DeviceIdleController | 设备idle(空闲状态)的控制，例如低功耗模式就基于这个服务
DevicePolicyManagerService | 提供一些系统级别的设置及属性
StatusBarManagerService | 系统状态栏服务
ClipboardService | 剪贴板服务
NetworkManagementService | 网络管理服务，提供对物理网络接口的管理服务。Android 系统网络连接和管理服务由四个系统服务ConnectivityService、NetworkPolicyManagerService、NetworkManagementService、NetworkStatsService共同配合完成网络连接和管理功能。
TextServicesManagerService | 文本服务，例如文本拼写检查服务
NetworkScoreService | 网络状态分服务？待理解
NetworkStatsService | 网络传输数据统计服务
NetworkPolicyManagerService | 网络策略管理服务
WifiP2pService | Wifi Direct服务
WifiService | Wifi服务
WifiScanningService | Wifi扫描服务
RttService | wifi RTT服务(Wi-Fi Round-Trip-Time)，让应用可以利用室内定位功能
EthernetService | 以太网服务
ConnectivityService | 网络连接状态服务
NsdService | 网络服务搜索
UpdateLockService | 锁屏更新服务
RecoverySystemService | recovery系统模式，在一些情况下可能会使用此服务重启系统
NotificationManagerService | 通知服务
DeviceStorageMonitorService | 设备存储器监听服务
LocationManagerService | 位置服务，用于GPS、定位等。
CountryDetectorService | 国家检测服务，类ComprehensiveCountryDetector.java，检测顺序：移动网络、位置、sim卡的国家、手机的位置
SearchManagerService | 全局搜索服务
DropBoxManagerService | 用于系统运行日志和错误日志的存储管理服务
WallpaperManagerService | 壁纸管理服务
AudioService | AudioFlinger的上层管理封装，主要是音量、音效、声道及铃声等的管理。
DockObserver | 看网上的解释，如果系统有个座子，当手机装上或拔出这个座子的话，可以靠该服务进行管理
UsbService | usb设备服务，管理所有USB设备相关的状态，包括host（UsbHostManager）和device（UsbDeviceManager）模式
SerialService | 串口服务
HardwarePropertiesManagerService | 提供访问设备硬件状态的机制：CPU，GPU和电池温度，每个内核的CPU使用率，风扇速度等
TwilightService | 指出用户当前所在位置是否为晚上，被UiModeManager等用来调整夜间模式。
NightDisplayService | 设备夜晚显示模式服务
JobSchedulerService | 这个没太明白是干啥用的
SoundTriggerService | 语音识别架构？
BackupManagerService | 备份服务
AppWidgetService | widge管理服务，Widget安装，删除，更新等等
VoiceInteractionManagerService | 手势启动服务
SensorNotificationService | 传感器通知服务
ContextHubSystemService | 主要是为了启动ContextHubService
DiskStatsService | 磁盘统计服务，使用adb shell dumpsys diskstats时会被调用
SamplingProfilerService | 用于耗时统计等
NetworkTimeUpdateService | 监视网络时间，当网络时间变化时更新本地时间
CommonTimeManagementService | 管理本地常见的时间服务的配置，在网络配置变化时重新配置本地服务
EmergencyAffordanceService | 紧急呼叫服务
DreamManagerService | 管理屏保
AssetAtlasService | 负责将预加载的bitmap组装成纹理贴图，生成的纹理贴图可以被用来跨进程使用，以减少内存。
GraphicsStatsService | 用来汇总屏幕卡顿数据的，通过adb shell dumpsys graphicsstats调用查看汇总结果
PrintManagerService | 打印服务
RestrictionsManagerService | 限制管理服务
MediaSessionService | 多媒体会话服务
HdmiControlService | hdmi控制服务
TvInputManagerService | 电视输入服务
MediaResourceMonitorService | 媒体资源监控服务
TrustManagerService | 信任管理器服务?
FingerprintService | 指纹服务
BackgroundDexOptService | dex后台优化服务，在PackManager中有调用
LauncherAppsService | Launcher应用相关的服务
ShortcutService | 快捷方式服务
MediaProjectionManagerService | 媒体投影服务，可用于录屏或者屏幕投射
WearBluetoothServicea | 穿戴设备蓝牙服务
WearWifiMediatorService | 穿戴设备Wi-Fi中介服务
WearCellularMediatorService | 穿戴设备蜂窝网络服务？
WearTimeService | 穿戴设备时间服务
MmsServiceBroker | 彩信服务
RetailDemoModeService | 这个不太理解

## 参考
[Android 系统服务一览表](https://www.jianshu.com/p/9c401741d51d?utm_campaign=maleskine&utm_content=note&utm_medium=seo_notes&utm_source=recommendation)
[Android8.1系统服务整理](http://bcoder.com/java/list-of-android-8-1-system-services)

