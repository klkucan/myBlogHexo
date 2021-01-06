---
layout: post
title: "Unity material 冗余数据清理"
date: 2021-1-4 00:39:52 +0800
thumbnail: "img/home-bg-o.jpg"
categories:
    - 游戏开发
tags:
    - Unity3D
---

## 问题

- 做如下操作：
  - 创建一个material文件，设置shader和texture。获得此时mat文件的内容。
  - 对这个material换几个不同的shader，并且分别在不同的shader中选中了一些texture后，最终还原为初次的shader和texture。获取此时mat文件的内容。
- 可以看到，文件中保存了一些过程中用到的shader的变量。例子中就是多了一些纹理和颜色的数据。
- 为什么原始的mat会多出这么多当前shader中没有的变量呢？这是因为一个material创建之初，默认给了standard这个shader，所以就保留了这个shader中的变量。
- PS：下面的YAML文件中，第一个是原始的mat文件，第二个是修改后再还原的。

<!--more-->

```yaml
%YAML 1.1
%TAG !u! tag:unity3d.com,2011:
--- !u!21 &2100000
Material:
  serializedVersion: 6
  m_ObjectHideFlags: 0
  m_CorrespondingSourceObject: {fileID: 0}
  m_PrefabInstance: {fileID: 0}
  m_PrefabAsset: {fileID: 0}
  m_Name: demo
  m_Shader: {fileID: 4800000, guid: 9d51725fb9bb3084e93cab0ba3655a49, type: 3}
  m_ShaderKeywords: _WIGGLE_ON _WIND_ON
  m_LightmapFlags: 4
  m_EnableInstancingVariants: 0
  m_DoubleSidedGI: 0
  m_CustomRenderQueue: -1
  stringTagMap: {}
  disabledShaderPasses: []
  m_SavedProperties:
    serializedVersion: 3
    m_TexEnvs:
    - _BumpMap:
        m_Texture: {fileID: 0}
        m_Scale: {x: 1, y: 1}
        m_Offset: {x: 0, y: 0}
    - _DetailAlbedoMap:
        m_Texture: {fileID: 0}
        m_Scale: {x: 1, y: 1}
        m_Offset: {x: 0, y: 0}
    - _DetailMask:
        m_Texture: {fileID: 0}
        m_Scale: {x: 1, y: 1}
        m_Offset: {x: 0, y: 0}
    - _DetailNormalMap:
        m_Texture: {fileID: 0}
        m_Scale: {x: 1, y: 1}
        m_Offset: {x: 0, y: 0}
    - _EmissionMap:
        m_Texture: {fileID: 0}
        m_Scale: {x: 1, y: 1}
        m_Offset: {x: 0, y: 0}
    - _MainTex:
        m_Texture: {fileID: 2800000, guid: 41bc79d092683d14da52878e3ee35663, type: 3}
        m_Scale: {x: 1, y: 1}
        m_Offset: {x: 0, y: 0}
    - _MetallicGlossMap:
        m_Texture: {fileID: 0}
        m_Scale: {x: 1, y: 1}
        m_Offset: {x: 0, y: 0}
    - _OcclusionMap:
        m_Texture: {fileID: 0}
        m_Scale: {x: 1, y: 1}
        m_Offset: {x: 0, y: 0}
    - _ParallaxMap:
        m_Texture: {fileID: 0}
        m_Scale: {x: 1, y: 1}
        m_Offset: {x: 0, y: 0}
    - _ToonRamp:
        m_Texture: {fileID: 2800000, guid: 23a1485968bc8b246b3c0db15cae6a71, type: 3}
        m_Scale: {x: 1, y: 1}
        m_Offset: {x: 0, y: 0}
    - _texcoord:
        m_Texture: {fileID: 0}
        m_Scale: {x: 1, y: 1}
        m_Offset: {x: 0, y: 0}
    m_Floats:
    - _BumpScale: 1
    - _Cutoff: 0.5
    - _DetailNormalMapScale: 1
    - _DstBlend: 0
    - _Float2: 1
    - _GlossMapScale: 1
    - _Glossiness: 0.5
    - _GlossyReflections: 1
    - _Metallic: 0
    - _Mode: 0
    - _OcclusionStrength: 1
    - _Parallax: 0.02
    - _RimOffset: 0.24
    - _RimPower: 0.5
    - _SmoothnessTextureChannel: 0
    - _SpecularHighlights: 1
    - _SrcBlend: 1
    - _UVSec: 0
    - _Wiggle: 1
    - _WiggleStrenght: 0.5
    - _Wind: 1
    - _WindStrenght: 0.5
    - _ZWrite: 1
    - __dirty: 1
    m_Colors:
    - _Color: {r: 1, g: 1, b: 1, a: 1}
    - _EmissionColor: {r: 0, g: 0, b: 0, a: 1}
    - _RimColor: {r: 0, g: 1, b: 0.8758622, a:
    
```

