---
layout: post
title:  "OpenGL学习笔记 第一周：环境构建、OpenGL基础"
date:   2020-9-6 20:48:00 +0800
header-img: "img/home-bg-o.jpg"
tags:
    - OpenGL
---

## 构建环境
#### windows上创建一个窗体
- 定义窗体类，用到了`WNDCLASSEX`和`RegisterClassEx`函数。

- 创建窗体
    - 计算一个合适的窗体大小，用到了`AdjustWindowRect`
    - 创建`HWND hwnd = CreateWindowEx(NULL, "GLWindow", "OpenGL Window", WS_OVERLAPPEDWINDOW,
		100, 100, windowWidth, windowHight, NULL, NULL, hInstance, NULL);`
    
- 展示窗体：`ShowWindow(hwnd, SW_SHOW);`

<!--more-->

#### 在窗体对象上使用OpenGL
- 获得上下文句柄：`HDC  dc = GetDC(hwnd);`
- 获得合适的像素格式：用到`PIXELFORMATDESCRIPTOR`结构体、 `ChoosePixelFormat`和`SetPixelFormat` 两个函数。
- 获得OpenGL render对象：`HGLRC rc = wglCreateContext(dc); `
- 设定当前的渲染：`wglMakeCurrent(dc, rc);`

#### VS设置

- 因为是window窗体，因此printf没办法看到。在属性-生成事件-生成后事件-命令行中输入：`editbin /subsystem:console $(OutDir)$(ProjectName).exe` 就可以启动一个console窗体，然后看到printf的输出了。

#### OpenGL错误捕获

```
    // 注意OpenGL只会保留最后一条的错误
	GLenum error = glGetError();
	if (error != GL_NO_ERROR)
	{
		printf("error happened 0x%x \n", errno);
	}
```

## OpenGL基础操作
- 初始设置，glMatrixMode这个函数其实就是对接下来要做什么进行一下声明，也就是在要做下一步之前告诉计算机我要对“什么”进行操作了，这个“什么”在glMatrixMode的“()”里的选项(参数)有，GL_PROJECTION，GL_MODELVIEW和GL_TEXTURE；

```
    //设置当前矩阵，投影矩阵。从现在开始所有能影响到矩阵的代码都会影响当前的矩阵。
	glMatrixMode(GL_PROJECTION);
	//看世界的时候垂直方向上的视角，画布宽高比， 最近看到的距离，最远看到的距离
	gluPerspective(50.0f, 800.0f / 600.0f, 0.1f, 1000.0f);
	// 设置模型视口矩阵
	glMatrixMode(GL_MODELVIEW);
	glLoadIdentity();

```

- 基本的点线面的画法都遵循一个套路
    - 确定操作的什么：`glBegin(GL_POINTS)`、`glBegin(GL_LINE_STRIP)`和`glBegin(GL_TRIANGLES); `   
    - 根据不同的图形设置数据，最常用的是`glColor4ub(255, 255, 255, 255)`和`glVertex3f(-0.2f, -0.2f, -1.5f);` 分别代表了颜色和位置。
    - 三角形是按照逆时针确定点的顺序。
    - 以`glEnd()`结尾

#### 纹理使用
- 纹理对象创建
    - 读取文件为二进制数据
    - BMP格式是bgr顺序存储的，opengl需要RGB，因此需要转码。
    - 从OpenGL创建一个纹理对象，设置对象的各种属性，调用glTexImage2D来填充读取的纹理数据。外部拿到的纹理对象是一个GLuint。

```
    GLuint texture;
	// 参数1生成几个，参数2是OpenGL中生成的纹理的标识符。
	glGenTextures(1, &texture);
	// 绑定当前的纹理对象，后面的操作都是针对这个对象。
	// PS:到这了的认知是OpenGL中都是先用API确定一个对象，然后后面的操作都是针对这个确定的对象的。
	glBindTexture(GL_TEXTURE_2D, texture);
	// 设置texture参数，
	// 参数说明：设置的是2d纹理，当一个纹理被放大时（128的图放到512的多边形上面）采用什么算法来采用，采用线性过滤
	// GL_NEAREST是采样是取最近的像素点的值，如果有多个应该是有个策略取的。
	glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MAG_FILTER, GL_LINEAR);
	// 一样，只不过设置的是当一个大图放到小的多边形上时如何过滤
	glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MIN_FILTER, GL_LINEAR);
	// 如果模型uv坐标超出1.0怎么采样，
	// GL_CLAMP设置的是到纹理坐标为1.0的位置取
	// GL_REPEAT时，是uv.u取小数部分，比如1.1，那么采样就是0.1的位置。
	glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_WRAP_S, GL_REPEAT);
	glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_WRAP_T, GL_REPEAT);
	// 2d, 用mipmap哪个层级，纹理数据在显卡上是什么格式，宽、高、0、纹理数据在内存是什么格式，像素中每个分量（rgba）是什么格式，像素数据源
	glTexImage2D(GL_TEXTURE_2D, 0, type, width, height, 0, type, GL_UNSIGNED_BYTE, pixelData);
	// 当前纹理重绑定，等于上面设置的不要再改了。
	glBindTexture(GL_TEXTURE_2D, 0);
```

- 纹理对象使用
    -  通过使用`glBindTexture(GL_TEXTURE_2D, texture);`来确定当前要操作的纹理是哪一个。
    -  设置四边形，然后开始设置点的位置和采样UV。

```
    glEnable(GL_TEXTURE_2D);
	glBindTexture(GL_TEXTURE_2D, texture);
	glBegin(GL_QUADS); // 设置一个面片
	glColor4ub(255, 255, 255, 255); // 面片背景色。最终颜色是这个颜色和纹理的混合
	glTexCoord2f(0.0f, 0.0f); // 第一个点纹理坐标（uv采样位置），00是纹理的左下角
	glVertex3f(-0.1f, -0.1f, -0.4f); // 第一个点空间坐标位置
	//2
	glTexCoord2f(1.0f, 0.0f);
	glVertex3f(0.1f, -0.1f, -0.4f);
	//3
	glTexCoord2f(1.0f, 1.0f);
	glVertex3f(0.1f, 0.1f, -0.4f);
	//4
	glTexCoord2f(0.0f, 1.0f);
	glVertex3f(-0.1f, 0.1f, -0.4f);
	glEnd();
```

