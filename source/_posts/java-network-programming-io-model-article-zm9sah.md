---
title: Java网络编程-IO模型篇
updated: '2023-03-18 23:34:54'
excerpt: >-
  java网络编程io模型篇io基本概念综述io的分类​io​以不同的维度划分可以被分为多种类型比如可以从工作层面划分成磁盘io​（本地io​）和网络io​_磁盘io​_指计算机本地的输入输出从本地读取一张图片一段音频一个视频载入内存这都可以被称为是磁盘io​。网络io​_指计算机网络层的输入输出比如请求响应下载上传等都能够被称为网络io​。也可以从工作模式上划分例如常听的bionioaio​还可以从工作性质上分为阻塞式io​与非阻塞式io​亦或从多线程角度也可被分为同步io​与异步io​这么看下来是不是感
tags: []
categories: []
permalink: /post/java-network-programming-io-model-article-zm9sah.html
comments: true
---

# Java网络编程-IO模型篇

# IO基本概念综述

## IO的分类

​`IO`​以不同的维度划分，可以被分为多种类型，比如可以从工作层面划分成磁盘`IO`​（本地`IO`​）和网络`IO`​：

* 磁盘`IO`​：指计算机本地的输入输出，从本地读取一张图片、一段音频、一个视频载入内存，这都可以被称为是磁盘`IO`​。
* 网络`IO`​：指计算机网络层的输入输出，比如请求/响应、下载/上传等，都能够被称为网络`IO`​。

也可以从工作模式上划分，例如常听的`BIO、NIO、AIO`​，还可以从工作性质上分为阻塞式`IO`​与非阻塞式`IO`​，亦或从多线程角度也可被分为同步`IO`​与异步`IO`​，这么看下来是不是感觉有些晕乎乎的？没关系，接下来我们对`IO`​体系依次全方位进行解析。

## IO工作原理

无论是Java还是其他的语言，本质上`IO`​读写操作的原理是类似的，编程语言开发的程序，一般都是工作在用户态空间，但由于`IO`​读写对于计算机而言，属于高危操作，所以`OS`​不可能`100%`​将这些功能开放给用户态的程序使用，所以正常情况下的程序读写操作，本质上都是在调用`OS`​内核提供的函数：`read()、 write()`​。  
 也就是说，在程序中试图利用`IO`​机制读写数据时，仅仅只是调用了内核提供的接口函数而已，本质上真正的`IO`​操作还是由内核自己去完成的。

​`IO`​工作的过程如下：

​![image](assets/image-20230316222838-xctg3qz.png)​

* ①首先在网络的网卡上或本地存储设备中准备数据，然后调用`read()`​函数。
* ②调用`read()`​函数厚，由内核将网络/本地数据读取到内核缓冲区中。
* ③读取完成后向`CPU`​发送一个中断信号，通知`CPU`​对数据进行后续处理。
* ④`CPU`​将内核中的数据写入到对应的程序缓冲区或网络`Socket`​接收缓冲区中。
* ⑤数据全部写入到缓冲区后，应用程序开始对数据开始实际的处理。

在上述中提到了一个`CPU`​中断信号的概念，这其实属于一种`I/O`​的控制方式，`IO`​控制方式目前主要有三种：​**忙等待方式****、**​**中断驱动方式以及****`DMA`****直接存储器方式**​，不过无论是何种方式，本质上的最终作用是相同的，都是读取数据的目的。

在上述`IO`​工作过程中，其实大体可分为两部分：​**准备阶段****和**​**复制阶段**​，准备阶段是指数据从网络网卡或本地存储器读取到内核的过程，而复制阶段则是将内核缓冲区中的数据拷贝至用户态的进程缓冲区。常听的`BIO、NIO、AIO`​之间的区别，就在于这两个过程中的操作是同步还是异步的，是阻塞还是非阻塞的。

## 内核态和用户态

​![image](assets/image-20230316223043-r1l77d6.png)​

​`Linux`​为了确保系统足够稳定与安全，因此在运行过程中会将内存划分为内核空间与用户空间，其中运行在用户空间的程序被称为“用户态”程序，同理，运行在“内核态”的程序则被称为“内核态”程序，而普通的程序一般都会运行在用户空间。

那么系统为什么要这样设计呢？因为如果内核与用户空间都为同一块儿，此时假设某个程序执行异常导致崩溃了，最终会导致整个系统也出现崩溃，而划分出两块区域的目的就在于：用户空间中的某个程序崩溃，那自会影响自身，而不会影响系统整体的运行。

