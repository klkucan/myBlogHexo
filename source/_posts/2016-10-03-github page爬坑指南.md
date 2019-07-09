---
layout: post
title:  "Github Page爬坑指南"
date:   2016-10-03 13:18:23 +0800
thumbnail: "img/home-bg-o.jpg"
categories: 
- 杂记
tags: 
- 杂记
---

#### 1.jekyll的使用

 - jekyll的核心命令就几个
   
   ```
   jekyll new blog // 选好文件夹位置
   jekyll build    // 在_site文件夹下生成网站内容
   jekyll serve    // 可以在本地生成一个服务查看生成网站的内容，IP:127.0.0.1:4000
   
   ```
   
 - 模板，参考别人的模板进行了修改，[地址](https://github.com/Huxpro/huxpro.github.io)。

<!--more-->

#### 2.github上的设置

- github上需要设置的反而是最少的，创建一个用户名下的repository，例如sam.github.io，然后直接将jekyll中_site下的文件提交到这个repository即可。如果代码没错就可以直接通过访问sam.github.io进入个人网站了。

#### 3.个人域名绑定

- 首先在sam.github.io这个repository下添加一个名为CNAME的文件，内容为你自己的网站地址，比如sam.tech

- 去自己的域名管理中进行解析设置，以阿里云为例。从左侧进入`云解析 DNS`，选择域名后点击解析。删除原来的几个默认生成的解析，然后进行如下设置![](http://images2015.cnblogs.com/blog/23250/201610/23250-20161003173328629-236251795.jpg)
- 记录值为`sam.github.io.`，注意io后面有个点。

- 经过上面两步如果你的域名和网站内容没有问题的话就可以通过域名访问了。


#### 4.https的问题

- 经过上面的几步已经可以正常访问网站了，但是会注意到在chrome中会提示网站不安全，因为不支持https。

- 解决办法参考了[这篇文章](https://zhuanlan.zhihu.com/p/22667528)，在阿里云中通过`域名→管理`然后设置新的nameservers即可。只不过一直提示`非万网DNS`。
![](http://images2015.cnblogs.com/blog/23250/201610/23250-20161003173334707-1998000533.jpg)

#### 5.图片链接问题

- 目前生成出来的HTML代码中图片的路径是错误的，从网上也找了很多办法但是都不理想，咋这篇文章中使用的图片其实是我把图片上传到博客园，然后拿到URL后使用的。也就是说使用图片存储服务来实现，这样的好处就在于github page容量有限，如果将来图片多了会很麻烦，不如一开始就使用第三方来存储图片。
