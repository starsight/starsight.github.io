---
layout: post
title: "Raspberry Pi 3 ngrok内网穿透"
date: 2016-04-28 13:48:51 +0800
tags: [ngrok,树莓派,内网穿透,内网]
comments: true
categories: 2016-04
---
现在很多宽带服务都不分公网ip，那怎么让外网访问树莓派的服务呢？
ngrok 就可以实现外网访问树莓派，ngrok的具体介绍和实现就不说了，主要来看看怎么使用。
<!--more-->

我用的是国内的ngrok.cc 的服务。

去ngrok.cc 下载arm版本的客户端，解压。
修改 ngrok.cfg 文件 

这里的token填写在 [Ngrok管理中心](http://www.ngrok.cc/index.php/Member/index.html) 的token

{% img /imgs/ngrok/1.jpg%}


我们需要在域名列表里面注册一下：

{% img /imgs/ngrok/2.jpg%}

我这里用的是自己的域名，还需要在dns解析 加一条记录

{% img /imgs/ngrok/3.jpg%}

不运行服务，直接输入域名，应该可以看到如下文字：

{% img /imgs/ngrok/4.jpg%}

说明这部分没有问题了~

接下来启动服务：  
```
 ./ngrok -config ngrok.cfg start test
```

这样表示运行成功：  

{% img /imgs/ngrok/5.jpg%}

打开浏览器，刷新一下刚才的界面：

{% img /imgs/ngrok/6.jpg%}

我们把上一次的owncloud也给加上
```
test.wenjiehe.com/owncloud
```

{% img /imgs/ngrok/7.jpg%}  
可以照常使用~~

可能遇到 ####正在访问来自不信任域名的服务器,请联系你的系统管理员。如果你是系统管理员，配置config/config.php文件中参数"trusted_domain"
则修改下config.php
```
sudo vi /var/www/html/owncloud/config/config.php
```  

{% img /imgs/ngrok/8.jpg%}

同时再提供下android版本的登录成功的截图：

{% img /imgs/ngrok/9.jpg%}

{% img /imgs/ngrok/10.jpg%}

如果没有自己的域名也可以使用ngrok.cc提供的二级域名。

参考:  
[Sunny-Ngrok内网转发](http://ngrok.cc/)  
["正在访问来自不信任域名的服务器"的解决办法](https://www.chiphell.com/thread-1534193-1-1.html)  
[如何让树莓派可以被外网访问？](https://www.zhihu.com/question/42433730)