> 同时为了防止普通程序去进行`IO`​、内存动态调整、线程挂起等一些高危操作引发系统崩溃，因此这些高危操作的具体执行，也只能由内核自己来完成，但程序中有时难免需要用到这些功能，因此内核也会提供很多的函数/接口提供给外部调用。

当处于用户态的程序调用某个内核提供的函数时，此时由于用户态自身不具备这些函数的执行权限，因此会发生用户态到内核态的切换，也就是说：当程序调用某个内核提供的函数后，具体的操作会切换成内核自己去执行。

> 但用户态与内核态切换时，由于需要处理操作句柄、保存现场、执行系统调用、恢复现场等等过程，因此状态切换其实也是一个开销较大的动作，因此在设计程序时，要尽量减少会发生状态切换的事项，比如Java中，解决线程安全能用`ReetrantLock`​的情况下则尽量不使用`Synchronized`​。

最后对于用户态和内核态的区别，用大白话来说就是：类似于做程序开发时，**普通用户和管理员的区别**，为了防止普通用户到处乱点，从而导致系统无法正常运转，因此有些权限只能开放给管理员身份执行，例如删库~

# NIO详解

Java NIO整体架构图

​![image](assets/image-20230316224700-5uto182.png)​

观察上述结构，其实`Buffer、Channel`​的定义并不算复杂，仅是单纯的三层结构，因此对于源码这块不再去剖析，有兴趣的根据给出的目录结构去调试源码，自然也能摸透其原理实现。

而最关键的是`Selector`​选择器，它是整个`NIO`​体系中较为复杂的一块内容，同时它也作为`Java-NIO`​与内核多路复用模型的“中间者”，但在上述体系中，却出现了之前未曾提及过的`SelectorProvider`​系定义，那么它的作用是干嘛的呢？主要目的是用于创建选择器，在Java中创建一般是通过如下方式：

```java
// 创建Selector选择器
Selector selector = Selector.open();

// Selector类 → open()方法
public static Selector open() throws IOException {
    return SelectorProvider.provider().openSelector();
}
```

从源码中可明显得知，选择器最终是由`SelectorProvider`​去进行实例化，不过值得一提的是：`Selector`​的实现是基于工厂模式与`SPI`​机制构建的。对于不同`OS`​而言，其对应的具体实现并不相同，因此在`Windows`​系统下，我们只能观测到`WindowsSelectorXXX`​这一系列的实现，而在`Linux`​系统时，对于的则是`EPollSelectorXXX`​这一系列的实现，所以要牢记的是，​**`Java-NIO`**​**在不同操作系统的环境中，提供了不同的实现**​，如下：

* ​`Windows`​：`select`​
* ​`Unix`​：`poll`​
* ​`Mac`​：`kqueue`​
* ​`Linux`​：`epoll`​

## 文件描述符

到目前可得知：Java中的`NIO`​最终会依赖于操作系统所提供的多路复用函数去实现，而`Linux`​系统下对应的则是`epoll`​模型，但`epoll`​的前身则是`select、poll`​，因此我们先分析`select、poll`​多路复用函数，再分析其缺点，逐步引出`epoll`​的由来，最终进一步对其进行全面剖析。

相信大家在学习`Linux`​时，都听说过“**`Linux`**​**本质上就是一个文件系统**”这句话，在`Linux-OS`​中，万事万物皆为文件，连网络连接也不例外，因此在分析多路复用模型之前，咱们首先对这些基础概念做一定了解。

在上述中提到过：`Linux`​的理念就是“一切皆文件”，在`Linux`​中几乎所有资源都是以文件的形式呈现的。如磁盘的数据是文件，网络套接字是文件，系统配置项也是文件等等，所有的数据内容在`Linux`​都是通过文件系统来管理的。  
 既然所有的内容都是文件，那当我们要操作这些内容时，又该如何处理呢？为了方便系统执行，`Linux`​都是通过文件描述符`File Descriptor`​对文件进行操作，对于文件描述符这个概念可以通过一个例子来理解：

```java
Object obj = new Object();
```

上述是Java创建对象的一行代码，类比`Linux`​的文件系统，后面`new Object()`​实例化出来的对象可以当成是具体的文件内容，而前面的引用`obj`​则可理解为是文件描述符。`Linux`​通过`FD`​操作文件，其实本质上与`Java`​中通过`reference`​引用操作对象的过程无异。

> 当出现网络套接字连接时，所有的网络连接都会以文件描述符的形式在内核中存在，也包括后面会提及的多路复用函数`select、poll、epoll`​都会基于`FD`​对网络连接进行操作，因此先阐明这点，作为后续分析的基础。


