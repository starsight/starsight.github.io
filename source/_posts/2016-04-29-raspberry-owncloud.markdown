---
layout: post
title: "Raspberry Pi 3 使用owncloud云服务"
date: 2016-04-29 10:48:51 +0800
tags: [云服务,owncloud,树莓派]
comments: true
categories: 2016-04
---
我们将要搭建自己的云系统平台，更精确的说是一个云存储系统，正如上面的产品所提供的功能。我们将使用开源软件ownCloud来搭建自己的私有云。ownCloud 起源于一个叫The KDE 云计算项目，现在已经适用于大多主流平台，它最早是KED的开发者Frank Karlitschek 创建的，现在由一个ownCloud team共同开发。

首先介绍一下ownCloud：
简单来说就是一个基于PHP的自建网盘。基本上是私人使用，没有用户注册功能，但是有用户添加功能，你可以无限制地添加用户，OwnCloud还提供了不少的免费应用，这些应用可以让你更好观看视频、倾听音乐等。
<!--more-->

ownCloud 内核是用PHP5写的，支持SQLite、MySQL、Oracle以及PostgreSQL等数据库。为了简单，我们将用MySQL数据库。在你的Linux系统下你需要安装以下软件：

PHP 安装包：php5, php5-gd, php-xml- parser,php5-intl  
数据库驱动：php5-mysql（如果你使用其他数据库，需要安装相应的数据库以及驱动）  
Curl 安装包：curl, libcurl3, php5-curl  
SMB 客户端：smbclient （这个用来挂载Windows共享文件夹的）  
Web 服务器：apache2  

####一键安装：
```
sudo apt-get install apache2 php5 php5-gd php-xml-parser php5-intl php5-sqlite php5-mysql smbclient curl libcurl3 php5-curl mysql-server
```

从 https://owncloud.org/install/ 下载最新的ownCloud Server 对于本文，我们使用owncloud-9.0.1 版本

{% img /images/owncloud/1.jpg%}

对于基于Debian发行版的Linux系统，web服务器的根目录为/var/www  我实际操作过程中似乎不可用，于是我就放在了html文件夹下。

```
cd /var/www/html  #（网页目录）
tar jxf owncloud-9.0.1.tar.bz2 -C  /var/www/html   #(解压至web目录)
cd /var/www/html/owncloud	 #（进入owncloud web目录）
mkdir data  	#(建立数据库目录)
cd data
```
####OwnCloud在安装的过程中需要对一些目录有写的权限，为此，web服务器用户（www-data对于基于Debian的系统）必须要拥有apps、data、config目录的权限。运行以下命令完成：

```
#data 目录下
sudo chown -R www-data:www-data data 
sudo chown -R www-data:www-data config 
sudo chown -R www-data:www-data apps
```

还需要修改下配置文件：
（网上说法不一,有说是 /etc/apache2/sites-enabled/000-default.conf ，还有说 /etc/httpd/conf.d/owncloud.conf 不过我没找到此文件）
```
sudo vi /etc/apache2/apache2.conf
```

修改如下：
```
<Directory /var/www/>
        Options Indexes FollowSymLinks
        AllowOverride All
        Require all granted
</Directory>
```

打开浏览器，输入http://IP/owncloud 

{% img /images/owncloud/2.jpg%}

电脑上装一下owncloud的客户端，然后操作比较简单

看我的同步效果：

{% img /images/owncloud/3.jpg%}

{% img /images/owncloud/4.jpg%}

{% img /images/owncloud/5.jpg%}

这是局域网的访问，如果没公网ip，则可用ngrok内网转发来实现外网访问。


参考:  
[使用OwnCloud创建私有云](http://jjliu.blog.ustc.edu.cn/198/)  
[树莓派Raspberry Pi安装ownCloud搭建私有云服务器](https://alexlee.cn/%e6%a0%91%e8%8e%93%e6%b4%beraspberry-pi%e5%ae%89%e8%a3%85owncloud%e6%90%ad%e5%bb%ba%e7%a7%81%e6%9c%89%e4%ba%91%e6%9c%8d%e5%8a%a1%e5%99%a8/)  
[Ubuntu 12.04下使用ownCloud搭建私人存储云](http://www.linuxidc.com/Linux/2013-08/89380.htm)  
[CentOS 6.3搭建个人私有云存储ownCloud](http://www.linuxidc.com/Linux/2014-03/98757.htm)  
