---
layout: post
title:  "关于AssetBundle解压的讨论结果"
date:   2017-3-5 23:15:13 +0800
thumbnail: "img/home-bg-o.jpg"
categories: 
- 游戏开发
tags:
- Unity3D
---

- 一切的起源在于[这篇文章](《http://blog.uwa4d.com/archives/ABTheory.html》)，在文中对AssetBundle的加载进行了详细的描述，但是疑问也由此而生。以下是我在UWA讨论群的提问：
> 在讲WWW的时候有这样一段话【解压后的内容，通常为原Bundle文件的4~5倍大小，纹理资源比例可能更大】，在我理解就是www这样的方式获取到的资源（压缩形式的）会被解压缩，并放置到webstream中。而在【AssetBundle加载进阶】部分的【前者劣势】部分，又有【每次加载都涉及到解压操作】，请问这个【加载】指的是什么？是www.assetBundle还是AssetBundle.Load？还有【解压缩】是指的什么？不是在WWW的时候已经解压缩了吗？

<!--more-->

- 张鑫大神是这样回答的
> 是这样的，我们所说的AssetBundle【加载】指的是New WWW（或其他加载AB的API）和www.assetbundle的统称，而真正的【解压】过程，是在New WWW进行的。同时，需要说明的是，这个是5.3版本之前Unity对于AssetBundle的处理方式，5.3版本后，由于Unity新增了LZ4压缩方式，所以如果AssetBundle在制作时已经是LZ4格式的，那么在加载时就已经不会在进行解压了。

- 身为一个程序猿一定要弄清楚才行，所以有了另一个疑问
> 那么【每次加载都涉及到解压操作】是不是可以这样理解：对于new WWW实际是解压到webstream。而如果是用的LoadFromCacheOrDownload，那么资源是在磁盘的，所以在调用www.assetbundle时才做解压。或者正如你说的只要是加载AB的API，都会有一个解压过程。那么有个问题，如果使用直接获取AssetBundle的那些API，解压过程是怎样的呢？CreateFromFile是在调用assetbundle.load的时候从磁盘解压，CreateFromMemory是直接解压到webstream？还有个问题，对于WebStream来说，同一个www对象多次调用www.assetbundle方法时，得到了栈上的多个变量，但是这些变量都指向同一个WebStream中的对象吗？

- 其实在这里我犯了一个错误，就是对于资源最终放到磁盘上的AB来说在调用LoadFromCacheOrDownload和LoadFromFile（5.3之前是CreateFromFile）方法的过程中就已经解压缩了，LoadFromCacheOrDownload是上面文章中说明了我自己看漏了，但是LoadFromFile是有一个演进的过程的。
> 在5.3之前，如果需要解压，都在四个API（New WWW、CreateFromFile、LoadFromMemory、LoadFromCache等）中进行，而不是在后面执行
>
> CreateFromFile在4.x的API，只能加载非压缩的AB，而在5.0以后，对应的是LoadFromFile，5.3之后可以加载任何压缩格式的AB了

- 至此可以总结一点就是***解压缩发生在New WWW、CreateFromFile、LoadFromMemory、LoadFromCache函数调用过程中***。

- 至于张鑫大神提到的LZ4不要解压缩这个事情，从[文档](https://docs.unity3d.com/Manual/AssetBundleCompression.html)上看LZ4还是需要解压缩的，只不过是基于块的，也就是说当你从LZ4文件中加载一个object的时候，只会解压缩这个object对应的压缩块，这个当然效率就高了很多。但是这句我甚是不解，如果一个LZ4文件压缩了一个很大的对象且只有这个对象，解压的时候难道不要时间？？在群里问完大神在来补充吧。
> This occurs on-the-fly, meaning there are no wait times for the entire bundle to be decompressed before use.

- *第二天的更新*：
- 看了文档也和群里讨论了一下，应该还是我理解上有误。所谓`5.3版本后，由于Unity新增了LZ4压缩方式，所以如果AssetBundle在制作时已经是LZ4格式的，那么在加载时就已经不会在进行解压了。`的前提是在AB的基础上的，5.3以后的AB压缩格式3种：LZMA、LZ4和不压缩（Uncompressed Format，原文如此），LZMA是默认的AB压缩格式，unity在使用socket下载LZMA格式的文件时，一边下载，一边解压，并且一边压缩为LZ4。和大家讨论了下认为应该是考虑到传输的流量成本、节约内存和磁盘的空间，所以解压后再压缩为LZ4。一是因为LZMA压缩比高，适合传输，但是解压后会占用内容和磁盘空间比较大，而且是整体解压并且过程较慢，不适合用时在解压。所以解压后使用LZ4压缩一下节约空间，而且LZ4前面也说了是按需逐块解压的速度比较快，因此适合已经是内存中或者本地的解压。从这个角度说，如果直接AB压成LZ4就可以直接在需要的时候解压了，没有了解压再压缩的过程。

- 最后还是要刷[这个](https://unity3d.com/cn/learn/tutorials/topics/best-practices/guide-assetbundles-and-resources?playlist=30089)才行。



 