```yaml
%YAML 1.1
%TAG !u! tag:unity3d.com,2011:
--- !u!21 &2100000
Material:
  serializedVersion: 6
  m_ObjectHideFlags: 0
  m_CorrespondingSourceObject: {fileID: 0}
  m_PrefabInstance: {fileID: 0}
  m_PrefabAsset: {fileID: 0}
  m_Name: demo
  m_Shader: {fileID: 4800000, guid: 9d51725fb9bb3084e93cab0ba3655a49, type: 3}
  m_ShaderKeywords: YG_ON _WIGGLE_ON _WIND_ON
  m_LightmapFlags: 4
  m_EnableInstancingVariants: 0
  m_DoubleSidedGI: 0
  m_CustomRenderQueue: -1
  stringTagMap: {}
  disabledShaderPasses: []
  m_SavedProperties:
    serializedVersion: 3
    m_TexEnvs:
    - _BumpMap:
        m_Texture: {fileID: 0}
        m_Scale: {x: 1, y: 1}
        m_Offset: {x: 0, y: 0}
    - _DetailAlbedoMap:
        m_Texture: {fileID: 0}
        m_Scale: {x: 1, y: 1}
        m_Offset: {x: 0, y: 0}
    - _DetailMask:
        m_Texture: {fileID: 0}
        m_Scale: {x: 1, y: 1}
        m_Offset: {x: 0, y: 0}
    - _DetailNormalMap:
        m_Texture: {fileID: 0}
        m_Scale: {x: 1, y: 1}
        m_Offset: {x: 0, y: 0}
    - _EmissionMap:
        m_Texture: {fileID: 0}
        m_Scale: {x: 1, y: 1}
        m_Offset: {x: 0, y: 0}
    - _MainTex:
        m_Texture: {fileID: 2800000, guid: 41bc79d092683d14da52878e3ee35663, type: 3}
        m_Scale: {x: 1, y: 1}
        m_Offset: {x: 0, y: 0}
    - _MainTex2:
        m_Texture: {fileID: 2800000, guid: 328688e6713a72148920f5b60c4a0a94, type: 3}
        m_Scale: {x: 1, y: 1}
        m_Offset: {x: 0, y: 0}
    - _MetallicGlossMap:
        m_Texture: {fileID: 0}
        m_Scale: {x: 1, y: 1}
        m_Offset: {x: 0, y: 0}
    - _OcclusionMap:
        m_Texture: {fileID: 0}
        m_Scale: {x: 1, y: 1}
        m_Offset: {x: 0, y: 0}
    - _ParallaxMap:
        m_Texture: {fileID: 0}
        m_Scale: {x: 1, y: 1}
        m_Offset: {x: 0, y: 0}
    - _TextureSample3:
        m_Texture: {fileID: 2800000, guid: 6935b77838fc2bb46abd6cc5b1f8edc7, type: 3}
        m_Scale: {x: 1, y: 1}
        m_Offset: {x: 0, y: 0}
    - _ToonRamp:
        m_Texture: {fileID: 2800000, guid: 23a1485968bc8b246b3c0db15cae6a71, type: 3}
        m_Scale: {x: 1, y: 1}
        m_Offset: {x: 0, y: 0}
    - _texcoord:
        m_Texture: {fileID: 0}
        m_Scale: {x: 1, y: 1}
        m_Offset: {x: 0, y: 0}
    m_Floats:
    - _BumpScale: 1
    - _Cutoff: 0.5
    - _DetailNormalMapScale: 1
    - _DstBlend: 0
    - _EndDitheringFade: 1
    - _Float2: 1
    - _GlossMapScale: 1
    - _Glossiness: 0.5
    - _GlossyReflections: 1
    - _Metallic: 0
    - _Mode: 0
    - _OcclusionStrength: 1
    - _Parallax: 0.02
    - _RimOffset: 0.24
    - _RimPower: 0.5
    - _SmoothnessTextureChannel: 0
    - _SpecularHighlights: 1
    - _SrcBlend: 1
    - _StartDitheringFade: 0
    - _UVSec: 0
    - _Wiggle: 1
    - _WiggleStrenght: 0.5
    - _Wind: 1
    - _WindStrenght: 0.5
    - _ZWrite: 1
    - __dirty: 1
    - yg: 1
    m_Colors:
    - _Color: {r: 1, g: 1, b: 1, a: 1}
    - _Color2: {r: 1, g: 1, b: 1, a: 0}
    - _Color9: {r: 0.17, g: 0.17, b: 0.17, a: 0}
    - _EmissionColor: {r: 0, g: 0, b: 0, a: 1}
    - _RimColor: {r: 0, g: 1, b: 0.8758622, a: 0}

```

## 这个问题带来什么结果

- 这一现象导致的问题是打包时会把已经不需要的资源进行打包。
- 通过unity的AssetBundleBrowser工具可以明确看到在打包时，出现了不需要的texture文件。

