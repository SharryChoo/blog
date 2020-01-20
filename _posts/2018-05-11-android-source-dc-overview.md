---
title: Android 系统架构 —— 数据通信篇 概览
permalink: android-source/dc-overview
key: android-source-dc-overview
tags: AndroidFramework
sidebar:
  nav: android-source
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
      - 匿名/命名
    - Socket
      - IMS 传递跨进程发送 InputEvent 
      - SurfaceFlinger 跨进程发送 Vsync
    - 消息队列
    - 共享内存

关于 Linux 的支持的, 这里就不再赘述了, 是属于操作系统的基础知识, 感兴趣的可以自行了解, 这里主要探究一下 Android 相关的通信方式

## 一. Handler 
Handler 我们平时用的是最多的, 在线程之间传输数据, 一般都用它来完成, 它也是三者之中最容易学习的, 它的使用场景如下
- 线程间信息的传递
- 在当前线程执行异步操作, 依靠 MessageQueue
- 监听主线程的卡顿

### Looper 的创建
Looper 的 prepare 操作主要是创建当前线程单例的 MessageQueue 对象, 保存在 mQueue 中
- 创建线程单例的 NativeMessageQueue 对象
  - 因为 Java 的 MessageQueue 是线程单例的, 因此 NativeMessageQueue 对象也是线程单例的 
- 创建线程单例的 Looper(Native)
  - 创建了一个名为 mWakeEventFd 的 eventfd 
    - Android 6.0 之前使用的 pipe
    - Android 6.0 之后使用的 eventfd
      - 比起管道, 它只有一个文件描述符, 更加轻量级
  - 创建了一个 IO 多路复用的 epoll 对象
  - 让 epoll 对其进行监听 mWakeEventFd

**Java 的 Looper 为死循环从 MessageQueue 中获取 Message 并执行**

**Native 的 Looper 主要是对 IO 多路复用的 epoll 的封装**
- 它除了可以监听 EventFd 来判断 MessageQueue 的消息有无, 还可以监听其他的文件描述符(如 InputChannel 的 Socket 端口和 Vsync 的 Connection Socket 端口), 当变更时回调指定的处理函数

### Looper 循环的启动
Looper.loop 主要是通过死循环, 不断地处理消息队列中的数据, 流程如下
- **通过 MessageQueue.next() 获取下一条要处理的 Msg**
  - **调用 nativePollOnce 根据 nextPollTimeoutMillis 时长, 阻塞在 epoll_wait 上**
  - **睡眠结束, 获取队列中可执行 Message**
    - 若有同步屏障消息, 找寻第一个异步的消息执行
    - 若未找到, 则找寻队首消息直接执行
      - 若当前的时刻 < 第一个同步的消息的执行时刻
        - 则更新 nextPollTimeoutMillis, 下次进行 for 循环时, 会进行睡眠操作
      - 若当前的时刻 >= 第一个同步的消息的执行时刻
        - 则将第一个同步消息的返回给 Looper 执行
    - 若未找到可执行消息
      - 将 nextPollTimeoutMillis 置为 -1, 下次 for 循环时可以无限睡眠, 直至消息队列中有新的消息为止
  - **未找到可执行 Message, 进行空闲消息的收集和处理**
    - 收集空闲消息
    - 处理空闲消息
- **通过 msg.target.dispatchMessage(msg); 分发处理消息**
  - 若是 Message 对象内部设置了 callback, 则**调用 handleCallback 方法直接处理, 不会再往下分发**
  - 若 Handler 设置 Callback, 则会**调用 Callback.handleMessage 方法**
    - Callback.handleMessage **返回 false, 则会将消息处理继续分发给 Handler.handleMessage** 

### 消息发送
Android 的消息机制, 主要有三种类型的消息
- 同步消息
  - 正常的消息, 我们一般都使用这个
- Android 4.1 配合 Choreographer 引入
  - 同步屏障: 从同步屏障起的时刻跳过同步消息, 优先执行异步消息
  - 异步消息: 用于和同步区分, 并非多线程异步执行的意思

## 二. Binder 驱动
Binder 驱动用于在 Android 中提供**高效**、**便捷**和**轻量级**跨进程通信服务
- 高效: 使用 mmap 实现用户空间和内核的通信, 无需多次拷贝
- 便捷: 提供了 AIDL 的辅助程序, 让我们在 Java 层也可以很方便的进行跨进程模型的搭建
- 轻量级: 无法传递大数据, 只能进行小规模的跨进程数据交互

### 使用场景
- 应用进程之间的跨进程通信
- 系统服务进程与服务管理进程之间的交互

### Binder 跨进程通信流程
#### 1. 系统调用
- **init**: 创建 Binder 驱动相关的文件
  - 注册 Binder 设备 
  - 创建 /proc/binder/ 
- **open**: 打开 binder 设备文件, 为当前进程创建 **binder_proc**, 可以理解为 binder 驱动上下文
  - binder 对象
    - **binder_node**: 实体对象
    - **binder_ref**: 引用对象
  - **binder_thread**: binder 线程
  - **binder_buffer**: binder 缓冲区  
