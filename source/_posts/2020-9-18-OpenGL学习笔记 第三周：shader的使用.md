---
layout: post
title:  "OpenGL学习笔记 第三周：shader的使用"
date:   2020-9-14 20:56:00 +0800
header-img: "img/home-bg-o.jpg"
tags:
    - OpenGL
---

#### 前言
- 一周时间学会了一件事，怎么把前两周的纯OpenGL绘制改成shader绘制。
- 之前对unityshader比较熟悉，因此编写shader本身没啥新收获。但是对于C++代码层面如何的包装shader和调用上收获颇丰。
- 学完后更加深入的体会到，shader作为在GPU上执行的函数的意义。同时对于渲染一个物体都需要哪些数据有了更深刻的认知。

<!--more-->

#### 编写shader

- 这两个shader更符合在“GPU运行的程序”的含义。Unity shader的写法隐藏了很多的细节，但是功能更加强大。

```
attribute vec4 position;
attribute vec4 color;
attribute vec2 texcoord;
attribute vec4 normal;

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
	V_WorldPos = ModelMatrix*position;
	V_Color = color;
	gl_Position=ProjectionMatrix*ViewMatrix*ModelMatrix*position;
}
```

```
#ifdef GL_ES
precision mediump float; //opengl es中需要定义浮点精度，否则可能运行不了
#endif

uniform sampler2D U_Texture;

uniform vec4 U_LightPos;        // 光源位置，W=0时就是平行光

uniform vec4 U_LightAmbient;    // 光的环境光分量
uniform vec4 U_AmbientMaterial; // 物体材质对环境光的反射系数

uniform vec4 U_LightDiffuse;
uniform vec4 U_DiffuseMaterial;

uniform vec4 U_LightSpecular;
uniform vec4 U_SpecularMaterial;

uniform vec4 U_CameraPos;
uniform vec4 U_LightOpt;

varying vec4 V_Color;
varying vec2 V_Texcoord;
varying vec4 V_Normal;
varying vec4 V_WorldPos;

vec4 GetPointLight()
{
	float distance=0.0;
	float constantFactor=1.0;
	float linearFactor=0.0;
	float quadricFactor=0.0;
	vec4 ambientColor=vec4(1.0,1.0,1.0,1.0)*vec4(0.1,0.1,0.1,1.0);
	vec3 L=vec3(0.0,1.0,0.0)-V_WorldPos.xyz;
	distance=length(L);
	float attenuation=1.0/(constantFactor+linearFactor*distance+quadricFactor*quadricFactor*distance);
	L=normalize(L);
	vec3 n=normalize(V_Normal.xyz);
	float diffuseIntensity=max(0.0,dot(L,n));
	vec4 diffuseColor=vec4(1.0,1.0,1.0,1.0)*vec4(0.1,0.4,0.6,1.0)*diffuseIntensity*attenuation;
	return ambientColor+diffuseColor;
}

// 接受的是vertex shader中生成的三角面，然后计算三角面上的像素的值。
// GPU中并行计算，并行的是计算像素这个事情
void main()
{
	/*  unity shader的写法
	fixed4 frag(v2f i) : SV_Target {
	// Get ambient term
	fixed3 ambient = UNITY_LIGHTMODEL_AMBIENT.xyz;

	fixed3 worldNormal = normalize(i.worldNormal);
	fixed3 worldLightDir = normalize(_WorldSpaceLightPos0.xyz);

	// Compute diffuse term
	fixed3 diffuse = _LightColor0.rgb * _Diffuse.rgb * saturate(dot(worldNormal, worldLightDir));

	// Get the reflect direction in world space
	fixed3 reflectDir = normalize(reflect(-worldLightDir, worldNormal));
	// Get the view direction in world space
	fixed3 viewDir = normalize(_WorldSpaceCameraPos.xyz - i.worldPos.xyz);
	// Compute specular term
	fixed3 specular = _LightColor0.rgb * _Specular.rgb * pow(saturate(dot(reflectDir, viewDir)), _Gloss);

	return fixed4(ambient + diffuse + specular, 1.0);
	}*/

	vec4 ambientColor = U_LightAmbient * U_AmbientMaterial;
	
	vec3 lightPos = U_LightPos.xyz;
	vec3 L = lightPos;
	// 这里直接对U_LightPos.xyz对了归一化
	L = normalize(L);

	vec3 n = normalize(V_Normal.xyz);

	float diffuseIntensity = max(0.0, dot(L,n));
	vec4 diffuseColor = U_LightDiffuse*U_DiffuseMaterial*diffuseIntensity;

	vec4 specularColor = vec4(0.0,0.0,0.0,0.0);
	if(diffuseIntensity!=0)
	{
		vec3 reflectDir = normalize(reflect(-L,n));
		vec3 viewDir = normalize(U_CameraPos.xyz - V_WorldPos.xyz);
		specularColor = U_LightSpecular*U_SpecularMaterial*pow(max(0.0, dot(viewDir, reflectDir)), U_LightOpt.x);
	}
	if(U_LightOpt.y==1.0){
		gl_FragColor=(ambientColor+diffuseColor)*texture2D(U_Texture,V_Texcoord.xy)+specularColor;
	}else if(U_LightOpt.z==1.0){
		gl_FragColor=(ambientColor+diffuseColor+GetPointLight())*texture2D(U_Texture,V_Texcoord.xy);
	}else if(U_LightOpt.w==1.0){
		gl_FragColor=ambientColor+diffuseColor+specularColor;
	}
}


```

