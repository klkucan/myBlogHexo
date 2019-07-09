---
layout: post
title:  "AssetBundle学习笔记：资产、对象和序列化"
date:   2017-7-16 00:13:52 +0800
thumbnail: "img/home-bg-o.jpg"
categories: 
- 游戏开发
tags:
- Unity3D
---


## 前言
- 这篇及后续的几篇笔记是我阅读[A guide to AssetBundles and Resources](https://unity3d.com/cn/learn/tutoriazals/topics/best-practices/guide-assetbundles-and-resources)的一些记录。官方的这个系列文章详细的阐述了unity是管理资源的。从一个asset的导入开始，unity就在背后开始了它的工作。它为每个资源创造了一个mate文件，给予了资源一个唯一的ID，从此这个资源的一切都与这个ID紧密结合到了一起。可能我们的一些操作会导致资源丢失它的mate文件，这会造成一些很不好的后果。
- 与此同时，系列文章中还重点阐述了什么是AssetBundle，如何下载AssetBundle和如何从AssetBundle中加载asset。
- 最关键的是最后一章，在这一章中详细的描写了AssetBundle的制作策略。AssetBundle可以说是unity中至关重要的一个概念，但是在日常的工作中按照怎么样的颗粒度来划分AssetBundle，如何平衡AssetBundle的精细度和方便性，这一直以来是一个让人头痛的问题，第四章中给出了一些答案。

<!--more-->

## 资产、对象和序列化

- 这是系列文章的第一部分（前面还有一个类似介绍的章节），讲述了unity如何处理导入的资源，如何去标识它们，管理它们。

### 资产的导入

- 首先asset这个词如果只是看翻译的话可以是**资产**，其实在我看来它可以是资源，和resource这个词的含义几乎一样。但是因为Resource这个词在unity中有着特殊的含义，它代表了一个特别的文件夹，所以可能需要用一个别的词语来表示资源的概念。
- asset可以是一个脚本、一个音频文件或者是一个美术从3D建模工具中导出的FBX文件，它保存在unity工程的Assets文件夹下，它的格式可能unity能够直接使用的(比如一个material)，也可能是不能直接使用的（比如FBX）。
- 每一个asset在导入到unity时都会进行一项工作，就是将asset序列化，因为只有只有unity引擎才能够使用它们。在这个导入的过程中可能会对纹理等进行压缩，所以这个过程会比较久。
- 这些被序列化后的数据保存在工程的Library文件夹下，我们不需要修改这些数据，因为这个是unity自动做的，每次你导入新的数据的时候都会生成这些文件。而且如果你留意的话，会发现你切换unity的目标平台时（Android→ios）会重现生成一次序列化文件，因为每个平台能够识别的数据类型不同。同一个纹理文件可能对于了不同的序列化文件。（没事不要删除Library文件夹，因为这个序列化过程非常耗时。）
- 通常我们在unity的IDE中看到的asset文件都是原始的文件，因为导入的过程中虽然做了序列化，但是其实是生成了新的文件，而原始的文件并没有被修改。[（给出一个参考文档）](https://docs.unity3d.com/Manual/BehindtheScenes.html)

### 资产导入的结果是什么

- 答案是序列化文件，那么我们通过什么使用这些序列化文件呢？答案是 UnityEngine.Object,或者是大写O的Object。
- Object是一套序列化数据用来描述一个具体的资源实例。它（object）的类型可以是任何引擎使用的类型，比如一个mesh、一个音频片段等。基本上所有的内置对象类型都是UnityEngine.Object的子类型，除了ScriptableObject和MonoScript。
- 需要注意的是，一个外部文件的导入可能产生了多个asset，比如FBX文件的导入，可能产了mesh、material和texture。

#### 什么是ScriptableObject和MonoScript
- ScriptableObject在我看来是一种内置的数据存储格式
- MonoScript代表了工程中的一个脚本文件，它是个editor类。用法

```
#if UNITY_EDITOR

                mediatorClass = RTEditorGUI.ObjectField<MonoScript>(mediatorClass, true);
                if (mediatorClass != null)
                {
                    mediatorName = mediatorClass.GetClass().Name;
                }
#endif

```

### unity如何区分这些导入的asset

- 这是一个核心的问题。unity在生成（以FBX文件导入为例，可能一个FBX生成了unity中的多个asset）一个asset时，会同时生成一个mate文件。这个asset会分配一个File GUID，而这个asset中的多个UnityEngine.Object会生成多个Local ID。这个理解起来比较容易，一个asset就是一栋房子，有一个唯一的门牌号。而一栋房子的每个房间又有一个唯一的房间号。
- unity editor将这个File GUID与asset的路径关联了起来。这样一来，如果你移动了asset的位置，unity会更新这个File GUID对应的路径，这也是为何unity建议我们移动或者重命名或者删除文件已经要在IDE中做，因为unity会监控文件的变化。如果你在外面改了文件名字或者路径，回到unity后会发现unity会给这个asset重新生成一个mate文件，显然这样会分配一个新的GUID。
- 后果是什么？如果你的这个asset中的内容已经被使用，那么会出现丢失的情况。总的来说，unity就是靠File GUID和Local ID来管理资源，一切变化它都会关注，而且也会帮我们处理好。但是外在的变化它无从得知，所以只能出现错误。有趣的一点是，如果你只是删除了mate文件，但是不改变asset的位置，那么unity在重新生成mate时会给asset一个原来的GUID。

### unity是如何在运行时管理资源的

#### PersistentManager缓存的概念
- 由于GUID在运行时比较的时候效率较低，所以unity使用了一套其它的机制来管理运行时的资源。这就是PersistentManager缓存。unity将File GUID和Local ID翻译（个人认为应该是经过某种变换）成一个叫做Instance ID的整型数据，它在单独的会话（single session，难以理解）中是唯一的。我个人的理解是在一个进程中这个Instance ID是唯一的。
- 缓存维护了一个InstanceID和内存中实例对象的映射图。UnityEngine.Objects通过这个做到了强引用。通过Instance ID可以快速的找到内存中的对象。

#### PersistentManager缓存的初始化和更新
- 这个映射图在游戏启动的时候就会初始化。数据包含：游戏启动时初始场景中用到的对象的映射，Resources文件中所有的对象信息都会被映射到缓存中（这个缓存建立是需要时间的，所以如果Resources文件中内容太多启动会变慢）。
- 在游戏运行的过程中这个缓存也会进行更新。比如在运行时新的asset被导入了（不能理解这是一个什么场景），或者一些对象从AssetBundle中加载的时候，会生成新的Instance ID，新的对象，也就有了新的映射。
- 还有当AssetBundle被卸载的时候（这个卸载应该指的是所有已经实例化对象也被卸载），对应的映射数据会被删除。而当对象重新从AssetBundle中加载时，会产生一个新的Instance ID。这个也是后面会说到的AssetBundle管理的一个重要内容，如何保证无效的资源在卸载时也被删除掉。
- 需要注意一点就是有时一些特别的事件会导致内存中的对象被删除了，比如ios的挂起，此时unity是没有办法重新加载这些数据的（比如材质数据），此时场景中看到的就是品红色的对象了。所以如果手机上出现这个问题了，就要考虑是不是内存中对象被删除了。

### Resource的生命周期

#### 加载

- 按照文中的说法有两种方式来加载UnityEngine.Objects，第一是被自动加载，这个需要一个实例ID映射到这个对象，同时这个对象没有加载进内存并被间接引用，而且对象的数据源存在。我对这个的理解是参考初始场景中的对象，前文说过它们会进入缓存中，而且在一开始肯定是内存中不存在的，而且如果这个实例ID对应的FILE ID和local ID是有效的，那么就会被自动的加载；至于说间接引用，讲真，没理解。
- 第二种加载就是代码加载了，这个API就多了，resources的load或者AssetBundle的load都可以做到。
- 当一个对象被加载，Unity 会尝试将所有引用从文件 GUID 和本地 ID 转换成实例 ID。（这个应该是为了把运行时的引用关系都保存在PersistentManager所管理的映射图中。）

#### 卸载
- 如果一个fileID和localID没有对应的实例ID，或者实例ID所对应的fileID和localID无效了，那么就会出现missing的情况，此时实例ID仍旧是存在的，在场景中它也被别的对象引用着，但是你可能看不到它或者它显示为品红色。
- 这个情况其实挺常见的，由于操作问题导致asset的fileID发生了变化，但是原来的实例ID没有跟着更新，就容易出现。
- 文中列举了三个被卸载的情况
    - 在未使用的asset被清理时对象会被自动卸载。这个过程会在场景切换的时候发生。
    - 从Resources文件夹中加载的对象在调用Resources.UnloadAsset时会卸载。但是对象对应的实例ID保持有效，而且也对应着有效的fileID和localID。**这些对象如果被其它的mono变了或者对象引用着，那么在调用Resources.UnloadAsset后这些对象会被重新加载。**（这个就有意思了，代表在Resources.UnloadAsset函数有时不会实现我们想要的效果。）

- 当调用AssetBundle.Unload(false)时，unity不会将从这个AssetBundle中加载的还处于激活状态的对象销毁掉，但是会使这些对象对应的实例ID与fileID和localID断开引用。如果这些对象从内存中卸载并且对这些已卸载的对象的引用依然保持着，Unity将无法重新加载对象。
- 上面这段讲的比较奇怪，因为在后面的最后一章中应该是描述了这样的情况，此时是可以重新加载asset的，只不过会造成内存泄漏。
- 在注视中讲到了这个情况的一个例子就是APP被挂起后，GPU内存中的纹理、网格被删除了，那么在APP恢复后需要重新加载这些内容。


### 加载大的层次结构

#### 几个点
- 在序列化层级结构时会将所有对象和组件都单独的序列化到序列化文件中。这对于层级的加载产生阴影。
- 层级加载时CPU时间消耗在以下内容上：
    - 读取序列化数据
    - 在新创建出的transform中设置层级结构
    - 实例化对象和组件
    - 唤醒对象和组件（awake）
- 后面三条基本上花费的时间是固定的，但是第一条不一样。层级越复杂加载的越慢。
- 在所有平台上，内存读取速度快过磁盘，PC读取快过手机。因此当层级过于复杂时可能读取的时间都要长于实例化的时间。
- 对于层级中数据相同的对象会多次进行序列化，比如UI上30个复制出来的相同的元素会序列化三十次，加载的时候也是加载30个不同的数据，因此很耗时。
- 5.4之后unity修改了transform在内存中的呈现形式。每个根节点下的所有子层级都保存在一段紧密连续的内存中。当实例化一个会导致重新指定父级的新对象时，考虑使用GameObject.Instantiate方法的带父对象参数的重载。使用这个重载可以避免给新物体新分配根 tranform 层次。测试结果中，这个可以提高 5 - 10 % 的实例化时间。


### 总结

- 这个章节讲述了unity中对UnityEngine.Objects和asset的管理，从资材的导入到运行时的加载卸载。读完后应该明白一下几点：

- UnityEngine.Objects与asset的关系
- asset导入后产生了mate文件，产生了fileID和localID的概念
- asset在运行时中产生了实例ID的概念，并且实例ID和fileID、localID是通过一个映射图结合到了一起，这个映射图也让内存中的object与磁盘上的asset产生了联系。
- asset可以被加载也可以被卸载，而它的卸载需要注意不要让资源出现内存泄漏的问题。
- 最后在场景中的层级结构需要合理的规划。
