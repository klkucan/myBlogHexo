---
layout: post
title:  "LuaFramework&PureMVC"
date:   2017-10-16 9:54:07 +0800
thumbnail: "img/home-bg-o.jpg"
categories: 
- 编程语言
tags:
- Lua
---

#### [LuaFramework](https://github.com/jarjin/LuaFramework_UGUI)是GitHub上一个基于tolua的人更新案例，里面除了tolua的功能外还使用了PureMVC的部分功能。这个文章是对代码中PureMVC部分的一些理解。

## PureMVC 
### Structure
#### Model&Proxy
- Model保存对Proxy对象的引用，Proxy负责操作数据模型，与远程服务通信存取数据。这样保证了Model层的可移植性。

<!--more-->

#### View&Mediator
- View保存对Mediator对象的引用。
- 由Mediator对象来操作具体的视图组件（View Component，例如Flex的DataGrid组件），包括：添加事件监听器，发送或接收Notification ，直接改变视图组件的状态。
- 这样做实现了把视图和控制它的逻辑分离开来。
- 在LuaFramework的初始代码中只有个APPView，在其中的


#### Controller&Command
- Controller保存所有Command的映射。Command类是无状态的，只在需要时才被创建。
- Command可以获取Proxy对象并与之交互，发送Notification，执行其他的Command。经常用于复杂的或系统范围的操作，如应用程序的“启动”和“关闭”。应用程序的业务逻辑应该在这里实现。
- Controller会注册侦听每一个Notification，当被通知到时，Controller会实例化一个该Notification对应的Command类的对象。最后，将Notification作为参数传递给execute方法。

#### Facade
- Façade类应用单例模式，它负责初始化核心层（Model，View和Controller），并能访问它们的Public方法。
- 在实际的应用中，只需继承Façade类创建一个具体的Façade类就可以实现整个MVC模式，并不需要在代码中导入编写Model，View和Controller类。
- Proxy、Mediator和Command就可以通过创建的Façade类来相互访问通信。
- Facade保存了Command与Notification之间的映射。当Notification（通知）被发出时，对应的Command（命令）就会自动地由Controller执行。Command实现复杂的交互，降低View和Model之间的耦合性。

#### Observer&Notification
- 使用一个简单的观察者模式实现。

## PureMVC in LuaFramework
##### NOTE：在LuaFramework的初始代码中是没有model、proxy、mediator这些内容的。
#### View
- view部分也只存在一个AppView对象，它监听了一个MessageList，如果需要监听自己的事件可以在messagelist继续添加。从目前情况看，一个AppView基本可以满足所有UI事件的需要。
- 如果需要多个view的时候，在每个view中可以通过RegisterMessage方法来注册多个事件。
- view的父类Base负责view的notification和command的映射。

#### Controller
- LuaFramework中Controller是一个singleton的，在一次性command的基础上加入了一个m_viewCmdMap用于处理由view负责处理的事件。
- 上面提到PureMvc中Command类是无状态的，只在需要时才被创建。所以在command的注册上LuaFramework采用的是名字+类型，因为有了类型就可以动态生成对象。
- 具体来说在LuaFramework的初始代码中注册了socketcommand，当facade send某个message时会调用这个command，采用的就是生成一个socketcommand对象，然后执行它的execute方法。这个方法调用了lua的network对象的OnSocket方法。
- 在command的注册上分为一次性的和view的
- 在command的执行上，在Controller的ExecuteCommand方法中可以看到，先从m_commandMap中查询是否有类型，如果有就实例化并执行execute方法。如果没有就在m_viewCmdMa中找，所有订阅了特定消息的view都会被执行OnMessage方法。

#### Facade
- AppFacade是其子对象。
- 在LuaFramework中facade不但负责将notification和command关联，而且负责管理Manager实例。本质一个字典，如果add的时候没有动静就是
- facade父类负责将一次性的notification和command关联起来。

#### Observer&Notification
- m_viewCmdMap是多播，而m_commandMap是单播。

## 总结
- 基本上在LuaFramework中的PureMvc只是使用了一部分功能，主要功能在于消息的注册发布和manager的管理。
- 在使用过程中，因为这个框架本身是要使用lua的，所以在源码中lua的部分在解耦上主要依赖event库来实现订阅发布。在C#部分反而不是很能体现MVC架构的优势。
- 在我自己的项目中为了最大化的使用热更新，所以很多逻辑都是写在lua中的，其实可以考虑在lua中实现一套PureMVC，但是这个语言在实现上应该不简单。
