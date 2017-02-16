---
layout: post
title: "Raspberry Pi 3 opencv+python的人脸识别"
date: 2016-05-01 16:44:58 +0800
tags: [树莓派,opencv,python,人脸识别]
comments: true
categories: 2016-05
---
上次安装了opencv库，这次我们来使用一下。

<!--more-->
安装PIL库

```
sudo pip install pil
```

```python  
xx.py:

#!/usr/bin/env python
#coding=utf-8
import os
from PIL import Image, ImageDraw
import cv

def detect_object(image):
    '''检测图片，获取人脸在图片中的坐标'''
    grayscale = cv.CreateImage((image.width, image.height), 8, 1)
    cv.CvtColor(image, grayscale, cv.CV_BGR2GRAY)

    cascade = cv.Load("/usr/local/opencv-2.4.9/data/haarcascades/haarcascade_frontalface_alt_tree.xml")
    rect = cv.HaarDetectObjects(grayscale, cascade, cv.CreateMemStorage(), 1.1, 2,
        cv.CV_HAAR_DO_CANNY_PRUNING, (20,20))

    result = []
    for r in rect:
        result.append((r[0][0], r[0][1], r[0][0]+r[0][2], r[0][1]+r[0][3]))

    return result

def process(infile):
    '''在原图上框出头像并且截取每个头像到单独文件夹'''
    image = cv.LoadImage(infile);
    if image:
        faces = detect_object(image)

    im = Image.open(infile)
    path = os.path.abspath(infile)
    save_path = os.path.splitext(path)[0]+"_face"
    try:
        os.mkdir(save_path)
    except:
        pass
    if faces:
        draw = ImageDraw.Draw(im)
        count = 0
        for f in faces:
            count += 1
            draw.rectangle(f, outline=(255, 0, 0))
            a = im.crop(f)
            file_name = os.path.join(save_path,str(count)+".jpg")
     #       print file_name
            a.save(file_name)

        drow_save_path = os.path.join(save_path,"out.jpg")
        im.save(drow_save_path, "JPEG", quality=80)
    else:
        print "Error: cannot detect faces on %s" % infile

if __name__ == "__main__":
    process("kobe.jpg")
```

接下来只要运行python xx.py

过一会，如果图片内有人的话生成一个文件夹，里面有一张人脸的截图和一张人脸的标识图。

以下为示例：

{% img /imgs/face/1.jpg%}

{% img /imgs/face/2.jpg%}

{% img /imgs/face/3.jpg%}

接下来用一下摄像头：

```
sudo apt-get install fswebcam 
sudo fswebcam --device /dev/video0  a.jpg
```

在 process("kobe.jpg") 前面加一句:

```
os.system("fswebcam --device /dev/video0 /home/pi/Desktop/kobe.jpg")
```

看一下效果：

{% img /imgs/face/4.jpg%}

{% img /imgs/face/5.jpg%}

光线不好还是能认出来，说明opencv自带的分类器算开源里面不错的了~

参考:  
[Tigerboard开发板试用体验 python+opencv的人脸识别](http://bbs.ickey.cn/group-topic-id-67165.html)   
[NanoPi2试用体验 简单人脸识别-结项](http://blog.csdn.net/u010873775/article/details/50615834)
