---
layout: post
title: "Raspberry Pi 3 中文显示及输入+百度云传输"
date: 2016-04-22 21:07:14 +0800
tags: [中文显示输入,树莓派,百度云]
comments: true
categories: 2016-04
---
树莓派系统显示中文的话，大部分都是一个的小方块，还好我们有非常无私的开源者：
```
sudo apt-get install ttf-wqy-zenhei
```
文泉驿的开源中文字体，树莓派全网唯一的开源中文字体库

能显示中文，输入也是必不可少的：<!--more-->
```
sudo apt-get install scim-pinyin
```
快捷键也是Ctrl+空格
```
sudo raspi-config
```

{% img /images/baiduyun/1.jpg%}  


然后选择第五项Internationalisation Options，change_locale，在Default locale for the system environment:中选择zh_CN.UTF-8

重启就可以生效啦~

下面装神器：百度云~

https://github.com/houtianze/bypy
这是地址，把它下载到树莓派上，解压。


我的树莓派好多python库都装了，所以直接：
```
./bypy.py info
```

在浏览器中粘贴那个网址，再输入你的百度账号，
会得到一串授权码。

{% img /images/baiduyun/2.jpg%}  

粘贴到刚才的终端上：

{% img /images/baiduyun/3.jpg%}  

好了，至此就搞定了~

下面测试下：
```
显示使用帮助和所有命令（英文）: 
bypy.py 

更详细的了解某一个命令： 
bypy.py help <command> 

显示在云盘（程序的）根目录下文件列表： 
bypy.py list 

把当前目录同步到云盘： 
bypy.py syncup 
or 
bypy.py upload 

把云盘内容同步到本地来： 
bypy.py syncdown 
or 
bypy.py downdir / 

## 比较本地当前目录和云盘（程序的）根目录（这个很有用）：## 
bypy.py compare
```



上传下，看一下我的百度云，有了！
  
{% img /images/baiduyun/4.jpg%} 

下载：

{% img /images/baiduyun/5.jpg%} 
 