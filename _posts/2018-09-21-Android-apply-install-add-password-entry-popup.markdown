---
layout: post
title:  "Android 应用安装添加密码输入弹窗"
author: "陈宇瀚"
date:   2018-09-21 11:07:20 +0800
categories: article
tags:
  - Android 
  - Framework
---
基于RK Android6.0-MID代码
packages/apps/PackageInstaller/src/com/android/packageinstaller/PackageInstallerActivity.java
为防止之后遗忘，记录一下
安装apk时会弹出输入弹窗，输入内容正确，才可以点击安装按钮，输入错误安装应用弹窗消失

先引用需要用到的类
```
import android.widget.EditText;
import android.app.AlertDialog;
import android.view.Display;
import android.view.WindowManager;
```
在startInstallConfirm方法内添加如下代码
```
//cyh add start
          try {
                String packageName = mPkgInfo.packageName;     
                final EditText et = new EditText(PackageInstallerActivity.this);
                AlertDialog.Builder builder = new AlertDialog.Builder(PackageInstallerActivity.this);
                builder.setTitle("请输入安装应用密码")
                .setIcon(android.R.drawable.sym_def_app_icon)
                .setView(et)
                .setPositiveButton("确定", new DialogInterface.OnClickListener() {
                    @Override
                    public void onClick(DialogInterface dialogInterface, int i) {
                         if (et.getText().toString().equals("12345678")) {
                            //按下确定键后的事件
                            android.widget.Toast.makeText(PackageInstallerActivity.this, "密码正确", android.widget.Toast.LENGTH_LONG).show();
                        } else {
                            android.widget.Toast.makeText(PackageInstallerActivity.this,"密码错误", android.widget.Toast.LENGTH_LONG).show();
                            finish();
                            return;
                        }
                    }
                }).setNegativeButton("取消", new DialogInterface.OnClickListener() {
                    @Override
                    public void onClick(DialogInterface dialogInterface, int i) {
                         finish();
              		 return;
                    }
                });
                final AlertDialog dialog = builder.create();
                //获取屏幕的长宽
                WindowManager window=getWindowManager();
                Display display=window.getDefaultDisplay();
                int screenheight=display.getHeight();
                int screenWidth=display.getWidth();
                dialog.setCancelable(false);
                dialog.show();
                //设置弹出框的长宽
                dialog.getWindow().setLayout(screenWidth/3,screenheight/4);
           }catch(NullPointerException e){
                   e.printStackTrace();
           }
        //cyh add end 
        TabHost tabHost = (TabHost)findViewById(android.R.id.tabhost);
        tabHost.setup();
        ........
```
