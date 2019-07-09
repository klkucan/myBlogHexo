---
layout: post
title:  "How I build games with Entitas 摘要"
date:   2019-2-13 20:51:00 +0800
thumbnail: /images/home-bg-o.jpg
categories: 
- 游戏开发
tags:
- Unity3D
---

- [这篇文章](https://github.com/sschmid/Entitas-CSharp/wiki/How-I-build-games-with-Entitas-%28FNGGames%29)并不会讲解entitas的功能，作者在文章中主要谈了谈他如何利用entitas开发清晰、稳定和灵活的代码结构。这篇文章可以说是对于ECS的一种实践的总结，同时也是对于Entitas的使用总结。
- 文章会介绍如下的话题：
    - 定义：数据、逻辑和视图层
    - 维护它们彼此的独立
    - 通过接口进行抽象
    - IOC
    - 视图层抽象（视图和视图控制器）

<!--more-->

## 定义
- **数据 Data**：游戏的 ***状态*** ，比如生命、存量、经验等。在entitas中这些数据存在于 ***Component*** 中。
- **逻辑 Logic**：数据变化的规则，在我们的代码中可能就是一些常见的函数，这些函数在改变着数据。在entitas中表现为 ***Systems***.
- **视图 View**：实际上就是游戏状态的表现，比如渲染、动画、音频、UI等。在作者的example中就是 *GameObjects* 上的 *MonoBehaviours*。
- **服务 Service**：ECS之外的功能，常见的就是寻路、物理、社会化分享等，在entitas的视角里面游戏引擎也是一种服务（entitas本质是与引擎无关的，这个需要对源码进行一些扩展，比如Math库就不能使用unity引擎相关的数据定义）。
- **输入Input**：ECS外部的输入，比如键盘、鼠标、网络等。个人认为input对于ECS的影响是通过修改component数据来实现的，也就要求了外部是输入是可以操作部分system的，因为我们修改component是通过system。
 

## 架构

- 任何游戏的核心只是CPU中的数值模拟。每一个游戏都只是数据集合（游戏状态）的周期性变化（游戏逻辑）。*逻辑* 规定了数据变化的规则。
- 纯粹的数据模拟与游戏的却别在于，游戏中有外部的玩家，能够使用逻辑来改变游戏的状态。
- 随之而来的需求是游戏数据与玩家的通信，或者说交互。而这个交互就是**视图层**，表现为渲染actor到屏幕、播放音频、更新UI等。


- 以上是作者对于游戏中数据、逻辑的理解，可谓之醍醐灌顶。而下面的图可以说是对于ECS（Entitas）使用的最佳实践。


![Architecture Diagram](https://raw.githubusercontent.com/klkucan/ImgLib/master/blog/EntitasCode.png?token=AC5fBlT3eFZWrd9UlHLjMabWRaNvEsIPks5cbtqmwA%3D%3D)

## Action中的抽象

- 首先抽象是一个移除**What you want to do**和**How you want to do**之间耦合的过程。如何理解what和how，简单来说what就是interface，how就是implementation。

- 文章中使用了一个常见的例子，就是在开发中的使用log。当然一般来说我们会避免把`Debug.Log`方法在代码的各处使用，弊端显而易见。作者提出这个例子和角色控制器是为了提出一个很重要的思想 ***关注点分离（Separate your concerns）***，这个也是ECS的一个重要的作用。

- 那么entitas是如何实现关注点分离的呢？一般会使用`LogMessageComponent (string message)`和一个 用于处理它的`ReactiveSystem`。代码如下，以后需要修改实现时只要修改这个system就行了。本质就是一个功能交给一个system，和之前定义一个专门负责log的函数或者类是一个概念。entitas中需要输出log只需要创建一个LogEntity就好了。


```csharp
using UnityEngine;
using Entitias; 
using System.Collections.Generic;

// debug message component
[Debug] 
public sealed class DebugLogComponent : IComponent {
    public string message;
}

// reactive system to handle messages
public class HandleDebugLogMessageSystem : ReactiveSystem<DebugEntity> {

    // collector: Debug.Matcher.DebugLog

    // filter: entity.hasDebugLog

    public void Execute (List<DebugEntity> entities) {
        foreach (var e in entities) {
            Debug.Log(e.debugLog.message);
            e.isDestroyed = true;
        }
    }
}
```

- 最后上面代码的弊端是依赖了unity API debug，除此之外如果这里的log功能比较复杂比如json解析、网络发送等时，就会需要更多的依赖甚至关联。如果画出UML可能惨目忍睹。为了解决这个问题作者提出了使用接口。这块其实就没啥特别的了，面向接口编程本身也是OOP里面重要的概念，在上面的图里面，只要不是在虚线框内的代码多数情况是OOP的。而外部代码与ECS代码的交互是基于接口的。


## Interfaces by example

- 在这部分中作者使用大量的例子来展示如何在entitas中使用接口来解耦。对于上面的log功能来说设计出来的接口可能只需要一个方法`LogMessage(string message)`。 代码如下：
- 可以看出`UnityDebugLogService`和`JsonLogService`对应了不同的log需要。

```csharp
// the interface 
public interface ILogService {
    void LogMessage(string message);
}

// a class that implements the interface
using UnityEngine;
public class UnityDebugLogService : ILogService {
    public void LogMessage(string message) {
        Debug.Log(message);
    }
}

// another class that does things differently but still implements the interface
using SomeJsonLib;
public class JsonLogService : ILogService {
    string filepath;
    string filename;
    bool prettyPrint;
    // etc...
    public void LogMessage(string message) {
        // open file
        // parse contents
        // write new contents
        // close file
    }
}
```

- 下面的代码展示了如何通过system的构造函数来注入interface，最后实现了system与interface的关联，但是不与具体实现产生依赖或关联。


```csharp
// the previous reactive system becomes 
public class HandleDebugLogMessageSystem : ReactiveSystem<DebugEntity> {

    ILogService _logService;
    
    // contructor needs a new argument to get a reference to the log service
    public HandleDebugLogMessageSystem(Contexts contexts, ILogService logService) {
         // could be a UnityDebugLogService or a JsonLogService
        _logService = logService; 
    }
    
    // collector: Debug.Matcher.DebugLog
    // filter: entity.hasDebugLog

    public void Execute (List<DebugEntity> entities) {
        foreach (var e in entities) {
            _logService.LogMessage(e.DebugLog.message); // using the interface to call the method
            e.isDestroyed = true;
        }
    }
}
```

- 下面的例子是一个较为复杂的`IInputService`，可以看到是对unity api的一个封装。


```csharp
// interface
public interface IInputService {
    Vector2D leftStick {get;}
    Vector2D rightStick {get;}
    bool action1WasPressed {get;}
    bool action1IsPressed {get;}
    bool action1WasReleased {get;}
    float action1PressedTime {get;}
    // ... and a bunch more
}

// (partial) unity implementation
using UnityEngine;
public class UnityInputService : IInputService {
   // thank god we can hide this ugly unity api in here
   Vector2D leftStick {get {return new Vector2D(Input.GetAxis('horizontal'), Input.GetAxis('Vertical'));} }
   // you must implement ALL properties from the interface
   // ... 
}
```


```csharp
public class EmitInputSystem : IInitalizeSystem, IExecuteSystem {    
    Contexts _contexts;
    IInputService _inputService; 
    InputEntity _inputEntity;
    
    // contructor needs a new argument to get a reference to the log service
    public EmitInputSystem (Contexts contexts, IInputService inputService) {
        _contexts = contexts;
        _inputService= inputService;
    }

    public void Initialize() {
        // use unique flag component to create an entity to store input components        
        _contexts.input.isInputManger = true;
        _inputEntity = _contexts.input.inputEntity;
    }

    // clean simple api, 
    // descriptive, 
    // obvious what it does
    // resistant to change
    // no using statements
    public void Execute () {
        inputEntity.isButtonAInput = _inputService.button1Pressed;
        inputEntity.ReplaceLeftStickInput(_inputService.leftStick);
        // ... lots more queries
    }
}
```



## Inversion of control 控制反转

- 这部分是对上面构造函数的进一步升级，无需在每个system的构造函数中注入服务实例，而是在entitas中使用一个helper类来引用每个服务实例。这个引入的时机最好是应用启动时。下面的例子是完整的过程。

- 可以看到下面是一个关联了很多service的类，通过构造函数注入实例。特别注意一下暴露出的全局变量都是只读的。

*Services.cs*
```csharp
public class Services
{
    public readonly IViewService View;
    public readonly IApplicationService Application;
    public readonly ITimeService Time;
    public readonly IInputService Input;
    public readonly IAiService Ai;
    public readonly IConfigurationService Config;
    public readonly ICameraService Camera;
    public readonly IPhysicsService Physics;

    public Services(IViewService view, IApplicationService application, ITimeService time, IInputService input, IAiService ai, IConfigurationService config, ICameraService camera, IPhysicsService physics)
    {
        View = view;
        Application = application;
        Time = time;
        Input = input;
        Ai = ai;
        Config = config;
        Camera = camera;
        Physics = physics;
    }
}
```

- 实例化Services类

```csharp
var _services = new Services(
    new UnityViewService(), // responsible for creating gameobjects for views
    new UnityApplicationService(), // gives app functionality like .Quit()
    new UnityTimeService(), // gives .deltaTime, .fixedDeltaTime etc
    new InControlInputService(), // provides user input
    // next two are monobehaviours attached to gamecontroller
    GetComponent<UnityAiService>(), // async steering calculations on MB
    GetComponent<UnityConfigurationService>(), // editor accessable global config
    new UnityCameraService(), // camera bounds, zoom, fov, orthsize etc
    new UnityPhysicsService() // raycast, checkcircle, checksphere etc.
);
```

- 下面的代码是一个不错的方法，使用一个特别的context（MetaContext）中的component中包含这些服务实例。注意这个component是单例的。

In my `MetaContext` I have a set of unique components that hold instances of these interfaces. For example: 

```csharp
[Meta, Unique]
public sealed class TimeServiceComponent : IComponent {
    public ITimeService instance;
}
```

- 下面的两段代码演示了如何将services中包含各个实例引入到ecs系统中。
    - 首先上面一步已经创建了一些负责持有这些service实例的component，并且是singleton的。
    - 使用 `Feature`来创建一些IInitializeSystem。`Feature`继承自system，但是从它包含了多个system。这个命名在我看来是极具深意的，因为只有多个system的协作才能形成一个特征（feature）。同时feature在执行顺序上非常靠前会先执行。
    - 在各个IInitializeSystem类中注入service
    - 在system会创建第一步中的component实例，并将service实例赋值进component的成员变量中。


*ServiceRegistrationSystems.cs*
```csharp
public class ServiceRegistrationSystems : Feature
{
    public ServiceRegistrationSystems(Contexts contexts, Services services)
    {
        Add(new RegisterViewServiceSystem(contexts, services.View));
        Add(new RegisterTimeServiceSystem(contexts, services.Time));
        Add(new RegisterApplicationServiceSystem(contexts, services.Application));
        Add(new RegisterInputServiceSystem(contexts, services.Input));
        Add(new RegisterAiServiceSystem(contexts, services.Ai));
        Add(new RegisterConfigurationServiceSystem(contexts, services.Config));
        Add(new RegisterCameraServiceSystem(contexts, services.Camera));
        Add(new RegisterPhysicsServiceSystem(contexts, services.Physics));
        Add(new ServiceRegistrationCompleteSystem(contexts));
    }
}
```

*Example of one of the registration systems*
```csharp
public class RegisterTimeServiceSystem : IInitializeSystem
{
    private readonly MetaContext _metaContext;
    private readonly ITimeService _timeService;

    public RegisterTimeServiceSystem(Contexts contexts, ITimeService timeService)
    {
        _metaContext = contexts.meta;
        _timeService = timeService;
    }

    public void Initialize()
    {
        _metaContext.ReplaceTimeService(_timeService);
    }
}
```
- 下面的代码是entitas中ReplaceComponent的源码，可以看到如果没有某个component对象时会创建一个。

```csharp
/// Replaces an existing component at the specified index
/// or adds it if it doesn't exist yet.
/// The prefered way is to use the
/// generated methods from the code generator.
public void ReplaceComponent(int index, IComponent component) {
    if (!_isEnabled) {
        throw new EntityIsNotEnabledException(
            "Cannot replace component '" +
            _contextInfo.componentNames[index] + "' on " + this + "!"
        );
    }

    if (HasComponent(index)) {
        replaceComponent(index, component);
    } else if (component != null) {
        AddComponent(index, component);
    }
}
```

- 经过上面的处理，在system中使用`_contexts.meta.timeService.instance`就能够获得一个service的实例了。这些实例都是在service的构造函数中创建的，便于管理和替换。
- 至于此处IOC的理解，可以回到上面的那个图。中间虚线框内可以认为是一个ECS framework，我们只向framework注入service实现，但是如何使用是framework内部的事情。这种把控制权交给一个framework的做法就是控制反转。


## View Layer Abstraction 视图层抽象
- 这部分是上面图中的右边部分，描述了如何将entitas系统与表现层结合一起。
- 需要注意的是，GameObject的实例是被一个ViewComponent持有的。这个和service是一样的，也就是说ECS系统之外的对象都可以让一个特别的component持有引用。不过需要通过使用interface，将entitas与引擎API进行了解耦。
- 在这里的View layer用于游戏状态的展示，可以是动画、音频、mesh、渲染器等等。当然也可以是GameObject，或者GameObject上的一个MonoBehaviour对象。

- 下面是这部分功能的代码，看起来比较零散，做了个UML的图可以便于理解。

![视图层抽象uml](https://raw.githubusercontent.com/klkucan/ImgLib/master/blog/%E8%A7%86%E5%9B%BE%E5%B1%82%E6%8A%BD%E8%B1%A1uml.jpg?token=AC5fBuUCXFYdugcg2T_kWbcGM1fJnDkMks5casvGwA%3D%3D)

- UML图中左边是一个用于创建GameObject的service，中间是entity、system等entitas内容，右边是view层的实现。完美符合开始的那张图。


```csharp
public interface IViewController {
    Vector2D Position {get; set;}
    Vector2D Scale {get; set;}
    bool Active {get; set;}
    void InitializeView(Contexts contexts, IEntity Entity);
    void DestroyView();
}
``` 

- IViewController的unity实现:

```csharp
public class UnityGameView : MonoBehaviour, IViewController {

    protected Contexts _contexts;
    protected GameEntity _entity;

    public Vector2D Position {
        get {return transform.position.ToVector2D();} 
        set {transform.position = value.ToVector2();}
    }

    public Vector2D Scale // as above but with tranform.localScale

    public bool Active {get {return gameObject.activeSelf;} set {gameObject.SetActive(value);} }

    public void InitializeView(Contexts contexts, IEntity Entity) {
        _contexts = contexts;
        _entity = (GameEntity)entity;
    }

    public void DestroyView() {
        Object.Destroy(this);
    }
}
```

- 持有viewController的component
```csharp
[Game]
public sealed class ViewComponent : IComponent {
    public IViewController instance;
}
```

- 使用`IViewService`来创建view，并且绑定到entitas

```csharp
public interface IViewService {   
    // create a view from a premade asset (e.g. a prefab)
    IViewController LoadAsset(Contexts contexts, IEntity entity, string assetName);
}
```

- service的unity实现
```csharp
public class UnityViewService : IViewService {
    public IViewController LoadAsset(Contexts contexts, IEntity entity, string assetName) {
        var viewGo = GameObject.Instantiate(Resources.Load<GameObject>("Prefabs/" + assetName));
        if (viewGo == null) return null;
        var viewController = viewGo.GetComponent<IViewController>();
        if (viewController != null) viewController.InitializeView(contexts, entity);
        return viewController;
    }
}
```

- LoadAssetSystem用于加载资源（创建view）和绑定view到component

```csharp
public class LoadAssetSystem : ReactiveSystem<GameEntity>, IInitializeSystem {
    readonly Contexts _contexts;
    readonly IViewService _viewService;

    // collector: GameMatcher.Asset
    // filter: entity.hasAsset && !entity.hasView

    public void Initialize() {    
        // grab the view service instance from the meta context
        _viewService = _contexts.meta.viewService.instance;
    }

    public void Execute(List<GameEntity> entities) {
        foreach (var e in entities) {
            // call the view service to make a new view
            var view = _viewService.LoadAsset(_contexts, e, e.asset.name); 
            if (view != null) e.ReplaceView(view);
        }
    }
}
```
- 在其它的system中使用抽象的view实例。需要注意entitas中并不知道view对象到底是个什么，下面代码能够改变GameObject的position是因为IViewController中的定义。

```csharp
public class SetViewPositionSystem : ReactiveSystem<GameEntity> {
    // collector: GameMatcher.Position;
    // filter: entity.hasPosition && entity.hasView
    public void Execute(List<GameEntity> entities) {
        foreach (var e in entities) {
            e.view.instance.Position = e.position.value;
        }
    }
}
```


**存在的问题**
- 上面的最后一段代码中，当需要修改位置时是直接修改了view的position属性的，这与开始时描述的一个原则相违背。我们希望ECS中执行的就是数据与逻辑，不需要知道这些数据到底是如何表现出来的。但是现在我们在修改数据的同时也必须手动的修改view的position属性。下面的**Event**章节会来解决这个问题。

## Events 事件
- 本质还是使用C#的事件功能
- view MonoBehaviours作为事件监听器

#### 下面是修改后的代码
- 首先需要给要监听的数据（component）添加Event标签，这个标签会在代码生成中生成监听器和事件系统。下面的代码会生成`PositionListenerComponent` 和 `IPositionListener`。文章作者写了个`IEventListener `，用来将监听器与entity绑定起来。

```csharp
// [Game, Event(true)] (Event(true) DEPRECATED as of Entitas 1.6.0) 
[Game, Event(EventTarget.Self)] // generates events that are bound to the entities that raise them
public sealed class PositionComponent : IComponent {
    public Vector2D value;
}
```


```csharp
public interface IEventListener {
    void RegisterListeners(IEntity entity);
}
```

- 不在需要viewcomponent，同时LoadAsset方法不需要在有返回值。然后需要在UnityViewService中添加一些代码来确定和初始化事件监听器。

```csharp
using Entitas;
public interface IViewService {
    // create a view from a premade asset (e.g. a prefab)
    void LoadAsset(
        Contexts contexts,
        IEntity entity,
        string assetName);
}
```


- 更新后的IViewService实现，前面还是生成GameObject，但是后面认为GameObject默认要有个实现了IEventListener的MonoBehaviour（这里单独写一个MonoBehaviour或者直接写在继承了IViewController的那个脚本中都可以），同时将监听器和entity绑定到一起。 

```csharp
using UnityEngine;
using Entitas;

public class UnityViewService : IViewService {
    // now returns void instead of IViewController
    public void LoadAsset(Contexts contexts, IEntity entity, string assetName) {

        //Similar to before, but now we don't return anything. 
        var viewGo = GameObject.Instantiate(Resources.Load<GameObject>("Prefabs/" + assetName));
        if (viewGo != null) {
            var viewController = viewGo.GetComponent<IViewController>();
            if(viewController != null) {
                viewController.InitializeView(contexts, entity);
            }

            // except we add some lines to find and initialize any event listeners
            var eventListeners = viewGo.GetComponents<IEventListener>();
            foreach(var listener in eventListeners) {
                listener.RegisterListeners(entity);
            }
        }
    }
}
```

- 在PositionListener中实现接口。此处可以看出来一个viewGo上可以监听了多个数据变化的事件，每个事件都要绑定一次entity。
- 同时`_entity.AddPositionListener(this);`方法会把这个MonoBehaviour的引用交给_entity的PositionListenerComponent。当数据发送变化时，system会调用OnPosition方法，**这些方法、接口都是添加了`[Event]`标签后自动生成的。**

```csharp
public class PositionListener : MonoBehaviour, IEventListener, IPositionListener {
    
    GameEntity _entity;
 
    public void RegisterEventListeners(IEntity entity) {
        _entity = (GameEntity)entity;
        _entity.AddPositionListener(this);
    }

    public void OnPosition(GameEntity e, Vector2D newPosition) {
        transform.position = newPosition.ToVector2();
    }
}
```

- 如此以来，GameObject的transform的改变就不需要system专门做什么处理了，view的position与ecs中的PositionComponent实现了解耦。



##### 附加一些文中没有贴出来的代码，便于阅读

- 自动生成的PositionListener相关的代码
```csharp
public void AddPositionListener(IPositionListener value) {
    var listeners = hasPositionListener
        ? positionListener.value
        : new System.Collections.Generic.List<IPositionListener>();
    listeners.Add(value);
    ReplacePositionListener(listeners);
}
```

- `AddPositionListener`是在PositionListener中调用的，但是最后它会调用ReplacePositionListener，然后在ECS中创建一个PositionListenerComponent对象。

- 同时生成了一个PositionEventSystem

```csharp
public sealed class PositionEventSystem : Entitas.ReactiveSystem<GameEntity> {

    readonly System.Collections.Generic.List<IPositionListener> _listenerBuffer;

    public PositionEventSystem(Contexts contexts) : base(contexts.game) {
        _listenerBuffer = new System.Collections.Generic.List<IPositionListener>();
    }

    protected override Entitas.ICollector<GameEntity> GetTrigger(Entitas.IContext<GameEntity> context) {
        return Entitas.CollectorContextExtension.CreateCollector(
            context, Entitas.TriggerOnEventMatcherExtension.Added(GameMatcher.Position)
        );
    }

    protected override bool Filter(GameEntity entity) {
        return entity.hasPosition && entity.hasPositionListener;
    }
    

    protected override void Execute(System.Collections.Generic.List<GameEntity> entities) {
        foreach (var e in entities) {
            var component = e.position;
            _listenerBuffer.Clear();
            _listenerBuffer.AddRange(e.positionListener.value);
            foreach (var listener in _listenerBuffer) {
                listener.OnPosition(e, component.value);
            }
        }
    }
}
```

##### 附一个Event的时序图

![PositionEvent](https://raw.githubusercontent.com/klkucan/ImgLib/master/blog/PositionEvent.jpg?token=AC5fBknEmIHZGZpDRTltukY7BOMrjAVQks5cbUxMwA%3D%3D)

## 总结
- entitas配合面向接口编程&Event，实现了数据+逻辑与表现层的解耦，同时解耦ECS与引擎。
- 需要注意的是，最开始的图是一个横向的，如果纵向的来看，Entitas是在底层，service和view是在上面的。也就是说后面两部分是可以直接持有entitas中的对象的，典型的就是Context、Entity等对象。而entitas则是通过interface、event等手段来操作service和view。
- 当然，较真来说，entitas其实也是持有了后两者的对象。这个目前看来是无法避免的。