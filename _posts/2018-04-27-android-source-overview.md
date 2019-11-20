---
layout: article
title: "Android 系统架构 —— 启动篇"
key: "Android 系统架构 —— 启动篇" 
tags: AndroidFramework
aside:
  toc: true
---

## 前言
从 17 年下半年, 学习 Android 系统架构已经有一段时间了, 学习的目的主要有如下几点
- 满足求知欲
- 提升源码阅读能力
- 了解 Android 系统运行方式, 更好的追踪和解决开发中的问题
- 学习 Android 系统中优秀的设计方案, 即时 Android 离开历史的舞台, 我们也能够将一些好的思想记录下来

在学习 Android 系统架构的过程中, 留下印象最深刻的便是, **很难将零散的 Android 知识组织成自己的知识体系**
- 比如在学习系统服务进程时, 因为其涉及到了 Binder 驱动, 于是不得不去补 Binder 驱动的知识, Binder 驱动定义在 Linux 内核中, 不得不去补 Linux 内核的知识

这里笔者记录一下自己的 Android 系统源码的学习之路, 在归纳总结中补缺补差, 希望能够和大家一起进步, 其需要的基础知识如下
- 应用层开发基础
- C/C++ 基础
- Linux 内核机制

<!--more-->

下面从广义上了解一下 Android 系统的组成

