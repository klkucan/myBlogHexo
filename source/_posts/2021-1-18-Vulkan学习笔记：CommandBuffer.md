---
layout: post
title: "Vulkan学习笔记：CommandBuffer"
date: 2021-1-18 21:18:13 +0800
thumbnail: "img/home-bg-o.jpg"
categories:
- 游戏开发
tags:
- Vulkan
---

## 什么是commandbuffer

- CPU发给GPU的命令，用于两者之间的通信。
- 通过Graphic Queue、Present Queue等队列发送到GPU
- 它是有顺序的

## 使用的流程

### 创建commandbuffer

<!--more-->

```cpp
// count：生成几个commandbuffer，level：多线程渲染时使用
void xGenCommandBuffer(VkCommandBuffer* commandbuffer, int count, VkCommandBufferLevel level /* = VK_COMMAND_BUFFER_LEVEL_PRIMARY*/)
{
	VkCommandBufferAllocateInfo cbai = {};
	cbai.sType = VK_STRUCTURE_TYPE_COMMAND_BUFFER_ALLOCATE_INFO;
	cbai.level = level;
	cbai.commandPool = GetCommandPool();
	cbai.commandBufferCount = count;
	vkAllocateCommandBuffers(GetVulkanDevice(), &cbai, commandbuffer);
}
```

### 开始和结束commandbuffer

- VkCommandBufferBeginInfo设置了commandbuffer的打开方式

```cpp
void xBeginOneTimeCommandBuffer(VkCommandBuffer* commandbuffer)
{
	// 生成一个commandbuffer
	xGenCommandBuffer(commandbuffer, 1);
	VkCommandBufferBeginInfo cbbi = {};
	cbbi.sType = VK_STRUCTURE_TYPE_COMMAND_BUFFER_BEGIN_INFO;
	// 这个flag设定使得commandbuffer是一个onetime
	cbbi.flags = VK_COMMAND_BUFFER_USAGE_ONE_TIME_SUBMIT_BIT;
	// 开始设置commandbuffer
	vkBeginCommandBuffer(*commandbuffer, &cbbi);
}

void xEndOneTimeCommandBuffer(VkCommandBuffer& commandbuffer)
{
	// commandbuffer设置完毕，等待发送到GPU
	vkEndCommandBuffer(commandbuffer);
	// 等待commandbuffer完成
	xWaitForCommmandFinish(commandbuffer);
	// 销毁commandbuffer
	vkFreeCommandBuffers(GetVulkanDevice(), GetCommandPool(), 1, &commandbuffer);
}
```

### 等待GPU执行

- 上面的步骤主要完成了commandbuffer的创建和设置，但是并没有真正的发送到GPU去执行。
- CPU和GPU之间的API都是异步的，因此需要等待GPU完成了某个动作后才执行后续的代码，因此存在一个等待的问题。
- Vulkan中使用VkFence作为同步的标记

```cpp
// 发送commandbuffer，等待完成
void xWaitForCommmandFinish(VkCommandBuffer commandbuffer)
{
	// commandbuffer提交信息，或者说是如何打开
	VkSubmitInfo submitinfo = {};
	submitinfo.sType = VK_STRUCTURE_TYPE_SUBMIT_INFO;
	submitinfo.commandBufferCount = 1;
	submitinfo.pCommandBuffers = &commandbuffer;
	// fence是一种同步的方式，它在整个命令管线里做一个标记，当GPU执行到了这个标记后，fence变成已被执行了的状态，
	// 也意味着fence前面的commandbuffer中的指令也全部已经被执行完成了。
	VkFence fence;
	VkFenceCreateInfo fenceinfo = {};
	fenceinfo.sType = VK_STRUCTURE_TYPE_FENCE_CREATE_INFO;
	// 创建一个标记
	vkCreateFence(GetVulkanDevice(), &fenceinfo, nullptr, &fence);
	// 提交commandbuffer或者信号量到GPU
	// args:提交路线，VkSubmitInfo结构体数量，VkSubmitInfo实例，fence
	vkQueueSubmit(GetGraphicQueue(), 1, &submitinfo, fence);
	// 等待commandbuffer执行完成：waitAll参数true是等待所有的fence执行完，false是等待的fence任一一个完成就执行后面的代码
	// 从参数看，一个wait可以等待多个fence
	vkWaitForFences(GetVulkanDevice(), 1, &fence, VK_TRUE, 1000000);
	vkDestroyFence(GetVulkanDevice(), fence, nullptr);
}
```

## 总结:

- 基本使用套路
  - 到此为止可以发现在Vulkan编程中，创建对象总是需要先准备一个XXXInfo的结构体
  - 然后根据结构体来创建对象，比如前面提到的VkBuffer、VkMemory，本文中的VkCommandBufferAllocateInfo等。
- 命名的规则
  - 可以看到对于对象创建有两种Info：VkXXXCreateInfo和VkXXXAllocateInfo
  - 对应的释放资源也是两种：VkDestroyXXX和VkFreeXXX
- 背后的内存位置
  - 这两组API格式背后的含义虽然课程中还没有提到，但是大胆推测一下应该是所有抽象对象的都是用的Create和Destroy；而所有真实的在GPU中的对象都是Allocate和Free。
  - 总结一下目的对象有：
    - VkBuffer
    - VkMemory
    - VkCommandBuffer
    - VkFence