- **mmap**: 初始化当前进程第一个 binder 缓冲区 binder_buffer
  - 为缓冲区分配一个物理页面, 将物理页映射到用户虚拟地址和内核虚拟地址
    - 更多的物理内存, 在用时动态分配, 首次只分配一个物理页面(4kb) 
  - 将缓冲区添加到 binder_proc 中缓存
- **ioctl**: 找寻空闲的线程, 处理 ioctl 请求 
  - BINDER_WRITE_READ: 数据的读写
  - BINDER_SET_CONTEXT_MGR: 注册 Binder 上下文的管理者

#### 2. 使用 Binder 驱动的条件
若想使用 Binder 驱动, 那么必须要在当前进程初始化 Binder 驱动
- open 打开驱动文件
- mmap 确定 Binder 驱动缓冲区的大小
- 启动并注册 Binder loop 线程, 通过内核的 binder_thread_read 函数确定是否存在其他进程发过来的待处理工作项

##### 不同进程的初始化策略
- **ServiceManager: main 函数直接初始化**
  - 设置 binder 缓冲区为 128 kb, 刚开始只会分配一个页面的物理内存(4kb)
  - 通过 binder_loop, 将当前线程注册为 Binder Loop 线程, 没有工作项时睡眠在 binder_thread_read 函数上, 有工作项插入时唤醒
- **SurfaceFlinger: main 函数直接初始化**
  - PrcoessState 是进程单例的, 构建时会初始化 Binder 驱动
    - 设置 binder 缓冲区为 1MB - 8k, 刚开始只会分配一个页面的物理内存(4kb)
    - 设置 Binder 线程池的数量为 4
  - 调用 ProcessState 的 startThreadPool 启动 Binder 驱动线程池的主线程
    - 将 Binder 线程池的主线程线程注册为 Binder 驱动的 loop 线程, 没有工作项时阻塞在 binder_thread_read 上, 有工作项插入时唤醒
- **Zygote 子进程: fork 之后立即调用 ZygoteInit.zygoteInit 到 Native 层初始化**
  - PrcoessState 是进程单例的, 构建时会初始化 Binder 驱动
    - 设置 binder 缓冲区为 1MB - 8k, 刚开始只会分配一个页面的物理内存(4kb)
    - 设置 Binder 线程池的数量为 4
  - 调用 ProcessState 的 startThreadPool 启动 Binder 驱动线程池的主线程
    - 将 Binder 线程池的主线程线程注册为 Binder 驱动的 loop 线程, 没有工作项时阻塞在 binder_thread_read 上, 有工作项插入时唤醒

#### 3. 通信
![流程图](https://i.loli.net/2019/10/23/OP862z4SdvbJcgo.png)

- 请求进程
  - 将数据封装成 Parcel
  - 将 Parcel 写入 Binder 驱动的缓冲区
  - 通过 ioctl 用 BC_TRANSACTION 指令通知 Binder 驱动处理处理这次跨进程通信
  - 通过 Binder 代理对象 binder_ref 找到 Binder 实体对象 binder_node
  - 通过 binder_node 找到目标进程, **将数据拷贝到目标进程的 Binder 缓冲区(物理内存不足, 会动态分配, 类似于缺页映射)**
    - 目标进程的空闲缓冲区不足, 则动态分配物理页面
    - 拷贝这一次, 是利用并发编程中的 C-O-W 的思想, 防止两个进程操作同一块缓冲区造成不可预估的数据错误
    - 又因为 Binder 缓冲区本就非常小, 因此拷贝一次并不会造成过高的性能损耗
  - 向目标线程的工作队列中添加 BINDER_WORK_TRANSACTION 工作项, 通知目标线程处理这个 Binder 通信
  - 当前进程的 waitForResponse 会在 binder_thread_read 上等待返回结果
- 目标进程
  - 睡眠在 binder_thread_read 上的线程被 BINDER_WORK_TRANSACTION 唤醒
  - 通知用户空间处理缓冲区传递过来的数据
  - 用户空间处理后通过同样的方式将返回数据发送给请求进程

## 三. Asheme 匿名共享内存
**Ashmem(Anonymous Shared Memory) 匿名共享内存是 Android 的 Linux 内核实现的一个驱动**, 它以驱动程序的形式实现在内核空间, 用于在进程间进行数据共享

###  使用场景
在 Android UI 渲染时, 我们需要将我们需要渲染的 Surface 数据发送到 SurfaceFlinger 进程进行最后的绘制, Binder 驱动传输显然是无法满足这么大的数据量, 因此这里使用到了 Ashmem 共享内存

### 工作流程
- init
  - 创建了用于分配小内存的 kmem_cache, 方便后续进行 ashmem_area 结构体的内存分配
- open
  - 打开驱动设备文件, 创建 ashmem_area 描述一块匿名共享内存，将其保存到共享内存驱动 fd 的 private 中 
- ioctl
  - ASHMEM_SET_NAME: 为这块匿名共享内存设置文件名
    - 只能在 mmap 之前调用
  - ASHMEM_SET_SIZE: 为这块匿名共享内存设置文件大小
    - 只能在 mmap 之前调用 
- mmap
  - 创建一个基于内存文件系统的共享内存文件 vmfile
  - 将这个 vmfile 文件映射到用户空间的虚拟地址上