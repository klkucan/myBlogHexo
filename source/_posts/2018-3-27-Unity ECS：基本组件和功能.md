---
layout: post
title:  "Unity ECS：基本组件和功能"
date:   2018-3-27 20:07:00 +0800
thumbnail: "img/home-bg-o.jpg"
categories: 
- 游戏开发
tags:
- Unity3D
---

### ECS的组成
#### Entity：
- 近似一个轻量级的GameObject对象。
- 内部没有什么东西，这个和GameObject还不一样，毕竟在GameObject的继承链中具有很多成员和函数。
- 可以添加和移除组件
- 具有一个ID，这个是唯一稳定的。这个ID是entity在被保存时的唯一引用。

<!--more-->

#### IComponentData和ISharedComponentData
- 前文说到过，ECS中的Component是一个结构体，里面只有数据。
- 这个结构体可以继承IComponentData或ISharedComponentData，从源码看这两个接口是空的。
- IComponentData可以理解为不同entity之间不同的数据，而ISharedComponentData代表了相同的数据。其实从后者的名称上也可以看到一些蛛丝马迹。

#### EntityArchetype
- 其实看到archetype（原型）这个词就大致明白这个类的作用了。
- EntityArchetype是一个具有唯一性的ComponentType数组，它可以在创建entity时被作为参数使用，一个entity具有什么能力，完全在于它上面挂了什么Component，而多个component组成了一个能力组，这个就是EntityArchetype了。

```
// Using typeof to create an EntityArchetype from a set of components
EntityArchetype archetype = EntityManager.CreateArchetype(typeof(MyComponentData), typeof(MySharedComponent));

// Same API but slightly more efficient
EntityArchetype archetype = EntityManager.CreateArchetype(ComponentType.Create<MyComponentData>(), ComponentType.Create<MySharedComponent>());

// Create an Entity from an EntityArchetype
var entity = EntityManager.CreateEntity(archetype);

// Implicitly create an EntityArchetype for convenience
var entity = EntityManager.CreateEntity(typeof(MyComponentData), typeof(MySharedComponent));
```

#### EntityManager
- 一个管理所有EntityData、Archetype、SharedComponentData  和ComponentGroup的类。与ComponentSystems是同等地位的，看来unity ECS确实是一套新的系统。
- 从下面的代码中可以看出，实际上所有的Entity的创建、添加组件、查询live状态等操作的API都在这个类里面。应该是核心类。


```
// Create an Entity with no components on it
var entity = EntityManager.CreateEntity();

// Adding a component at runtime
EntityManager.AddComponent(entity, new MyComponentData());

// Get the ComponentData
MyComponentData myData = EntityManager.GetComponentData<MyComponentData>(entity);

// Set the ComponentData
EntityManager.SetComponentData(entity, myData);

// Removing a component at runtime
EntityManager.RemoveComponent<MyComponentData>(entity);

// Does the Entity exist and does it have the component?
bool has = EntityManager.HasComponent<MyComponentData>(entity);

// Is the Entity still alive?
bool has = EntityManager.Exists(entity);

// Instantiate the Entity
var instance = EntityManager.Instantiate(entity);

// Destroy the created instance
EntityManager.DestroyEntity(instance);
```

```
// EntityManager also provides batch APIs
// to create and destroy many Entities in one call. 
// They are significantly faster 
// and should be used where ever possible
// for performance reasons.

// Instantiate 500 Entities and write the resulting Entity IDs to the instances array
var instances = new NativeArray<Entity>(500, Allocator.Temp);
EntityManager.Instantiate(entity, instances);

// Destroy all 500 entities
EntityManager.DestroyEntity(instances);
```

#### 对entity的一个小总结

- EntityManager在生成一个entity时实际使用的是EntityDataManager类中的方法，而EntityDataManager中Entity

- 当一个entity添加或者移除一个component时，实际上改变了其archetype的结构，此时会创建（优先从已有的archetype集合中查询）一个新的archetype来赋值给entity。参考EntityManager源码中相关的代码。


#### Chunk
- chunk是所有ComponentData存储的方式，或者说在内存中的排列方式。
- chunk连接着（或者说对应着）一个EntityArchetype。所有使用同一个EntityArchetype的entity都具有相同的内存布局，在内存中这些entity上的components的布局也比较奇特（原谅我用这个词）。它们（components）在存储时按照类型紧密排列的，。同一类型的component实例在内存中被连续的放在一起，后面是另一个类型的所有component实例。


#### 从内存角度看看archetype和chunk

- 从函数GetOrCreateArchetype可以看到每当新创建一个archetype时本质还是mallco一块内存，但是比较复杂的是一个archetype还包含了很多其它的数据，比如所有ISharedComponentData的类型。
- 但是正如上面提到的，chunk对于着一个archetype，其实chunk不过是一个内存地址。ChunkAllocator中的m_LastChunk维护这一个chunk的链表。
    

#### JobComponentSystem
- 上一篇最后的代码中提到了JobComponentSystem，它主要是用于自动管理job的依赖。一个例子就是在多个job在操作同一个ComponentData时，如果前面几个是在并行的read数据，突然出现一个write数据的job，此时就要等已经执行的read全部结束后暂停其它的job，让write job完成后在执行其它的。这个东西本质上与.NET的Parallel API是一样的，而大多数并行操作都会采用这样的方式，比如一个典型的例子就是SQLit的操作。
- 同时系统内部还使用Injection和ComponentGroup在管理依赖。