---
layout: post
title: "Vulkan学习笔记：CPU数据写入GPU"
date:   2021-1-12 21:18:13 +0800
thumbnail: "img/home-bg-o.jpg"
categories:
- 游戏开发
tags:
- Vulkan
---

## 从物理硬件获取内存类型索引

- GPU里有存储纹理的内存、存储通用数据的内存。
- 当申请一个vkBuffer的时候，它只是定义了一个逻辑上的对象，真正需要开辟物理内存时，需要根据要存储的是对象类型来决定给与它什么样的内存。
- 获取类型有两个条件
  - 32位掩码：来自VkMemoryRequirements
  - VkMemoryPropertyFlags：这个设置代表着内存的访问权限，是GPU访问还是CPU访问。当CPU可见时，GPU访问会很慢。

<!--more-->

```cpp
uint32_t xGetMemoryType(uint32_t type_filters, VkMemoryPropertyFlags properties)
{
	VkPhysicalDeviceMemoryProperties memory_propertice;
	// 获取物理显卡上所有的内存信息描述的集合
	vkGetPhysicalDeviceMemoryProperties(GetVulkanPhysicalDevice(), &memory_propertice);
	for (uint32_t i = 0; i < memory_propertice.memoryTypeCount; i++)
	{
		uint32_t flag = 1 << i;
		if ((type_filters & flag) && (memory_propertice.memoryTypes[i].propertyFlags & properties) == properties)
		{
			return i;
		}
	}
	return 0;
}
```

## 从GPU获取一块内存

### 流程

- 申请一个buffer
  - 定义VkBufferCreateInfo，给出size
  - 定义VkBuffer对象：用于管理一块GPU物理内存的逻辑对象
  - 调用vkCreateBuffer
- 获取这个buffer的内存请求信息
  - 定义VkMemoryRequirements：GPU上的物理内存是被分成各种类型的，存储纹理的，存储VBO的
  - 调用vkGetBufferMemoryRequirements获取info
- 获取分配器信息
  - 定义VkMemoryAllocateInfo
  - 设置allocationSize和memoryTypeIndex
  - 定义VkDeviceMemory对象：代表是一块GPU上的真实内存
- 调用vkAllocateMemory获取内存
- 调用vkBindBufferMemory把VkBuffer对象和VkDeviceMemory对象绑定

### 代码

- 其中FindMemoryType后续补完

```cpp
VkResult xGenBuffer(VkBuffer &buffer, VkDeviceMemory &buffermemory, VkDeviceSize size, VkBufferUsageFlags usage, VkMemoryPropertyFlags properties)
{
	// vulkan中创建一个对象，都是先要创建一个info，用法就是VkXXXCreateInfo
	VkBufferCreateInfo bufferinfo = {};
	bufferinfo.sType = VK_STRUCTURE_TYPE_BUFFER_CREATE_INFO; // 显然是一个buffer创建的信息
	bufferinfo.size = size;
	bufferinfo.usage = usage;

	// 调用API创建buffer
	VkResult ret = vkCreateBuffer(GetVulkanDevice(), &bufferinfo, nullptr, &buffer);
	if (ret != VK_SUCCESS)
	{
		printf("faild to create buffer\n");
		return ret;
	}

	// 如果成功了就分配内存
	// 这里很有意思，buffer创建成功并没有分配内存，似乎只是告诉GPU我要个buffer，你看行不行，可以的话再去要内存。
	VkMemoryRequirements requirements;
	// vulkan根据buffer给requirements写值，意思就是获取到了这个buffer对于内存的需求
	vkGetBufferMemoryRequirements(GetVulkanDevice(), buffer, &requirements);

	VkMemoryAllocateInfo memoryAllocateInfo = {};
	memoryAllocateInfo.sType = VK_STRUCTURE_TYPE_MEMORY_ALLOCATE_INFO;
	// 这里需要注意，创建buffer的时候有个size，那个只是真实需要的大小。但是显存是有内存对齐的，比如对齐数是256KB，
	// 那么即使我只申请了10KB，最少也会给我256KB。
	memoryAllocateInfo.allocationSize = requirements.size;
	memoryAllocateInfo.memoryTypeIndex = xGetMemoryType(requirements.memoryTypeBits, properties);

	// 真正申请内存
	ret = vkAllocateMemory(GetVulkanDevice(), &memoryAllocateInfo, nullptr, &buffermemory);
	if (ret != 0)
	{
		printf("faild to allocate memory\n");
		return ret;
	}

	// 最后，把内存与buffer绑定到一起
	// 注意 ：Vulkan中的bind纯粹就是把两者关联起来，没有状态机的概念。这也是Vulkan适合多线程的地方，它是没有状态的。
	//		 opengl中bind一般都是设置当前的渲染状态，比如BindTexture，BindProgram
	vkBindBufferMemory(GetVulkanDevice(), buffer, buffermemory, 0);
	return VK_SUCCESS;
}
```

