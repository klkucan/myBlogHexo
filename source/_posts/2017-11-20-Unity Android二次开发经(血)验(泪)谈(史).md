---
layout: post
title:  "Unity Android二次开发经(血)验(泪)谈(史)"
date:   2017-11-20 00:47:54 +0800
thumbnail: "img/home-bg-o.jpg"
categories: 
- 游戏开发
tags:
- Unity3D
---

#### 主要参考的是[这篇文章](http://www.cnblogs.com/xtqqkss/p/6387271.html)，不过在这个期间遇到了各种的问题，在这里写出来，希望看到的人能够少走弯路。

## 准备工作
- 安装Android Studio（以下用AS代替），然后通过它下载Android SDK和tools。
- AS目前已经是3.0版本，在3之前不支持java8，也就是说没有lambda和一些新的特性。
- 从AS上可以下载到的build-tool是27，但是最新的tools是unity不支持，会在输出工程的时候报错，目前我使用的是26。
- PS:实际上在这几天的尝试过程中使用27在unity端会有一个`can not get xx list`的错误，通过网上查询说是build tools版本的问题。而AS端似乎也有问题，不过没有记清楚。

<!--more-->

## AS中做的事情
- 按照文中的说法做即可。
- 其实作者本人也提到了几个点：
    - 在建立module后要删除原来的app内存。
    - module名字和unity工程名字相同。
    - 这是一个很坑的地方，在最开始的时候，我不断的遇到一个提示错误：library和project使用了同样的package name，而且unity的文档也说了不能同名。但是实际上是完全可以同名的。而且不管是直接用module还是把APP改成module，这两种方式都需要包名相同。


## build AAR
- 其实在AS中只要把工程设置成了module，那么选择build apk出来的也是AAR。
- AAR文件放到`Assets\Plugins\Android\libs`
- 或者通过gradle直接生成，这个方式是目前用这最方便的。看下图，双击即可build。

