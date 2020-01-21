---
title: Android 系统架构 —— 图形架构篇 概览
permalink: android-source/graphic-overview
key: android-source-graphic-overview
tags: AndroidFramework
sidebar:
  nav: android-source
---

## 前言
Android 的图像处理架构是非常庞大且复杂的, 并且经历了好几个版本的变更, 相关信息如下

Android 版本 | 渲染变更
---|---
Android 3.0 阶段 | 开始支持硬件加速
Android 4.0 阶段 | 默认开启硬件加速
Android 4.1 阶段 | 1. 引入了 VSYNC 垂直同步信号<br> 2. Triple Buffering 三缓冲机制(为了解决 A 帧被屏幕占用, B 帧被 GPU 占用, CPU 此时无法准备下一帧的问题)
Android 4.2 阶段 | 开发者选项中引入了过度渲染监控工具
Android 5.0 阶段 | 1. 引入了 RenderNode 来保存 View 的绘制动作 DisplayList<br>2. 引入了 RenderThread, 所有的 GL 命令都在 RenderThread 中进行, 减轻了 UI 线程的工作量 
Android 7.0 阶段 | 引入了  Vulkan 的硬件渲染引擎

<!--more-->

## 一. 图形渲染架构
Android 的图形渲染是一个生产者消费者模型, Android 图形渲染的组件结构图如下

