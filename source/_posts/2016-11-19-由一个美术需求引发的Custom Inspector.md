---
layout: post
title:  "由一个美术需求引发的Custom Inspector"
date:   2016-11-19 15:52:25 +0800
thumbnail: "img/home-bg-o.jpg"
categories: 
- 游戏开发
tags:
- Unity3D
---

### 需求
- Editor模式下，在运行或者非运行状态下，能够按照指定的变化率来自动改变material中属性数值。

### 需求分析

- 如何在Editor模式下获得一个游戏对象及其组件，尤其是在非运行状态下？我们知道在Unity IDE运行起来后是很容易获得一个对象和组件的，在GameObject上挂一个脚本即可。但是在非运行状态下呢，`transform.GetComponent `这样的方法怎么执行？好在unity已经为我们考虑到了这个问题，提供了`[ExecuteInEditMode]`Attribute，通过指定这个attribute使得组件类中的方法可以在edit模式下执行，并且是在非运行状态下的。

- 如何在非运行状态下匀速改变数值呢？update方法中配合`Time.deltaTime`是一个完美的方案，但是即使设置了`[ExecuteInEditMode]`，update的表现在非运行和运行时也是完全不同的，查资料看到[ is only called when something in the scene changed. ](https://docs.unity3d.com/ScriptReference/ExecuteInEditMode.html)这句话时也有种吐槽的冲动。好在unity又为大家考虑到了这个问题（话说unity editor确实功能强大，AssetAtore里面那些插件真是厉害）,Edit模式下提供了`EditorApplication.update`，这是一个事件，我们注册一个自己的方法就可以在非运行状态下实现update的功能。我个人比较推荐使用[EditorCoroutine](https://gist.github.com/benblo/10732554)，一个基于`EditorApplication.update`的协程实现。

<!--more-->

### 功能实现

- 使用一个自定义组件来实现material中数值的修改，这个类在UI上要体现出能够设置变化速率和初始值。并且在UI上通过点击按钮的形式来触发改变。
- 使用Custom Inspector来实现组件UI的自定义。
- 在运行状态下通过使用默认的update来实现匀速变化，在非运行状态下通过使用EditorCoroutine来实现。

### 代码实现

```
using UnityEngine;
using System.Collections;
using UnityEditor;
public class UVAnimation : MonoBehaviour
{

    public Vector2 TilingSpeed = new Vector2(1, 1);
    public Vector2 OffsetSpeed = new Vector2(0.1f, 0.1f);

    public Vector2 Tiling = new Vector2(1, 1);
    public Vector2 Offset = new Vector2(0, 0);

    float rate = 0.02f;

    EditorCoroutine coroutineOffset;
    EditorCoroutine coroutineTiling;

    bool isOffset = false;
    bool isTiling = false;

    // Use this for initialization
    void Start()
    {
    }

    // Update is called once per frame
    void Update()
    {

    }

    void FixedUpdate()
    {
        if (isOffset)
        {
            transform.GetComponent<Renderer>().sharedMaterials[0].mainTextureOffset += OffsetSpeed * Time.deltaTime;
        }
        if (isTiling)
        {
            transform.GetComponent<Renderer>().sharedMaterials[0].mainTextureScale += TilingSpeed * Time.deltaTime;
        }
    }

    public void ChangeOffset()
    {
        if (EditorApplication.isPlaying)
        {
            isOffset = true;
        }
        else
        {
            if (coroutineOffset != null)
            {
                coroutineOffset.stop();
            }
            coroutineOffset = EditorCoroutine.start(ChangeOffsetCoroutine());
        }

    }

    IEnumerator ChangeOffsetCoroutine()
    {
        while (true)
        {
            yield return new WaitForSeconds(rate);
            transform.GetComponent<Renderer>().sharedMaterials[0].mainTextureOffset += OffsetSpeed * rate;
        }
    }

    public void ChangeTiling()
    {
        if (EditorApplication.isPlaying)
        {
            isTiling = true;
        }
        else
        {
            if (coroutineTiling != null)
            {
                coroutineTiling.stop();
            }
            coroutineTiling = EditorCoroutine.start(ChangeTilingCoroutine());
        }
    }

    IEnumerator ChangeTilingCoroutine()
    {
        while (true)
        {
            yield return new WaitForSeconds(rate);
            transform.GetComponent<Renderer>().sharedMaterials[0].mainTextureScale += TilingSpeed * rate;
        }
    }

    public void SetOffset()
    {
        transform.GetComponent<Renderer>().sharedMaterials[0].mainTextureOffset = Offset;
    }

    public void SetTiling()
    {
        transform.GetComponent<Renderer>().sharedMaterials[0].mainTextureScale = Tiling;
    }

    public void Reset()
    {
        isOffset = false;
        isTiling = false;
        transform.GetComponent<Renderer>().sharedMaterials[0].mainTextureScale = new Vector2(1, 1);
        transform.GetComponent<Renderer>().sharedMaterials[0].mainTextureOffset = new Vector2(0, 0);
        if (coroutineOffset != null)
        {
            coroutineOffset.stop();
        }
        if (coroutineTiling != null)
        {
            coroutineTiling.stop();
        }
    }
}

```

```
using UnityEngine;
using System.Collections;
using UnityEditor;

[CustomEditor(typeof(UVAnimation))]
public class UVAnimationBuilderEditor : Editor {


	public override void OnInspectorGUI ()
	{
	   base.OnInspectorGUI ();
		//DrawDefaultInspector ();
		UVAnimation uva = (UVAnimation)target;

		if (GUI.changed) {
			uva.SetTiling ();
			uva.SetOffset ();
		}

		if (GUILayout.Button("Change Tiling")) {
			uva.ChangeTiling ();	
			EditorUtility.SetDirty (target);
		}

		if (GUILayout.Button("Change Offset")) {
			uva.ChangeOffset ();
		}

		if (GUILayout.Button("Reset")) {
			uva.Reset ();
		}
	}
}

```

### 代码解析

- `[CustomEditor(typeof(UVAnimation))]`为UVAnimation创建的Editor类，在这个类里面可以修改UVAnimation类的UI，可以调用UVAnimation类中的方法。

- `OnInspectorGUI`方法，顾名思义在里面可以对UI进行编程，注意一下这个方法会自己生产一句代码`base.OnInspectorGUI ();`，我所注释掉的`DrawDefaultInspector ();`这句代码都是用来绘制默认UI的，二者只可留其一。

- `transform.GetComponent<Renderer>().sharedMaterials`在edit模式下获取材质球的对象需要用`sharedMaterials`。

- `mainTextureScale`对应的就是UI上的Tiling。命名这让人吐槽。