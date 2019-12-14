---
title: Android 系统架构 —— 图形架构篇 概览
permalink: android-source/graphic-overview
key: android-source-graphic-overview
tags: AndroidFramework
---

## 前言
Android 的图像处理架构是非常庞大且复杂的, 其核心的设计思想为**生产者-消费者**模式

![SurfaceFlinger 渲染流程](https://i.loli.net/2019/12/14/3oW1BfFqnzKm9bc.png)

- **生产端**
  -  Surface 是渲染数据的载体, Surface 代表 BufferQueue 的生产者
- **消费端**
  - SurfaceFlinger 作为负责绘制应用 UI 的核心, 运行在单独的进程中, 用于处理 Surface 的合成, 是图形架构的消费者

<!--more-->

## 一. 图形渲染架构
Android 的图形渲染是一个生产者消费者模型, Android 图形渲染的组件结构图如下

![图形渲染架构](https://i.loli.net/2019/10/23/iGaKNuE63vpqBwJ.png)

### 一) 图像生产者
生产者为 Media Player 视频解码器, OpenGL ES 等产生的图像缓存数据, 他们通过 BufferData 的形式传递到缓冲队列中
- OpenGL ES: 通过渲染管线之后, 将数据输出到 EGLSurface 中绑定的 GraphicBuffer 中
- Media Player: 解码视频数据, 获取到 YUV 视频帧
- Camera Preview: 获取预览数据, 输出到 SurfaceTexture 上

### 二) 图像消费者
图像数据的消耗者主要是 SurfaceFlinger, 该系统服务会消耗 Surface 提供的数据缓冲流, 并且使用窗口管理器中提供的数据, 把他们合并输出绘制到屏幕上

### 三) 窗口管理器
**控制窗口的 Android 系统服务，它是视图容器。**

窗口总是由 Surface 提供支持。该服务会监督生命周期、输入和聚焦事件、屏幕方向、转换、动画、位置、变形、Z-Order 以及窗口的其他许多方面。

窗口管理器会将所有窗口元数据发送到 SurfaceFlinger，以便 SurfaceFlinger 可以使用该数据在显示部分合成 Surface。

### 四) 各个组件之间的映射关系
- **画笔**: Skia 和 OpenGL, 我们通过 Canvas API 进行绘制最终都会调用到外部链接库的 Skia 和 OpenGL
  - Skia: 2D 图像绘制, 关闭硬件加速时使用该引擎
  - OpenGL: 2D/3D 图像绘制, 开启硬件加速时使用该引擎

- **画纸**: Surface, Android 中所有的元素都在 Surface 这张画纸上进行绘制
  - Window 是 View 的容器, **每个 Window 会关联一个 Surface**
  - WindowManager 用于管理所有的 Window
    - 它将 Window 的 Surface 信息传递给 Graphic Buffer
    - 将 Window 其他数据信息发送给 SurfaceFlinger

- **画板**: Graphic Buffer, 它用于图像数据的缓冲, 将数据发送给 SurfaceFlinger 进行绘制
  - 4.1 之前使用双缓冲机制, 4.1 之后使用三缓冲机制 

- **显示**: SurfaceFlinger, 它将 WindowManager 提供的所有的 Surface, 通过硬件合成输出到屏幕上


## 二. 图形运算引擎
Android 图像运算的方式主要有两种一是 CPU 运算, 另一种是 GPU 运算

### 一) CPU 运算
Android CPU 渲染引擎框架为 **[Skia](https://skia.org/)**, 它是一款在底端设备上呈现高质量的 2D 跨平台图形框架, Google 的 Chrome、Flutter 内部都有使用这个图形渲染框架

### 二) GPU 运算
**通过 GPU 进行图形运算, 称之为硬件加速**

#### 1. OpenGL
市面上最常用于图形运算的引擎莫过于 [OpenGL](https://developer.android.com/guide/topics/graphics/opengl) 了, Android 系统架构中的外部链接库中有 OpenGL ES 的依赖, 并且提供了应用层的 API, 用于做高性能的 2D/3D 图形渲染, Android 中对 OpenGL 的支持如下

##### OpenGL 版本支持
Android 版本 | OpenGL ES 支持
---|---
Android 1.0 | OpenGL ES 1.0、1.1
Android 2.2 | OpenGL ES 2.0
Android 4.3 | OpenGL ES 3.0
Android 5.0 | OpenGL ES 3.1
Android 7.0 | OpenGL ES 3.2

##### OpenGL API 支持
![OpenGL API 支持](https://i.loli.net/2019/10/23/rqv5mWc3YXlF4Vz.png)

#### 2. Vulkan
Android 7.0 之后除了添加 OpenGL ES3.2 的支持, 同时添加了 Vulkan 图像引擎, Vulkan 是用于高性能 3D 图形的低开销、跨平台 API, 它与 OpenGL 不同, 它被添加到 Android 运行时库中, 目前支持面稍窄

## 三. 学习思路
### 生产进程
- [图形渲染的准备](https://sharrychoo.github.io/blog/android-source/graphic-ready)
  - [Choreographer 的工作机制](https://sharrychoo.github.io/blog/android-source/graphic-choreographer)
- View 的 Traversals 操作
  - [测量与布局](https://sharrychoo.github.io/blog/android-source/graphic-view-traversals-measure-layout)
  - [Surface 的创建](https://sharrychoo.github.io/blog/android-source/graphic-surface-create)
  - [软件渲染](https://sharrychoo.github.io/blog/android-source/graphic-draw-software)
  - [硬件渲染](https://sharrychoo.github.io/blog/android-source/graphic-draw-hardware)

### 消费进程
- [SurfaceFlinger 的启动](https://sharrychoo.github.io/blog/android-source/surfaceflinger-launch)
- [SurfaceFlinger Hotplug 的处理](https://sharrychoo.github.io/blog/android-source/surfaceflinger-hotplug)
- [SurfaceFlinger Vsync 信号处理](https://sharrychoo.github.io/blog/android-source/surfaceflinger-vsync-dispatch)