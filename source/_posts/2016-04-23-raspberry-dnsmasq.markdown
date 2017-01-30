---
layout: post
title: "Raspberry Pi 3 搭建本地DNS服务器"
date: 2016-04-23 21:39:04 +0800
tags: [树莓派,dnsmasq,dns,局域网缓存]
comments: true
categories: 2016-04
---
需求：上网时莫名地弹出广告，或者莫名的流量被消耗掉导致网速变慢。其次是部分网站域名不能正常被解析，莫名其妙地打不开，或者时好时坏。<!--more-->

管理下局域网的DNS（双十一的时候，把某宝网站直接给解析到本地ip，打不开网页，O(∩_∩)O哈哈~）

这里我用的是dnsmasq


安装比较简单：
```
sudo apt-get install -y dnsmasq
```

下面我们就需要配置dnsmasq了，配置文件一般位于路径/etc/dnsmasq.conf

这个文件里全是注释的内容，相当于空文件。我先备份了此文件，然后修改了一些配置：
```
resolv-file=/etc/wenjie_dns.conf
strict-order
cache-size=1500
listen-address=127.0.0.1,218.195.54.90
address=/taobao.com/218.195.54.90
```

```
resolv.conf:文件主要的作用是DNS客户机配置文件，设置DNS服务器的IP地址及DNS域名.如果我们不配置的话，默认使用 /etc/resolv.conf 网上有人说每次开机此文件都会被重写，所以就自己指定/etc/wj_dns.conf

strict-order: 按照resolv.conf中nameserver顺序依次使用，本行被注释后会随机的调用nameserver

cache-size: 缓存解析条数，默认是150

listen-address: 监听地址，listen-address=127.0.0.1，表示这个 dnsmasq 本机自己使用有效。
	注意：如果你想让本机所在的局域网的其它电脑也能够使用上Dnsmasq，应该把本机（树莓派）的局域网IP加上去：listen-address=218.195.54.90（树莓派的ip）,127.0.0.1

address:很明显就是我们想劫下的域名了
```

在/etc/wenjie_dns.conf 写入 DNS服务器，本地的当然得放在第一个，下面的写其它稳定的就行
```
nameserver 127.0.0.1
nameserver 202.206.240.13
nameserver 202.206.240.12
```

重启服务：
```
sudo service dnsmasq restart
```

将Dnsmasq作为本地DNS服务器使用，直接修改电脑的本地DNS的IP地址即可。

{% img /images/dns/1.jpg%}

ping 一下试试

{% img /images/dns/2.jpg%}  

大功告成。

拦截一些广告，也可以把域名劫到127.0.0.1 
bogus-nxdomain 这个配置文件里的选项可以反dns劫

对于dns缓存，可以用dig 命令看一下效果。
树莓派没有安装，可以安装一下：
```
sudo apt-get install dnsutils
```

还有no-hosts 选项 ，默认情况下这是注释掉的, dnsmasq 会首先寻找本地的 /etc/hosts 文件，再去寻找缓存下来的域名, 最后去上游 dns 服务器寻找。所以/etc/hosts才是dnsmasq第一个寻找的地方。

参考:  
[关于dnsmasq的使用配置](http://www.tuicool.com/articles/bUn2Uz)  
[Raspberry Pi （树莓派）折腾记之一](http://skypegnu1.blog.51cto.com/8991766/1641149)  
[树莓派瞎玩~9~dns服务器](http://www.xiaobaidonghui.cn/?p=400#more-400)  
[Dnsmasq安装与配置-搭建本地DNS服务器 更干净更快无广告DNS解析](http://www.freehao123.com/dnsmasq/)  
