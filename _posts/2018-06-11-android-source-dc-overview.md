---
layout: article
title: "Android 系统架构 —— 数据通信篇 概览"
key: "Android 系统架构 —— 数据通信篇 概览" 
tags: AndroidFramework
aside:
  toc: true
---
## 前言
任何一个操作系统, 都需要解决线程和进程之间数据交互的问题, 基于 Linux 内核的 Android 自然也不例外, 它能够使用的跨进程通信手段如下
- 线程间通信
  - Handler
- 进程间通信
  - Android 特色
    - Binder 驱动
    - Ashmem 共享内存
  - Linux 支持
    - 信号
    - pipe
    - Socket
    - 消息队列
    - 共享内存 + 信号量

关于 Linux 的支持的, 这里就不再赘述了, 是属于操作系统的基础知识, 感兴趣的可以自行了解, 这里主要探究一下 Android 相关的通信方式

<!--more-->

## 一. Handler 
Handler 我们平时用的是最多的, 在线程之间传输数据, 一般都用它来完成, 它也是三者之中最容易学习的

### 学习路线
- [Looper 的创建与启动](https://sharrychoo.github.io/blog/2018/06/12/android-source-dc-handler1.html)
- [消息的发送与处理](https://sharrychoo.github.io/blog/2018/06/13/android-source-dc-handler2.html)
- [MessageQueue 同步屏障技术](https://sharrychoo.github.io/blog/2018/06/14/android-source-dc-handler3.html)

## 二. Binder 驱动
Binder 这个名词我们经常能够听到, 也知道 AIDL 是对 Binder 的封装, 但是 Binder 到底是什么呢? 

这个问题在我初始看 Android 源码的时候, 一度困扰着我, 对它的概念一直是处于模糊的状态, 其实很简单 **Binder 是一个驱动程序**

### 什么是 Binder 驱动
Binder 驱动并非鼠标、键盘、硬盘等与硬件相关联的驱动程序, 它是一个纯软件实现的驱动程序

### Binder 驱动的作用
Binder 驱动用于在 Android 中提供**高效**、**便捷**和**轻量级**跨进程通信服务
- 高效: 使用 mmap 实现用户空间和内核的通信, 无需多次拷贝
- 便捷: 提供了 AIDL 的辅助程序, 让我们在 Java 层也可以很方便的进行跨进程模型的搭建
- 轻量级: 无法传递大数据, 理论最大为 4 M(应用进程一般不可超过 1MB - 8k, ServiceManager 不可超过 128 kb), 只能进行小规模的跨进程数据交互

### 使用场景
- 应用进程之间的跨进程通信
- 系统服务进程与服务管理进程之间的交互

### 学习路线
笔者学习 Binder 驱动, 是由浅入深的方式进行
- 应用层
  - [通信实例](https://sharrychoo.github.io/blog/2018/07/01/android-source-dc-binder1.html)
  - [AIDL 与 Binder](https://sharrychoo.github.io/blog/2018/07/05/android-source-dc-binder2.html)
- 运行时库层
  - [IBinder 对象的实例化](https://sharrychoo.github.io/blog/2018/07/07/android-source-dc-binder3.html) 
- Linux 内核层
  - [Binder 驱动](https://sharrychoo.github.io/blog/2018/07/10/android-source-dc-binder4.html)
  - [ServiceManager 启动](https://sharrychoo.github.io/blog/2018/07/15/android-source-dc-binder5.html)
  - [Binder 通信完整流程](https://sharrychoo.github.io/blog/2018/07/25/android-source-dc-binder6.html)

## 三. Asheme 匿名共享内存
**Ashmem(Anonymous Shared Memory) 匿名共享内存是 Android 的 Linux 内核实现的一个驱动**, 它以驱动程序的形式实现在内核空间, 用于在进程间进行数据共享

###  使用场景
在 Android UI 渲染时, 我们需要将我们需要渲染的 Surface 数据发送到 SurfaceFlinger 进程进行最后的绘制, Binder 驱动传输显然是无法满足这么大的数据量, 因此这里使用到了 Ashmem 共享内存

### 相关文章
- [Ashmem 驱动共享内存驱动](https://sharrychoo.github.io/blog/2018/08/05/android-source-dc-ashmem.html)

## 总结
Android 数据交互中 Handler 是开发过程中最为常用的, Binder 是在 Android 系统源码中最为常用的, 进程之间的数据交互, 基本上都是使用 Binder
- 这也是为什么我们在系统启动篇中没有分析 ServiceManager 进程启动的原因, 就是因为它与 Binder 的关联性太强

后面这段时间我们就按照 Handler, Binder 和 Asheme 的顺序依次的学习, 知晓其实现细节, 帮助我们在开发过程中根据场景更好的选用合适数据交互方式