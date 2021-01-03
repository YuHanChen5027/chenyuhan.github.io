---
layout: post
title:  "Android 系统中新增广播提供给应用设置静态ip"
author: "陈宇瀚"
date:   2020-03-09 19:31:00 +0800
categories: article
tags:
  - Android 
  - Framework
---

# 系统中新增静态ip设置
    本文主要目的是开发提供给应用开发使用修改连接Wifi或以太网时的静态ip设置，采用广播的方式控制，在Setting是内同实现时仿照Settings应用内部的修改方式实现。  
    新增两个广播：  
    1.com.cyh.wifi_static_ip：wifi静态ip设置广播
    2.com.cyh.eth_static_ip：以太网静态ip设置广播
    在Settings内新增一个广播接收NetworkStaticReceiver.java
## AndroidManifest.xml 
在**AndroidManifest.xml**文件内添加上我们新增的**NetworkStaticReceiver**和相关广播
```
 <receiver android:name="NetworkStaticReceiver">
            <intent-filter android:priority="1000">
                <action android:name="com.cyh.wifi_static_ip"/>
                <action android:name="com.cyh.eth_static_ip" />
            </intent-filter>
       </receiver>
```
之后来看**NetworkStaticReceiver.java**文件的内容
## NetworkStaticReceiver.java
路径：packages/apps/Settings/src/com/android/settings/NetworkStaticReceiver.java，只发逻辑部分，省略了import部分。
```
public class NetworkStaticReceiver extends BroadcastReceiver{
     private final String WIFI_STATIC_IP = "com.cyh.wifi_static_ip";
     private final String ETH_STATIC_IP = "com.cyh.eth_static_ip";
     private final String TAG="NetworkStaticReceiver";
     
     @Override
        public void onReceive(Context context, Intent intent){
                String action = intent.getAction();
                switch(action){
                    if(WIFI_STATIC_IP.equals(action)){
                //Wi-Fi静态设置
                 //获得是否设置为静态模式
                        boolean isStatic =  intent.getBooleanExtra("Static",false);
                        //获取连接wifi的config
                        WifiManager mWifiManager = (WifiManager)context.getApplicationContext().getSystemService(Context.WIFI_SERVICE);
                        WifiConfiguration config = new WifiConfiguration();
                        WifiInfo connectionInfo= mWifiManager.getConnectionInfo();
                                List<WifiConfiguration> configuredNetworks=mWifiManager.getConfiguredNetworks();
                                for (WifiConfiguration conf : configuredNetworks) {
                                        if (conf.networkId==connectionInfo.getNetworkId()) {
                                                config=conf;
                                                break;
                                        }
                                }
                        if(isStatic){
                                 //获得设置的Wi-Fi静态ip，网关，DNS1，DNS2
                                String ip = intent.getStringExtra("staticIp");
                                String gateway = intent.getStringExtra("gateway");
                                String dns1 = intent.getStringExtra("dns1");
                                String dns2 = intent.getStringExtra("dns2");
                                //判断传入参数是否为空，空的话给默认值
                                ip = ip ==null?"192.168.0.123":ip;
                                gateway = gateway ==null?"192.168.1.1":gateway;
                                dns1 = dns1 ==null?"8.8.8.8":dns1;
                                dns2 = dns2 ==null?"8.8.4,4":dns2;
                                //validateIpConfigFields用于验证ip，网管，dns1，dns2，并将相关设置项转换成StaticIpConfiguration
                                StaticIpConfiguration mStaticIpConfiguration = validateIpConfigFields(ip,gateway,dns1,dns2);
                               
                               if(mStaticIpConfiguration == null){
                                        //设置相关内容有问题时mStaticIpConfiguration会为null
                                        return;
                                }
                                config.setStaticIpConfiguration(mStaticIpConfiguration);
                                mWifiManager.updateNetwork(config);
                                boolean saveConfiguration = mWifiManager.saveConfiguration();

                                int netId = mWifiManager.addNetwork(config);
                                //断开网络
                                mWifiManager.disableNetwork(netId);
                                //重新连接
                                mWifiManager.enableNetwork(netId, true);
                        }else{
                                //动态IP模式（DHCP）
                                config.setIpAssignment(IpAssignment.DHCP);
                                mWifiManager.updateNetwork(config);
                                mWifiManager.saveConfiguration();
                                int netId = mWifiManager.addNetwork(config);
                                //断开网络
                                mWifiManager.disableNetwork(netId);
                                //重新连接
                                mWifiManager.enableNetwork(netId, true);
                        }
                } else if(ETH_STATIC_IP.equals(action)){
                        //获得是否设置为静态模式
                        boolean isStatic =  intent.getBooleanExtra("Static",false);
                        Log.i(TAG,"ETH_STATIC_IP isStatic:"+isStatic);
                        EthernetManager mEthManager = (EthernetManager)context.getApplicationContext().getSystemService(Context.ETHERNET_SERVICE);
                        if(isStatic){
                                String ip = intent.getStringExtra("staticIp");
                                String gateway = intent.getStringExtra("gateway");
                                String dns1 = intent.getStringExtra("dns1");
                                String dns2 = intent.getStringExtra("dns2");
                                //判断传入参数是否为空，空的话给默认值
                                ip = ip ==null?"192.168.0.123":ip;
                                gateway = gateway ==null?"192.168.1.1":gateway;
                                dns1 = dns1 ==null?"8.8.8.8":dns1;
                                dns2 = dns2 ==null?"8.8.4,4":dns2;
                                StaticIpConfiguration mStaticIpConfiguration = validateIpConfigFields(ip,gateway,dns1,dns2);
                                  if(mStaticIpConfiguration == null){
                                        //设置相关内容有问题时mStaticIpConfiguration会为null
                                        return;
                                }
                                IpConfiguration mIpConfiguration = new IpConfiguration(IpAssignment.STATIC, ProxySettings.NONE,mStaticIpConfiguration,null);
                                mEthManager.setConfiguration(mIpConfiguration);
                        }else{
                                //动态IP模式（DHCP）
                                mEthManager.setConfiguration(new IpConfiguration(IpAssignment.DHCP, ProxySettings.NONE,null,null));
                        }
                }
        }
    }
    //验证ip，网管，dns1，dns2，并将相关设置项转换成StaticIpConfiguration
    public StaticIpConfiguration validateIpConfigFields(String ipAddr,String gateway,String dns1,String dns2){
        StaticIpConfiguration staticIpConfiguration = new StaticIpConfiguration();
         //ip地址
        if (TextUtils.isEmpty(ipAddr)){
                Log.i(TAG,"Type a valid IP address.");
                return null;
        }
        Inet4Address inetAddr = getIPv4Address(ipAddr);
        if (inetAddr == null) {
            Log.i(TAG,"Type a valid IP address.");
            return null;
        }
        
        int networkPrefixLength = -1;
        try {
            networkPrefixLength = 24;
            if (networkPrefixLength < 0 || networkPrefixLength > 32) {
                Log.i(TAG,"Type a network prefix length between 0 and 32.");
                return null;
            }
            staticIpConfiguration.ipAddress = new LinkAddress(inetAddr, networkPrefixLength);
        } catch (NumberFormatException e) {
             Log.i(TAG,"24");
        }
        //网关
        if (TextUtils.isEmpty(gateway)) {
            try {
                //Extract a default gateway from IP address
                InetAddress netPart = NetworkUtils.getNetworkPart(inetAddr, networkPrefixLength);
                byte[] addr = netPart.getAddress();
                addr[addr.length-1] = 1;
            } catch (RuntimeException ee) {
                }
        } else {
            InetAddress gatewayAddr = getIPv4Address(gateway);
            if (gatewayAddr == null) {
                  Log.i(TAG,"ype a valid gateway address.");
                return null;
            }
            staticIpConfiguration.gateway = gatewayAddr;
        }
        //dns1
        InetAddress dnsAddr = null;
        if (TextUtils.isEmpty(dns1)) {
             Log.i(TAG,"8.8.8.8");
        } else {
            dnsAddr = getIPv4Address(dns1);
            if (dnsAddr == null) {
                Log.i(TAG,"Type a valid DNS address.");
                return null;
            }
            staticIpConfiguration.dnsServers.add(dnsAddr);
        }
        
        //dns2
        if (!TextUtils.isEmpty(dns2)) {
            dnsAddr = getIPv4Address(dns2);
            if (dnsAddr == null) {
                Log.i(TAG,"Type a valid DNS address.");
                return null;
            }
            staticIpConfiguration.dnsServers.add(dnsAddr);
        }
        return staticIpConfiguration;
    }
    
    private Inet4Address getIPv4Address(String text) {
        try {
            return (Inet4Address) NetworkUtils.numericToInetAddress(text);
        } catch (IllegalArgumentException|ClassCastException e) {
            return null;
        }
    }
    
}
```
以上就是NetworkStaticReceiver.java的逻辑内容，大体逻辑步骤可简述为：  
接收广播->辨别广播是WI-FI设置还是以太网设置  
1. WI-FI设置：
- 获得当前连接wifi的config，获得isStatic对应的值->
- true则获取设置的ip，网关,dns等设置项的值,false则直接设置为DHCP->   
- 判断设置信息是否正常，同时封装成**StaticIpConfiguration**->
- 将**StaticIpConfiguration**设置进当前Wi-Fi->
- 通过WifiManager更新并保存配置config->
- 断开网络连接，之后重连（只有这样设置后才有效）。
2. 以太网设置
- 获得isStatic对应的值->
- true则获取设置的ip，网关,dns等设置项的值,false则直接设置为DHCP->
- 判断设置信息是否正常，同时封装成**StaticIpConfiguration**->
- 创建一个**IpConfiguration**用来存放**StaticIpConfiguration**，同时设置为STATIC模式->
- 通过**EthernetManager**保存配置项;

