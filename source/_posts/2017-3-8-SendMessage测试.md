---
layout: post
title:  "SendMessage测试"
date:   2017-3-8 15:34:13 +0800
thumbnail: "img/home-bg-o.jpg"
categories: 
- 游戏开发
tags:
- Unity3D
---

- 5.5.2中没有了以前那种参数是对象的SendMessage方法，WTF.
- obj.SendMessage("Foo", 1, SendMessageOptions.DontRequireReceiver);如果设置了DontRequireReceiver，那么找不到Foo方法不会报错，如果没有写这个参数默认是RequireReceiver，则报错。
- 如果Foo的参数不对，会提示`MissingMethodException: The best match for method Foo has some invalid parameter.`


<!--more-->

```
    public void Foo(string s)
    {
        Debug.Log(s);
    }

```
- 如果Foo是没有参数的，SendMessage有参数也没关系。
- **总之最好指定的函数是存在的，参数是匹配的。**



 