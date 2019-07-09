---
layout: post
title:  "AssetBundle学习笔记：3、AssetBundle内部结构"
date:   2017-4-4 12:04:45 +0800
thumbnail: "img/home-bg-o.jpg"
categories: 
- 游戏开发
tags:
- Unity3D
---

#### 本质
- AssetBundle在本质上是一些被分组保存在一个序列化文件中的对象。被部署为一个数据文件，这些文件按照普通包和场景包的不同，在结构上有些差别。

#### Normal AssetBundle structure
![image](https://docs.unity3d.com/550/Documentation/uploads/Main/AssetBundleStructureNormal.png)

- 对于第一个块中的main serialized file应该是可以通过AssetBundle.mainAsset获取的，而谁是mainAsset呢？在build的时候第一个放置进去的资源就是mainAsset，所以可以采用将AssetBundle中所有的资源的名称写入一个文件，然后文件作为mainAsset来使用。[具体代码参考引用的第二部分](https://docs.unity3d.com/560/Documentation/ScriptReference/BuildPipeline.BuildAssetBundles.html)

<!--more-->

#### Streamed scene AssetBundle structure
![image](https://docs.unity3d.com/550/Documentation/uploads/Main/AssetBundleStructureStreamedScene.png)

- 虽然结构不同，但是是一样的压缩方式。
- 
#### AssetBundle compression
![image](https://docs.unity3d.com/550/Documentation/uploads/Main/AssetBundleArchiveFileSystem.png)

- LZ4（ chunk-based compression）将原始数据分隔后独立压缩，在解压的时候是随机读取的开销很小。而LZMA(stream-based compression)提供了高压缩比，但是解压时必须按照顺序读取。

- 总结:
	- 这篇的内容基本都是图，但是从normal bundle和Streamed scene bundle的区分可以看出对于一般的资源和场景资源，unity在AssetBundle中是有很大差别的。一般的资源中，可以将每个不同的类型的资源打包成一个AssetBundle(当然我们也可以将多个资源打包到一个AssetBundle中)。而在场景的AssetBundle中，主场景序列化文件是预加载的数据(场景中对象相关的InstanceID，对于的prefab对象等)、其它场景中使用的AssetBundle资源和对象。而后面的内容包含共享的数据，光照数据和其它一些资源文件。
	- 通过使用unity官方的AssetBundleBrowser可以看到不同类型的资源打包所带的资源文件。在使用这个工具打包的时候场景资源会默认包含光照的信息。

