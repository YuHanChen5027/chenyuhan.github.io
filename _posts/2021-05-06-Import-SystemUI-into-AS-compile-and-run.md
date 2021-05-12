---
layout: post
title:  "将SystemUI导入AS编译运行"
author: "陈宇瀚"
date:   2021-05-06 19:31:00 +0800
header-img: "img/img-head/img-head-boot-process"
categories: article
tags:
  - Android
  - SystemUI
  - 源码编译
---
# 将SystemUI导入AS编译运行

## 前期准备
**以Android9.0源码（我这里用的是MT2712，win10环境，AS 4.1.3）为例**

首先需要将源码进行一次编译，因为导出的SystemUI之后会引入一些编译后生成的库。

**SystemUI**代码路径为**mt2712/frameworks/base/packages/SystemUI**

将**SystemUI**代码拉到本地，删除代码中test和example相关目录
即删除shared/tests/、plugin/ExamplePlugin/和tests/三个目录。

同样的，将**mt2712/frameworks/base/packages/SettingsLib**、**package/service/Car/car_lib**和**framework/support/car**下拉到本地，删除其中的**test**目录。

## SystemUI项目搭建
### 创建SystemUI项目
使用AS创建一个包名为com.android.systemui的No Activity项目
![image](/img/in_post/systemui_to_as_create_module.png)
将**SystemUI**目录中**的AndroidManifest.xml**、**Android.mk**文件、**res**、**res-keyguard**和**src**目录下的所有文件移到刚创建的app目录的的src/main/java目录下，移动完目录结构如下：
* app
  * src
    * main
      * java
        * com.android.keyguard
        * com.android.systemui
      * res
      * res-keyguard
      * Android.mk
      * AndroidManifest.xml

在build.gradle文件内添加
```
    sourceSets {
        main {
            res.srcDirs += "src/main/res-keyguard"
        }
    }
```
引用res-keyguard目录下的资源
### 创建Module
根据SystemUI源代码目录下的**AndroidManifest.xml**文件数量可以看出部分需要创建的Module(library形式的)，我这里需要创建两个：

![image](/img/in_post/systemui_to_as_androidmanifest.png)

1. shared:com.android.systemui.shared
2. plugins:com.android.systemui.plugins

此外还有两个需要创建
3. settingsLib:com.android.settingslib
4. car_support:androidx.car

![image](/img/in_post/systemui_to_as_folder.png)

 1. 删除**Module**中**src**目录下的自动生成的目录（如**test**）
 2. 将**SystemUI/shard**、**SystemUI/plugin**、**SettingsLib**和**car**目录中**的AndroidManifest.xml**和**Android.mk**文件(没有则不用)和**src**目录下的所有文件移到对应**Module**的src/main/java目录下。
 
移动完毕后得到的目录结构如下(只列出了转移的文件):
* plugin
  * src
    * main
      * java
        * com.android.systemui.plugins
      * Android.mk
      * AndroidManifest.xml
* shared
  * src
    * main
      * java
        * com.android.systemui.shared
      * Android.mk
      * AndroidManifest.xml
* SettingsLib
  * src
    * main
      * java
        * com.android.systemui.shared
      * res
      * Android.mk
      * AndroidManifest.xml
* car_support
  * src
    * main
      * java
        * androidx.car
      * res
      * AndroidManifest.xml

## SystemUI库的引入

### Android.mk中的库
下面是plugin和shared的Android.mk文件：
```mk
########### plugin Android.mk ###########
LOCAL_PATH := $(call my-dir)
include $(CLEAR_VARS)
....
LOCAL_MODULE := SystemUIPluginLib
LOCAL_SRC_FILES := $(call all-java-files-under, src)
LOCAL_RESOURCE_DIR := $(LOCAL_PATH)/res
LOCAL_JAR_EXCLUDE_FILES := none

include $(BUILD_STATIC_JAVA_LIBRARY)
....
########### shared Android.mk ###########
LOCAL_PATH := $(call my-dir)
include $(CLEAR_VARS)
....
LOCAL_MODULE := SystemUISharedLib

LOCAL_SRC_FILES := $(call all-java-files-under, src) $(call all-Iaidl-files-under, src)

LOCAL_RESOURCE_DIR := $(LOCAL_PATH)/res
LOCAL_JAR_EXCLUDE_FILES := none

include $(BUILD_STATIC_JAVA_LIBRARY)
....
```
可以看出**plugin**和**shared**分别产生**SystemUIPluginLib**和**SystemUISharedLib**库。这两个库会被**SystemUI**所使用到。因为我们已经将这两部分以Module形式提出，所以直接在**SystemUI****的build.gradle**中引入即可。
```
    implementation project(':plugin')
    implementation project(':shared')
```

