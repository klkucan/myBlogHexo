---
layout: post
title:  "Unity ECS：C# Job System"
date:   2018-3-23 16:07:00 +0800
thumbnail: "img/home-bg-o.jpg"
categories: 
- 游戏开发
tags:
- Unity3D
---

### Job system 
#### Job system解决了什么问题
- unity是支持多线程的，虽然有些缺陷。但是我们在代码里面编写大量的线程，即使使用thread pool也无法避免上下文切换的问题。而Job System本质上还是线程(wrok thread)，只不过它的工作线程的数量和CPU逻辑上的核数量一致，这样就避免了上下文切换。

<!--more-->

#### Job System如何工作
- 在这个系统中真正发挥作用的是job单元，它类似方法的调用，包含了参数和数据。job被填入Job Queue中，然后work thread负责调用它。
- job system中有dependency（依赖）的概念，如果job A依赖于job B，那么系统会保证B在A执行前完成。
- job system不是现存的任何C#线程模型中的一种。
- 它被集成在引擎内部，这意味着我们开发者所编写的代码会让unity引擎共享work thread，这样才能避免对于CPU的竟态使用。
- PS: 我很怀疑是否代码写的很糟糕的情况下，会影响引擎本身的效率。


#### Job System如何避免竟态
- 我觉得文档想描述的很大程度上是数据原子性的问题
- job system会检查所有潜在的竟态。
- 让所有的job操作同一份数据拷贝
- job只能访问blittable data(这种数据在托管和非托管中具有相同的内存结构)，而不是managed types。
- 鉴于上面提到的数据拷贝的就行，为了解决现实世界复杂的问题，提供了NativeContainers：
    - 类型包括：NativeArray, NativeList, NativeHashMap, and NativeQueue.
    - 这些类型能够被unity追踪，谁在读写它们。比如两个job同时写入时需要使用Schedule来安排执行顺序，安全系统会抛出一个明确的异常来说明这些。
    - 多个job同时读取一个数据时是并发的
    - 读写的限制同样适用于main thread
    - 一些container有特殊的规则，以便适应来自`ParallelFor jobs`的安全、明确的写入访问。比如`NativeHashMap.Concurrent`允许使用`IJobParallelFor`并发的添加item。

#### 调度job
- job system依赖blittable data和NativeContainer
- 实现步骤：
    - 定一个实现了IJob接口的struct
    - 创建struct对象，填充数据，调用Schedule方法。
    - 调用方法后得到一个job handle对象。它可以做为其它job的依赖对象；也可以等待它完成工作。如果说需要在主线程访问传入job的NativeContainer，那就等待这个handle完成。这部分具体看代码比较好理解。
    - 需要注意的是在主线程想要访问NativeContainers数据之前，需要让所有的 job依赖都完成，此时只用JobHandle.IsDone来判断是不够的，需要手动调用 JobHandle.Complete方法。Complete方法还会清空jobs debugger中的状态。
    - 如果每一帧都调用一个新的job，这个job还依赖于上一帧的job的话，会出现内存泄漏。

- 从下面的代码可以看出，一个job就是一个实现了IJob的struct。
    
- 定义struct：

``` 
// Job adding two floating point values together
public struct MyJob : IJob
{
    public float a;
    public float b;
    NativeArray<float> result;
    public void Execute()
    {
        result[0] = a + b;
    }
}

public struct AddOneJob : IJob
{
    public NativeArray<float> result;
    public void Execute()
    {
        result[0] = result[0] + 1;
    }
}

```

- 创建单个job实例，进行调用：

```
// Create a native array of a single float to store the result in. This example will wait for the job to complete, which means we can use Allocator.Temp
NativeArray<float> result = new NativeArray<float>(1, Allocator.Temp);
// Setup the job data
MyJob jobData = new MyJob();
jobData.a = 10;
jobData.b = 10;
jobData.result = result;
// Schedule the job
JobHandle handle = jobData.Schedule();
// Wait for the job to complete
handle.Complete();
// All copies of the NativeArray point to the same memory, we can access the result in "our" copy of the NativeArray
float aPlusB = result[0];
// Free the memory allocated by the result array
result.Dispose();
```

- 使用schedule调用多个job

```
NativeArray<float> result = new NativeArray<float>(1, Allocator.Temp);
// Setup the job data
MyJob jobData = new MyJob();
jobData.a = 10;
jobData.b = 10;
jobData.result = result;
// Schedule the job
JobHandle firstHandle = jobData.Schedule();
AddOneJob incJobData = new AddOneJob();
incJobData.result = result;
JobHandle handle = incJobData.Schedule(firstHandle);
// Wait for the job to complete
handle.Complete();
// All copies of the NativeArray point to the same memory, we can access the result in "our" copy of the NativeArray
float aPlusB = result[0];
// Free the memory allocated by the result array
result.Dispose();
```

#### ParallelFor jobs
- ParallelForJob适用于对于多个物体的并发操作，比如集合中的item。
- 并不是每个item对应一个job，而是每个CPU CORE对应一个job。这个也是上文说的为了保证减少上下文切换。
- 在使用ParallelForJob时需要传递两个参数：
    - 数组的长度，这个数组应该是你要用来迭代的数组。
    - PS：实际上这部分文档上写的很奇怪，通过长度来区分数组？
    - 一次批处理的个数，这个决定了job的个数。建议一开始使用的一个job然后慢慢的增加。直到性能提升的不明显了。
    - PS：其实在并行问题上，线程越多未必能够效率越高。以为还存在一个锁的问题。

```
// Job adding two floating point values together
public struct MyParallelJob : IJobParallelFor
{
    [ReadOnly]
    public NativeArray<float> a;
    [Readonly]
    public NativeArray<float> b;
    public NativeArray<float> result;
    public void Execute(int i)
    {
        result[i] = a[i] + b[i];
    }
}
```

```
var jobData = new MyParallelJob();
jobData.a = 10;  
jobData.b = 10;
jobData.result = result;
// Schedule the job with one Execute per index in the results array and only 1 item per processing batch
JobHandle handle = jobData.Schedule(result.Length, 1);
// Wait for the job to complete
handle.Complete();
```


#### Job System还存在的问题
- 无法访问static数据，这个功能后续应该会有。

### 参考
- [Blittable and Non-Blittable Types](//docs.microsoft.com/en-us/dotnet/framework/interop/blittable-and-non-blittable-types)