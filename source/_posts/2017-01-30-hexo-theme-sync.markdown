---
layout: post
title: "基于Hexo的博客同步中的一些问题"
date: 2017-01-30 21:42:02 +0800
comments: true
categories: 2017-01
tags: [Hexo]
---
这篇文章讲讲在我的Octopress博客更换成基于Hexo的博客后，为了在多台电脑能够同步中遇到的问题。<!--more-->

我基于[这个回答](https://www.zhihu.com/question/21193762)来解决这个问题的。问题主要出在theme上，由于NexT主题引自第三方，所以这就牵扯到git中的submodule问题了。想偷懒把不引入，但是好像不行，于是按以下思路进行解决：clone NexT官方的github到自己的仓库，然后引入子模块，我这边不知道为什么有“[already exists in the index](https://my.oschina.net/jerikc/blog/513039)”的问题，执行如下命令：
```
git rm -r --cached theme/next
```

由于我是已经配置好了自己的NexT，为了不让它遗失，我先把它剪贴出来，再添加submodlue：
```
git submodule add git@github.com:starsight/hexo-theme-next.git themes/next
```

然后再把自己的配置覆盖fork下来的NexT仓库。
我们先要push submodule，在theme/next目录下依次执行：
```
git add .
git commit -m "next settings in fork next rep"
git push origin master //这是提交到fork后主题的仓库
```

这样，是提交到starsight/hexo-theme-next仓库。然后我们再更新[starsight.github.io](https://github.com/starsight/starsight.github.io)仓库：
```
cd ../../ //切到仓库的根目录
git add .
git commit -m "update next settings in blog sources branch"
git push origin hexo //注意hexo分支
```

以后写文章，只需要在根目录下（hexo分支）进行git add,commit,push(hexo)操作，例如：
```
git add .
git commit -m"new post hexo theme sync solution"
git push origin hexo
```

然后再更新master分支，即对外显示的html部分：
```
//hexo s -g
hexo d -g  // g为generate 生成，s为本地预览，d为deploy 部署到远程分支
```
参考文章：
[关于博客同步的解决办法](http://devtian.me/2015/03/17/blog-sync-solution/)  
[手动配置Git的Submodule](https://cragod.github.io/2016/GitSubmodule/)
[使用Git Submodule管理子模块](https://segmentfault.com/a/1190000003076028)