## 一. Android 系统架构
![Android 层级图](https://i.loli.net/2019/10/19/BuXSCfDb3hsMd65.png)

Android 是基于 Linux 内核的, 从广义的角度说, 它可以分为 **Linux 内核层**和**用户空间层**

### 一) Linux 内核层
Linux 内核层主要管理底层驱动程序, 用于和设备硬件直接交互, 除了 Linux 内核的进程、内存管理等, 还包含 Android 添加的特色驱动程序如
- Binder: IPC 通信驱动
- Logger: 日志打印驱动
- Ashmem: 共享内存驱动

![Linux 内核层](https://i.loli.net/2019/10/19/9vVfYIx8bu7g5Xo.jpg)

### 二) 用户空间层
用户空间从由底部至上包括
- 硬件抽象层(HAL)
  - 是 Linux 内核与用户空间交互的纽带
- 外部链接库(Native Library)
  - OpenGL
  - SQLite
  - SKIA
  - ......
- 运行时库(Android Runtime)
  - Binder
  - Parcel
  - Ashmem
  - 智能指针
  - ......
- 应用框架层(Android Framework)
  - 即 Android 为应用层开发提供的 Java API
- 应用开发层(Application)

![用户空间层](https://i.loli.net/2019/10/19/bDvXL4MBZwTnzHC.jpg)

## 二. 学习思路
笔者探索过很多 Android 源码的学习思路, 后来发现由底向上的学习是最适合自己的, 因为**底层的知识比较基础**而且**底层的知识点与比较独立, 与其他模块的耦合度较低**

### [一) 数据通信篇](https://sharrychoo.github.io/blog/2018/05/11/android-source-dc-overview.html)
#### 1. Handler 线程间通信
- [Looper 的创建与启动](https://sharrychoo.github.io/blog/2018/05/12/android-source-dc-handler1.html)
- [消息的发送与处理](https://sharrychoo.github.io/blog/2018/05/13/android-source-dc-handler2.html)
- [MessageQueue 同步屏障技术](https://sharrychoo.github.io/blog/2018/05/14/android-source-dc-handler3.html)

#### 2. Binder 驱动
- 应用层
  - [通信实例](https://sharrychoo.github.io/blog/2018/06/01/android-source-dc-binder1.html)
  - [AIDL 与 Binder](https://sharrychoo.github.io/blog/2018/06/05/android-source-dc-binder2.html)
- 运行时库层
  - [IBinder 对象的实例化](https://sharrychoo.github.io/blog/2018/06/07/android-source-dc-binder3.html) 
- Linux 内核层
  - [Binder 驱动](https://sharrychoo.github.io/blog/2018/06/10/android-source-dc-binder4.html)
- [Binder 上下文管理者的启动]
  - 见系统启动中的 [ServiceManager 启动](https://sharrychoo.github.io/blog/2018/06/15/android-source-servicemanager-start.html)
- [Binder 通信完整流程](https://sharrychoo.github.io/blog/2018/06/25/android-source-dc-binder5.html)

#### 3. Asheme 共享内存驱动
- [Ashmem 驱动共享内存驱动](https://sharrychoo.github.io/blog/2018/07/05/android-source-dc-ashmem.html)

### 二) 系统启动
- [Init 进程的启动](https://sharrychoo.github.io/blog/2018/07/30/android-source-init-process-start.html)
- [ServiceManager 启动](https://sharrychoo.github.io/blog/2018/06/15/android-source-servicemanager-start.html)
- [SurfaceFlinger 的启动](https://sharrychoo.github.io/blog/2019/10/11/android-source-graphic-consumer1.html)
   - [SurfaceFlinger Hotplug 的处理](https://sharrychoo.github.io/blog/2019/10/15/android-source-graphic-consumer2.html)
   - [SurfaceFlinger 对 Vsync 信号护理](https://sharrychoo.github.io/blog/2019/10/16/android-source-graphic-consumer3.html)
   - [SurfaceFlinger 渲染图层](https://sharrychoo.github.io/blog/2019/10/17/android-source-graphic-consumer4.html)
- [Zygote 进程的启动](https://sharrychoo.github.io/blog/2018/08/03/android-source-zygote-process-start.html)
- [SystemServer 进程的启动](https://sharrychoo.github.io/blog/2018/08/04/android-source-systemserver-process-start.html)
  - [AMS 的启动](https://sharrychoo.github.io/blog/2018/08/05/android-source-ams_start.html)
  - [PMS 的启动](https://sharrychoo.github.io/blog/2019/11/12/android-source-pkms-launch1.html)
    - [解析备份文件](https://sharrychoo.github.io/blog/2019/11/12/android-source-pkms-launch1.html)
    - [扫描安装目录](https://sharrychoo.github.io/blog/2019/11/13/android-source-pkms-launch2.html)
    - [应用程序的安装](https://sharrychoo.github.io/blog/2019/11/14/android-source-pkms-install.html) 
  - [IMS 的启动](https://sharrychoo.github.io/blog/2019/11/19/android-source-ims-launch.html)
    - [IMS 的事件分发](https://sharrychoo.github.io/blog/2019/11/20/android-source-ims-dispatch.html) 
  - [WMS 的启动](https://sharrychoo.github.io/blog/2018/08/06/android-source-wms-start.html)
- 应用进程
  - Activity 的启动
    - [请求方的暂停](https://sharrychoo.github.io/blog/2018/08/07/android-source-activity-launch1.html)
    - [应用进程的创建](https://sharrychoo.github.io/blog/2018/08/08/android-source-activity-launch2.html)
    - [目标 Activity 的启动](https://sharrychoo.github.io/blog/2018/08/09/android-source-activity-launch3.html)

### [三) 图形架构篇](https://sharrychoo.github.io/blog/2018/08/10/android-source-graphic-overview.html)
- [Window 和 WindowManager 的关系](https://sharrychoo.github.io/blog/2018/08/11/android-source-graphic-producer1.html)
- [Window 与 View 的关系](https://sharrychoo.github.io/blog/2018/08/12/android-source-graphic-producer2.html)
- [ViewRootImpl 与 WMS 的通信](https://sharrychoo.github.io/blog/2018/08/20/android-source-graphic-producer3.html)
- [View 的测量](https://sharrychoo.github.io/blog/2018/09/01/android-source-graphic-producer4.html)
- [ViewRootImpl 的 Surface 创建](https://sharrychoo.github.io/blog/2018/09/20/android-source-graphic-producer5.html)
- [View 的布局](https://sharrychoo.github.io/blog/2018/09/25/android-source-graphic-producer6.html)
- [View 的软件渲染](https://sharrychoo.github.io/blog/2018/10/10/android-source-graphic-producer7.html)
- [View 的硬件渲染](https://sharrychoo.github.io/blog/2019/08/14/android-source-graphic-producer8.html)

### 四) 资源管理篇
待补充

## 三. 学习资料
- <<Android 系统源代码情景分析(第三版)>>
- <<深入理解 Linux 内核>>
- https://blog.csdn.net/Luoshengyang/article/details/8923485
- http://gityuan.com/android/

在个人学习的过程中, 以上的资料对我产生了极大的帮助, 尤其是罗老师 **<<Android 系统源码情景分析>>**, 它几乎陪我度过了 2017 年到 2018 年上半年所有的闲暇时光, 在这里对作者表示衷心的感谢

## 结语
**"独学而无友，则孤陋而寡闻"**, 关于上述的文章, 笔者会花时间花心思去完善, 虽然不是高深的知识, 但也是自己积累的点滴, 能够分享给大家还是还开心的, 也希望大家多多批评帮助笔者一起进步