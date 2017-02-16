---
layout: post
title: "Raspberry Pi 3 编译opencv"
date: 2016-04-30 10:18:51 +0800
tags: [编译opencv,树莓派]
comments: true
categories: 2016-04
---
OpenCV的全称是：Open Source Computer Vision Library。OpenCV是一个基于（开源）发行的跨平台计算机视觉库，可以运行在Linux、Windows和Mac OS操作系统上。它轻量级而且高效——由一系列C函数和少量C++类构成，同时提供了Python、Ruby、MATLAB等语言的接口，实现了图像处理和计算机视觉方面的很多通用算法。

需要完成此次的项目，离不开opencv的支持，接下来我们就在树莓派上安装opencv。
<!--more-->

安装OpenCV的依赖包：

```
sudo apt-get install libqt4-dev libvtk5-qt4-dev
```

接下来需要从OpenCV官方网站：http://opencv.org 下载Linux版本的OpenCV的源代码：

{% img /imgs/opencv-install/1.jpg%}

我选择Linux平台下的2.4.9版本的源码包，将压缩包解压到/usr/local目录下。

进入opencv-2.4.9目录，新建一个build目录： 
 
```
sudo mkdir build
```

先安装 cmake  

```
sudo apt-get install cmake
```

进入build目录，利用下面的cmake命令进行编译设置：
  
注意下python部分：

{% img /imgs/opencv-install/2.jpg%}
 
```
cmake -D CMAKE_BUILD_TYPE=RELEASE -D .. 
```

等待检测和设置完成,就可以开始编译了:

```
sudo make
sudo make install
#更新搜索动态链接库
sudo ldconfig 
```

{% img /imgs/opencv-install/3.jpg%}

{% img /imgs/opencv-install/4.jpg%}

在python环境下执行

```
import cv
```

没有报错则安装成功~

参考:  
[GoBian安装OpenCV2.4.10](http://jjliu.blog.ustc.edu.cn/198/)   
[树莓派学习笔记—— 源代码方式安装opencv ](http://blog.csdn.net/xukai871105/article/details/40988101)  
