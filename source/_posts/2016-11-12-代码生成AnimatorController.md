---
layout: post
title:  "代码生成AnimatorController"
date:   2016-11-12 17:18:16 +0800
thumbnail: "img/home-bg-o.jpg"
categories: 
- 游戏开发
tags:
- Unity3D
---

### 0.出发点
- 现在的项目需要设置多套动画组合，全部是由策划在XML文件中设置完成，如果完全的手动在AnimatorController中去做不但工作量大而且如果将来有配置修改了还要一个个去找到对应的自状态机并且修改。因此就萌生了用代码去生成状态机的想法，而且在网上也有了很多的教程可以参考，只是每个项目都不同，且对于一些参数和属性的设置也不尽相同，因此还是把自己的代码进行一些修改后分享出来，基本上应该是包含了状态机常用的功能。

- 需要注意我的具体代码中是在一个已有的AnimatorController基础上创建的。如果完全是从0开始可以参考别的资料，其实道理是一样的都是代码创建对象。
  
### 1.数据来源

一个典型的XML文件

``` 
<?xml version="1.0" encoding="ISO-8859-1"?>
<config>
	<datas>
		<data INDEX="1" Clip1="jump" Clip1Count="1" Clip2="BackLeap" Clip2Count="2" Clip3="die" Clip3Count="1"></data>
		<data INDEX="2" Clip1="BackLeap" Clip1Count="3" Clip2="jump" Clip2Count="0" Clip3="jump" Clip3Count="0"></data>
		<data INDEX="3" Clip1="BackLeap" Clip1Count="1" Clip2="ForwardLeap" Clip2Count="1" Clip3="jump" Clip3Count="1"></data>
	</datas>
</config>
```

<!--more-->

### 2.动画控制器中的主要元素  
- Unity中editor的功能十分的强大，能够加载项目中的各种资源，而AnimatorController就是其中之一。
   
