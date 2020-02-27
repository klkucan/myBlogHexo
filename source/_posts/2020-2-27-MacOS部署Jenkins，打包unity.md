---
layout: post
title:  "MacOS部署Jenkins，打包unity"
date:   2020-2-27 15:13:00 +0800
thumbnail: "img/home-bg-o.jpg"
categories: 
- 游戏开发
tags: 
- Unity3D
---

## 准备Jenkins
- 终端输入`brew install jenkins`，用这个方式安装本质还是个war包，但是不用再安装Tomcat
- 安装好后直接在终端输入`jenkins`启动应用，浏览器输入`localhost:8080`可以打开。
- 网上有8080端口被占用的处理办法
- brew安装失败大概率是因为githubusercontent被墙了，要么梯子，要么修改Mac上的hosts，方法如下：
    - `sudo vi /etc/hosts`
    - 在`https://www.ipaddress.com/`查询`raw.githubusercontent.com`的当前地址
    - 然后在hosts里面加入，当前的IP是`199.232.28.133  raw.githubusercontent.com`
- 网页打开后第一次要注册，然后会提示安装插件，最后就可以进入Jenkins。基本上准备工作没啥难度，除了网速。

<!--more-->

## 准备自动打包

#### Jenkins中配置

- 创建一个任务后，点击任务的配置
- 构建部分，这些暴露的参数就是给下面脚本用的，而脚本中调用unity命令时的参数又是在C#中通过GetCommandArgValue函数获得。

![image](https://note.youdao.com/yws/res/30807/D8B6A45F998D4332A6EDA84CE66F86B4)

```
export Project_Path=$WORKSPACE
export PRODUCT_NAME="Test"
export PROJ_OUTPUT="$Project_Path/build/android"
export UNITY_PATH="/Applications/Unity/Unity.app/Contents/MacOS/Unity"
export PROJ_PATH="$Project_Path/HelloUnity/"
export UNITY3D_BUILD_METHOD="PerformBuild.CommandLineBuildAndroid"
sh /Users/m/work/JenkinsTest/MacBuildAndroid.sh
```

#### 准备sh脚本

- 脚本主体就是调用unity的命令，其中大量的参数都来自jenkins的设置。

```
#！/bin/bash
echo "Delete build/android"
rm -rf $Project_Path/build/android
echo "Delete Success"

echo "Build Android"

${UNITY_PATH} -quit -batchmode -nographics -projectPath $PROJ_PATH -executeMethod ${UNITY3D_BUILD_METHOD} -logFile "log.txt" -PROJ_OUTPUT $PROJ_OUTPUT -PRODUCT_NAME $PRODUCT_NAME
echo "Build Success!"
```

#### 准备C#脚本

```
    static readonly string OUTPUT_PATH_NAME = "-PROJ_OUTPUT";
    static readonly string PRODUCTT_NAME = "-PRODUCT_NAME"; //包名称
    
    ...
    
    static string GetCommandArgValue(string key)
    {
        string defaultValue = string.Empty;
        // 获取参数的值
        string[] args = System.Environment.GetCommandLineArgs();
        for (int i = 0; i < args.Length - 1; i++)
        {
            if (args[i].Equals(key))
            {
                defaultValue = args[i + 1];
                break;
            }
        }
        return defaultValue;
    }
    
    static string[] GetBuildScenes()
    {
        List<string> names = new List<string>();

        foreach (EditorBuildSettingsScene e in EditorBuildSettings.scenes)
        {
            if (e == null)
                continue;

            if (e.enabled)
                names.Add(e.path);
        }
        return names.ToArray();
    }
    
    static string GetBuildPathAndroid()
    {
        string dirPath = GetCommandArgValue(OUTPUT_PATH_NAME) + "/" + GetCommandArgValue(PRODUCTT_NAME) + ".apk";
        if (!System.IO.Directory.Exists(dirPath))
        {
            System.IO.Directory.CreateDirectory(dirPath);
        }
        return dirPath;
    }

   
    static void CommandLineBuildAndroid()
    {
        Debug.Log("Command line build android version\n------------------\n------------------");

        string[] scenes = GetBuildScenes();
        string path = GetBuildPathAndroid();
        if (scenes == null || scenes.Length == 0 || path == null)
        {
            Debug.LogError("Please add scene to buildsetting...");
            return;
        }

        Debug.Log(string.Format("Path: \"{0}\"", path));
        for (int i = 0; i < scenes.Length; ++i)
        {
            Debug.Log(string.Format("Scene[{0}]: \"{1}\"", i, scenes[i]));
        }

        Debug.Log("Starting Android Build!");
        SetUnityBuildArgs(BuildTargetGroup.Android);
        BuildPipeline.BuildPlayer(scenes, path, BuildTarget.Android, BuildOptions.None);
    }
```



## 各种

#### 删除构建历史记录

- jenkins–系统管理----脚本命令行

```
// NOTE: uncomment parameters below if not using Scriptler >= 2.0, or if you're just pasting
// the script in manually.

// The name of the job.
def jobName = "Test"

// The range of build numbers to delete.
def buildRange = "1-30"

import jenkins.model.*;
import hudson.model.Fingerprint.RangeSet;
def j = jenkins.model.Jenkins.instance.getItem(jobName);

def r = RangeSet.fromString(buildRange, true);

j.getBuilds(r).each { it.delete() }
```

#### 修改时区

- jenkins–系统管理----脚本命令行

```
System.setProperty('org.apache.commons.jelly.tags.fmt.timeZone', 'Asia/Shanghai')
```
