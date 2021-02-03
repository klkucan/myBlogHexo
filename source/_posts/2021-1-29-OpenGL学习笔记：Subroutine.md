---
layout: post
title: "OpenGL学习笔记：Subroutine"
date: 2021-1-29 19:55:09 +0800
thumbnail: "img/home-bg-o.jpg"
categories:
- 游戏开发
tags:
- OpenGL
---

## 概念

- Subroutine赋予了代码动态调用shader中函数的能力
- 写法比较固定，直接上代码

```glsl
#version 400 core
subroutine vec4 SurfaceColor();
uniform sampler2D U_MainTexture;
subroutine uniform SurfaceColor U_SurfaceColor;
varying vec3 V_Normal;
varying vec4 V_WorldPos;
varying vec2 V_Texcoord;

subroutine (SurfaceColor) vec4 Ambient()
{
	return texture2D(U_MainTexture,V_Texcoord)*ambientColor;
}

subroutine (SurfaceColor) vec4 Diffuse()
{
	return ambientColor+texture2D(U_MainTexture,V_Texcoord)*diffuseColor;
}

subroutine (SurfaceColor) vec4 Specular()
{
	return ambientColor+texture2D(U_MainTexture,V_Texcoord)*diffuseColor+specularColor+vec4(1f, 0.7f, 0.72f,1.0f);
}

void main()
{
	gl_FragColor=U_SurfaceColor();
}
```

<!--more-->

- cpp代码中获得这些函数的index

```cpp
GLuint amibentLightIndex = glGetSubroutineIndex(program, GL_FRAGMENT_SHADER, "Ambient");
GLuint diffuseLightIndex = glGetSubroutineIndex(program, GL_FRAGMENT_SHADER, "Diffuse");
GLuint specularLightIndex = glGetSubroutineIndex(program, GL_FRAGMENT_SHADER, "Specular");
```

```cpp
model = glm::translate(1.0f, -0.5f, -4.0f)*glm::rotate(-90.0f, 0.0f, 1.0f, 0.0f)*glm::scale(0.01f, 0.01f, 0.01f);
glUniformMatrix4fv(MLocation, 1, GL_FALSE, glm::value_ptr(model));
// 选择用哪个函数
glUniformSubroutinesuiv(GL_FRAGMENT_SHADER, 1, &specularLightIndex);
glBindBuffer(GL_ELEMENT_ARRAY_BUFFER, ibo);
glDrawElements(GL_TRIANGLES, indexCount, GL_UNSIGNED_INT, 0);
glBindBuffer(GL_ELEMENT_ARRAY_BUFFER, 0);
```