![图形渲染架构](https://i.loli.net/2019/10/23/iGaKNuE63vpqBwJ.png)

### 一) 图像生产者
生产者为 Media Player 视频解码器, OpenGL ES 等产生的图像缓存数据, 他们通过 BufferData 的形式传递到缓冲队列中
- OpenGL ES: 通过渲染管线之后, 将数据输出到 EGLDisplay 上
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
图像运算的方式主要有两种, 一种是 CPU 运算, 另一种是 GPU 运算

### 一) CPU 运算
Android CPU 渲染引擎框架为 **[Skia](https://skia.org/)**, 它是一款在底端设备上呈现高质量的 2D 跨平台图形框架, Google 的 Chrome、Flutter 内部都有使用这个图形渲染框架

### 二) GPU 运算
**通过 GPU 进行图形运算, 称之为硬件加速**

#### 1. OpenGL
市面上最常用于图形运算的引擎莫过于 [OpenGL](https://developer.android.com/guide/topics/graphics/opengl) 了, Android 系统架构中的外部链接库中有 OpenGL ES 的依赖, 并且提供了应用层的 API, 用于做高性能的 2D/3D 图形渲染, Android 中对 OpenGL 的支持如下

##### 1) OpenGL 版本支持

Android 版本 | OpenGL ES 支持
---|---
Android 1.0 | OpenGL ES 1.0、1.1
Android 2.2 | OpenGL ES 2.0
Android 4.3 | OpenGL ES 3.0
Android 5.0 | OpenGL ES 3.1
Android 7.0 | OpenGL ES 3.2

##### 2) OpenGL API 支持

Canvas | First supported API level
---|---
drawBitmapMesh() (colors array) |	18
drawPicture()|	23
drawPosText()|	16
drawTextOnPath()|	16
drawVertices()|	✗
setDrawFilter()	|16
clipPath()|	18
clipRegion()|	18
clipRect(Region.Op.XOR)	|18
clipRect(Region.Op.Difference)	|18
clipRect(Region.Op.ReverseDifference)|	18
clipRect() with rotation/perspective|	18

Paint| First supported API level
---|---
setAntiAlias() (for text) |	18
setAntiAlias() (for lines) |	16
setFilterBitmap() |	17
setLinearText()	 |✗
setMaskFilter() |	✗
setPathEffect() (for lines)	 |28
setShadowLayer() (other than text)	 |28
setStrokeCap() (for lines) |	18
setStrokeCap() (for points)	 |19
setSubpixelText() |	28

Xfermode | First supported API level
---|---
PorterDuff.Mode.DARKEN (framebuffer)|	28
PorterDuff.Mode.LIGHTEN (framebuffer)|	28
PorterDuff.Mode.OVERLAY (framebuffer)|	28

Shader| First supported API level
---|---
ComposeShader inside ComposeShader	| 28
Same type shaders inside ComposeShader	| 28
Local matrix on ComposeShader	| 18

更多详情, 可以查看 https://developer.android.com/guide/topics/graphics/hardware-accel#drawing-support

#### 2. Vulkan
Android 7.0 之后除了添加 OpenGL ES3.2 的支持, 同时添加了 Vulkan 图像引擎, Vulkan 是用于高性能 3D 图形的低开销、跨平台 API, 它与 OpenGL 不同, 它被添加到 Android 运行时库中, 目前支持面稍窄

## 三. 图形渲染原理

![图像渲染流程](B15104AAEE9D439BA2FF937BBE559798)

其核心的设计思想为**生产者-消费者**模型
- 应用进程 与 SurfaceFlinger 进程
  - 生产端: Surface 持有队列的 Producer 对象, 是队列的生产者
  - 消费端: SurfaceFlinger 进程与 Surface 对应的 Layer, 它持有 Consumer 对象, 是队列的消费者
- SurfaceTexture 模型: 它是一个自成一体的生产者消费者模型
  - 相机可以将构建好的纹理投入 SurfaceTexture, 同时会回调 onFrameAvailable 通知外界有了一个新的 GraphicBuffer
  - 通过 updateTexImage 可以从其队列中获取一个新的 GraphicBuffer 交由 OpenGL 进行加工

### 一) 生产进程
#### 1. Window 的创建与初始化
##### 1) 初始化 Window
Activity 初始化 Window 的时机在 Activity.attach 方法中
- 创建 PhoneWindow 实例保存在 Activity 中
- 为 Window 绑定 WindoManager
  - 获取 Context 缓存的 WindowManager
  - 通过 Context 缓存的 WindowManager 创建为当前 PhoneWindow 创建对应的 WindowManagerImpl
    - 每一个 Window 都有自己对应的 WindowManager 对象 

##### 2) Window 填充 View
为 Window 填充 View 的时机在 setContentView 中
- 给 **Window 安装 DecorView**
  - 创建 DecorView
  - 为 DecorView 填充布局 mContentRoot
  - 将 mContentRoot 中 R.id.content 的 View 保存到 mContentParent
- 将我们传入的布局填充到 mContentParent

##### 3) 通知到 WMS
与 WMS 建立联系的时机在 onResume 之后
- WindowManager.addView, 将 Window 中的 DecorView 缓存到 WindowManagerGlobal
- **创建 DecorView 的管理实现类 ViewRootImpl**
  - **获取 IWindowSession 用于和 WMS 交互**
  - **创建一个 IWindow 的 Binder 实体对象 W, 描述当前 View 所在窗体**
  - **获取 Choreographer 示例对象, 用于调度 UI 渲染**
    - Native 层创建一个与 SurfaceFlinger 进程中 EventThread-app 的 Socket 连接
    - 通过 Looper 的 epoll 监听 Socket 端口, 有数据时回调 handleEvent 函数获取 VSYNC 事件
    - 应用进程渲染请求会添加到 CallbackQueue 队列, 同时会向 EventThread-app 请求一个 VSYNC 信号, 最终通过 handleEvent 获取之后再进行分发
       - CallbackQueue 的优先级: 触摸事件, 动画, Traversals(measure, layout, draw), Commit
- 调用 ViewRootImpl.setView 将 DecorView 注入
  - 通过 requestLayout 将 View 的遍历的操作投放到消息队列
  - **通过 addToDisplay 通知 WMS 有一个新的窗体创建了**
    - 在 WMS 端创建 WindowState 并加入 mWindowMap 中缓存
    - **创建一对 InputChannel, 让 ViewRootImpl 以 socket 的方式与 IMS 数据通信**

#### 2. View 的 Traversals

![View 的 Traversals](12E53135554048B5929E9BC735AED942)

上面我们也说道 Choreographer 会将 View 的 Traversals 操作同步到 VSYNC 时间线上, 其具体的流程如下

##### 1) View 的测量
- 构建 DecorView 的测量说明书
- 遍历 View 树进行测量(以 FrameLayout 为例)
  - **测量子 View**
    - 构建子 View 测量说明书
    - 分发到子 View 执行测量操作
      - 确定 View 的最小尺寸
      - 根据测量说明书的类型, 确认 View 最终的大小
   - **测量容器自身**
     - 根据容器的特性进行测量即可

##### 2) Surface 的创建

![Surface 创建示意图](https://i.loli.net/2019/12/13/Vz5iMlC8NQvFG6W.png)

Surface 虽然在 ViewRootImpl 创建的时候便会创建, 但此时它没有注入 IGraphicBufferProducer, 处于不可用的状态; 在 measure 确定了根 View 的宽高之后, 才能够确定 Surface 所需的缓冲区的大小, 这时会调用 relayout 通知 WMS 中窗体的尺寸变更了, 同时为 Surface 注入 IGraphicBufferProducer, 具体流程如下
- 系统服务进程创建 SurfaceControl 对象
- SurfaceFlinger 创建 GraphicBufferProducer
  - 创建 BufferLayer
    - 创建 GraphicBuffer 队列
    - 创建队列生产者 mProducer
    - 创建队列消费者 mConsumer
      - **Android 4.1 之后默认支持获取 3 个 GraphicBuffer 缓冲**
  - 缓存 BufferLayer
    - 最大 Layer 数量为 4096 
- 将 mProducer 的 Binder 代理对象保存到 SurfaceControl 中
- 系统服务进程将 SurfaceControl 中的 IGraphicBufferProducer 拷贝会应用进程的 Surface 中


##### 3) View 的布局
- 调用 setFrame 来更新当前 View 的坐标值
- 调用了 onLayout 操作去确定其子视图的位置

##### 4) View 的渲染
###### 软件渲染

![软件渲染依赖关系图](https://i.loli.net/2019/10/23/eXrF1g3EiWPxkuR.png)

Android 端软件渲染的引擎为 Skia, 其**软件渲染的工作机制即使用 Canvas 将数据绘制到 Surface 的 GraphicBuffer 中**, 它的工作流程如下
- **//////////////////////// Step1 /////////////////////////**
- Surface 锁定一个 GraphicBuffer 缓冲区
- 让 SkiaCanvas 的 SkBitmap 绑定缓冲区的共享内存
  - 意味着 Bitmap 的数据直接存储在缓冲区上了
- **//////////////////////// Step2 /////////////////////////**
- 分发 View 绘制
- **//////////////////////// Step3 /////////////////////////**
- Surface 释放 GraphicBuffer 缓冲区, 并将其推入 SurfaceFlinger 的渲染队列
- SurfaceFlinger 获取渲染数据输出到屏幕

###### 硬件渲染

![DisplayList 的重构](https://i.loli.net/2019/12/08/7zwjfk3KrON4LEX.jpg)

硬件渲染, 即通过 GPU 来进行图形运算, 进行帧准备的操作
- 硬件渲染是由 ThreadRenderer 执行的, 它初始化的时候
  - 会将 Surface 绑定到 RenderPipeline 中
- 每个 View 中都有一个 RenderNode, 需要重绘时会将标记为 dirty 的 View 重构其 RenderNode 中的 DisplayList
  - DisplayList 用于捕获 View 的绘制动作, 并为真正开始绘制
- 构建完成之后, ThreadRenderer 会到 native 层的 RendererThread 中使用 OpenGL/Vulkan 执行 DisplayList 中的渲染操作
- 完成之后同过 swapBuffer 推到 Surface 对应 Layer 的缓冲队列


######  两者差异
- 从渲染机制
  - 硬件绘制使用的是 OpenGL/ Vulkan, 支持 3D 高性能图形绘制
  - 软件绘制使用的是 Skia, 仅支持 2D 图形绘制
- 渲染效率上: 硬件绘制较之软件绘制会更加流畅
  - 硬件绘制
    - 在 Android 5.0 之后引入了 RendererThread, 它将 OpenGL 图形栅格化的操作全部投递到了这个线程
    - 硬件绘制会跳过渲染数据无变更的 View, 直接分发给子视图
  - 软件绘制
    - 在将数据投入 SurfaceFlinger 之前, 所有的操作均在主线程执行
    - 不会跳过无变化的 View
- 从内存消耗上
  - 硬件绘制消耗的内存要高于软件绘制, 但在当下大内存手机时代, 用空间去换时间还是非常值得的
- 从兼容性上
  - 硬件绘制的 OpenGL 在各个 Android 版本的支持上会有一些不同, 常有因为兼容性出现的系统 bug
  - 软件绘制的 Skia 库从 Android 1.0 便屹立不倒, 因此它的兼容性要好于硬件绘制

### 二) 消费进程

![SurfaceFlinger 渲染流程](https://i.loli.net/2019/12/14/3oW1BfFqnzKm9bc.png)

通过 View 的 Traversals 操作, 我们应用进程准备的帧数据就投递到 Surface 对应在 SurfaceFlinger 的 Layer 中了, 当 SurfaceFlinger 的 EventThread-sf 发送一个 sf-VSYCN 信号时, 便会触发图层的合成, 将图层合成到屏幕的 Buffer 上

#### 1. 屏幕的生产者消费者
- **生产者为 NativeWindowSurface** 
  - RE::Surface 它是一个 EGL 的工具类
    - **其内部的 EGLSurface 绑定了生产者 NativeWindowSurface 中的缓冲区**
    - 通过 EGL 的渲染之后的图像数据会存储到 EGLSurface 绑定的缓冲区中
    - 通过 eglSwapBuffer, 将生产好的数据推送到屏幕的 Buffer 队列
- **消费者为 FramebufferSurface**
  - 它负责从屏幕 Buffer 队列中取数据, 将其推送给硬件屏幕呈现出来

#### 2. 图层的合成
sf-Vsync 的信号分发到 EventThread-sf从而 唤醒 SurfaceFlinger 的主线程, 处理 INVALIDATE 消息
- INVALIDATE 消息类型
  - **从 Layer 的队列中锁定一个 GraphicBuffer**
- REFESH 消息类型
  - rebuildLayerStacks 负责 Layzer 的排序
    - **每个显示设备的 Layer 按照 Z 轴进行排序**
  - doComposition 负责合并 Layer, 并且输出到屏幕
    - **通过 RE::Surface 将 Layer 中的 Buffer 当做纹理合成到 EGLSurface 中**
    - **调用 RE::Surface 的 swapBuffers, 将 EGLSurface 中的数据通过 IGraphicBufferProducer 推入屏幕渲染队列**
    - **调用了 FramebufferSurface 的 advanceFrame, 将缓冲推给 HAL 进行展示**

## 参考
- [Android ProjectBuffer 计划](https://blog.csdn.net/innost/article/details/8272867)