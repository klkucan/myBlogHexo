---
layout: post
title:  "Unity返回Value的协程"
date:   2017-7-23 23:31:11 +0800
thumbnail: "img/home-bg-o.jpg"
categories: 
- 游戏开发
tags:
- Unity3D
---


### 起因
- 其实一直以来就有个想法，希望能让unity中的coroutine返回一个具体的数值，比如int，比如string或者一个GameObject。最近也查了些资料，然后总结一下这个功能如何实现。

<!--more-->

### 回调函数实现

- 这个算是一个取巧的办法，在调用协程时传递一个回调函数，在执行过程中根据情况调用回调即可。不废话，直接上代码：

```
 IEnumerator CoroutineWithCallback(Action<string> callback)
{
    WWW www = new WWW("https://caihua.tech");
    yield return www;
    if (string.IsNullOrEmpty(www.error))
    {
        callback(www.text);
    }
    else
    {
        callback("www fail");
    }
}

// 调用方式

IEnumerator Start()
{
    yield return StartCoroutine(CoroutineWithCallback(s =>
    {
        Debug.Log(s);
    }));
}
```

### 利于协程的原理来制作一个可以返回数值的包装器

- 协程本身就是利用IEnumerator的可遍历来实现的，在运行期间如果约到yield语句就会等后面的代码执行完成后返回结果。如果在代码块的后面还有yield就会继续执行。


```
// 包装器
public class CoroutineWithData
{
    public Coroutine coroutine { get; private set; }
    public object result;
    private IEnumerator target;
    public CoroutineWithData(MonoBehaviour owner, IEnumerator target)
    {
        this.target = target;
        this.coroutine = owner.StartCoroutine(Run());
    }

    private IEnumerator Run()
    {
        while (target.MoveNext())
        {
            result = target.Current;
            yield return result;
        }
    }
}


// 真正执行逻辑的代码
IEnumerator LoadSomeStuff()
{
    WWW www = new WWW("https://caihua.tech");
    yield return www;
    if (string.IsNullOrEmpty(www.error))
    {
        yield return www.text;
    }
    else
    {
        yield return "fail";
    }
}

// 使用
...
CoroutineWithData cd = new CoroutineWithData(this, LoadSomeStuff());
yield return cd.coroutine;
Debug.Log("result is " + cd.result);
...
```