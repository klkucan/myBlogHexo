---
layout: post
title:  "开发ECS style项目的思考"
date:   2018-7-20 15:52:19 +0800
thumbnail: "img/home-bg-o.jpg"
categories: 
- 游戏开发
tags:
- Unity3D
---

## 概述
- 这篇文章是近期在学习Entitas时对于ECS的进一步思考
- 其中会包含在使用ECS时需要注意的一些重要原则
- 这些原则在使用Entitas时如何实现
- 这篇文章不会讲Entitas的具体代码和功能，这需要每个使用者自己去看源码和demo。

<!--more-->

## ECS的使用原则

#### 数据驱动一切

- ECS的目的就是将数据与行为进行解耦，而ECS中行为（System）的执行是依赖于数据（Component）的。
- 这个依赖应该分为两个方向来看，第一是变化的数据，它造成了事件的发生；第二是非变化的数据，它提供了Update中逻辑执行的数据。
- 外部的输入（Input、Net、Bluetooth）改变的是数据，这些数据经过System的处理（非必须）后可能被用来驱动View层，或者用来改变其它的数据，从而产生新一轮的System处理数据。


#### ECS只包含数据与逻辑
- ECS是一种开发模式、是一种思维方式，它的实现是不依赖与具体语言、引擎或特定环境的。ECS系统中应该只包含entity、component、system，以及其它辅助的类。
- 这样的认知就要求我们在使用时要将ECS的数据、逻辑与具体实现分离开，请细细品味 `I am abstracting the logic from the implentation, the what from the how. `

#### ECS与外部实现解耦
- ECS与外部实现交互时遵从面向接口编程的原则：从函数调用层面讲不要在ECS系统中直接调用外部对象的函数，因为这必然要求component持有一个具体的对象，导致ECS引用了外部的类定义。此时ECS应该是持有一个接口的实现对象。
- 从数据层面讲，ECS中的类型不依赖于任何第三方的定义。比如表示位置的Vector3，它应该是ECS内部定义的，而不应该依赖于外部定义的类型（比如Unity中的Vector3）。
- 尽量避免component持有对象的引用，哪怕是基于接口的。使用事件机制可以做到更好的解耦。
- 在事件机制的基础上，应该尽量做到上层功能依赖底层功能，反之则不行。也就是说外部实现可以使用ECS的方法、修改component的数据。但是ECS内部可以做到不知道有外部对象这回事，因为在完全使用事件系统的情况下，ECS调用外部函数都可以通过触发事件的方式来调用。


#### 忘却OOP
- 这点对于大家来说比较难，MonoBehaviour自身也是OOP的。但是从和大家的讨论结果来看，忘却继承、多态有助于更好的写出ECS style的代码。

## 如何使用Entitas

### 它山之石可以攻玉

- 下图是一位国外开发者对于使用Entitas，或者说是使用ECS的经验总结。里面用了非常多的代码例子来讲述如何解耦ECS与具体的实现，建议所有使用Entitas的人都看一下，链接在最后。

![image](https://camo.githubusercontent.com/c7565ff2582568e06e529dd81eb0bafbc0ac4224/68747470733a2f2f692e696d6775722e636f6d2f5244576b6150582e706e67)

- 在这幅图中ECS通过interface与View和Service交互，具体的实现可以是unity或者是其它的引擎。
- 里面的箭头也很好定义了一个workflow，service层改变了ECS中component的数据，引发system的处理，最终引发VIEW层的变化。

### Entitas中的System
- 说到System就要提一下Entitas中两种处理数据变化的方式，一种是响应式的，一种是轮询式的。
- 响应式处理需要使用继承自ReactiveSystem的system，虽然在底层它还是会去轮询，但是在使用层面上它只是处理发生了变化的component。
- 而轮询式的则要求system继承自IExecuteSystem，它会在每帧去调用一次Execute方法。
- 对于不同功能的component应该合理的去选择它对应的system，从之前讨论的结果看，绝大多数component都应该是对应的ReactiveSystem。因为响应式会比较高效，只有当系统中entity上的component发生了Add/Remove操作，或者component中的数据发生了变化后才会执行相关的操作。

### 一些使用ECS实现的功能
- 下面会对一些常见的功能如何使用ECS实现来进行讨论，目前为止这些实现只存在于讨论的层面，当然这些实现是基于Entitas的。

#### FSM
- FSM的核心是状态（state）和状态变化所引发的函数调用。
- 以点击跳跃按钮触发玩家播放跳跃动画为例：
    - ECS中添加一个PlayerAnimatorStateEntity，添加PlayerAnimatorStateComponent，里面的变量是一个string类型的state，利用它的改变来驱动玩家身上动画状态机的切换。
    - 在PlayerGameObject创建时需要监听state改变的事件。
    - View层识别到一个按钮的点击（Unity自己的事件机制），ECS中产生一个PlayerJumpEntity，它上面add了一个PlayerJumpComponent。组件中无需任何数据，因为当ECS中出现这个组件时就已经代表jump按钮被按下了。
    - 在这里需要注意，如果点击按钮只是改变动画，那么VIEW里面按钮点击可以直接改PlayerAnimatorStateComponent中state变量的值。
    - PlayerJumpSystem是一个ReactiveSystem，它在执行Execute函数时改变state的值引发事件，从而使得PlayerGameObject可以改变animator的状态。
    
![image](https://raw.githubusercontent.com/klkucan/XMind/master/ECS%E5%AE%9E%E7%8E%B0%E5%8A%9F%E8%83%BD.png)

#### 伤害技能系统
- 未完待续

## 引用
[How I build games with Entitas (FNGGames)](https://github.com/sschmid/Entitas-CSharp/wiki/How-I-build-games-with-Entitas-%28FNGGames%29)