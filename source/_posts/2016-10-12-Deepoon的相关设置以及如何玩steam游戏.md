---
layout: post
title:  "Deepoon的相关设置以及如何玩steam游戏"
date:   2016-10-12 21:21:12 +0800
thumbnail: "img/home-bg-o.jpg"
categories: 
- 游戏开发
tags:
- VR/AR
---

### 基本
- 保证Deepoon助手、显卡驱动是最新的。
- 安装Deepoon platform sdk。在platform sdk的安装过程中需要阅读其自带的文档，sdk path中不要带中文。

### DpnnGetLastError的处理
- 目前发现在安装了steamVR的情况下会出现DpnuGetLastError错误，在启动状态下出现错误后，退出steamVR。然后重启游戏。
- 在setting→play中关闭VR support，这个东西在启动后会被steamVR开启（如果你的程序中用到了steamVR）。
- 按照上述两部则应该不会出现DpnnGetLastError错误，并且只能通过头盔来看游戏内容。

### 玩steam
- 在steam的安装目录下放置xxx\Steam\steamapps\common\SteamVR\drivers\deepoon。如果没有放置这个驱动可能导致steam下无法设置VR房间。设置完成后应该是
![image](http://images2015.cnblogs.com/blog/23250/201610/23250-20161012092230578-1855749103.jpg)

- 经过上面的设置就可以用大朋眼镜玩steam游戏了。

### 开发相关
- 目前发现如果用大朋眼镜可以玩steam，则在unity中是不需要使用DpnCameraRig的，而且也不需要开启大朋助手。
- 从这些表现来看大朋应该用的是Oculus的底层。
  