- 公式的几何意义

![image](https://note.youdao.com/yws/res/32764/0F542835D53C46C484FE376B0705F6A2)

##### uniform

- uniform变量在vertex和fragment两者之间声明方式完全一样，则它可以在vertex和fragment共享使用。（相当于一个被vertex和fragment shader共享的全局变量）
 
- uniform变量一般用来表示：变换矩阵，材质，光照参数和颜色等信息。以下是例子：

##### attribute
- attribute变量是只能在vertex shader中使用的变量。（它不能在fragment shader中声明attribute变量，也不能被fragment shader中使用）
 
- 一般用attribute变量来表示一些顶点的数据，如：顶点坐标，法线，纹理坐标，顶点颜色等。
 
- 程序中，一般用函数glBindAttribLocation（）来绑定每个attribute变量的位置，然后用函数glVertexAttribPointer（）为每个attribute变量赋值。

##### varying变量
- varying变量是vertex和fragment shader之间做数据传递用的。一般vertex shader修改varying变量的值，然后fragment shader使用该varying变量的值。因此varying变量在vertex和fragment 
- shader二者之间的声明必须是一致的。application不能使用此变量。

#### 使用OpenGL编译shader

- 创建shader对象
- 将shader代码赋值给shader对象
- 编译
- 错误处理

```cpp
GLuint CompileShader(GLenum shaderType, const char* shaderCode)
{
	GLuint shader = glCreateShader(shaderType);
	// GLuint shader, GLsizei count, const GLchar *const* string, const GLint* length
	// shader ，      有几句代码，    代码段，                       代码段长度，如果是多久代码的话，这里是一个长度数组
	glShaderSource(shader, 1, &shaderCode, nullptr);
	// 编译
	glCompileShader(shader);
	// 检查编译结果
	GLint compileResult = GL_TRUE;
	// 获取编译结果
	glGetShaderiv(shader, GL_COMPILE_STATUS, &compileResult);

	if (compileResult == GL_FALSE)
	{
		char szLog[1024] = { 0 };
		GLsizei logLen = 0;
		// 第二个是错误日志的缓冲区有多大，第三个参数实际写的log的长度
		glGetShaderInfoLog(shader, 1024, &logLen, szLog);
		printf("Compile shader error : %s \n, shader code : \n%s\n", szLog, shaderCode);
		glDeleteShader(shader);
		shader = 0;
	}
	return shader;
}
```

- shader的编译是驱动提供的功能，上面的代码中是最基本的用法。
- 除此之外在使用FeedBack技术的时候，也可以只传递vertex shader。

#### 创建Program

- 创建Program对象
- attach shader
- link
- 错误检查

```cpp
GLuint CreateProgram(GLuint vsShader, GLuint fsShader)
{
	GLuint program= glCreateProgram();
	glAttachShader(program, vsShader);
	glAttachShader(program, fsShader);
	glLinkProgram(program);
	glDeleteShader(vsShader);
	glDeleteShader(fsShader);
	
	GLint nResult;
	glGetProgramiv(program, GL_LINK_STATUS, &nResult);
	if (nResult == GL_FALSE)
	{
		char log[1024] = { 0 };
		GLsizei writed = 0;
		// 第二个是错误日志的缓冲区有多大，第三个参数实际写的log的长度
		glGetProgramInfoLog(program, 1024, &writed, log);
		printf("create gpu program fail , link error : %s \n", log);
		glDeleteProgram(program);
		program = 0;
	}
	
	return program;
}
```

- 后续的参数获取和数据赋值都是围绕program来做的。

#### 使用Shader

- shader的使用可以分成两个步骤。第一，获取参数的location。第二是在每次渲染的时候给与shader中的参数赋值。
- 这些参数我个人理解分成三大类：
    - VBO中提取的数据：position、texcoord、color、normal这些
    - 管线中变换矩阵，也就是MVP三个矩阵。
    - 其它主要用于fragment的参数：纹理、光源信息、物体材质信息，其它用于逻辑的参数。

##### VBO

- 模型的顶点数据，常见的数据有vertex的position、texcoord、normal、color

```
    glGenBuffers(1, &vbo); // 显存中申请一块内容，地址为vbo
	float data[] = {
		-0.2f, -0.2f, -0.6f, 1.0f, /* color: */ 1.0f, 1.0f, 1.0f, 1.0f, /* texcoord: */ 0.0f, 0.0f,
		0.2f, -0.2f, -0.6f, 1.0f,  /* color: */ 0.0f, 1.0f, 0.0f, 1.0f, /* texcoord: */ 1.0f, 0.0f,
		0.0f, 0.2f, -0.6f, 1.0f,   /* color: */ 1.0f, 0.0f, 0.0f, 1.0f, /* texcoord: */ 0.5f, 1.0f
	};

	// 操作当前的vbo，类比glBindTexture
	glBindBuffer(GL_ARRAY_BUFFER, vbo);
	// GL_STATIC_DRAW指的是放到GPU后不再动的数据
	// 函数完成后数据从内存赋值到了显存，此时这个数据已经可以删除了
	glBufferData(GL_ARRAY_BUFFER, sizeof(float) * 30, data, GL_STATIC_DRAW);
	glBindBuffer(GL_ARRAY_BUFFER, 0); 
```

##### 参数赋值
- 获得shader中参数的location，这一步得到的位置是某个program中的，因此执行一次就可以了。
- 给uniform、attribute变量赋值。这种赋值的操作都是针对某个program，可能在一次渲染中针对多个program赋值。因此按照套路，都是先bind一个对象。
- glUseProgram(program)后，在赋值默认都是给这个program的。


```
    GLuint vbo;
    GLuint program;
    GLint positionLocation;
    GLint modelMatrixLocation;
    GLint viewMatrixLocation;
    GLint projectionMatrixLocation;
    GLint colorLocation;
    GLint texcoordLocation;
    GLint textureLocation;
    glm::mat4 modelMatrix, viewMatrix, projectionMatrix;
    GLuint texture;
    
    ...
    
    // 位置不是按照shader代码里写的变量的顺序确定的
    positionLocation = glGetAttribLocation(program, "position");
	colorLocation = glGetAttribLocation(program, "color");
	texcoordLocation = glGetAttribLocation(program, "texcoord");

	modelMatrixLocation = glGetUniformLocation(program, "ModelMatrix");
	viewMatrixLocation = glGetUniformLocation(program, "ViewMatrix");
	projectionMatrixLocation = glGetUniformLocation(program, "ProjectionMatrix");
	textureLocation = glGetUniformLocation(program, "U_Texture");
	
	... 
	
	glUseProgram(program);
	// 参数含义：插槽位置，几个矩阵（单插槽可以多矩阵），CPU传到GPU时是否需要转置，矩阵地址(从定义看是第一个数据的位置)
	glUniformMatrix4fv(modelMatrixLocation, 1, GL_FALSE, glm::value_ptr(modelMatrix));
	glUniformMatrix4fv(viewMatrixLocation, 1, GL_FALSE, glm::value_ptr(viewMatrix));
	glUniformMatrix4fv(projectionMatrixLocation, 1, GL_FALSE, glm::value_ptr(projectionMatrix));
	
	// 使用纹理
	glBindTexture(GL_TEXTURE_2D, texture);
	glUniform1i(textureLocation, 0);
	
	glBindBuffer(GL_ARRAY_BUFFER, vbo);
	// attribute插槽中某一个位置生效
	glEnableVertexAttribArray(positionLocation);
	
	// GLuint index, GLint size, GLenum type, GLboolean normalized, GLsizei stride, const void* pointer
	// 插槽位置 ， 插槽中数据有几个分量（现在的位置是4个，xyzw），分量类型，
	// 是否归一化（传入的如果不是float会变成float，e.g. 255,255,255,255→ 1.0,1.0,1.0,1.0)
	// 紧挨着的两个点起始地址相距多远，这个位置信息从vbo的什么地址开始取值（从0号位置开始取值）
	glVertexAttribPointer(positionLocation, 4, GL_FLOAT, GL_FALSE, sizeof(float) * 10, 0);
	
	// 给color变量赋值
	glEnableVertexAttribArray(colorLocation);
	glVertexAttribPointer(colorLocation, 4, GL_FLOAT, GL_FALSE, sizeof(float) * 10, (void*)(sizeof(float) * 4));

	// 给texcoord赋值
	glEnableVertexAttribArray(texcoordLocation);
	glVertexAttribPointer(texcoordLocation, 2, GL_FLOAT, GL_FALSE, sizeof(float) * 10, (void*)(sizeof(float) * 8));
	
	// 绘制什么，从第几个点开始绘制，绘制几个三角形。
	glDrawArrays(GL_TRIANGLES, 0, 3);

	glBindBuffer(GL_ARRAY_BUFFER, 0);

	glBindTexture(GL_TEXTURE_2D, 0);

	glUseProgram(0);
```

- glVertexAttribPointer的作用是告诉GPU如何读取VBO的数据
- glDrawArrays的作用是告知GPU如何把VBO中的数据分发到不同的shader去执行

#### 面向对象的设计

- 遵照数据（VBO）与逻辑分离（shader）分离的思想，有了VertexBuffer和Shader 两个类。
- 又按照渲染前准备的数据（location、一些空对象的创建）和渲染中需要的数据（每个shader中参数的赋值），在每个类中有两个核心的参数。
- 实际上两个维度就将OpenGL shader的使用整得明明白白的。

## 已经忘记的差不多的知识

#### 点积几何意义
- 设二维空间内有两个向量 和 ， 和 表示向量a和b的大小，它们的夹角为 ，则内积定义为以下实数：
  
![image](https://note.youdao.com/yws/res/32760/6FD6E25DEAAB4022B3B3E06FFC8C0DEB)

![image](https://note.youdao.com/yws/res/32762/C4DC95E76DC24F1A8B576CB98EB113B7)



