---
layout: post
title:  "AssetBundle学习笔记：2、压缩"
date:   2017-3-20 18:00:00 +0800
thumbnail: "img/home-bg-o.jpg"
categories: 
- 游戏开发
tags:
- Unity3D
---

#### 格式
- 谈压缩就要谈到格式，unity目前在制作AssetBundle时默认就是压缩的，当然也可以设置为不压缩，但是显然不科学尤其是在移动端的开发上。

- LZMA：这个是一个序列化数据文件流，也是AssetBundle的默认压缩格式。LZMA提供最佳的压缩比，但是在使用的时候需要完全解压后才能使用，因此加载时需要的事件比较多。
- LZ4：5.3以后才加入的一种压缩格式，被unity自己大量使用。这是一种基于块的压缩方式，当一个对象从LZ4压缩的AssetBundle中加载时，会按照需要解压部分的块。这样的好处在于快速的加载对象，后面会提到一个最佳实践。当然，按照这个原理，你也可以想象如果一个LZ4AssetBundle中只包含了一个GameObject，那么也不存在什么按需加载了。当使用AssetBundle.LoadFromFile加载LZ4格式的文件时，其实不会将问价加载到内容，而是加载AssetBundle的Header。
- AssetBundle也可以选择不压缩，这样体积最大但是加载最快。

<!--more-->

#### 缓冲被压缩的AssetBundle
- WWW.LoadFromCacheOrDownload方法下载并缓冲AssetBundle到磁盘，5.3以后缓冲的文件可以是LZ4格式的压缩文件了，这直接导致了一个很2的事情。就是当你下载了LZMA格式的AssetBundle后，每当你通过socket下载到足够的数据后，unity就在后台悄悄的解压然后重新压缩为LZ4格式的文件直到下载结束，这个压缩发生在下载流（download streaming）中。当用到缓冲中的数据时，就会按照解压LZ4的套路走了。也就是按需按块解压。
- 缓冲压缩是默认的，然后可以通过 Caching.compressionEnabled属性来控制。会影响到包缓冲到磁盘和保存在内存中。

#### 指导方针，满满的干货

- 如果是将AssetBundle打包到游戏中，也就是AssetBundle跟着安装包走的话，推荐LZ4+AssetBundle.LoadFromFileAsync，既有压缩省空间，又有最快的加载性能可能性，给你读取内存buff的快感。
- 如果是网络下载可下载内容（DLC），则使用LZMA+LoadFromCacheOrDownload/WebRequest，这样可以获得最好的压缩比。缓存到本地后使用AssetBundle.LoadFromFile加载。这里需要注意，如果是一些需要持久化的内容，是需要实实在在的保存到本地的，否则进程结束后内存中的cache会清除。
- 如果是加密的AssetBundle，则使用LZ4+LoadFromMemoryAsync。其实也可以理解，毕竟需要加载到内容后进行解密。而且这个差不多是LoadFromMemory[Async]唯一的使用场景。
- 当使用自定义压缩时，请使用不压缩来创建AssetBundle，然后自己解压后使用AssetBundle.LoadFromFileAsync来加载。
- 为了节约内存，如果是不含prefab的AssetBundle，比如是texture的。最好用LoadFromCacheOrDownload，前提是通过网络获取，因为只有少量的序列化文件在内存。如果是保护prefab比较多的，用new WWW可能占用的内存会比序列化文件的都少。
- 总之，AssetBundle的使用需要进行灵活的应用。