![image](http://imglf5.nosdn.127.net/img/Nld0N0tacnNuUGtyd2tYaDIzbUF2MEYyMUkyRlJuZlExM3VTUUJCUkpVeWdBc3JxQ3JkcDJ3PT0.jpg?imageView&thumbnail=500x0&quality=96&stripmeta=0&type=jpg)

- 图中的classes.jar就是我们自己的代码，libs文件夹下的是我们拷贝unity的，要删除掉。
![image](http://imglf4.nosdn.127.net/img/Nld0N0tacnNuUGx4UzhpR2QzUzFkejc3S3V3eGVEL3hUVWRtczVhU3NGMnFieGJkcjFJdXhBPT0.png?imageView&thumbnail=500x0&quality=96&stripmeta=0&type=jpg)


## unity中build apk

- 这块分开说下，因为我在整个尝试的过程中尝试了直接使用module方法（以下用方法1替代），这个方法不需要用到AS中的manifest。还有就是将app改造成module（以下用方法2替代），这个需要提供manifest。

#### 方法1因为不需要提供manifest，所以省了很多事。
- 首先按照文中的方法先在AS中改造下manifest文件。
- 拷贝到Plugins/Android文件夹下。
- 在unity的PlayerSetting中进行设置。保证包名一致，保证minimum api level和AS中设置的一样。
- 在AS中我设置了`compileSdkVersion`和`targetSdkVersion`为25，开始按照文中和其他人的说法，在unity中target api level也设置的是25，==但是导出project时遇到android:configChanges错误：==

    > unity  CommandInvokationFailure: Gradle build failed. 'configChanges' with value...
        
- 真正的解决这个问题应该是在unity中Player Setting中将target API level设置为auto，这个要比minimum高才行。


#### 方法2会遇到的问题：

- 因为开始没有把AndroidManifest.xml放到Plugin下，导致每次unity编译的时候都生成一个新的build-id，与AS中的不一致。这个问题其实只要把AS中的manifest文件放到unityPlugin中，就保证了不会每次编译都ID变的问题。
    
- ==合并manifest==：AS的Gradle插件默认会启用Manifest Merger Tool，若Library项目中也定义了与主项目相同的属性（例如默认生成的android:icon和android:theme），则此时会合并失败。

- 解决方法有以下2种：1有效，2没试：
    - 方法1：在Manifest.xml的application标签下添加tools:replace="android:icon, android:theme"（多个属性用,隔开，并且记住在manifest根标签上加入xmlns:tools="http://schemas.android.com/tools"，否则会找不到namespace哦）
    - PS：其实是这样写`<category tools:replace=" android:name" android:name="com.google.intent.category.CARDBOARD" />`
    - 方法2：在build.gradle根标签上加上useOldManifestMerger true （懒人方法）

## 各种错误的处理

- [Failed to resolve:com.android.support:appcompat-v7:27+:报错处理](https://kotlintc.com/articles/3938)

- 这个问题本质在于AS或者说本地的工具版本低，从下图中可以找到本地的版本。可以看到时26，所以需要在build.gradle中设置`compile 'com.android.support:appcompat-v7:26.+'`，将27改为26即可。

![image](http://imglf3.nosdn.127.net/img/Nld0N0tacnNuUG5EUnpPWTVFTzMrZzJGdm5VMHhqWmUveENURXZnajJPUTRPcWpvY010eldBPT0.png?imageView&thumbnail=500x0&quality=96&stripmeta=0&type=jpg)

- 在project structure中可以设置java语言的版本，1.7的时候是不支持lambda的。需要注意。如果仍旧抽风般的提示lambda的问题就重启AS.

- AS2.3使用java8，本来3.0直接用的，2.3需要做如下处理。设置成功后也解决了`unity Can't read classes.jar`这个问题，`是因为项目使用的Java（JDK）版本比较高，而ProGuard, version 4.4支持的版本（最高到1.6），所以产生该问题。`，看了AS是一样的问题。

```
android {
  ...
  defaultConfig {
    ...
    jackOptions {
      enabled true
    }
  }
  compileOptions {
    sourceCompatibility JavaVersion.VERSION_1_8
    targetCompatibility JavaVersion.VERSION_1_8
  }
}
```

- 下面这个错误印象中是改AS中工程的target版本，或者删除了对应的代码。

> Error:(4) Error retrieving parent for item: No resource found that matches the given name 'android:TextAppearance.Material.Widget.Button.Inverse'.

- AS3版本好像不能支持gradle4.1，找到dependencies改成如下。

```
dependencies {
		classpath 'com.android.tools.build:gradle:3.0.0'
	}
	
...	

apply plugin: 'com.android.library'

....

defaultConfig {
		targetSdkVersion 27
		//applicationId 'xxxx'
	}
```
- 这个错误的情况是用unity生成并导出Android的代码，在2017.2p1的情况下输出的代码中gradle是4.1的。
- 如果是用AS自己生成APP或者module就没这样的问题。 

##### adb.exe 已停止工作

- 这个问题最终发现是360手机助手占用了5037端口造成的
- 排查过程记录一下：
    - 在\platform-tools目录下调用adb devices命令
    - 提示这个错误：`adb server version (31) doesn't match this client (39); killing...
error: could not install *smartsocket* listener: cannot bind to 127.0.0.1:5037: 通常每个套接字地址(协议/网络地址/端口)只允许使用一次。 (1
0048)`
    - `CMD:netstat -ano | findstr "5037"` //查看占用这个端口的APP，发现是进程15128
    - `CMD:tasklist | findstr "14152"`    //查看这个进程对应的应用
    - 下图中的14152进程是adb.exe，此时adb就允许正常了。

![image](http://imglf3.nosdn.127.net/img/Nld0N0tacnNuUG16ZHdGdmd1MEpFMkxkdWJUTEdTTURSc1NYYnhuSkUzZkY3MURnYnhRZ0JRPT0.png?imageView&thumbnail=500x0&quality=96&stripmeta=0&type=jpg)

## 后记

- 其实在整个的使用过程中还遇到了非常多的问题，都是遇到一个搜索一个。所以也不可能完全的记录下来。
- 另外遗留了一个问题就是目前还不能调用Android中的类方法。