---
layout: post
title: "OpenGL学习笔记：Compute Shader"
date: 2021-1-30 16:34:36 +0800
thumbnail: "img/home-bg-o.jpg"
categories:
- 游戏开发
tags:
- OpenGL
---

## 概念

- 区别于vs 、fs，compute shader虽然从名字上有个shader，但是在用途上是与shader区别开的。
- shader从词义上是着色器的意思，因此它的核心用途是参与渲染。而compute shader可以看做是跑在GPU上的program，它可以完全是不参与渲染，而仅仅是运行一些我们想要的计算。

## 用法

- 下面是一个用compute shader反转图片的例子

### shader上的变化

- gl_GlobalInvocationID是一个全局的变量，标记当前是在绘制哪个顶点，依靠这个变量来进行采样，compute shader专用。
- In the compute language, gl_GlobalInvocationID is a derived input variable containing the n-dimensional index of the work invocation within the global work group that the current shader is executing on. The value of gl_GlobalInvocationID is equal to gl_WorkGroupID * gl_WorkGroupSize + gl_LocalInvocationID.
- image2D：a handle for accessing a 2D texture

<!--more-->

```glsl
#version 430
layout(local_size_x = 16, local_size_y = 16)in;
layout(binding = 0, rgba8)uniform mediump image2D inputImage;
layout(binding = 1, rgba8)uniform mediump image2D outImage;

void main()
{
	float u = float(gl_GlobalInvocationID.x);
	float v = float(gl_GlobalInvocationID.y);
	vec4 inputColor = imageLoad(inputImage,ivec2(u,v));
	imageStore(outImage,ivec2(u,v),vec4(1.0,1.0,1.0,1.0)-inputColor);
}
```

### Cpp代码

- 生成shader，与vs编译时的区别就是glCreateShader的第一个参数是GL_COMPUTE_SHADER
- 生成两个texture，一个用于存储原图，一个用于存储经过shader变色后的图。
- 此处注意写法上的区别，以前设置图片属性都是用的glTexImage2D，但是这里得用glTexStorage2D。而且对于上传到GPU的图片在设置数据时是用的glTexSubImage2D。
- 这里glTexStorage2D对标的是上面shader的写法

```cpp
  GLuint computeProgram = CreateComputeProgram("res/shader/simple.compute");
	GLuint computeDstTexture;
	glGenTextures(1, &computeDstTexture);
	glBindTexture(GL_TEXTURE_2D, computeDstTexture);
	glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_WRAP_T, GL_CLAMP_TO_EDGE);
	glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_WRAP_S, GL_CLAMP_TO_EDGE);
	glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MIN_FILTER, GL_LINEAR);
	glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MAG_FILTER, GL_LINEAR);
	glTexStorage2D(GL_TEXTURE_2D,1,GL_RGBA8,imgWidth,imgHeight);
	glBindTexture(GL_TEXTURE_2D, 0);

	GLuint mainTexture;

	glGenTextures(1, &mainTexture);
	glBindTexture(GL_TEXTURE_2D, mainTexture);
	glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_WRAP_T, GL_CLAMP_TO_EDGE);
	glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_WRAP_S, GL_CLAMP_TO_EDGE);
	glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MIN_FILTER, GL_LINEAR);
	glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MAG_FILTER, GL_LINEAR);
	glTexStorage2D(GL_TEXTURE_2D, 1, GL_RGBA8, imgWidth, imgHeight);
	glTexSubImage2D(GL_TEXTURE_2D,0,0,0,imgWidth,imgHeight,GL_RGB,GL_UNSIGNED_BYTE,pixelData);
	//glTexImage2D(GL_TEXTURE_2D, 0, GL_RGB, imgWidth, imgHeight, 0, GL_RGB, GL_UNSIGNED_BYTE, pixelData);
	glBindTexture(GL_TEXTURE_2D, 0);
```

- 使用compute shader
- 此处注意GL_READ_ONLY和GL_WRITE_ONLY，用来说明哪个纹理是读取用的，哪个纹理是我们要写入并在后面用的。
- glDispatchCompute ：launch one or more compute work groups。这个API需要注意参数不能过大，可以通过[glGet]([https://www.khronos.org/registry/OpenGL-Refpages/gl4/html/glGet.xhtml](https://www.khronos.org/registry/OpenGL-Refpages/gl4/html/glGet.xhtml))(GL_MAX_COMPUTE_WORK_GROUP_COUNT)来获取上限。
- glDispatchCompute是异步操作（事实上CPU对GPU都是异步操作），因此需要glMemoryBarrier来做一个同步的等待，然后执行后面的代码。

```cpp
  glUseProgram(computeProgram);
	glBindImageTexture(0, mainTexture, 0, GL_FALSE, 0, GL_READ_ONLY, GL_RGBA8);
	glBindImageTexture(1, computeDstTexture, 0, GL_FALSE, 0, GL_WRITE_ONLY, GL_RGBA8);
	glDispatchCompute(imgWidth / 16, imgHeight / 16, 1);
	//sync cpu with gpu
	glMemoryBarrier(GL_SHADER_IMAGE_ACCESS_BARRIER_BIT);
```

- 得到结果后，在绘制模型时，绑定纹理：glBindTexture(GL_TEXTURE_2D, computeDstTexture); 就是用计算后的结果作为纹理来用了。

## 参考

- [Image Load Store]([https://www.khronos.org/opengl/wiki/Image_Load_Store](https://www.khronos.org/opengl/wiki/Image_Load_Store))