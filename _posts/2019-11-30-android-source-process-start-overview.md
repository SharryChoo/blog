---
title: Android 系统架构 —— 开机启动篇 概览
permalink: android-source/process-start-overview
key: android-source-init-process-overview
tags: AndroidFramework
sidebar:
  nav: android-source
---

## 前言
Android 的系统是基于 Linux 的, 因此它的启动与 Linux 是相似的, 主要分为两个阶段
- **BIOS 阶段**
  - BIOS 在主板的 ROM 中存储, 加电之后便会启动
- **bootloader 阶段**, 读取磁盘上的操作系统
  - **0 号进程**
    - 所有进程的祖先, 这是 Linux 唯一一个没有通过 fork 或者 kernel_thread 产生的进程
    - 初始化进程管理、内存管理, 初始化驱动程序 Display, Camera Driver, Binder Driver 等相关工作
  - **1 号进程(init 进程)**
    - 为用户进程, 是所有用户态进程的祖先
  - **2 号进程**
    - 为内核进程, 负责内核态所有进程的管理和调度, 是所有 Linux 内核进程的祖先

## 一. init 进程
![启动进程树](https://i.loli.net/2019/10/19/abnUSCpxGR7m6zZ.png)

从 Init 进程的启动主要的职责如下
- 挂载文件系统, 创建驱动目录和驱动设备文件
- 解析启动脚本
- 执行脚本指令
  - 启动服务管理进程
  - 启动图像渲染进程
  - 启动 Zygote 进程

## 二. ServiceManager
ServiceManager 为服务管理进程, 我们系统的服务的 Binder 代理对象都会注册到 ServiceManager 中, 由它统一对应用进程提供
- 打开 Binder 驱动
  - open: 打开 Binder 设备文件
  - mmap: 初始化一个 128 kb 的缓冲区
- 注册为当前进程为 Binder 上下文的管理者
  - binder_node 对象通过静态变量 binder_context_mgr_node 保存
- 将当前线程注册为 Binder Loop 线程
  - 死循环通过 ioctl 从 Binder 驱动中读取数据
    - 没有任务, 则阻塞在 binder_thread_read 的 wait_event_freezable_exclusive 函数上, 等待任务到来了再唤醒
  - 读到任务之后, 通过 binder_parse 处理任务数据

## 三. SurfaceFlinger
![SurfaceFlinger 初始化相关类](https://i.loli.net/2019/12/13/cISm1EbwlqLXH8z.png)

SurfaceFlinger 进程主要负责 VSYNC 信号的分发和图层的合成
- main 函数初始化 Binder 驱动
  - 启动 Binder 的 Loop 线程 
- **SurfaceFlinger 的创建**
  - **DispSyncThread**: 用于将 DispSync 处理好的 SW-VSYNC 分发给监听者 DisplaySyncSource
  - **EventThread-app**: 维护了与 App 进程的 Connection 连接, 用于接收 DispSyncThread 发送的信号, 并且发送给 App 端, 会触发 Choreographer 执行 View 的绘制流程
  - **EventThread-sf**: 维护了与 SurfaceFlinger 的 Connection, 接收到 VSYNC 后, 会调用 cb_eventReceiver 执行图层的合成操作
  - **EventControlThread**: 用于控制硬件 Vsync 的开关
  - **StartPropertySetThread**: 执行属性列表, 会启动 BootAnimation 进程展示开机启动动画

## 四. Zygote
Zygote 是在 init 进程启动过程中启动的, 主要负责进程的孵化
- SystemServer 进程
- 应用进程

启动之后会调用 app_main.cpp 的 main 函数, 主要有两个方面的初始化
- Native 层调用 AndroidRuntime::start
  - **启动 ART 虚拟机**
- Java 层调用 ZygoteInit 的 main 方法
  - **预加载资源**
  - **创建 Socket**
    - 用于后续监听是否有 fork 请求
  - **创建 SystemService 系统服务进程**
  - **启动死循环**
    - 监听 Socket 端口, 响应进程创建请求

### 一) SystemServer
系统服务进程是由 Zygote 进程孵化的第一个进程, 它的初始化和启动流程如下
- **创建**
  - 通过 Zygote.fork 创建
- **初始化**
  - Native 初始化
    - startThreadPool 启动 Binder 驱动线程池的主线程
    - 主线程的 run 方法会执行 joinThreadPool 进入一个 loop 不断的与 Binder 驱动交互
      - 内核中阻塞在 binder_thread_read 上, 有工作项时唤醒  
  - Java 初始化
    - 找到 SystemServer 的 main 函数
- **启动**
  - 引导服务
    - **启动 ActivityManagerService**
    - **启动 PackageManagerService**
  - 核心服务
  - 其他服务
    - **启动 InputManagerService**
    - **启动 WindowManagerService**
    - **回调 AMS 的 systemReady**
      - 启动 [SystemUi](http://androidxref.com/9.0.0_r3/xref/frameworks/base/packages/SystemUI/) 服务进程, 为 App 提供通用的 UI 组件
        - 状态栏 StatusBar, 通知 NotificationChannels, 最近任务列表 Recents...
      - 启动 Launcher 进程
  - 进入 Looper 死循环, 保证 SystemServer 进程长久存活

#### PKMS
##### 1) PKMS 的启动
- **解析应用程序备份信息 package.xml/package-backup.xml**
  - 为应用程序分配 UserID
- **扫描应用程序安装目录**
  - PKMS 使用 Package 对象来描述一个已经安装的应用程序, 它会从 base.apk 中解析 AndroidManifest.xml 中四大组件的信息, 将它封装到 Pacakge 中
- **发布应用数据**
  - 将安装包内的 Android 四大组件发布到 PMS 的 mActivity, mServices... 集合中

##### 2) PKMS 安装应用

![安装好的应用目录](https://i.loli.net/2019/11/15/Nn2OJy3ZaH6XLMo.png)

我们 Android 端, 整个应用程序的安装主要有两种方式:
- **通过 Session 发起(adb, 手动点击安装)**
  - 提前将安装包拷贝到 "data/app/vmdl${sessionId}.tmp/" 目录下 
  - 再通过 Session commit, 通知 PKMS 的 PKMS.installStage 进行安装
- **通过系统的软件商店自动发起**
  - 直接通过 PKMS 的 PKMS.installStage 进行安装

**通过 Session 拷贝安装包**
- 获取 SessionId 描述一个安装任务
  - 分配 SessionId
  - 创建临时目录 stageDir
    - "data/app/vmdl{sessionId}.tmp"
  - 创建服务进程的 PackageInstallerSession 对象
- 将安装包拷贝到 "data/app/vmdl${sessionId}.tmp/PackageInstaller" 位置下暂存
- 提交安装任务
  - 调用 sealAndValidateLocked 对拷贝过来的安装包进行验证
    - **将 "data/app/vmdl${sessionId}.tmp/PackageInstaller" 重命名为 "data/app/vmdl${sessionId}.tmp/base.apk"**
  - 解压 Native 库
  - 调用 PKMS.installStage 发起这次应用安装的请求

**PKMS 安装应用程序**
- **连接远程应用安装服务 DefaultContainerService**
  - 获取 IMediaContainerService 的 Binder 代理对象
- 安装应用
  - 拷贝安装包
    - 若是已经通过 Session 拷贝过了, 跳过拷贝的过程
    - 若未拷贝, 调用了 IMediaContainerService 的 copyPackage, 在远程服务中执行 apk 拷贝操作
      - 拷贝到 **"data/app/vmdl${sessionId}.tmp/base.apk"** 中
    - 解压 Native 库文件
  - 安装应用程序
    - 调用了 PackageParser.Package 解析 base.apk 安装包文件
      - 即将 apk 中的信息发布到 PKMS 
    - 解析 apk 的 dex 文件
    - 调用 InstallArgs.doRename 更换目录名称
      - 更名前: "data/app/vmdl${sessionId}.tmp/"
      - 更名后: "data/app/${PackageName}${Base64 随机码}/"
    - 优化 dex 文件


#### IMS
![Android 系统事件分发流程图](https://i.loli.net/2019/11/21/OVAPohwv2H8t9CW.png)

**IMS 的初始化**
- 创建线程 InputReaderThread, 使用 InputReader 读取输入设备产生事件到 QueuedInputListener
- 创建线程 InputDispatcherThread, 使用 InputDispatcher 分发数据

**IMS 与应用进程的交互**
- 当应用进程通知 WMS 创建了一个窗体时, 会创建 InputChannel 输入通道组
  - 创建最大缓冲为 32 kb 的 Socket
  - 服务端的 InputChannel 保留 Socket 服务端的文件描述符
  - 客户端的 InputChannel 保留 Socket 客户端的文件描述符
- 将服务端的新建的 WindowState 和 InputChannel 注册到 IMS 中
  - InputDipatcher 将 WindowState 和 InputChannel 封装到 Connection 中用于后续的事件分发
- 客户端监听 InputChannel
  - 应用进程拿到 WMS 传递过来的 InputChannel 后便会构建一个 InputChannelReceived
  - **InputChannelReceived 在 Native 层通过 Looper 的 epoll 监听了 InputChannel 的 Socket 端口**
  - 有数据写入便会通过 JNI 调用 dispatchInputEvent 分发到 Java 层

#### WMS
WMS 的启动流程如下
- WMS 的创建
  - 在 DisplayThread 执行创建操作 
- WMS.onInitReady
  - 执行了 PhoneWindowManager 的初始化 
- WMS.displayReady
  - 初始化 RootWindowContainer 子窗体
- WMS.systemReady
  - 执行了 PhoneWindowManager 的初始化

当这些服务都启动完毕之后, 会**回调 AMS 的 systemReady 启动 Launcher 这个 App**

### 二) 应用进程
#### 1. Activity 启动
- 准备阶段
  - 获取目标 Activity 的 ActivityRecord
    - 不存在则创建一个
  - 获取目标 Activity 所在的任务栈 ActivityStack
    - 不存在则创建一个
- 若发起方 Activity 未暂停, 则通知发起方的应用进程处理请求 Activity 的暂停
  - **暂停成功之后, 通知 AMS, 继续进行目标 Activity 的启动操作**
- 若目标 Activity 的进程未启动, 则先通过 Socket 请求 Zygote fork  应用进程
  - 应用进程初始化阶段会启动 Binder 的 loop 线程
  - **ActivityThread.main 中会回调 AMS.attachApplication 继续执行后续操作**
- AMS 发送请求到应用进程, 执行目标 Activity 的启动
  - 构建 App 的资源包
    - 优先从缓存中获取 LoadedApk
    - 不存在, 则构建一个对象, 添加到缓存 mPackages 中
  - **创建一个 Activity 的 Context**
    - 创建 ContextImpl
    - 注入 Resource 资源信息
  - 构建 Activity 实例
    - **创建进程唯一的 Application 对象**
      - 创建 Resource, 并注入 Application
    - 创建 ContextImpl
    - 注入 Resource 资源信息
  - 让 appContext 绑定 Activity
  - **为 Activity.attach 绑定参数**
    - 绑定 ContextImpl
    - 创建 Window 和 WindowManager
  - **回调 onCreate**

#### 2. Service
##### start
- 未启动 
  - 未创建进程, 则先创建进程
  - 回调 onCreate, 启动服务
  - 回调 onStartCommand
- 已经启动
  - 直接回调 onStartCommand
- 结束服务
  - 回调 onDestroy 

##### bind
- 未启动 
  - 未创建进程, 则先创建进程
  - 回调 onCreate, 启动服务
  - AMS 调用 onBind, 获取一个 Binder 实体对象
  - AMS 将 Binder 实体对象通过 Binder 驱动发送给请求方的 ServiceConnection
- 已经启动
  - 直接回调 onRebind
- 结束服务
  - 回调 onUnbind
  - 回调 onDestroy