## 应用内控制方法
```
  /**
     * 控制Wi-Fi的状态
     * @param isStatic true为STATIC模式，false为DNCP模式。只有TRUE时之后的配置项才有效，false时后面的参数设置null即可
     * @param staticIp STATIC模式下的静态ip
     * @param gateway  STATIC模式下的网关
     * @param dns1     STATIC模式下的dns
     * @param dns2     STATIC模式下的备用dns
     */
    public void controlWifiStatic(boolean isStatic,String staticIp,String gateway,String dns1,String dns2){
        Intent intent = new Intent("com.cyh.wifi_static_ip");
        intent.putExtra("Static",isStatic);
        intent.putExtra("staticIp",staticIp);
        intent.putExtra("gateway",gateway);
        intent.putExtra("dns1",dns1);
        intent.putExtra("dns2",dns2;
        sendBroadcast(intent);
    }
    
    //调用示例
    controlWifiStatic(true,"192.168.0.190","192.168.0.1","8.8.8.8","8.8.4.4");
     
     /**
     * 控制以太网的状态
     * @param isStatic true为STATIC模式，false为DNCP模式。只有TRUE时之后的配置项才有效，false时后面的参数设置null即可
     * @param staticIp STATIC模式下的静态ip
     * @param gateway  STATIC模式下的网关
     * @param dns1     STATIC模式下的dns
     * @param dns2     STATIC模式下的备用dns
     */
    public void controlEthStatic(boolean isStatic,String staticIp,String gateway,String dns1,String dns2){
        Intent intent = new Intent("com.cyh.eth_static_ip");
        intent.putExtra("Static",isStatic);
        intent.putExtra("staticIp",staticIp);
        intent.putExtra("gateway",gateway);
        intent.putExtra("dns1",dns1);
        intent.putExtra("dns2",dns2;
        sendBroadcast(intent);
    }
     //调用示例
    controlEthStatic(true,"192.168.0.190","192.168.0.1","8.8.8.8","8.8.4.4");
```


    
    

