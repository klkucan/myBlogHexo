---
layout: post
title:  "侯捷C++学习笔记：Reference"
date:   2018-1-7  23:45:30 +0800
thumbnail: "img/home-bg-o.jpg"
categories: 
- 编程语言
tags:
- CPP
---

#### 本质
- 通过指针实现，所以本质就是指针

<!--more-->

#### 意义
- 是某个变量的代表，因此下面的代码中，x的值为5。在给x赋值后i和x都是6。

```
int i = 5;

int& x = i;

x = 6;
```

#### 编译器的一些做法
- 以一下代码为例，如果有一个变量b是a的引用，那么有`sizeof(a) == sizeof(b)`而且`&a == &b`。
- 上面也说了引用是个指针，在32位系统中sizeof指针应该是4个byte。但是因为b是a的引用，它代表了a，所以代码中`sizeof(b)`是16byte。而且a和b取地址得到的值也是一样的，虽然我们知道这个在内存中肯定是不一样的。只不过编译器做了这些的处理。

```
typedef struct A {int a, b, c, d};

A a;
A& b = a;
```

#### reference的常用方式
- 主要用于参数的修饰，很少用来声明变量。
- 参考以下的代码，在使用上引用传递在写法上和值传递是一样的，当然本质不同。但是从代码的优雅性上引用会比指针的参数形式好。


```
void Func1 (Cls a)
void Func2 (Cls& a)
void Func3 (Cls* a)

...

Cls obj;

Func1(obj);
Func2(obj);
Func3(&obj);
```

- 需要注意以下两个函数不能同时定义，因为在调用上会出现歧义。但是const修饰符可以实现重载。

```
void Func (Cls a)
void Func (Cls& a)

......

Cls obj;
Func(obj);

......

void Func (Cls a) const
void Func (Cls& a)
```
