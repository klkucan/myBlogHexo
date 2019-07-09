---
layout: post
title:  "Unity ECS：概述"
date:   2018-3-22 1:07:00 +0800
thumbnail: "img/home-bg-o.jpg"
categories: 
- 游戏开发
tags:
- Unity3D
---

#### 前言

- 看来unity已经认识到现在编写代码的一些问题了：OO的模型、mono所编译的糟糕的机器码、GC和单线程。emmm
- ECS的推出就是为了解决上面的问题，同时使用ECS是为了能够利用C# Job System和Burst compiler。job system是支持多线程的（( Ĭ ^ Ĭ )）

<!--more-->

#### ComponentSystem

- 在后面的文章中会说到ECS中的Component，这里简单说下，新的component模型只包含了数据，而ComponentSystem包含了行为。
- 一个ComponentSystem负责在每帧中对物体进行操作，这些物体必须匹配ComponentSystem所定义的一组component。有点类似后面讲到的的EntityArchetype。看下代码的例子：

```
class Rotator : MonoBehaviour
{
    // The data - editable in the inspector
    public float Speed;
}

class RotatorSystem : ComponentSystem
{
    struct Group
    {
        // Define what components are required for this 
        // ComponentSystem to handle them.
        Transform Transform;
        Rotator   Rotator;
    }
    
    override protected OnUpdate()
    {
        // We can immediately see a first optimization.
        // We know delta time is the same between all rotators,
        // so we can simply keep it in a local variable 
        // to get better performance.
        float deltaTime = Time.deltaTime;
        
        // ComponentSystem.GetEntities<Group> 
        // lets us efficiently iterate over all GameObjects
        // that have both a Transform & Rotator component 
        // (as defined above in Group struct).
        foreach (var e in GetEntities<Group>())
        {
            e.Transform.rotation *= Quaternion.AxisAngle(e.Rotator.Speed * deltaTime, Vector3.up);
        }
    }
}
```

- Rotator是现在系统中的一个MonoBehaviour，而RotatorSystem是一个ComponentSystem。后者包含了一个Group用于匹配GameObject，而OnUpdate方法则是处理行为的函数。

#### 如何在现有的系统中使用ComponentSystem

- 目前是需要在每个GameObject上挂一个GameObjectEntity脚本，它在OnEnable方法中会创建一个entity，挂上GameObject上的所有组件。然后就能被ComponentSystem使用了。
- 这意味着你可以将所有的行为处理从MonoBehaviour.Updata中转移到 ComponentSystem.OnUpdate中，数据仍旧可以保持在MonoBehaviour中。

- 这样做的优势    
    - 数据（MonoBehaviour）与行为（ComponentSystem）的分离
    - 一些对象上的系统操作是批量的（应该是基于Job System），便于优化代码。比如上面代码中的deltaTime的使用。
    - 可以继续使用现存的inspector、editor工具等。

- 劣势
    - 实例化时间没有改善
    - 加载的时间没有改善
    - 数据时随机访问的，非线性。线性数据访问的问题下面会说到。
    - 非多核的
    - 非[SIMD](https://en.wikipedia.org/wiki/SIMD)

- 总结来说这样的混合方案性能提升有限

#### Pure ECS

- ECS不支持托管类型，支持struct和NativeContainer类型。所以只有IComponentData可以被C# Job安全的访问。
- EntityManager（后面会提到）保证了线性的内存布局 ，这是性能提升很重要的部分。通过job和IComponentData可以做到。
- 目前想将ComponentData添加到GameObject上需要使用ComponentDataWrapper

```
// The rotation speed component simply stores the Speed value
[Serializable]
public struct RotationSpeed : IComponentData
{
    public float Value;
}

// This wrapper component is currently necessary to add ComponentData to GameObjects.
// In the future we want to make this wrapper component automatic.
public class RotationSpeedComponent : ComponentDataWrapper<RotationSpeed> { } 
```

```
// Using IJobProcessComponentData to iterate over all entities matching the required component types.
// Processing of entities happens in parallel. The main thread only schedules jobs.
public class RotationSpeedSystem : JobComponentSystem
{
    // IJobProcessComponentData is a simple way of iterating over all entities given the set of required compoenent types.
    // It is also more efficient than IJobParallelFor and more convenient.
    [ComputeJobOptimization]
    struct RotationSpeedRotation : IJobProcessComponentData<Rotation, RotationSpeed>
    {
        public float dt;

        public void Execute(ref Rotation rotation, [ReadOnly]ref RotationSpeed speed)
        {
            rotation.Value = math.mul(math.normalize(rotation.Value), math.axisAngle(math.up(), speed.Value * dt));
        }
    }

    // We derive from JobComponentSystem, as a result the system proviides us 
    // the required dependencies for our jobs automatically.
    //
    // IJobProcessComponentData declares that it will read RotationSpeed and write to Rotation.
    //
    // Because it is declared the JobComponentSystem can give us a Job dependency, which contains all previously scheduled
    // jobs that write to any Rotation or RotationSpeed.
    // We also have to return the dependency so that any job we schedule 
    // will get registered against the types for the next System that might run.
    // This approach means:
    // * No waiting on main thread, just scheduling jobs with dependencies (Jobs only start when dependencies have completed)
    // * Dependencies are figured out automatically for us, so we can write modular multithreaded code
    protected override JobHandle OnUpdate(JobHandle inputDeps)
    {
        var job = new RotationSpeedRotation() { dt = Time.deltaTime };
        return job.Schedule(this, 64, inputDeps);
    } 
}
```

- 这段代码和ComponentSystem的部分有类似的结构，但是它结合了JobSystem，做到了并发执行。而且里面还有依赖注入的部分，下一篇在将ECS功能的时候会详细来说。


#### 总结
- Unity ECS做到了数据与行为的分离，而且更加轻量级。
- 通过结合JobSystem实现了CPU多核的利用。
- 目前来看在已有系统结构上使用ECS有点累赘的感觉。


