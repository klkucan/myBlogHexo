---
layout: post
title:  "浅析Timeline结构"
date:   2017-11-2 19:59:13 +0800
thumbnail: "img/home-bg-o.jpg"
categories: 
- 游戏开发
tags:
- Unity3D
---

## 结构

<table>
   <tr>
      <td>PlayableDirector</td>
      <td colspan="6" align="center">Timeline</td>
   </tr>
   <tr>
      <td>PlayableGraph</td>
      <td colspan="6" align="center">Graph</td>
   </tr>
   <tr>
      <td>…</td>
      <td colspan="4" align="center">TrackGroup</td>
      <td>TrackGroup</td>
   </tr>
   <tr>
      <td>TrackAsset</td>
      <td>AnimationTrack</td>
      <td>PlayableTrack</td>
      <td>CinemachineTrack</td>
      <td>CustomTrack</td>
      <td>…</td>
   </tr>
   <tr>
      <td>…</td>
      <td>TimelineClips</td>
      <td>TimelineClips</td>
      <td>TimelineClips</td>
      <td>TimelineClips</td>
      <td>…</td>
   </tr>
   <tr>
      <td>PlayableAsset</td>
      <td>AnimationClip (Motion)</td>
      <td>CustomPlayableAsset</td>
      <td>CinemachineShot</td>
      <td>CustomClip</td>
      <td>…</td>
   </tr>
   <tr>
      <td>PlayableBehaviour</td>
      <td>PlayableBehaviour</td>
      <td>PlayableBehaviour</td>
      <td>CinemachineShotPlayable</td>
      <td>PlayableBehaviour</td>
      <td>…</td>
   </tr>
</table>

<!--more-->

- 从表中可以看出是一个树结构，在Timeline Editor中也是一目了然。核心的类就是最左边一列中列出的。

## 组件
### Timeline
##### Timeline & PlayableDirector的关系
- 本质是个PlayableDirector，作用顾名思义。

##### PlayableGraph & PlayableDirector的关系
- 从测试情况看，一个PlayableDirector对应一个PlayableGraph
- 通过`PlayableDirector pd = graph.GetResolver() as PlayableDirector;`可以反向得到PlayableDirector，在PlayableAsset中可能会比较有用。

### TrackAsset
- TrackAsset从视觉上看就是Timeline Editor中左边preview中的一项，每个track可以约束它所影响的 GameObject的类型，也可以设置它上面clip的类型。
- **从实际的操作上来看，PlayableTrack上可以放置任何继承自PlayableAsset的clip，但是其它的track上就必须放置约定类型的clip了。**

##### 如何自定义一个Track

```
[TrackColor(1f, 1f, 0f)]
[TrackClipType(typeof(LightControlClip))]
[TrackBindingType(typeof(Light))]
public class LightControlTrack : TrackAsset {}
```

- 从上面的代码看TrackBindingType定义了绑定的类型，TrackClipType定义了clip的类型。而TrackColor是定义了在Timeline Editor中最左边的那条很细的彩色线条。
- 经过测试，在自定义的TrackAsset中可以不实现CreateTrackMixer方法，但是如果要去override它，代码中不能用`return Playable.Create(graph);`,而是用`return ScriptPlayable<CustomMixer>.Create(graph, inputCount);`哪怕这个CustomMixer是个空类都可以，很坑爹。

### PlayableAsset（Clip）

- 要说PlayableAsset就离不开PlayableBehaviour。在旧版本的Timeline中BasePlayableBehaviour实现了PlayableAsset和PlayableBehaviour的功能。但是因为已经被舍弃了，因此想实现既可以放到track上，又能监控状态的对象需要用PlayableAsset配合PlayableBehaviour。做法为在自定义的PlayableAsset中写：

```
public override Playable CreatePlayable(PlayableGraph graph, GameObject go)
{
    return ScriptPlayable<CustomBehaviour>.Create(graph, template);
}
```

### PlayableBehaviour
- 核心类，其定义了事件函数涵盖了自身的状态变化、graph的状态变化和PlayableDirector创建销毁时触发的事件。
- PrepareFrame函数可以在每一帧对timeline中的元素进行访问和设置。可以说是在做自定义blend中不可缺失的功能。

## 执行顺序
- 目前来看应该是：

**CreateTrackMixer->CreatePlayable->OnPlayableCreate->OnGraphStart->OnBehaviourPause->OnBehaviourPlay->OnGraphStop->OnPlayableDestroy**

