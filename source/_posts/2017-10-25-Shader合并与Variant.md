---
layout: post
title:  "Shader合并与Variant"
date:   2017-10-25 00:02:11 +0800
thumbnail: "img/home-bg-o.jpg"
categories: 
- 游戏开发
tags:
- UWA
---

#### 前言
- 写这个的起因是因为看了唐建伟大佬在UWA上发表的文章[合并Shader系列_如何合并渲染状态](https://blog.uwa4d.com/archives/USparkle_Shader.html)。看完后受益良多，但是也对文中这样合并shader会不会产生shader的变种（shader variant）有所担心，因此在UWA的问答上提出了问题，最后唐建伟也做出了[回答](https://answer.uwa4d.com/question/59dd6f350461bc6f45206ad5/%E3%80%8A%E5%90%88%E5%B9%B6Shader%E7%B3%BB%E5%88%97-%7C-%E5%A6%82%E4%BD%95%E5%90%88%E5%B9%B6%E6%B8%B2%E6%9F%93%E7%8A%B6%E6%80%81%E3%80%8B%E8%BF%99%E7%AF%87%E6%96%87%E7%AB%A0%E5%BC%95%E5%8F%91%E7%9A%84shader-variant%E9%97%AE%E9%A2%98)，经过大佬授权在这里根据回答进行一些总结，想看原文的可以直接点击链接。

<!--more-->

#### Sahder Variant的概念


> 其实一个variant可以理解为一个具体的在GPU上执行的小程序，而一个Shader通常会编译出非常多的variant来应对不同的情况，比如单说雾效就有如下：没有雾效、有线性雾、有指数雾1、有指数雾2这样的4个variant(ps:这里只考虑雾效，其他条件一致)。至于原因嘛，有很多，粗略归纳一下是因为GPU需要更多的并行处理、逻辑单元少，因此Shader里面要尽可能规避各种判断、循环语句等等，最后本来可以通过逻辑判断来处理的雾效就需要编译成不同的执行程序来对应不同的情况(ps：这是一种高级优化，背后的原理很多很多，建议自行查询相关资料，查明前因后果，我懂的也不多)。

#### 问题1:这样通过多参数来设置渲染状态不会造成shader variant吗？

> 在这篇文章中，我们只合并了渲染状态，渲染状态的合并不会导致Unity编译出更多的shader variant。
口说无凭，那么我们就先拿一个示例Shader来做测试，我选用了文章中的“ShaderCombine/01.ShaderCombineSimpleZTest”来做测试，Unity版本为 5.5.4p3，使用Unity的Shader Variant Collection来算取Variant数量，不管是否加入合并的代码，Variant的数量都是259，如下图所示：

![image](https://uwa-public.oss-cn-beijing.aliyuncs.com/answer/image/public/101228/1507705049102.png)

> 即便是使用“ShaderCombine/02.ShaderCombineCommonState”来测试，Variant的数量也是是259，如下图所示

![image](https://uwa-public.oss-cn-beijing.aliyuncs.com/answer/image/public/101228/1507705061601.png)

> 从上面的图片可以看出，不管是否有渲染状态的参数在里面，Variant的数量都不会改变。

#### 问题2：什么情况下在才会造成shader variant呢？

> 宏定义(Keyword)，“#pragma multi_compile XXX YYY ZZZ”，“#pragma multi_compile_xxx”，SubShader，Pass，Fallback及一些特殊不常用命令等的增减会造成variant的数量变化。

> 至于一个Shader到底会生成多少variant呢？精确的计算方法，Unity并没有给出，但是我的归纳总结一下就是几组不同的编译宏的组合了，比如雾效、光照图、光源、阴影等等。另外还可以通过Unity的工具Shader Variant Collection来查看一个Shader到底有多少个variant，也可以在里面来自己组合和预编译自己想要的variant。

#### 福利：为什么修改渲染状态不会产生variant？

> 刚刚我们说了造成variant数量增加主要是需要生成不同的variant来应对不同的情况，那么不同的渲染状态是不是不同的情况呢？

> **其实不是。生成不同的variant主要是为了消灭Shader内部的逻辑判断（ps：Shader的真正逻辑是CGPROGRAM…ENDCG中间的东西）。**

> 继续用雾效举例，先只考虑有雾效和没雾效，按一般的游戏逻辑写法，我们通常会在逻辑里用一个if判断来搞定，但是由于GPU的特殊性，这样的做法非常低效、不可取，那么就会使用Keyword这样的编译宏在编译的时候就分别编译为有雾效和没雾效的2个执行程序，也就是2个variant。

> 现在说会渲染状态，看任何的Shader，我们都不会在CGPROGRAM…ENDCG里面有关于渲染状态的处理代码，当然不需要为不同的渲染状态编译不同的variant，也就不会造成variant的增加。（这部分可以参看Unity的渲染流水线，渲染一个物体需要非常多的步骤，我们写的Shader编译成的variant只在流水线中的两个可编程部分执行，而渲染状态是设置其他步骤的，与variant是完全隔离开、互不干扰。也可以说CGPROGRAM…ENDCG内的逻辑决定了variant的数量，CGPROGRAM…ENDCG外的是给Unity配置状态用的，不会引起Shader的逻辑变化，因此没变化）。
另附上一张简化版渲染流程图：
![image](https://uwa-public.oss-cn-beijing.aliyuncs.com/answer/image/public/101228/1507705163873.png)