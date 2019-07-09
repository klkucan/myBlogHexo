---
layout: post
title:  "AssetBundle学习笔记：1、概述和创建"
date:   2017-3-14 11:40:00 +0800
thumbnail: "img/home-bg-o.jpg"
categories: 
- 游戏开发
tags:
- Unity3D
---

## 前言
- 打算彻底搞清楚AssetBundle的原理和使用，目前计划刷完官方文档和最佳实践系列文章，而且笔记中的内容会大量是官方文档的内容。

## 概述

- AssetBundle可以包含任意unity能识别的类型文件，甚至是一个场景。如果想包含自定义的二进制文件，需要文件后缀名是.bytes，unity会将这样的文件导入为TextAsset。

<!--more-->

#### AB流程
![image](https://docs.unity3d.com/550/Documentation/uploads/Main/AssetBundlesBuildPlusUpload.jpg)

- Editor编辑，场景中使用。
- 上传AB到服务器，其实这个不是必须的，也可以放到本地，后面会讲到这种case下怎么做才是最高效的。程序会按需加载AB，然后每个AB中的asset各自处理。其实就是在AssetBundle中按照需要加载不同的资源。
- 下载AB到本地，方法也非常多，各有优劣，后面会说。
- 从AssetBundle中加载GameObject，这个有一个AssetBundle→asset→GameObject的过程。而AB之前还可能有www。

#### 友情提示

- 你必须看[这个](http://unity3d.com/learn/tutorials/topics/best-practices/guide-assetbundles-and-resources)


## 创建AssetBundle

#### 设置AssetBundle
- 定义AssetBundle名称，在一个资源的最下方会有一个AssetBundle的设置。

![](https://docs.unity3d.com/550/Documentation/uploads/Main/AssetBundleInspectorNewBundle.png)
- 点击New按钮可以创建新包，并给它命名。下图的例子中资源被添加到一个名为environment/desert的AssetBundle中，这里面可能包含了之前已经放置的资源。另外在AssetBundle命名时unity会当名称转为小写。

![image](https://docs.unity3d.com/550/Documentation/uploads/Main/AssetBundleEditor.png)

- 如果创建了一些AssetBundle，但是没有放置任何的资源通过Remove Unused Names选项可以删除定义的名字。

> 从上述的说明感觉这个new出来的是一个空文件夹识，在设置资源时这个文件夹的名字作为了标识来使用。而Remove Unused Names按钮就是一键删除空文件夹。

#### 代码生成AssetBundle

- 这里有一段代码。代码很简单，而且没有很多容错机制。而且说实话这个代码很粗暴，所有被标记（就是上面说的设置）了AssetBundle的资源：prefab、scene文件等都会被无脑的build进指定的AssetBundle中。
- 
```
using UnityEditor;

public class CreateAssetBundles
{
    [MenuItem ("Assets/Build AssetBundles")]
    static void BuildAllAssetBundles ()
    {
        BuildPipeline.BuildAssetBundles ("Assets/AssetBundles", BuildAssetBundleOptions.None, BuildTarget.StandaloneOSXUniversal);
    }
}

```
- 同时生成的manifest文件包含一下内容：
    - [CRC](https://zh.wikipedia.org/wiki/%E5%BE%AA%E7%92%B0%E5%86%97%E9%A4%98%E6%A0%A1%E9%A9%97)
    - AssetFileHash：AssetBundle中所有asset文件的hash，一个单独的hash。只用来做增量build时的检查。
    - TypeTreeHash：AssetBundle中所有类型的hash，只用来做增量build时的检查。
    - ClassTypes： AssetBundle中包含的所有类类型，这些数据用来得到一个新的单独的hash当typetree做增量build检测。
    - Asset names.所有在AssetBundle中的asset的名称。
    - Dependent AssetBundle names. AssetBundle所依赖的其它AssetBundle的名字
    - 这个manifest文件只是用来做增量build的，非运行时必须。
    
- 放一个自己的测试数据

```
ManifestFileVersion: 0
CRC: 1763426742
Hashes:
  AssetFileHash:
    serializedVersion: 2
    Hash: eed209af1be31231fa135faaff2ab7b6
  TypeTreeHash:
    serializedVersion: 2
    Hash: 31d6cfe0d16ae931b73c59d7e0c089c0
HashAppended: 0
ClassTypes: []
Assets:
- Assets/Test.unity
Dependencies:
- E:/CodeDemo/New Unity Project 2/Assets/AssetBundles/sphere
```
 
- 还有两个文件生成，文件名就是打包目的地文件夹的名字，比如上面的代码就在AssetBundles文件夹下生成，名字是AssetBundles，一个是没有后缀的一个是以manifest为后缀，打开manifest文件可以看到下面的内容，可以看出来是这个目录下AssetBundle的信息。***其实它只包含两个  信息，一个是所有的AssetBundle名字和它们的依赖。***
```
ManifestFileVersion: 0
CRC: 4270654667
AssetBundleManifest:
  AssetBundleInfos:
    Info_0:
      Name: cube
      Dependencies: {}
    Info_1:
      Name: scene
      Dependencies: {}
```

> 做了个小测试，添加了两个新的AssetBundle，名字也不一样，每次添加这个文件的内容跟着变，并且CRC也在变。

```
ManifestFileVersion: 0
CRC: 2591254959
AssetBundleManifest:
  AssetBundleInfos:
    Info_0:
      Name: cube
      Dependencies: {}
    Info_1:
      Name: scene
      Dependencies:
        Dependency_0: sphere
        Dependency_1: capsule
    Info_2:
      Name: sphere
      Dependencies: {}
    Info_3:
      Name: capsule
      Dependencies: {}
```

> 继续小测试，把Capsule的AssetBundle从capsule改为sphere。果然只是简单记录了下目录中的AssetBundle信息。


```
ManifestFileVersion: 0
CRC: 3856405611
AssetBundleManifest:
  AssetBundleInfos:
    Info_0:
      Name: cube
      Dependencies: {}
    Info_1:
      Name: scene
      Dependencies:
        Dependency_0: sphere
    Info_2:
      Name: sphere
      Dependencies: {}
      
```

> 再次小测试，在UI的AssetBundle设定中第一栏设定了多层次的名称结果如下。而且从文件夹结构上看one和two都是两个文件夹，而在two里面则是一个名为xx.three的文件。

```
ManifestFileVersion: 0
CRC: 2167406161
AssetBundleManifest:
  AssetBundleInfos:
    Info_0:
      Name: scene
      Dependencies:
        Dependency_0: sphere
    Info_1:
      Name: sphere
      Dependencies: {}
    Info_2:
      Name: one/two/xx.three
      Dependencies: {} 
```


#### shader剥离（Shader stripping）

- 如果AssetBundle中包含shader，unity编辑器会根据当前场景和光照贴图的设置来决定使用哪种Lightmap modes。这还意味着你在打包的时候需要将打开着一个配置的场景。我的理解是因为需要根据当前场景来作为因素来做某个事情，所以这个场景需要时打开的。
- 也可以指定一个场景用来计算光照贴图模式，这个在用命令行buildbundle时是必须的。
![image](https://docs.unity3d.com/550/Documentation/uploads/Main/AssetBundleShaderStripping.png)

#### 引擎代码剥离
- 代码剥离这个事情是将没有在任何场景（特指Build Settings > Scene list中的场景）中使用的代码移除，减少了包的大小。不过只能用在iOS、Android和webGL。选择IL2CPP的是时候是默认打开的。而且顾名思义，只剥离引擎代码，你自己的代码不会被剥离。
![image](https://docs.unity3d.com/550/Documentation/uploads/Main/BuildingAssetBundles-enginecodestripping.png)

- 如果一个引擎代码在build的时候被剥离了，但是在运行时又被请求使用，那么就会报错，因为被剥离的代码无法再被访问。如何避免这个错呢？把player（此处我认为不是玩家的意思，而是引擎）可能用到的AssetBundle列出来，放置到一个manifest文件中，代码剥离系统使用这个文件中列出来的AssetBundle (along with the Scenes in the build)来确定哪些code需要保留。当一些平台或build不支持代码剥离，又或者没有勾选剥离时assetBundleManifestPath属性是被忽略的。

- 从代码角度来看就是

```
using UnityEngine;

using UnityEditor;

namespace AssetBundles

{

    public class BuildScript

    {

        public static void BuildPlayer()

        {

            BuildPlayerOptions buildPlayerOptions = new BuildPlayerOptions();   

            // example hard-coded platform manifest path

            buildPlayerOptions.assetBundleManifestPath = "AssetBundles/iOS/iOS.manifest";

            // build the Player ensuring engine code is included for 

            // AssetBundles in the manifest.

            BuildPipeline.BuildPlayer(buildPlayerOptions);

        }

    }

}
```

#### 几个小工具代码

- 列出所有在build process中产生的AssetBundle的名称

```
using UnityEditor;
using UnityEngine;

public class GetAssetBundleNames
{
    [MenuItem ("Assets/Get AssetBundle names")]
    static void GetNames ()
    {
        var names = AssetDatabase.GetAllAssetBundleNames();
        foreach (var name in names)
            Debug.Log ("AssetBundle: " + name);
    }
}
```

- 一个asset发生改变时的监听，讲道理这个函数从说明到函数名都看着真么不协调。*从函数名看感觉是AssetBundle名字变化，但是说明又是AssetBundle中一个asset发生变化。* 

```
using UnityEngine;
using UnityEditor;

public class MyPostprocessor : AssetPostprocessor {

    void OnPostprocessAssetbundleNameChanged ( string path,
            string previous, string next) {
        Debug.Log("AB: " + path + " old: " + previous + " new: " + next);
    }
}
```

#### AssetBundle变体

- 这个玩意有什么意义呢？
- 下图中在后一个label中填入一个名字，就可以形成变体。

![image](https://docs.unity3d.com/550/Documentation/uploads/Main/AssetBundle50Editor.png)

- 从文档上看可以同一批asset创建不同的变体，而且在这些变体AssetBundle中asset有意义的内部ID。需要注意这样的变体AssetBundle的全名是由AssetBundle name + the variant name组成的，如果你创建了一个名为MyAssets.hd的AssetBundle，那它只是一个普通的AssetBundle而不是变体的。而且 “MyAssets”+“hd”和“MyAssets.hd”+""不能同时存在因为它们具有相同的全称。


#### 脚本建议

- 妈蛋总算快完了。
- 呵呵，我不打算写了。

#### Typetrees
- 默认写到AssetBundle中，但是metro除外，因为不同的序列化方案。


#### 一些实验

- 以熊作为测试，如果将bear文件夹下的所有asset打包，会出现明显的依赖。而且贴图的AssetBundle大小比prefab本身要大。

```
ManifestFileVersion: 0
CRC: 519453993
AssetBundleManifest:
  AssetBundleInfos:
    Info_0:
      Name: xiong.assetbundle
      Dependencies:
        Dependency_0: baixiong.assetbundle
        Dependency_1: baixiongyachi.assetbundle
        Dependency_2: xiong@run.assetbundle
        Dependency_3: xiong@stand.assetbundle
    Info_1:
      Name: baixiong.assetbundle
      Dependencies: {}
    Info_2:
      Name: baixiongyachi.assetbundle
      Dependencies: {}
    Info_3:
      Name: xiong@run.assetbundle
      Dependencies: {}
    Info_4:
      Name: xiong@stand.assetbundle
      Dependencies: {}
```

- 如果只是打包xiong这个prefab，会产生一个100多K的xiaong的AssetBundle。可以正常使用。
- 但是如果做一个xiong2，其实就是xiong的复制体的话。会发现运行的时候avatar、mesh、shader等都是双份的。这个就是不把依赖单独打包的坏处了。
- 对于shader的双份问题可以通过`Editor->Project Settings->Graphics->Always Included Shaders`添加你用的shader，这样就不会出现了。