根据**Android.mk**文件中**LOCAL_STATIC_ANDROID_LIBRARIES**、**LOCAL_STATIC_JAVA_LIBRARIES**和**LOCAL_JAVA_LIBRARIES**属性，可以了解到**SystemUI**引用到了哪些库，然后需要将这些库都找到放入**app/libs**目录下，然后在**build.gradle**文件中引入。

其中v4、v7、v14、v17的库可以先不引入，之后会将**SystemUI**中使用到这些库转成**AndroidX**的库，不引入可以减少一些麻烦。

**SystemUI**根目录**Android.mk**文件资源相关内容和对应的**jar**包如下：
```mk
# LOCAL_STATIC_ANDROID_LIBRARIES 里面引用到的库,大部分都可以在
# platform\out\soong\.intermediates\prebuilts\sdk\current目录下找到
# 比如android-support-car，对应的就是platform\out\soong\.intermediates\prebuilts\sdk\current\support\android-support-car\android_common\turbine-combined\android-support-car.jar
LOCAL_STATIC_ANDROID_LIBRARIES := \
    SystemUIPluginLib \  # 对应plugin Module
    SystemUISharedLib \  # 对应shared Module
    android-support-car \  # android-support-car.jar
    android-support-v4 \  # android-support-v4.jar
    android-support-v7-recyclerview \  #  android-support-v7-recyclerview.jar
    android-support-v7-preference \  # 之后以此类推
    android-support-v7-appcompat \
    android-support-v7-mediarouter \
    android-support-v7-palette \
    android-support-v14-preference \
    android-support-v17-leanback \
    android-slices-core \
    android-slices-view \
    android-slices-builders \
    android-arch-core-runtime \
    android-arch-lifecycle-extensions \

# LOCAL_JAVA_LIBRARIES  里面的jar，是只在编译的时候引用即可，不需要打包进apk
# 这些jar是系统本身的jar，所以我们build.gradle以compileOnly方式引用
LOCAL_JAVA_LIBRARIES := telephony-common \
    android.car
```
这些库中有一些是必须引入的，下方列出了库和其在源码中对应的jar包路径
### 导入framework.jar
**framework.jar**
文件路径为：out\target\common\obj\JAVA_LIBRARIES\framework_intermediates\classes.jar

将其放入app/libs文件目录下，然后在根目录**build.gradle**在**allproject**下添加
```
tasks.withType(JavaCompile) {
    options.compilerArgs.add('-Xbootclasspath/p:app\\libs\\framework.jar')
}
```
在**app**、**shared**、**plugin**和**SettingsLib**,**car_support**目录下的**build.gradle**下添加
```
preBuild {
    doLast {
        def imlFile = file(project.name + ".iml")
        println('Change ' + project.name + '.iml order')
        try {
            def parsedXml = (new XmlParser()).parse(imlFile)
            def jdkNode = parsedXml.component[1].orderEntry.find { it.'@type' == 'jdk' }
            parsedXml.component[1].remove(jdkNode)
            def sdkString = "Android API " + android.compileSdkVersion.substring("android-".length()) + " Platform"
            new groovy.util.Node(parsedXml.component[1], 'orderEntry', ['type': 'jdk', 'jdkName': sdkString, 'jdkType': 'Android SDK'])
            groovy.xml.XmlUtil.serialize(parsedXml, new FileOutputStream(imlFile))
        } catch (FileNotFoundException e) {
            // nop, iml not found
        }
    }
}
```
这样就能优先使用我们导入的**framework.jar**

### 导入telephony-common.jar
**telphont-common.jar**
文件路径为：out/target/common/obj/JAVA_LIBRARIES/telephony-common_intermediates/classes-header.jar

### 导入core-libart.jar
**core-libart.jar**
文件路径为：out\target\common\obj\JAVA_LIBRARIES\core-libart_intermediates\classes.jar

### 导入core-oj.jar
**core-oj.jar**
文件路径为：out\target\common\obj\JAVA_LIBRARIES\core-oj_intermediates\classes.jar

### 导入car-lib.jar
**car-lib.jar**
文件路径为：out\target\common\obj\JAVA_LIBRARIES\android.car_intermediates\classes.jar

### 导入android-slices-view.jar
**android-slices-view.jar**
文件路径为：out\target\common\obj\JAVA_LIBRARIES\android-slices-view_intermediates\classes.jar

### 导入android-slices-builders.jar
**android-slices-builders.jar**
文件路径为：out\target\common\obj\JAVA_LIBRARIES\android-slices-builders_intermediates\classes.jar

### 导入android-slices-view.jar
**android-slices-view.jar**
文件路径为：out\target\common\obj\JAVA_LIBRARIES\android-slices-view_intermediates\classes.jar

### 导入systemui-tags.jar
**systemui-tags.jar**
文件路径为：out\target\common\obj\JAVA_LIBRARIES\SystemUI-tags_intermediates\classes.jar

