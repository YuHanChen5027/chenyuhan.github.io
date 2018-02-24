---
layout: post
title:  "Android引入IjkPlayer无法播放mkv格式视频的解决方案"
date:   2017-09-14 12:48:20 +0800
categories: article
---
## 写在前面
项目中直接引用或者直接编译源码得到的ijkplayer在播放mkv文件时出现（-10000）的错误，去项目github查看了才知道，默认是不支持mkv和rmvb格式视频的播放的。
用了一天时间解决（为什么用了一天，因为我蠢啊），这里记录一下解决的方法（官方上面其实已经有了详细的教程，无奈我当时没有很认真看。)，这里为我自己这个新手做个记录：
仍然是采用编译源码的方式引入，只是需要按照官方的方法更改一下脚本文件
ijkplayer官方地址：https://github.com/Bilibili/ijkplayer
运行系统：Mac OS

## 第一步 安装 homebrew, git, yam,ndk
这个网上教程很多，要不就不写了吧。。。。
![](http://upload-images.jianshu.io/upload_images/4273129-aca8d434f081d2ba.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

ndk的版本不要使用15，可以去网上下一个14的版本，否则可能会出现编译错误的状况。
ndk r14下载地址：https://developer.android.google.cn/ndk/downloads/index.html
##第二步 进行源码的下拉
- 在终端内输入以下命令：
```
git clone https://github.com/Bilibili/ijkplayer.git ijkplayer-android
cd ijkplayer-android
git checkout -B latest k0.8.4
./init-android.sh
```
## 第三步 修改编译ffmpeg用的脚本文件
- 删除默认的脚本文件，复制module-default.sh脚本文件，将复制副本更改为默认脚本文件名module.sh
```
cd config
rm module.sh
ln -s module-default.sh module.sh
cd ..
```
## 第四步 编译源码
```
cd android/contrib
./compile-ffmpeg.sh clean
./compile-ffmpeg.sh all

cd ..
./compile-ijk.sh all
```
##### 若执行编译失败，提示
```
LOCAL_SRC_FILES points to a missing file
```
尝试更换ndk版本为r14

##### 若执行编译失败，提示
```
Android NDK: android-9 is unsupported. Using minimum supported version android-14。
```
尝试去对应错误文件夹（如android/ijkplayer/ijkplayer-armv5/src/main下）修改AndroidManifest.xml中 android:minSdkVersion="9"改为 android:minSdkVersion="14";

同时对应的jni文件下的Application.mk 中 APP_PLATFORM := android-9 改为
APP_PLATFORM := android-14。

## 第五步 项目中加入对应的so库和引用





- 编译完成后我们将ijkplayer项目导入Android Studio中运行一下，
导入这个操作一定要做，不然不会生成
**ijkplayer-java-release.aar**文件
导入的操作如下（选中build.gradle文件）：
![](http://upload-images.jianshu.io/upload_images/4273129-6b29a223cf7d76e6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![](http://upload-images.jianshu.io/upload_images/4273129-1061525861a30b7f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

- 之后将我们所需架构所对应的包含so文件的文件夹（例：ijkplayer-x86/src/main/libs/下的x86文件夹）和**ijkplayer-java-release.aar**文件(在ijkplayer-java/build/output/aar文件夹下)拷贝到我们的项目libs目录下
![](http://upload-images.jianshu.io/upload_images/4273129-cf93e742fe67dfae.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
- 然后在build.gradle文件中添加
```
android{
...
sourceSets {
main {
jniLibs.srcDirs = ['libs']
}
}
}
repositories {
mavenCentral()
flatDir {
dirs 'libs'
}
}
dependencies {
...
compile(name: 'ijkplayer-java-release', ext: 'aar')
...
}
```

到此就搞定了，然后使用就好了，怎么使用，这个网上教程也很多，我也不讲了吧。。。。。。。
![](http://upload-images.jianshu.io/upload_images/4273129-aca8d434f081d2ba.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
