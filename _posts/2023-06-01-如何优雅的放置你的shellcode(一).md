---
title: 如何优雅的放置你的shellcode(一)
date: 2023-06-01
categories: [红蓝对抗]
tags: [红蓝对抗, pe, shellcode]
---

## 0X00 前言 

在我们编写自己的loader的时候，通常会把shellcode放在pe文件的不同地方，例如`.data`、`.rdata` 、`.text` 、`.rsrc` 等地方来实现自己的需求，在这里笔者将探讨这几种方式的实现。

## 0X01 .data区段

`.data` 是pe文件的一个区段，它包含已初始化的全局变量和静态变量，这个区段是可读可写的。当我们把shellcode设置为全局或局部变量，它就会存储在 `.data`  区段中，当然这些具体还要取决于编译器设置。

这里笔者使用cs生成一段shellcode用作演示：

![](https://github.com/ring0rl/blog_pic/blob/main/2023-06-01/1.png?raw=true)

然后将其设置为全局变量并打印出shellcode的内存首地址(方便我们后续研究查看)

![](https://github.com/ring0rl/blog_pic/blob/main/2023-06-01/2.png?raw=true)

将其编译成exe后运行查看我们想要看到的内存地址

![](https://github.com/ring0rl/blog_pic/blob/main/2023-06-01/3.png?raw=true)

这里可以看到我们的shellcode内存首地址为`0x007AA000` ，然后去看内存

![](https://github.com/ring0rl/blog_pic/blob/main/2023-06-01/4.png?raw=true)

可以清楚的看到在`.data` 段，并且是可读可写的。

局部变量也是一样的，笔者在此不做赘述。

## 0X02 .rdata区段

`.rdata` 和`.data` 仅仅有一个字母之差，但是他们的实现方式和内存页属性是不一样的，`.rdata` 仅仅是可读的。其实现只需要在前面加一个`const` ，也就是把他变为一个常量即可。

同样我们以上面的shellcode和方法做演示：

![](https://github.com/ring0rl/blog_pic/blob/main/2023-06-01/5.png?raw=true)

我们可以看到`.rdata` 区段仅仅是可读的

![](https://github.com/ring0rl/blog_pic/blob/main/2023-06-01/6.png?raw=true)

可以在其二进制中清楚看到我们的shellcode。

![](https://github.com/ring0rl/blog_pic/blob/main/2023-06-01/7.png?raw=true)

## 0X03 .text区段

`.text` 区段与前面的`.data` 和`.rdata` 有所不同，`.data` 和`.rdata`只需要声明变量即可，而`.text` 则需要我们"告诉编译器"：我们要将shellcode放在`.text` 区段中才可以。

![](https://github.com/ring0rl/blog_pic/blob/main/2023-06-01/8.png?raw=true)

与`.data` 的可读可写和`.rdata` 的可读不同的是，`.text` 区段具有内存页的执行权限，从下图中可以验证，我们的`.text` 区段具有执行和读的权限。

![](https://github.com/ring0rl/blog_pic/blob/main/2023-06-01/9.png?raw=true)

## 0x04 小结

​	本文介绍了在pe中放置shellcode的四种办法中的三种，探讨了怎么在`.data` 区段、`.rdata`  区段和`.text` 区段存放shellcode的方法，`.data` 和 `.rdata` 二者的区别也仅仅是一个`const` 关键词，而`.text` 区段则需要我们去"告诉"编译器。三者的内存页权限也是不同的，对此读者可以根据自己的需求去选择使用哪一种方式来"优雅"的放置自己的shellcode。而对于`.rsrc` 区段的实现方法，笔者会在后续的博客中进行探讨。

### 注：本文仅用于安全研究和学习之用