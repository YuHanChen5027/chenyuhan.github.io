---
layout: post
title:  "Android 设置标志位限制应用安装"
date:   2018-06-06 12:01:20 +0800
author: "陈宇瀚"
categories: article
tags:
  - Android 
  - Framework
---
只有标志位为1的时候的apk才可以安装，否则apk都不能安装
需要修改的文件有以下几个
/frameworks/base/core/java/android/provider/Settings.java
/frameworks/base/packages/SettingsProvider/src/com/android/providers/settings/DatabaseHelper.java
/frameworks/base/packages/SettingsProvider/res/values/defaults.xml
/frameworks/base/services/core/java/com/android/server/pm/PackageManagerService.java
/frameworks/base/core/java/android/content/pm/PackageManager.java

## 1 Settings.java
数据分类
SettingsProvider对数据进行了分类，分别是Global、System、Secure三种类型，它们的区别如下：

Global：所有的偏好设置对系统的所有用户公开，第三方APP有读没有写的权限；
System：包含各种各样的用户偏好系统设置；
Secure：安全性的用户偏好系统设置，第三方APP有读没有写的权限。
参考文章：https://blog.csdn.net/myfriend0/article/details/59107989
我们这次添加在Global中，要添加另外两个类中方法也是类似的
首先我们在Settings的内部类Global中加入我们用来限制应用安装的标志位
```  
/**
* cyh add 
* app install lock
* @hide
*/
public static final String APP_INSTALL_LOCK = "app_install_lock";
```
之后在SETTINGS_TO_BACKUP字符串数组中加入这个标志位
```
public static final String[] SETTINGS_TO_BACKUP = {
...
...
APP_INSTALL_LOCK
};

```
## 2 DatabaseHelper.java
设定标志位初始值
在
/frameworks/base/packages/SettingsProvider/res/values/defaults.xml
资源文件内加入默认值
```
...
<!-- cyh add app install lock -->
<bool name="def_app_install_lock">true</bool>
...
```
在DatabaseHelper的loadGlobalSettings()方法内（System类型就在loadSystemSettings()方法内）加入初始值设置
```
...
//cyh add start
//设定默认值
loadBooleanSetting(stmt, Settings.Global.APP_INSTALL_LOCK,
R.bool.def_app_install_lock);
//cyh add end 
...
```
## 3 PackageManagerService.java
在scanPackageLI方法下加入如下代码
一定要将NullPointerException抛出，在标志位数据库未生成时会运行到这个方法内，若不抛出异常，则会卡在android开机画面
```
boolean success = false;
//cyh add start
//获取应用安装锁
if(mContext!=null&&mContext.getContentResolver()!=null){
try {
int  appInstallLock = android.provider.Settings.Global.getInt(mContext.getContentResolver(),android.provider.Settings.Global.APP_INSTALL_LOCK , 1);
if (appInstallLock==0){
int mLastScanError = PackageManager.INSTALL_FAILED_INVALID_LOCK;
throw new PackageManagerException(mLastScanError,
"禁止安装，安装模式未开启动");
}
}catch(NullPointerException e){
//一定要将NullPointerException抛出，在标志位数据库未生成时会运行到这个方法内，若不抛出异常，则会卡在android开机画面
e.printStackTrace();
}
}
//cyh add end 
```
## 4 PackageManager.java 
加入错误标志，其实影响不大，看你要不要在自己的应用内做判断
```
/**
* cyh add
* app install failed lock noopen
* cyh end 
* @hide
*/
public static final int INSTALL_FAILED_INVALID_LOCK = -27;
```

使用方式：
在自己的应用内加入权限
```
<!-- 用于读取，修改标志位-->
<uses-permission android:name="android.permission.READ_EXTERNAL_STORAGE" />
<uses-permission android:name="android.permission.WRITE_SETTINGS" />
<uses-permission android:name="android.permission.WRITE_SECURE_SETTINGS" />
```
在方法中
```
private final String APP_INSTALL_LOCK = "app_install_lock";
...
//方法内
//读取标志位的值
int keyValue;
try {
int keyValue = Settings.Global.getInt(getContentResolver(),APP_INSTALL_LOCK);
} catch (Settings.SettingNotFoundException e) {
e.printStackTrace();
}
//修改标志位的值
Settings.Global.putInt(getContentResolver(),APP_INSTALL_LOCK,value==0? 
1:0);
...
```
