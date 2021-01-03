---
layout: post
title:  "Mac下搭建nginx http点播服务器"
author: "陈宇瀚"
date:   2017-09-30 13:57:42 +0800
categories: article
tags:
  - nginx
---
## 第一步 下载nginx和nginx_mod_h264_streaming-2.2.7
nginx下载地址：http://nginx.org/en/download.html

nginx_mod_h264_streaming-2.2.7 下载地址:
http://h264.code-shop.com/download/nginx_mod_h264_streaming-2.2.7.tar.gz

解压nginx 和 nginx_mod_h264_streaming 到同一目录下

![](http://upload-images.jianshu.io/upload_images/4273129-49feda242d8eafc2.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 第二步 配置nginx
进入文件夹nginx文件夹内，执行配置命令
```
cd nginx-1.13.5
./configure --add-module=../nginx_mod_h264_streaming-2.2.7 --with-http_flv_module --with-http_mp4_module
```
配置命令中 我们引入了第们刚才下载的三方模块nginx_mod_h264_streaming-2.2.7，以及nginx自带的mp4，flv模块

## 第三步 编译安装nginx
### 编译make
```
make
```
如果出现以下的错误，我们直接找到对应的文件进行修改：
####错误1:
![错误1](http://upload-images.jianshu.io/upload_images/4273129-cf852fdfba918904.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
我们进入”nginx_mod_h264_streaming-2.2.7/src/ “找到“ngx_http_streaming_module.c”文件并将zero_in_uri的方法注释或者删除

![](http://upload-images.jianshu.io/upload_images/4273129-df482fc56491eab1.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
保存后再次make

#### 错误2:

![错误2](http://upload-images.jianshu.io/upload_images/4273129-e21be49c01aedac2.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

提示我们文件中有未使用的变量，未使用的那直接注释或者删除掉。

![](http://upload-images.jianshu.io/upload_images/4273129-2559f062cd07404a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

”nginx_mod_h264_streaming-2.2.7/src/ “找到“mp4_io.c”文件并将aac_channels的注释或者删除
![](http://upload-images.jianshu.io/upload_images/4273129-213c4db80df76ae9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
保存后再次make
之后如果还有这种类型的错，采用同样的方基本都能解决了

### 安装install
```
make install
```
此时可能出现"Permission denied" 权限问题
那我们就加上sudo命令再执行
```
sudo make install
```

![](http://upload-images.jianshu.io/upload_images/4273129-3dd3b4e3f7333959.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
我们可以看到安装目录是 “usr/local/nginx”

### 第四步 配置nginx.conf
nginx.conf文件在nginx目录下的conf文件夹下(即“usr/local/nginx/conf”)，
我们需要修改nginx.conf（直接修改需要权限，同样通过sudo命令开启vi编辑器进行修改）
```
cd /
cd usr/local/nginx/conf
sudo vi  nginx.conf
```
这里有许多相关的配置信息（要了解各种配置可以去看一下这个网页：http://www.cnblogs.com/hunttown/p/5759959.html），我们先不用管，直接进入http 的 server下修改为如下代码：
```
server {
listen       80;            #设置端口号
server_name  localhost;
root usr/local/nginx/;      #设置文件路径，默认也是nginx路径下

charset utf-8;              #设置编码

location /{                 #设置首页地址
root html;           #此处地址是usr/local/nginx/html
index index.html；   #对应打开的文件
}

location ~ \.mp4$ {
root movie;         #此处地址是usr/local/nginx/movie(电影就放在该文件夹下)
mp4;
}
location ~ \.flv {
root movie;
flv;
}
```
### 第五步 启动nginx
我们设定的视频读取路径是 "usr/local/nginx/movie"，将1.mp4视频文件放入该文件夹，启动nginx(nginx启动文件放在nginx下的sbin文件夹内)
```
cd /
cd usr/local/nginx/sbin
sudo ./nginx
```
此时在浏览器内输入 http://localhost:80/ 会显示如下界面(80为端口号，默认80不需要输入，如果修改了的话就要输入对应的端口号)：

![](http://upload-images.jianshu.io/upload_images/4273129-a89d30a00ff2b89f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

即代表开启成功，此时我们在浏览器内输入地址"http://localhost:80/1.mp4"即可以播放对应的视频了

![](http://upload-images.jianshu.io/upload_images/4273129-888380bf0a3563f3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### nginx 启动，关闭，重启命令
启动：./nginx
关闭：./nginx -s stop （快速停止nginx）
&emsp;&emsp;&emsp;./nginx -s quit     (完整有序的停止nginx)
重启：./nginx -s reload （修改配置后重启）
