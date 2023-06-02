---
title: 如何优雅的放置你的shellcode(二)
date: 2023-06-02
categories: [红蓝对抗]
tags: [红蓝对抗, pe, shellcode]
---

## 0X00 前言 

在[昨天的文章](https://ring0rl.github.io/posts/%E5%A6%82%E4%BD%95%E4%BC%98%E9%9B%85%E7%9A%84%E6%94%BE%E7%BD%AE%E4%BD%A0%E7%9A%84shellcode(%E4%B8%80)/)中我们探讨了shellcode放置的几种方式，今天将接着昨天的文章，完成我们对`.rsrc` 区段的探讨。在我们实际的shelcode放置中，这也是一种比较好的方法，因为这个区段的优点有很多，比如可以存放更多的shellcode。

## 0X01 rsrc实现

我们先添加一个`.rc` 资源文件

![](https://raw.githubusercontent.com/ring0rl/blog_pic/main/2023-06-02/1.png)

然后导入一个ico

![](https://raw.githubusercontent.com/ring0rl/blog_pic/main/2023-06-02/2.png)

在到这一步的时候，我们输入`RCDATA` 

![](https://raw.githubusercontent.com/ring0rl/blog_pic/main/2023-06-02/3.png)

导入之后，我们的shellcode便会以二进制展示出来

![](https://raw.githubusercontent.com/ring0rl/blog_pic/main/2023-06-02/4.png)

同时会出现一个`resource.h` 的头文件，这个文件包含一个定义语句，这个语句能够引用资源部分中的shellcode ID

![](https://raw.githubusercontent.com/ring0rl/blog_pic/main/2023-06-02/5.png)

当我们编译后，shelcode将存储在该`.rdata` 区段，但无法直接访问。这个时候我们必须使用多个 Win32API 来访问。

| [FindResourceW](https://learn.microsoft.com/zh-cn/windows/win32/api/libloaderapi/nf-libloaderapi-findresourcew) | 获取传入ID的资源段中指定数据的存储位置（这个在头文件中定义）  |
| ---------------------------------------- | -------------------------------- |
| [**LoadResource**](https://learn.microsoft.com/zh-cn/windows/win32/api/libloaderapi/nf-libloaderapi-loadresource) | **检索可用于获取指向内存中指定资源的第一个字节的指针的句柄** |
| [**LockResource**](https://learn.microsoft.com/zh-cn/windows/win32/api/libloaderapi/nf-libloaderapi-lockresource) | **检索指向内存中指定资源的指针**               |
| [**SizeofResource**](https://learn.microsoft.com/zh-cn/windows/win32/api/libloaderapi/nf-libloaderapi-sizeofresource) | **返回指定资源的字节数大小**                 |

```c
#include <stdio.h>
#include <Windows.h>
#include "resource.h"


int main() {

	HRSRC		hRsrc = NULL;
	HGLOBAL		hGlobal = NULL;
	PVOID		pPayloadAddress = NULL;
	SIZE_T		sPayloadSize = NULL;


	// 获取存储在.rsrc中的数据的位置
	hRsrc = FindResourceW(NULL, MAKEINTRESOURCEW(IDR_RCDATA1), RT_RCDATA);
	if (hRsrc == NULL) {
		printf("FindResourceW Failed With Error : %d \n", GetLastError());
		return -1;
	}

	// 获取句柄
	hGlobal = LoadResource(NULL, hRsrc);
	if (hGlobal == NULL) {
		printf("LoadResource Failed With Error : %d \n", GetLastError());
		return -1;
	}

	// 在.rsrc区段获取shellcode的地址
	pPayloadAddress = LockResource(hGlobal);
	if (pPayloadAddress == NULL) {
		printf("LockResource Failed With Error : %d \n", GetLastError());
		return -1;
}

	// 在.rsrc区段获取shellcode的大小
	sPayloadSize = SizeofResource(NULL, hRsrc);
	if (sPayloadSize == NULL) {
		printf("SizeofResource Failed With Error : %d \n", GetLastError());
		return -1;
	}


	printf("Address is : 0x%p \n", pPayloadAddress);
	printf("Size is : %ld \n", sPayloadSize);

	return 0;
}
```

编译并运行上述代码后，会把shellcode地址及其大小打印出来。但是这个地址位于 `.rsrc` 区段，这个区段是只读的，我们任何更改或编辑其中数据的操作都会导致访问错误。想要编辑shellcode，我们只能分配一个与shellcode大小相同的临时缓冲区并把shellcode复制过来，然后我们对这个缓冲区进行操作即可。

```c
	// 分配内存
	PVOID pTmpBuffer = HeapAlloc(GetProcessHeap(), 0, sPayloadSize);
	if (pTmpBuffer != NULL) {
		memcpy(pTmpBuffer, pPayloadAddress, sPayloadSize);
	}

	printf("new Address is : 0x%p \n", pTmpBuffer);
```

资源中的内存情况

![](https://raw.githubusercontent.com/ring0rl/blog_pic/main/2023-06-02/6.png)

临时缓冲区内存情况

![](https://raw.githubusercontent.com/ring0rl/blog_pic/main/2023-06-02/7.png)

## 0x02 小结

​	本文接着[昨天的文章](https://ring0rl.github.io/posts/%E5%A6%82%E4%BD%95%E4%BC%98%E9%9B%85%E7%9A%84%E6%94%BE%E7%BD%AE%E4%BD%A0%E7%9A%84shellcode(%E4%B8%80)/) 继续探讨了`.rsrc` 区段放置shellcode的方法，区别于昨天的方法，`.rsrc` 区段的实现方法较为繁琐，需要多个Win32API来辅助我们实现对shellcode的访问。但是效果比较好，在很多时候也是一种行之有效的办法。

### 注：本文仅用于安全研究和学习之用