- 一个AnimatorController的结构基本如下
![](http://imglf3.nosdn.127.net/img/Nld0N0tacnNuUG1RQlNwUzB6eW1PQm01cUplRzFJRXFRbHRCVWtFMmNrN1VYVFVnTk5qa3RRPT0.jpg?imageView&thumbnail=500x0&quality=96&stripmeta=0&type=jpg)

- AnimatorControllerLayer：一个AnimatorController由多个Layer组成，但是除了BaseLayer外其它的Layer并不主要负责动画逻辑，而是多用于动画遮罩。

- AnimatorControllerParameter：顾名思义是状态机中使用的参数，这个参数可以在不同的Layer和子状态机中使用。在代码添加参数时会选择参数类型，它是个枚举`AnimatorControllerParameterType`。

- AnimatorStateMachine：动画状态机，核心逻辑实线层。在一个状态机中可以有多个state，也可以有多个Sub AnimatorStateMachine。通过AddStateMachine方法来生成并添加子状态机。

- AnimatorState：动画状态，也是这个系统中的基础单元。其可以设定各种属性，比较常用的是AnimationClip和Speed等。

- AnimatorStateTransition：也就是动画转换，其中可以设定触发参数，而且其中还有一个很重要的东西就是动画过度的设定。

### 3.完整代码

```
using UnityEngine;
using System.Collections;
using UnityEditor;
using System;
using UnityEditor.Animations;
using System.IO;
using System.Xml;
using System.Text.RegularExpressions;
using System.Xml.Serialization;
using System.Collections.Generic;
using System.Linq;

//[CustomEditor(typeof(EditorTools))]
public class EditorTools : MonoBehaviour
{
	#region 创建动画控制器

	/// <summary>
	///  记录上一个state，用于自状态机中
	/// </summary>
	static AnimatorState lastAnimatorState = null;
	static string ParameterName;
	// 动画片段
	static AnimationClip die;
	static AnimationClip jump;
	static AnimationClip BackLeap;
	static AnimationClip ForwardLeap;
	/// <summary>
	/// base layer AnimatorStateMachine
	/// </summary>
	static AnimatorStateMachine mainASM;
	static int stateHeight = 100;
	static int stateWidth = 220;

	/// <summary>
	/// 根据配置文件创建特技组
	/// </summary>
	[MenuItem ("Tools/CreateAnimatorState")]
	static void CreateAnimatorState ()
	{
		// 获取动画片段
		List<object> allAssets = new List<object> (AssetDatabase.LoadAllAssetsAtPath ("Assets/Charactors/player2.FBX"));
		var animationClips = allAssets.Where (o => o.GetType () == typeof(AnimationClip)).ToList ();
		foreach (var item in animationClips) {
			AnimationClip x = item as AnimationClip;
			switch (x.name) {
			case "die":
				die = x;
				break;
			case "jump":
				jump = x;
				break;
			case "BackLeap":
				BackLeap = x;
				break;
			case "ForwardLeap":
				ForwardLeap = x;
				break;
			default:
				break;
			}
		}

		// 当每个动画是一个单独的FBX文件中时可以用下面的方法来获取
		//die = AssetDatabase.LoadAssetAtPath ("Assets/Charactors/player2.FBX", typeof(AnimationClip)) as AnimationClip;


		// 获取状态机
		AnimatorController animatorController = AssetDatabase.LoadAssetAtPath ("Assets/AnimatorController/demo.controller", typeof(AnimatorController)) as AnimatorController;
		AnimatorControllerLayer layer = animatorController.layers [0];
		mainASM = layer.stateMachine;

		// 获取当前所有的参数
		AnimatorControllerParameter[] paras = animatorController.parameters;
		List<AnimatorControllerParameter> listParas = new List<AnimatorControllerParameter> (paras);

		// 删除指定的参数
		var acps = listParas.Where (p => p.name.Contains ("GroupParameter")).ToArray ();
		foreach (AnimatorControllerParameter item in acps) {
			animatorController.RemoveParameter (item);
		}

		// 删除指定的子状态机
		ChildAnimatorStateMachine[] childASM = mainASM.stateMachines;
		List<ChildAnimatorStateMachine> listCASM = new List<ChildAnimatorStateMachine> (childASM);
		var casms = listCASM.Where (c => c.stateMachine.name.Contains ("Group")).ToArray ();
		foreach (ChildAnimatorStateMachine item in casms) {
			mainASM.RemoveStateMachine (item.stateMachine);
		}

		// 读配置文件
		XmlConfig xc = ReadXml ();

		Vector3 startPos = mainASM.anyStatePosition;


		// 根据配置创建自状态机
		for (int index = 0; index < xc.datas.Count; index++) {
			Data data = xc.datas [index];

			// 设置特技参数，
			ParameterName = "GroupParameter" + data.INDEX.ToString ();
			animatorController.AddParameter (ParameterName, AnimatorControllerParameterType.Trigger);

			// 创建子状态机
			AnimatorStateMachine sub = AddSubStateMachine<AnimatorEvent> ("Group_" + data.INDEX, ParameterName, mainASM, startPos + new Vector3 (stateWidth * index, -stateHeight, 0));
			// 创建子状态机中的state
			SetStateInSubMachine (sub, data);
			lastAnimatorState = null;
		}
	}

	/// <summary>
	///  创建sub state machine用于放置特效组中的动画
	/// </summary>
	/// <typeparam name="T"></typeparam>
	/// <param name="stateName"></param>
	/// <param name="sm"></param>
	/// <param name="position"></param>
	/// <param name="data"></param>
	/// <returns></returns>
	private static AnimatorStateMachine AddSubStateMachine<T> (string stateName, string para, AnimatorStateMachine sm, Vector3 position) where T : StateMachineBehaviour
	{
		AnimatorStateMachine sub = sm.AddStateMachine (stateName, position);
		sub.AddStateMachineBehaviour<T> ();
		AnimatorStateTransition transition = mainASM.defaultState.AddTransition (sub, false);
		transition.AddCondition (AnimatorConditionMode.If, 0, para);
		return sub;
	}

	/// <summary>
	///  根据配置数据在子状态机中创建state
	/// </summary>
	/// <typeparam name="T"></typeparam>
	/// <param name="subSM"></param>
	/// <param name="data"></param>
	private static void SetStateInSubMachine (AnimatorStateMachine subSM, Data data)
	{
		AnimatorState newState;
		string stateName;
		Vector3 pos;
		List<AnimationClip> acArray = new List<AnimationClip> ();
		SetAnimationClip (data.Clip1, data.Clip1Count, ref acArray);
		SetAnimationClip (data.Clip2, data.Clip2Count, ref acArray);
		SetAnimationClip (data.Clip3, data.Clip3Count, ref acArray);

		for (int x = 1; x <= acArray.Count; x++) {
			stateName = "GroupState_" + data.INDEX + "_" + x.ToString ();
			pos = subSM.entryPosition + new Vector3 (stateWidth, -stateHeight * x, 0);
			newState = AddState (stateName, subSM, pos, acArray [x - 1], x, acArray.Count);
			lastAnimatorState = newState;
		}
	}

	static void SetAnimationClip (string clipName, int count, ref List<AnimationClip> acArray)
	{
		for (int i = 0; i < count; i++) {
			if (clipName == die.name) {
				acArray.Add (die);
			}
			if (clipName == jump.name) {
				acArray.Add (jump);
			}
			if (clipName == BackLeap.name) {
				acArray.Add (BackLeap);
			}
			if (clipName == ForwardLeap.name) {
				acArray.Add (ForwardLeap);
			}
		}
	}

	static AnimatorState AddState<T> (string stateName, AnimatorStateMachine sm, float threshold, string parameter, Vector3 position,
	                                  AnimationClip clip, bool first = false, bool last = false) where T : StateMachineBehaviour
	{
		AnimatorStateTransition animatorStateTransition;
		//  生成AnimatorState
		AnimatorState animatorState = sm.AddState (stateName, position);
		// 设置动画片段
		animatorState.motion = clip;
		// 创建AnimatorStateTransition
		// entry连接到特技组的第一个动画
		if (first) {
			animatorStateTransition = sm.AddAnyStateTransition (animatorState);
			animatorStateTransition.AddCondition (AnimatorConditionMode.Equals, threshold, parameter);
		}
		// 最后一个动画连接到stand
		if (last) {
			animatorStateTransition = animatorState.AddTransition (mainASM.defaultState);
		}

		// 特技组内的连接创建
		if (!first && !last) {
			animatorStateTransition = animatorState.AddTransition (mainASM.defaultState);
		}

		animatorStateTransition = lastAnimatorState.AddTransition (animatorState, true);

		//AnimatorStateTransition 的设置
		animatorStateTransition.canTransitionToSelf = false;
		animatorState.AddStateMachineBehaviour<T> ();
		return animatorState;
	}

	static AnimatorState AddState (string stateName, AnimatorStateMachine sm, Vector3 position, AnimationClip clip, int index, int count)
	{
		AnimatorStateTransition animatorStateTransition = null;
		//  生成AnimatorState
		AnimatorState animatorState = sm.AddState (stateName, position);
		// 设置动画片段
		animatorState.motion = clip;
		// 创建AnimatorStateTransition
		// AnyState连接到特技组的第一个动画
		if (index == 1) {
			//animatorStateTransition = sm.AddAnyStateTransition(animatorState);
			//animatorStateTransition.canTransitionToSelf = false;
		}
		// 最后一个动画连接到main animator machine的default state
		if (index == count) {
			animatorStateTransition = animatorState.AddTransition (mainASM.defaultState);
			animatorStateTransition.hasExitTime = true;
		}

		// 特技组内的连接创建
		if (lastAnimatorState != null) {
			animatorStateTransition = lastAnimatorState.AddTransition (animatorState, true);
		}

		return animatorState;
	}

	#endregion

	#region public method

	static XmlConfig ReadXml ()
	{
		//string xmlStr = File.ReadAllText(Application.dataPath.ToString() + "/StreamingAssets/XMLConfigFiles/Stunt.xml");
		//Debug.Log(xmlStr);
		//string objTxt = Regex.Replace(xmlStr, @"<!--[^-]*-->", string.Empty, RegexOptions.IgnoreCase);
		//Debug.Log(objTxt);
		return DeserializeFromXml<XmlConfig> (Application.dataPath.ToString () + "/StreamingAssets/XMLConfigFiles/data.xml");
	}

	/// <summary>
	/// 从某一XML文件反序列化到某一类型
	/// </summary>
	/// <param name="filePath">待反序列化的XML文件名称</param>
	/// <param name="type">反序列化出的</param>
	/// <returns></returns>
	public static T DeserializeFromXml<T> (string filePath)
	{
		try {
			if (!System.IO.File.Exists (filePath))
				throw new ArgumentNullException (filePath + " not Exists");

			using (System.IO.StreamReader reader = new System.IO.StreamReader (filePath)) {
				System.Xml.Serialization.XmlSerializer xs = new System.Xml.Serialization.XmlSerializer (typeof(T));
				T ret = (T)xs.Deserialize (reader);
				return ret;
			}
		}
		catch (Exception ex) {
			return default(T);
		}
	}

	#endregion
}


#region 序列化需要的model
[XmlType (TypeName = "config")]
public class XmlConfig
{
	[XmlArray ("datas")]
	public List<Data> datas { get; set; }
}

[XmlType (TypeName = "data")]
public class Data
{
	[XmlAttribute]
	public int INDEX;
	[XmlAttribute]
	public string Clip1;
	[XmlAttribute]
	public int Clip1Count;
	[XmlAttribute]
	public string Clip2;
	[XmlAttribute]
	public int Clip2Count;
	[XmlAttribute]
	public string Clip3;
	[XmlAttribute]
	public int Clip3Count;
}

#endregion
```

### 4.最后的说明

- 其实整个过程基本就是读取XML文件内容，然后按照第二部分中描述的结构来一点一点构建状态机。

- 在设定具体属性时需要按照具体情况来做。

- 有个天坑，就是如果在Base Layer界面多次点击CreateAnimatorState按钮时会出现Unity的crash，或者出现界面所有元素消失并报错。我找了很多资料应该是UnityEditor的bug。有一个很简单的解决办法，就是创建一个新的Layer，切换到新Layer的界面，然后点击CreateAnimatorState按钮，再切回Base Layer，这样就不会出错了。