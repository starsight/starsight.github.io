---
layout: post
title: "Raspberry Pi 3 wifi问题"
date: 2016-04-21 19:59:09 +0800
tags: [树莓派,wifi,树莓派3]
comments: true
categories: 2016-04
---
树莓派自带wifi模块，应该说是很实用的功能，用法其实和USB wifi一样使用，但是树莓派3死活连接不上我的wifi，于是我就参照USB wifi的使用。
<!--more-->
先用SSH登陆：
  
{% img /images/wifi/1.jpg%}  

{% img /images/wifi/2.jpg%} 

```
sudo vi /etc/wpa_supplicant/wpa_supplicant.conf
```

由于回宿舍，没有显示器，所以需要直接修改此文件：
```
network={
        ssid="ssid"
        psk="password"
        key_mgmt=WPA-PSK
}
```

按照道理说，直接点图形界面的wifi连接，输入密码就可以了，然而我的死活就是不行

尝试加入一些配置选项：
```
proto=WAP2
pairwise=TKIP
group=TKIP
```
依旧不行

于是我就换一种方法就行修改：
```
sudo vim /etc/network/interfaces

auto wlan0
iface wlan0 inet dhcp
    wpa-conf /etc/wpa_supplicant/wpa_supplicant.conf
```
依旧不行。。


后来

……

……

……

……

（这是个悲伤的故事）

发现路由器是斐讯的，网上有人说不支持、、不支持。。。

于是我试了下我手机开的热点，直接在图形化界面就连上了……

折腾了一天，好了，wifi算是可以用了（悲伤好大）

总结：wifi好不好用看路由器，TP的可以，腾达的也行，同时最好使用最新的3-18的镜像，之前版本可以有问题。