- 这个顺序是一个clip的。当一个track上有多个clip时OnPlayableCreate、OnGraphStart、OnBehaviourPause会无序的出现，但是一定是在OnBehaviourPlay之前。
- 在真正的clip的执行期间，OnBehaviourPlay和OnBehaviourPause是按顺序执行的。
- 基本上可以判断当一个PlayableBehaviour准备好后，会先被pause。然后按照设计好的顺序执行，当开始执行时触发play，结束后再次触发pause。所以如果是要在一个clip结束后处理什么事情需要做一个判断，是否是第一次触发pause。


## 组件之间互相获取的方法
##### Timeline(PlayableDirector)的获取方式
- 通过脚本中设置PlayableDirector类型变量获得。
- 如果是自定义的TrackAsset，则通过CreateTrackMixer方法的go参数获得。

```
public override Playable CreateTrackMixer(PlayableGraph graph, GameObject go, int inputCount)
{
    PlayableDirector playableDirector = go.GetComponent<PlayableDirector>();
}
```
- 如果是自定义的PlayableAsset，则通过

```
public override Playable CreatePlayable(PlayableGraph graph, GameObject go)
{
    var pd = graph.GetResolver() as PlayableDirector;
}        
```

- PlayableBehaviour脚本中函数都是有playable参数，通过这个参数也可以获得

```
public override void OnBehaviourPlay(Playable playable, FrameData info)
{
    PlayableDirector pd = playable.GetGraph<Playable>().GetResolver() as PlayableDirector;
}
```
##### Timeline中获取track、clip和PlayerBehaviour
- 从上面的代码可以看到，获取director的过程比较符合最上面的那个表里的层级关系。但是从timeline获取其它的元素会比较困难。或者说比较不符合这个层级关系。我个人认为在API的设计上是有问题的。
- 那么要如何通过timeline获取这些元素呢？

```
public PlayableDirector pd;
// Use this for initialization
void Start()
{
    var binding = pd.playableAsset.outputs;
    foreach (var item in binding)
    {
        switch (item.sourceObject.GetType().Name)
        {
            case "AnimationTrack":
                {
                    var at = item.sourceObject as AnimationTrack;
                    foreach (TimelineClip clip in at.GetClips())
                    {
                        Debug.Log(clip.animationClip.name + "\n");
                    }
                }
                break;
            case "CinemachineTrack":
                var item2 = item.sourceObject as CinemachineTrack;
                foreach (TimelineClip clip in item2.GetClips())
                {
                    CinemachineShot cs = clip.asset as CinemachineShot;
                    Debug.Log(cs.VirtualCamera.Resolve(pd) + "\n");
                }
                break;
            case "PlayableTrack":
                var pt = item.sourceObject as PlayableTrack;
                foreach (TimelineClip clip in pt.GetClips())
                {
                    NewPlayableAsset cs = clip.asset as NewPlayableAsset;
                }
                break;
            case "CustomTrack":
                var ct = item.sourceObject as CustomTrack;
                foreach (TimelineClip clip in ct.GetClips())
                {
                    CustomClip cs = clip.asset as CustomClip;
                    CustomBehaviour cb = cs.template;
                    cb.Foo();
                }
                break;
            default:
                break;
        }
    }
}
```
- 从代码上看`pd.playableAsset.outputs`获得了一个`IEnumerable<PlayableBinding>`类型的集合。这个设计让人非常费解。然后遍历集合，得到`item.sourceObject`，这个对象就是track了。然后可以根据不同的类型转换成不同的Track。然后就和表中的结构一致了，获得clip->playableasset->playablebehaviour。

##### 获得场景对象
- Timeline中获取对象在API设计上也不合理，在unity的通用做法是在代码中定义一个Public变量或者使用[SerializeField]标记一个private的变量，然后拖拽。又或者用过GameObject.Find来获取。但是对于策划和美术来说最多的还是拖拽。
- Timeline相关的脚本中可以继续使用GameObject.Find。但是如果你想用拖拽的形式需要这样定义对象：

```
public ExposedReference<T> object;
```
- 而当你想真正使用这个变量的值的时候需要用

```
object.Resolve (graph.GetResolver());
```

- 需要说明的是在Timeline的那些脚本类里面是可以用public定义变量的，在inspector面板上也可以显示出来，但是退拽无效。

##### 获取Track绑定的对象
- 目前来看似乎只有用GameObject.Fine或者直接用属性值了。

