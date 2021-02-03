---
layout: post
title: "OpenGL学习笔记：Instance"
date: 2021-1-27 19:55:09 +0800
thumbnail: "img/home-bg-o.jpg"
categories:
- 游戏开发
tags:
- OpenGL
---

## 步骤

- 定义多个实例之间不同的数据（posOffsets），类似unity中的constantbuffer数据。
- shader中定义一个变量，代表这这些变化的数据。例子代码中用了位置的偏移量。
- 在draw的时候给这个变量赋值，这个和position、normal那些变量的赋值方式是一样的。
- 使用glVertexAttribDivisor说明posOffsets的使用频率
- 使用glDrawArraysInstanced函数，替换glDrawArrays

<!--more-->

```cpp
void Model::DrawInstance(float x, float y, float z, glm::mat4 viewMatrix, glm::mat4 projectionMatrix, unsigned int instance)
{
	glEnable(GL_DEPTH_TEST);
	mVertexBuffer->Bind();
	glm::mat4 it = glm::inverseTranspose(mModelMatrix);
	mShader->SetVec4("U_CameraPos", x, y, z, 1.0f);
	mShader->Bind(glm::value_ptr(mModelMatrix), glm::value_ptr(viewMatrix), glm::value_ptr(projectionMatrix));

	glUniformMatrix4fv(glGetUniformLocation(mShader->mProgram, "IT_ModelMatrix"), 1, GL_FALSE, glm::value_ptr(it));
	mVertexBuffer->Unbind();

	float posOffsets[] = {
		0.01f,0.0f,0.0f,1.0f,
		0.0f,0.0f,0.0f,1.0f,
		-0.01f,0.0f,0.0f,1.0f
	};

	GLuint offsetVBO = CreateBufferObject(GL_ARRAY_BUFFER, sizeof(float) * 12, GL_STATIC_DRAW, posOffsets);
	// instance数据设置到GPU
	GLuint offsetLocation = glGetAttribLocation(mShader->mProgram, "offset");
	glBindBuffer(GL_ARRAY_BUFFER, offsetVBO);
	glEnableVertexAttribArray(offsetLocation);
	glVertexAttribPointer(offsetLocation, 4, GL_FLOAT, GL_FALSE, sizeof(float) * 4, (void*)0);
	glBindBuffer(GL_ARRAY_BUFFER, 0);
	// 指定offset数据变化的频率。0时代表每绘制一个顶点就变一次，1和大于1时表示绘制多少个实例后变一次。这个变是指的换下一行（组）数据。
	// https://www.khronos.org/registry/OpenGL-Refpages/gl4/html/glVertexAttribDivisor.xhtml
	glVertexAttribDivisor(offsetLocation, 1);	
	glDrawArraysInstanced(GL_TRING, 0, mVertexBuffer->mVertexCount, instance);

	mShader->Unbind();
}
```

```glsl
attribute vec4 position;
attribute vec4 color;
attribute vec2 texcoord;
attribute vec4 normal;
attribute vec4 offset;

uniform mat4 ModelMatrix;
uniform mat4 ViewMatrix;
uniform mat4 ProjectionMatrix;
uniform mat4 IT_ModelMatrix; // 模型空间到世界空间的转置矩阵，等同于Unity中的UnityObjectToWorldNormal

varying vec4 V_Color;
varying vec2 V_Texcoord;
varying vec4 V_Normal;
varying vec4 V_WorldPos;

void main()
{
	V_Normal = IT_ModelMatrix*normal;
	V_Texcoord = texcoord;
	V_WorldPos = ModelMatrix*(position)+offset;
	V_Color = color;
	gl_Position=ProjectionMatrix*ViewMatrix*V_WorldPos;
}
```