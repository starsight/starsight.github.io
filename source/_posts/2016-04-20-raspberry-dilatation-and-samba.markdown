---
layout: post
title: "Raspberry Pi 3 开箱+扩容+samba"
date: 2016-04-20 20:33:34 +0800
tags: [树莓派,扩容,samba]
comments: true
categories: 2016-04
---
昨儿树莓3到手，今天先用了下电阻屏，帖子在此 ，花了些时间，今晚重点吧树莓基础环境先搭建下。<!--more-->
开箱图：
Pi3和微雪的屏

{% img /imgs/samba/1.jpg%} 

{% img /imgs/samba/2.jpg%} 

说明书多国语言：

{% img /imgs/samba/3.jpg%} 

正面照：

{% img /imgs/samba/4.jpg%} 

背面：

{% img /imgs/samba/5.jpg%} 

这个3的卡槽由以前的弹出式变成现在直插的，安装时候没有弹簧按压声。


安装很简单，去 http://downloads.raspberrypi.org/raspbian_latest 下载最新镜像即可，用Win32DiskImager 软件写入TF卡。

到显示器上显示：

{% img /imgs/samba/6.jpg%} 

接下来扩展下TF卡:
树莓派有傻瓜式方法：
```
sudo raspi-config
```

直接修改下重启就0k了~
如果想自己弄可以看这篇帖子：
http://bbs.hqchip.com/group-topic-id-35428.html

扩张后如下：

{% img /imgs/samba/7.jpg%} 

安装samba的话和Nanopi什么的大同小异


```
sudo apt-get install samba
sudo vi /etc/samba/smb.conf

[global]
        guest ok = yes
        security =share
[wj]
        comment = User
        path = /home/pi
        create mask = 0777
        directory mask = 0777
        guest ok = yes
        browseable = yes

sudo smbd restart
```
修改下权限：
```
chmod -R go+rwx /home/pi
```
然后查看下ip，就可以在电脑上连接啦~

{% img /imgs/samba/8.jpg%} 