---
layout: post
title:  "AssetBundle学习笔记：4、AssetBundle的下载与asset的加载"
date:   2017-4-4 12:06:27 +0800
thumbnail: "img/home-bg-o.jpg"
categories: 
- 游戏开发
tags:
- Unity3D
---

## AssetBundle的下载
#### 下载AssetBundle

- 首先需要理解的是这个下载就是指的从网络下载。因此只有WWW和 WWW.LoadFromCacheOrDownload两个方法。
- www是不带缓存的，而后者是带缓存的。
- LoadFromCacheOrDownload的version参数，如果缓存中不存在，或者version低于指定的版本就会重新下载，否则直接从缓存拿数据。**缓存数据应该是始终在磁盘上的，而5.3以后缓存文件也压缩为LZ4(好像是可配置的)，所以读取cache的数据很可能与LoadFromFile是等效的。**
- 如果多个AssetBundle使用WWW.LoadFromCacheOrDownload下载，每帧只能完成一个。
- [LoadFromCacheOrDownload](https://docs.unity3d.com/ScriptReference/WWW.LoadFromCacheOrDownload.html)中如果发现缓存文件夹已经满了，会优先删除最近没有使用的cache，但是如果磁盘本身就满了或者说cache folder已经满了（这个东西是有容量限制的）那么会把AssetBundle数据直接按照stream的形式放到内存，这个和new WWW的做法完全一样。

<!--more-->

#### 生成AssetBundle
- 当访问` .assetBundle`属性时，会从下载的数据中提取并生产AssetBundle对象。此时就可以加载AssetBundle中的asset了。

#### 其它
- 后面在editor中加载asset的demo中API已经过时，可以用`AssetDatabase.LoadAssetAtPath`代替。


## Asset加载

### AssetBundle加载API概述

- 具体表格可以参考[原文](https://docs.unity3d.com/Manual/AssetBundleCompression.html)

#### WWW
- 在使用WWW时需要及时释放，因为它会在内存（WebStream）中保留下载的文件的大小。

#### LoadFromCacheOrDownload
- LoadFromCacheOrDownload没有什么额外的内存使用（其实应该有序列化文件（SerializedFile）的内存使用，而且如果prefab过多，有可能SerializedFile比WebStream还大，[来源](http://blog.uwa4d.com/archives/ABTheory.html)）
- 从表中可以看到在性能上也只有读磁盘的操作，然而LoadFromCacheOrDownload本质上是个网络操作，因此它会产生一些例如CRC检测的操作，所以unity的建议是如果是本地的AssetBundle不要用这个函数加载。
- 如果是加载缓存行为与 LoadFromFile一致，但是按照上面的说法看应该有差异吧。
- 补充：关于SerializedFile在4.7上每个的大小并不一定。`在5.3-5.5期间Unity引擎目前的做法，为了保证Android端apk解压速度而保证的两个256KB的buffer，在pc上只有7KB`。而到了5.6会改为2x8KB。但是经过测试这个不是必然的。

#### LoadFromMemory (Async)
- LoadFromMemory (Async)，不推荐使用，因为它的操作几乎都是在内存中完成的。加载AssetBundle需要占用大概AB size两倍的内存，一个API创建的内容还有就是参数里面的数据内存。如果是load一个asset会在内存中出现三个asset的拷贝，一个是托管代码字节数组，一个是 AssetBundle 的本地内存，第三个是在 GPU 或者系统内存中的资产本身。（第一个是参数、一个是AssetBundle本身、一个是Load函数产生的对象内存）。

#### LoadFromFile(Async)
- LoadFromFile(Async)，**++不能加载LZMA（特么的你敢信）。而且在Android上5.3和之前的unity无法加载StreamingAssets中的AssetBundle。++**
- 移动端上：当使用AssetBundle.LoadFromFile加载LZ4格式的文件时，其实不会将问价加载到内容，而是加载AssetBundle的Header。
- Editor下会将AssetBundle加载到内容，因此在Profiler中会出现峰值，不用担心。

#### WebRequest
- [WebRequest](https://docs.unity3d.com/Manual/UnityWebRequest-HLAPI.html)。
 (also supports caching)，这个东西算是WWW的替代品，很好用，回头写个总结。

#### 其它
- 使用WWW, WebRequest下载AssetBundle时，还有一个8x64KB的缓冲池buff来保存socket中的数据。so，这也是内存啊。

#### 从压缩格式角度看:
- 不压缩没啥优势，除了访问速度快。其实如果是不考虑内存或者磁盘的占用问题，倒是可以用这个。
- LZ4，从文档看unity应该用的是[LZ4HC](http://www.findbestopensource.com/product/lz4hc)，一个LZ4的高压缩版本。由于unity从5.3以后缓存也可以压缩了，而且这个算法是基于块的，所以在综合性上比较好。
- LZMA，必须先解压在压缩，如果不是因为网络传输需要节省流量，不建议使用。


#### 加载二进制数据
- 按照建议应该将原始的二进制文件保存为后缀为`.bytes`的文件，unity会视这个后缀的文件为TextAsset，作为一个TextAsset文件然后打包到AssetBundle。加载的时候也是一样，从AssetBundle中按照TextAsset类型得到asset，然后TextAsset.bytes属性得到byte数组。
```
string url = "http://www.mywebsite.com/mygame/assetbundles/assetbundle1.unity3d";
IEnumerator Start () {
    while (!Caching.ready)
        yield return null;

    // Start a download of the given URL
    WWW www = WWW.LoadFromCacheOrDownload (url, 1);

    // Wait for download to complete
    yield return www;

    // Load and retrieve the AssetBundle
    AssetBundle bundle = www.assetBundle;

    // Load the TextAsset object
    TextAsset txt = bundle.Load("myBinaryAsText", typeof(TextAsset)) as TextAsset;

    // Retrieve the binary data as an array of bytes
    byte[] bytes = txt.bytes;
}
```

#### 对于加密资源的处理

- 三个套路：
  - 加密自己的原始数据，然后改后缀为.byte，按照上面说的二进制的套路走。客户端拿到二进制数据后进行解密。
  - 加密AssetBundle，后缀名不限。用WWW的形式拿到数据后解密，然后用AssetBundle.CreateFromMemory生成AssetBundle。
  - 一个普通的AssetBundle中包含一个加密的AssetBundle（蛋疼不？）。需要经历获取TextAsset->拿到二进制数据->解密二进制数据->生成未加密AssetBundle->加载AssetBundle中的asset。

#### Script Asset的处理

- script->assembly(pre-compiled)->TextAsset->AssetBundle->TextAsset->byte[]->

```
var assembly = System.Reflection.Assembly.Load(txt.bytes);
var type = assembly.GetType("MyClassDerivedFromMonoBehaviour");

// Instantiate a GameObject and add a component with the loaded class
GameObject go = new GameObject();
go.AddComponent(type);
```

#### 卸载asset或者说AssetBundle
- AssetBundle.Unload方法是目前AssetBundle唯一能够使用的方法，与Resources.UnloadAsset比起来缺乏灵活性。
- 参数是个bool，传递false`unload the compressed data from memory`，按照第二部分的说明，什么是压缩的数据，那就是LZ4的数据，也就是AssetBundle。如果传递true，那么已经load出来的asset也会被清理。

## 一些策略

- 一个AssetBundle对象在同一时刻只能存在一份，所以当反复使用www下载并调用www.assetBundle时会报错` Cannot load cached AssetBundle. A file of the same name is already loaded from another AssetBundle`。这个时候要么自己unload要么就代码控制只保留一个对象。unity的建议是尽快删除不用的AssetBundle。
- unity5之前bundle在unload之前如果有load，需要等到load结束才会执行unload。so，可能会卡线程。
- [文中](https://docs.unity3d.com/560/Documentation/Manual/keepingtrackofloadedassetbundles.html)有一个完整的解决方案。


## 优化AssetBundle的磁盘寻道(Seek)

- unity序列化数据是为使得数据在读取时是线性的，减少寻道的次数。因为HDDs的硬盘比SSD慢很多。
- 一个场景的数据按照对象类型进行排序，在场景加载时，场景中的asset先按顺序加载，然后画面中的对象才加载。（完全可以理解，场景中摆放的多是prefab，肯定要先加载prefab所需要的资源。）而Scene AssetBundle也是这个顺序。
- 一个来自non-Scene AssetBundles中的asset在序列化上是和Scene AssetBundles一样的，但是加载方式取决于调用的API。LoadAllAssets这样的API会采用线性的方式加载所有的资源。但是如果是LoadAsset会随机读取的形式加载。
- DeterministicAssetBundles不破坏线性读取和类型排序，但是保留对象位置在一个特点的范围(没看懂)。
- LZ4不影响线性读取，但是会影响读取的颗粒度。（想来是因为按需加载，所以。。。）

#### 一些需要注意的行为

- 一些异步加载API会打破线性读取的模式，即使是读取场景。
- 因为有依赖的存在，所即使AssetBundle.LoadAllAssets这样的API也不能保证是线性读取的。因为可能出现依赖的资源在别的AssetBundle中，所以最好是将所有依赖的资源都读取出来放好了，这样可以使用最小的寻道次数。

#### 参考
- [MMAP](http://www.cnblogs.com/huxiao-tee/p/4660352.html)
- [一些内存测试数据](http://blog.csdn.net/lodypig/article/details/51879702)。第二个测试结果符合unity使用MMAP的结论。
- 一些不错的文章
  - [简单好理解](http://minhhh.github.io/posts/unity-asset-bundle)
  - [震惊！男人看了会沉默，女人看了会流泪，不看不是中国人之事实的真相系列。](http://blog.shuiguzi.com/2016/12/15/GuideToABAndRes/)