## 解决问题

- 参考了网上的一些代码，修改后测试可以。
- 目前看到的mat文件有两种格式，下面的代码通用。

 

```csharp
using UnityEditor;
using UnityEngine;

public class MatTools : Editor
{
    [MenuItem("Tools/ClearMatProperties")]
    static void ClearMatProperties()
    {
        UnityEngine.Object[] objs = Selection.GetFiltered(typeof(Material), SelectionMode.DeepAssets);
        for (int i = 0; i < objs.Length; ++i)
        {
            Material mat = objs[i] as Material;

            if (mat)
            {
                SerializedObject psSource = new SerializedObject(mat);
                SerializedProperty emissionProperty = psSource.FindProperty("m_SavedProperties");
                SerializedProperty texEnvs = emissionProperty.FindPropertyRelative("m_TexEnvs");
                SerializedProperty floats = emissionProperty.FindPropertyRelative("m_Floats");
                SerializedProperty colos = emissionProperty.FindPropertyRelative("m_Colors");

                CleanMaterialSerializedProperty(texEnvs, mat);
                CleanMaterialSerializedProperty(floats, mat);
                CleanMaterialSerializedProperty(colos, mat);

                psSource.ApplyModifiedProperties();
                EditorUtility.SetDirty(mat);
            }
        }

        AssetDatabase.SaveAssets();
    }

    /// <summary>
    /// true: has useless propeties
    /// </summary>
    /// <param name="property"></param>
    /// <param name="mat"></param>
    private static void CleanMaterialSerializedProperty(SerializedProperty property, Material mat)
    {
        for (int j = property.arraySize - 1; j >= 0; j--)
        {
            string propertyName = property.GetArrayElementAtIndex(j).displayName;
            //string propertyName = property.GetArrayElementAtIndex(j).FindPropertyRelative("first").FindPropertyRelative("name").stringValue;
            Debug.Log("Find property in serialized object : " + propertyName);
            if (!mat.HasProperty(propertyName))
            {
                if (propertyName.Equals("_MainTex"))
                {
                    //_MainTex是内建属性，是置空不删除，否则UITexture等控件在获取mat.maintexture的时候会报错
                    if (property.GetArrayElementAtIndex(j).FindPropertyRelative("second").FindPropertyRelative("m_Texture").objectReferenceValue != null)
                    {
                        property.GetArrayElementAtIndex(j).FindPropertyRelative("second").FindPropertyRelative("m_Texture").objectReferenceValue = null;
                        Debug.Log("Set _MainTex is null");
                    }
                }
                else
                {
                    property.DeleteArrayElementAtIndex(j);
                    Debug.Log("Delete property in serialized object : " + propertyName);
                }
            }
        }
    }

}
```

- 工具执行完后的mat文件，对照shader文件可以发现清理了所有的不必要的变量。

```yaml
%YAML 1.1
%TAG !u! tag:unity3d.com,2011:
--- !u!21 &2100000
Material:
  serializedVersion: 6
  m_ObjectHideFlags: 0
  m_CorrespondingSourceObject: {fileID: 0}
  m_PrefabInstance: {fileID: 0}
  m_PrefabAsset: {fileID: 0}
  m_Name: demo
  m_Shader: {fileID: 4800000, guid: 9d51725fb9bb3084e93cab0ba3655a49, type: 3}
  m_ShaderKeywords: YG_ON _WIGGLE_ON _WIND_ON
  m_LightmapFlags: 4
  m_EnableInstancingVariants: 0
  m_DoubleSidedGI: 0
  m_CustomRenderQueue: -1
  stringTagMap: {}
  disabledShaderPasses: []
  m_SavedProperties:
    serializedVersion: 3
    m_TexEnvs:
    - _MainTex:
        m_Texture: {fileID: 2800000, guid: 41bc79d092683d14da52878e3ee35663, type: 3}
        m_Scale: {x: 1, y: 1}
        m_Offset: {x: 0, y: 0}
    - _ToonRamp:
        m_Texture: {fileID: 2800000, guid: 23a1485968bc8b246b3c0db15cae6a71, type: 3}
        m_Scale: {x: 1, y: 1}
        m_Offset: {x: 0, y: 0}
    - _texcoord:
        m_Texture: {fileID: 0}
        m_Scale: {x: 1, y: 1}
        m_Offset: {x: 0, y: 0}
    m_Floats:
    - _Cutoff: 0.5
    - _Float2: 1
    - _RimOffset: 0.24
    - _RimPower: 0.5
    - _Wiggle: 1
    - _WiggleStrenght: 0.5
    - _Wind: 1
    - _WindStrenght: 0.5
    - __dirty: 1
    m_Colors:
    - _Color: {r: 1, g: 1, b: 1, a: 1}
    - _RimColor: {r: 0, g: 1, b: 0.8758622, a: 0}
```