---
layout: post
title:  "AssetBundle学习笔记：Resources文件夹"
date:   2017-7-23 23:28:45 +0800
thumbnail: "img/home-bg-o.jpg"
categories: 
- 游戏开发
tags:
- Unity3D
---


## Resources文件夹
- 这个章节内容不多，但是也很有用。我们可以从这个章节中学习如何更合理的使用它。

### Resources系统的最佳实践

> **不要使用它**

- 是的，不要使用它。因为它有以下问题：
    - 使用 Resources 文件夹会让内存颗粒度管理变得更困难
    - 不正确的使用 Resources 文件夹会增加应用启动时间（因为要创建上一章讲的映射图）和包的大小，随着在 Resources 文件中的文件增加，管理这些文件会变得很困难。
    - Resources系统降低了项目自定义分发内容到具体平台的能力，消除了增量更新的可能。AssetBundle 变体是 Unity 用来在不同设备上调整内容的基础。

<!--more-->

### 正确的使用资源文件夹

> 下面两种情况 Resources 系统很有用处：

> 1.因为 Resources 系统很容易使用，它很适合是在快速原型制作和试验。但是当项目要转成产品时，强烈建议不要使用 Resources 文件夹.

> 2.Resources 文件夹对一些满足如下条件的案例很有用：
> - 存储在 Resources 文件夹下的内容不要很大的内存
> - 存储在 Resources 文件夹下的内容在整个项目周期都有用
> - 内容基本不用升级
> - 在各平台都一样的

### 序列化资源

- 一个工程中可能包含了多个名叫Resources的文件夹，所有这些文件夹下的对象和资材会在build的时候全部序列化到一个文件中，这个文件可以理解为一个特殊的AssetBundle。想来这就是为何不利于增量更新的原因了。
> AssetBundle包中索引信息里面包括了处理对象名字到对应的文件 GUID 和本地ID 的查找树。它也用来定位对象在序列化文件中的偏移位置。

> 用于查找的数据结构是平衡搜索树[1]（在大多数平台上），它的构建时间增长到了 O(nLog(N)), 其中 N 是在查找树内的对象的个数。这个增长也使 Resources 文件夹内的对象增加时，索引的加载时间会比线性增长时间长。

- 说了这么多就是告诉大家，应用启动时初始化resources文件夹（按照上一章说的应该是建立实例ID的映射图）这个事是不可避免的，当resources文件夹中内容太多时会严重影响启动速度。