## 向GPU写入数据

### 需要知道的点

- Vulkan把memory分成两种，一种是hostmemory，就是CUP可见的内存区域；第二种是device memory，也就是显存。当显卡没有显存时，需要使用内存给它当显存用，通常嵌入式设备上就是这么干的，还比如集成显卡。
- 在使用Vulkan API的时候会发现很多地方都用到一个参数VkAllocationCallbacks，这个回调的用途是让程序员有机会去自己分配host memory。如果传空，Vulkan就会调用它自己实现的分配host memory的逻辑。通常，设置这个回调是为了debug，监测内存的使用情况，调试程序。
- CPU无法直接向显存写入数据，它只能将数据写入到显存外的一块CPU和GPU都可见、可读取的内存中，然后发图形命令让GPU去这块内存copy数据到显存。

### 流程

- 将数据写到一块不是显存的内存块中
  - 创建一块Vulkan管理的上的buffer和memory，但是这块内存不在显存，因为显卡会拿一部分内存自己用，这就是个中转区域，传递的时候用DMA控制器，不需要耗费cpu的时钟。
  - 将一个VkDeviceMemory映射到一个实际的写入物理地址，这地址是应用程序可以识别或者说可以使用的。参考：[vkMapMemory(3)](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/man/html/vkMapMemory.html)

  - 写出数据
  - 解除绑定
- 使用commandbuffer写入GPU
  - 创建commandbuffer
  - 调用vkCmdCopyBuffer让GPU来copy内存
  - 清理临时数据

```cpp
// 拷贝数据从CPU到GPU
// 注意：整体来说从CPU传递数据到GPU是一个间接的、异步的过程。有点类似零拷贝的逻辑，利用DMA将内核态两块数据进行传递。
void xBufferSubData(VkBuffer buffer, VkBufferUsageFlags usage, const void* data, VkDeviceSize size)
{
	VkBuffer tempBuffer;
	VkDeviceMemory tempMemory;
	// 临时申请一块内存，这块内存应该是Vulkan管理的，CPU可见的。
	// VK_MEMORY_PROPERTY_HOST_COHERENT_BIT | VK_MEMORY_PROPERTY_HOST_VISIBLE_BIT，内存属性为cpu可见的，连贯的内存。
	xGenBuffer(tempBuffer, tempMemory, size, usage, VK_MEMORY_PROPERTY_HOST_COHERENT_BIT | VK_MEMORY_PROPERTY_HOST_VISIBLE_BIT);
	void* host_memory;
	// 将一个VkDeviceMemory映射到一个实际的写入物理地址，往物理地址写值时，就写进这个VkDeviceMemory里面了。
	// 最后一个参数是绑定完成后，写入数据的起始位置
	vkMapMemory(GetVulkanDevice(), tempMemory, 0, size, 0, &host_memory);
	// 把CPU数据copy到这块临时数据
	memcpy(host_memory, data, (size_t)size);
	vkUnmapMemory(GetVulkanDevice(), tempMemory);

	// 使用commandbuffer将临时区域的数据传递到GPU
	VkCommandBuffer commandbuffer;
	aBeginOneTimeCommandBuffer(&commandbuffer);
	// args: 看源码吧，很清晰
	VkBufferCopy copy = { 0, 0, size };
	vkCmdCopyBuffer(commandbuffer, tempBuffer, buffer, 1, &copy);
	aEndOneTimeCommandBuffer(commandbuffer);
	vkDestroyBuffer(GetVulkanDevice(), tempBuffer, nullptr);
	vkFreeMemory(GetVulkanDevice(), tempMemory, nullptr);
}
```

## 总结:
- 到此，完成了CPU数据到GPU数据写入的过程，后两段代码分别对标了OpenGL中的glGenBuffers和glBufferSubData。
- 一个buffer是干什么用的，它对应的物理内存的位置是哪里，由VkBufferUsageFlags和VkMemoryPropertyFlags决定。
- vkCmdCopyBuffer是可以任意拷贝的，从CUP到GPU，从GPU到GPU，从GPU到CUP。只看参数的设置。从这个API可以看到Vulkan的高自由度。
- 学习Vulkan，可以看到OpenGL背后的细节