### 导入systemui-proto.jar
**systemui-proto.jar**
文件路径为：out\target\common\obj\JAVA_LIBRARIES\SystemUI-proto_intermediates\classes.jar

最后贴上我个人引入
## 常见错误
### Found item Attr/longIntent more than one time
删除xml中重复的定义,非常多，需要花些时间。

### AAPT: warn: multiple substitutions specified in non-positional format
将%s 改为 %1$s

### AAPT: error: resource previously defined here.
文件中定义的属性与系统的属性值重名了，删掉即可。

### AAPT: error: resource (color/dimen)/xxxxx (aka com.android.systemui:(color/dimen)/xxxx) not found.
在源码中找到对应的color值，写入res/values/colors.xml或者dimens.xml中中。

### Error:Could not resolve all dependencies for configuration ':app:debugRuntimeClasspath'
![image](/img/in_post/systemui_to_as_error.png)
gradle版本改成其他版本，我这里Android9.0，从4.1.3改为3.6.0就可以编译通过了。
```
classpath "com.android.tools.build:gradle:4.1.3"
->
classpath "com.android.tools.build:gradle:3.6.0"
```

### 程序包android.support.annotation、android.arch.lifecycle不存在等问题
可以尝试将这些包转成androidx，systemui项目和SettingsLibMoudle都有部分需要转换，可以通过AS的Refactor->Migrate to AndroidX转换。

###  程序包libcore.util不存在
这个可以直接取消引用，对应调用方法返回值设置为null就可以了，如果有用到就要去找对应的库。

### AAPT: error: resource android:dimen/notification_extra_margin_ambient not found.
类似以下这种资源在**framework/core/res/res/values/dimens.xml**文件内的情况：
```xml
    # 引用非 public 资源文件 (没在 public.xml 中声明的资源)
    # xml: @*android:<resource_type>/<resource_name>
    # code: com.android.internal.R.<resource_type>.<resource_name>
    <dimen name="group_overflow_number_extra_padding_dark">@*android:dimen/notification_extra_margin_ambient</dimen>
```
将每个模块的 build.grale 中属性配置为当前 API 级别
```
compileSdkVersion 28
buildToolsVersion '28.0.1'
```

### AAPT: error: duplicate value for resource 'attr/xxxx' with config ''.
删除attr.xml文件中对应的定义即可

### Duplicate class android.xxx.xxxx found in modules 
引入的jar包存在重复的类，删除重复的jar包

### style attribute 'attr/passwordStyle (aka com.android.systemui:attr/passwordStyle)' not found.
在attrs.xml中添加对应的属性，例:
```xml
    // 对应的fromat要对应
    <attr name="passwordStyle" format="reference" />
```

### resource style/PasswordTheme (aka com.android.systemui:style/PasswordTheme) not found.
与上面的问题类似，这个要在styles.xml中添加对应的style，例：
```xml
    <style name="PasswordTheme" parent="Theme.SystemUI">
        <item name="android:textColor">?attr/wallpaperTextColor</item>
        <item name="android:colorControlNormal">?attr/wallpaperTextColor</item>
        <item name="android:colorControlActivated">?attr/wallpaperTextColor</item>
    </style>
```

### no default product defined for resource com.android.systemui:string/power_remaining_duration_only_shutdown_imminent.
将values/string.xml中对应值的product改为default

### import com.android.systemui.shared.recents.IOverviewProxy;/错误: 程序包IRecentsSystemUserCallbacks不存在

systemUI和module shared包内部有aidl文件，在src/main下创建aidl文件夹，将对应的文件按照源文件夹包名路径转过去即可，我这边转移后目录结构如下：
* app
  * src
    * main
      * aidl
        * com.android.systemui.recents
          * IRecentsNonSystemUserCallbacks.aidl
          * IRecentsSystemUserCallbacks.aidl
      * java
* shared
  * src
    * main
      * aidl
        * com.android.systemui.shared
          * recents
            * IOverviewProxy.aidl
            * ISystemUiProxy.aidl
          * system
            * GraphicBufferCompat.aidl
      * java


### 找不到符号：Thread.setUncaughtExceptionPreHandler(uncaughtExceptionHandler)/Thread.getUncaughtExceptionPreHandler()
```
Thread.getUncaughtExceptionPreHandler() -> Thread.getDefaultUncaughtExceptionHandler()
Thread.setUncaughtExceptionPreHandler -> Thread.setDefaultUncaughtExceptionHandler
```

### 找不到符号import com.xxxx.xxx
因为我们转成androidx，所以引入的类的包名会有一些变动，重新引入即可。

### 权限问题
没有对应的权限就加权限，有的话还报错就在引用位置加一个try-catch抛出错误

