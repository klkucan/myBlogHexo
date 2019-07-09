---
layout: post
title:  "Unity ECS：C# Job System文档总结"
date:   2018-9-26 10:32:00 +0800
thumbnail: "img/home-bg-o.jpg"
categories: 
- 游戏开发
tags:
- Unity3D
---

### 背景、初衷、目标
- 出现的背景、初衷就是：天下苦无多线程久已
- 当然这个说法要进一步的约束，首先unity中可以编写多线程代码，但是最大的问题是多线程中无法使用unity API，当然也无法修改component中的数据了。
- 目前来看似乎这个技术就是为了能够让开发者使用multithread来操作unity API和component 数据。

### 优势和劣势
- 多线程就是优势
- 可操作的数据类型限制比较大，目前只能是blittable data type
- 这个多线程是和unity引擎共享work thread，所以变相的侵占了引擎层的性能。

<!--more-->

### 组成部分和关键点

#### 组成
- job system
    - job system管理一组worker thread，通常每个逻辑上的CPU的核（core）可以跑一个worker。这是为了避免上下文切换带来的性能问题。
    - job system会把一些job放入一个队列，然后worker thread从队列中取出job进行执行。
- job：最小工作单元，接受参数和操作作为数据。job是可以自足的，也可以依赖其他的job。
- NativeContainer：
    - 一个托管的值类型，对native memory提供安全的C#的包装器。
    - 包含一个指向非托管内存地址的指针
    - job system中NativeContainer允许job和主线程共享数据，而不是copy的数据。

#### 关键点

##### 保障执行顺序和数据竟态
- job system管理job之间的依赖，保证执行的正确。
- job system通过深拷贝（memcpy）数据来解决竟态问题，带来的问题就是只能访问blittable data type。
- job system在托管和native之间传递copy的data，调度job时使用memcpy将数据放到native memory，然后在job执行时给托管代码这边访问数据的能力。


##### NativeContainer
- safety system的缺点是因为采用了copy数据的方式，所以数据结果在job之间是独立的。可以想象对于有执行顺序依赖的job，这样的结果是错误的。NativeContainer就是以为了解决这个问题。
- NativeContainer包含的数据结果：
    - NativeList - a resizable NativeArray.
    - NativeHashMap - key and value pairs.
    - NativeMultiHashMap - multiple values per key.
    - NativeQueue - a first in, first out (FIFO) queue.
- Allocator：分配器，不同的分配器区别主要在对于NativeContainer对象lifetime的影响。
    - Allocator.Temp：NativeContainer的lifespan只在一帧内，不能将这样的NativeContainer传递给job。而且要在某个使用了它的函数结束前调研Dispose方法来销毁NativeContainer。内存分配的速度很快（怀疑是在一个已经分配好的shared memory中）。
    - Allocator.TempJob：中速，lifespan 4帧，线程安全。多数比较小的job都使用这个。如果4帧后没有销毁，会有一个来自native code的警告输出。
    - Allocator.Persistent：最慢，NativeContainer长存。本身是malloc的包装器。

##### NativeContainer中的safety system
- **注意：所有NativeContainer中的安全监测只在Editor和Play Mode有效。也就是真机无需。**
- DisposeSentinel：发现内存泄漏，然后给出error。
- AtomicSafetyHandle：转移NativeContainer的控制权。这个有效需要使用宏：ENABLE_UNITY_COLLECTIONS_CHECKS
- 不管是主线程还是job对于NativeContainer的访问仍旧是同一时间只允许一个写操作，但是支持并发读操作。
- 如果NativeContainer被设计为只读的可以使用`[ReadOnly]`
- **注意：job中访问静态数据绕过了所有的保护系统，可能引起unitycrash。**

##### 创建job
- 创建一个inherit IJob的struct
- 添加成员变量，只能是NativeContainer或者blittable types
- 创建Execute函数，job运行时Execute函数在CPU core中会执行一次。 
- **注意：job中操作的数据除了NativeContainer的外都是一份拷贝。job中访问主线程中数据的唯一方式就是用操作NativeContainer类型的数据。（调用job的代码中很好的体现了这个）**
- 代码参考原文文档

##### 使用job
- 直接调用job的Schedule方法
- 调用Schedule方法将job放到一个队列中，一旦放入就不能中断。
- Schedule方法只能在主线程调用。
- 注意源码最后释放了NativeArray类型的result，因为它的分配器是TempJob。
- 代码参考原文文档

##### JobHandle和依赖
- JobHandle是Schedule函数的返回值
- 在job存在依赖时利用它来设置依赖
- 多依赖时使用CombineDependencies
- 当在主线程需要等待job完成，可以使用Complete方法。

##### ParallelFor jobs
- 利用多和并发执行job
- 可以通过这种参数决定在native层有多少个batch
- 参数中的lenght应该是作为输出结果的NativeContainer的lenght，它告诉job system有多少execute函数要执行。
- innerloopBatchCount从字面看应该是batch中有多少个job，从下面代码的注释看也是这个意思。   
`// Schedule the job with one Execute per index in the results array and only 1 item per processing batch`     
`JobHandle handle = jobData.Schedule(result.Length, 1);`

![image](https://docs.unity3d.com/uploads/Main/jobsystem_parallelfor_job_batches.svg)

### 底层原理和关键实现

- 底层实现全部被封装到了DLL中，SHIT!

### 对比
- 对比C#中的Thread类
    - Thread使用的是从系统申请来的现场，而job system是与引擎共享的线程，在没有源码的情况下姑且认为是有不同的。
    - 对于可操作对象的不同，Thread只能操作非引擎的对象，而job system是操作的引擎对象，虽然是受限的。
    - 从管理上也不一样，Thread需要自己控制执行的顺序但是job system的顺序是通过设置后引擎来管理的。

- 对比协程
    - 单线程与多线程的差异
    - 但是协程是可以控制所有的unityengine对象的，例如物理、动画等。这些job system还做不到，只能依赖MonoBehaviour。
    - 因此在大规模计算的数据是NativeContainer或者blittable types的时候在性能上肯定是碾压协程的。

#### 原文
[C# Job System](https://docs.unity3d.com/Manual/JobSystemOverview.html)