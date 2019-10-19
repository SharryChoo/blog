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

这里笔者记录一下自己的 Android 系统的学习方式, 其需要的基础知识如下
- 应用层开发基础
- C/C++ 基础
- Linux 内核机制

<!--more-->

接下来我们从广义上了解一下 Android 系统的组成

## Android 系统架构
![Android 层级图](https://i.loli.net/2019/10/19/BuXSCfDb3hsMd65.png)

Android 是基于 Linux 内核的, 从广义的角度说, 它可以分为 **Linux 内核层**和**用户空间层**

### Linux 内核层
Linux 内核层主要管理底层驱动程序, 用于和设备硬件直接交互, 除了 Linux 内核的进程、内存管理等, 还包含 Android 添加的特色驱动程序如
- Binder: IPC 通信驱动
- Logger: 日志打印驱动
- Ashmem: 共享内存驱动

![Linux 内核层](https://i.loli.net/2019/10/19/9vVfYIx8bu7g5Xo.jpg)

### 用户空间层
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

## 学习思路
笔者学习的思路主要是流程分析法, 先走通一条 Line, 然后抽丝剥茧的学习重要的功能实现
- 系统启动篇
  - Init 进程的启动
  - Zygote 进程的启动
    - 系统服务进程的启动
    - 应用进程的启动
  - 服务管理进程的启动
    - 服务管理进程为 Binder 驱动的上下文管理者, 因此需要和 Binder 驱动的知识串联
- 数据交互篇Å
  - Handler 线程间通信
  - Binder 进程间通信
  - Asheme 进程间通信
- 图形架构篇
  - 生产者
  - 消费者
- 输入系统篇
  - 窗体的事件分发
  - View 的事件分发
- 流程分析篇
  - Activity 启动
  - Service 启动/绑定
  - 应用的安装
  - 资源管理
- 虚拟机
  - Dalvik
  - ART

## 参考
- http://gityuan.com/android/
- https://blog.csdn.net/Luoshengyang/article/details/8923485