---
title: Android 系统架构 —— SurfaceFlinger 的启动
permalink: android-source/surfaceflinger-launch
key: android-source-surfaceflinger-launch
tags: AndroidFramework
---

## 前言
在之前学习 Android 图形渲染的时候, 我们看到了最终所有的渲染数据都会通过 Surface 推送给 SurfaceFlinger, 然后在 SurfaceFlinger 进程中真正的进行渲染

这里我们学习一下 SurfaceFlinger 进程的启动流程

<!--more-->

## SurfaceFlinger 的启动
在系统启动篇, 我们知道 SurfaceFlinger 是由 init 进程 fork 出来的, 它的[启动脚本](http://androidxref.com/9.0.0_r3/xref/frameworks/native/services/surfaceflinger/SurfaceFlinger.rc)定义如下
```
// frameworks/native/services/surfaceflinger/surfaceflinger.rc
service surfaceflinger /system/bin/surfaceflinger
    class core animation
    user system
    group graphics drmrpc readproc
    onrestart restart zygote
    writepid /dev/stune/foreground/tasks
    socket pdx/system/vr/display/client     stream 0666 system graphics u:object_r:pdx_display_client_endpoint_socket:s0
    socket pdx/system/vr/display/manager    stream 0666 system graphics u:object_r:pdx_display_manager_endpoint_socket:s0
    socket pdx/system/vr/display/vsync      stream 0666 system graphics u:object_r:pdx_display_vsync_endpoint_socket:s0
```
当 SurfaceFlinger 被 fork 出来后, 会调用 [main_surfaceflinger.cpp](http://androidxref.com/9.0.0_r3/xref/frameworks/native/services/surfaceflinger/main_surfaceflinger.cpp) 的 main 方法, 接下来看看它的启动
```
// frameworks/native/services/surfaceflinger/main_surfaceflinger.cpp
int main(int, char**) {
    ......
    // 配置 Binder 驱动相关数据
    // 设置 Binder 线程池的数量为 4 
    ProcessState::self()->setThreadPoolMaxThreadCount(4);
    // 启动 Binder 线程池
    sp<ProcessState> ps(ProcessState::self());
    ps->startThreadPool();
    
    // 1. 创建 SurfaceFlinger
    sp<SurfaceFlinger> flinger = new SurfaceFlinger();
    setpriority(PRIO_PROCESS, 0, PRIORITY_URGENT_DISPLAY);
    set_sched_policy(0, SP_FOREGROUND);
    ......
    
    // 2. 初始化 SurfaceFlinger
    flinger->init();
    
    // 3. 将 SurfaceFlinger 发布到 ServiceManager 中, 方便其他进程与之建立连接
    sp<IServiceManager> sm(defaultServiceManager());
    sm->addService(String16(SurfaceFlinger::getServiceName()), flinger, false,
                   IServiceManager::DUMP_FLAG_PRIORITY_CRITICAL);
                   
    ......
    
    // 4. 启动 SurfaceFlinger
    flinger->run();
    return 0;
}
```
从这里可以看出 SurfaceFlinger 启动的主要流程为如下 4 个步骤
- **SurfaceFlinger 的创建**
- **SurfaceFlinger 的初始化**
- **SurfaceFlinger 的发布**
  - 将这个 Binder 本地对象发布到 ServiceManager 进程 
- **SurfaceFlinger 的启动**

其中, 将 SurfaceFlinger Binder 本地对象发布到 ServiceManager 这里就不再赘述了, 我们主要关心一下其他三个方面

## 一. [SurfaceFlinger](http://androidxref.com/9.0.0_r3/xref/frameworks/native/services/surfaceflinger/SurfaceFlinger.h) 的创建
```
// frameworks/native/services/surfaceflinger/SurfaceFlinger.h
class SurfaceFlinger : public BnSurfaceComposer,
                       public PriorityDumper,
                       private IBinder::DeathRecipient,
                       private HWC2::ComposerCallback
{
    ......
}

// frameworks/native/services/surfaceflinger/SurfaceFlinger.cpp
SurfaceFlinger::SurfaceFlinger() : SurfaceFlinger(SkipInitialization) {
    ......
}

SurfaceFlinger::SurfaceFlinger(SurfaceFlinger::SkipInitializationTag)
      : BnSurfaceComposer(),
      .......
      // 创建 DispSync 对象
      mPrimaryDispSync("PrimaryDispSync"),
      ....... {
      
    // 初始化 DispSync 对象
    mPrimaryDispSync.init(SurfaceFlinger::hasSyncFramework, SurfaceFlinger::dispSyncPresentTimeOffset); 
          
}
```
从 SurfaceFlinger 的构造中我们了解到它是继承了 BnSurfaceComposer Binder 本地对象, 这也是为何它能够发布到 ServiceManager 的原因

在其构造方法中, 我们看到它创建并初始化了 DispSync mPrimaryDispSync 这个对象, **DispSync 类似于一个 PLL（phase lock loop，锁相回路），它通过接收硬件 VSYNC，然后给其他关心硬件 VSYNC 的组件（SurfaceFlinger 和需要渲染的 app）在指定的偏移以后发送软件 VSYNC，并且当误差在可接受的范围内，将会关闭硬件 VSYNC**

![VSYNC 信号转移](https://i.loli.net/2019/12/08/7gcVndjkAWJMs8L.png)

- HW-Vsync: 由硬件直接产生的垂直同步信号
- SW-Vsync: 由 DispSync 转移后的软件垂直同步信号
- SF-Vsync: 供 SurfaceFlinger 端使用的垂直同步信号
- APP-Vsync: 供 App 端 Choreographer 使用的垂直同步信号

**也就是说 DispSync 主要是负责将硬件垂直同步信号转为软件垂直同步信号, 然后供上层使用**

关于为何不直接使用硬件垂直同步信号, 笔者从这篇博文中找到了满意的答案 https://simowce.github.io/2019/10/07/all-about-dispsync/
- 在 Android 4.1 的时候，Google 提出了著名的 “Project Butter”，引入了 VSYNC, 把 app 画图，SurfaceFlinger 合成的时间点都规范了起来，减少了掉帧，增强了渲染的流畅度。
- 假设有这么一种需求，我希望在 VSYNC 偏移一段时间以后再干活，那么这个是硬件 VSYNC 提供不了，所以这个时候就必须引入软件模型。而 DispSync 就是为了解决这个需求引入的软件模型。

因此 DispSync 还是非常重要的, 下面我们看看 DispSync 的初始化操作

### 一) DispSync 的初始化
```
// frameworks/native/services/surfaceflinger/DispSync.cpp
DispSync::DispSync(const char* name)
      : mName(name), mRefreshSkipCount(0), mThread(new DispSyncThread(name)) {}
      
void DispSync::init(bool hasSyncFramework, int64_t dispSyncPresentTimeOffset) {
    ......
    // 调用了 mThread 的 run 方法
    mThread->run("DispSync", PRIORITY_URGENT_DISPLAY + PRIORITY_MORE_FAVORABLE);
    ......
}
```
可以看到 DispSync 的构造函数中创建了一个 DispSyncThread 对象, 而后在其初始化时, 调用了这个线程的 run 方法, 我们继续探究
```
// frameworks/native/services/surfaceflinger/DispSync.cpp
class DispSyncThread : public Thread {
public:
    explicit DispSyncThread(const char* name)
          : mName(name),
            mStop(false),
            mPeriod(0),
            mPhase(0),
            mReferenceTime(0),
            mWakeupLatency(0),
            mFrameNumber(0) {}
```
可以看到, 这个 DispSyncThread 继承自 Thread 类, 这个 Thread 是 Native 层对 p_thread_t 的封装, 其操作方式与 Java 相似,  当调用 run 函数时, 会创建一个新的线程, 并且调用 threadLoop 函数
```
virtual bool threadLoop() {
    status_t err;
    nsecs_t now = systemTime(SYSTEM_TIME_MONOTONIC);
    while (true) {
        Vector<CallbackInvocation> callbackInvocations;
        nsecs_t targetTime = 0;
        { // Scope for lock
            // 若 mPeriod 为 0, 则循环等待
            if (mPeriod == 0) {
                err = mCond.wait(mMutex);
                if (err != NO_ERROR) {
                    ALOGE("error waiting for new events: %s (%d)", strerror(-err), err);
                    return false;
                }
                continue;
            }
            ......
            // 收集 vsync 信号的所有回调方法
            callbackInvocations = gatherCallbackInvocationsLocked(now);
        }
        if (callbackInvocations.size() > 0) {
            // 回调所有对象的 onDispSyncEvent 方法
            fireCallbackInvocations(callbackInvocations);
        }
    }
    return false;
}
```
可以看到 threadLoop 中实现了一个死循环, 从循环的实现中可以得知, **DispSyncThread 的职责是分发 sw-Vsync 信号**的, 关于其循环内部的相关函数功能实现, 到后面的章节在具体分析

除此之外 SurfaceFlinger 是支持智能指针的, 当他第一次被引用的时候, 会回调 onFirstRef 方法, 下面看看 onFirstRef 中做了什么

### 二) onFirstRef
```
// frameworks/native/services/surfaceflinger/SurfaceFlinger.cpp
void SurfaceFlinger::onFirstRef()
{
    // 初始化事件队列
    mEventQueue->init(this);
}

// frameworks/native/services/surfaceflinger/MessageQueue.cpp
void MessageQueue::init(const sp<SurfaceFlinger>& flinger) {
    mFlinger = flinger;
    // 创建了一个 Looper
    mLooper = new Looper(true);
    // 创建了一个 Handler
    mHandler = new Handler(*this);
}
```
可以看到 onFirstRef 的实现很简单, 即初始化了当前线程的 MessageQueue, 也就是说 **SurfaceFlinger 中是存在与其他线程协作交互的**, 这个我们到后面逐步的验证

接下来看看 SurfaceFlinger 的初始化操作

## 二. SurfaceFlinger 的初始化
```
// frameworks/native/services/surfaceflinger/SurfaceFlinger.cpp

// Do not call property_set on main thread which will be blocked by init
// Use StartPropertySetThread instead.
void SurfaceFlinger::init() {
    ......
    
    Mutex::Autolock _l(mStateLock);

    // 1. 创建 APP 端对 vsync 信号的处理线程
    // DispSyncSource 为 DispSync 与 EventThread 沟通的桥梁
    mEventThreadSource =
            std::make_unique<DispSyncSource>(&mPrimaryDispSync, SurfaceFlinger::vsyncPhaseOffsetNs,
                                             true, "app");
    mEventThread = std::make_unique<impl::EventThread>(mEventThreadSource.get(),
                                                       [this]() { resyncWithRateLimit(); },
                                                       impl::EventThread::InterceptVSyncsCallback(),
                                                       "appEventThread");
    // 2. 创建 SF 端对 vsync 信号的处理线程
    mSfEventThreadSource =
            std::make_unique<DispSyncSource>(&mPrimaryDispSync,
                                             SurfaceFlinger::sfVsyncPhaseOffsetNs, true, "sf");
    mSFEventThread =
            std::make_unique<impl::EventThread>(mSfEventThreadSource.get(),
                                                [this]() { resyncWithRateLimit(); },
                                                [this](nsecs_t timestamp) {
                                                    mInterceptor->saveVSyncEvent(timestamp);
                                                },
                                                "sfEventThread");
                                                
    // 3. 将 mSFEventThread 设置为处理 EventQueue 要监听的线程
    mEventQueue->setEventThread(mSFEventThread.get());

    // 初始化 GL 渲染引擎
    getBE().mRenderEngine =
            RE::impl::RenderEngine::create(HAL_PIXEL_FORMAT_RGBA_8888,
                                           hasWideColorDisplay
                                                   ? RE::RenderEngine::WIDE_COLOR_SUPPORT
                                                   : 0);
    // 4. 初始化 HWComposer
    getBE().mHwc.reset(
            new HWComposer(std::make_unique<Hwc2::impl::Composer>(getBE().mHwcServiceName)));
    // 5. 注册监听回调
    getBE().mHwc->registerCallback(this, getBE().mComposerSequenceId);
    
    // 将默认的 GLContext 绑定为当前线程的 EGL 上下文
    getDefaultDisplayDeviceLocked()->makeCurrent();

    ......
    
    // 6. 创建 EventControlThread 用来开关 vsync 信号
    mEventControlThread = std::make_unique<impl::EventControlThread>(
            [this](bool enabled) { setVsyncEnabled(HWC_DISPLAY_PRIMARY, enabled); });
    
    ......

    // 5. 初始化显示设备
    initializeDisplays();

    ......
}
```
上述代码即为 SurfaceFlinger 初始化操作的大体流程, 其重点步骤如下
- **创建 APP 的 Vsync 信号处理线程 mEventThread-app**
  - 负责接收 sw-Vsync 并且发送信号给 App 进程的 Choreographer 执行 View 的绘制 
- **创建 SurfaceFlinger 的 Vysnc 信号处理线程 mSFEventThread-sf**
  - 负责接收 sw-Vsync 并且发送给 SurfaceFlinger 触发图层的合成
- **绑定 mSFEventThread**
- **初始化 HWComposer**
   - 注册 HWComposer 的回调
- **创建 EventControlThread 用来开关 Vsync 信号**

这里可以看到 SurfaceFlinger 初始化的过程中创建了三个线程, 其中 EventThread 非常重要, 它是 sw-Vsync 的分发者, 我们先看看 EventThread 的创建过程

### 一) EventThread 的创建
```
// frameworks/native/services/surfaceflinger/EventThread.cpp
EventThread::EventThread(VSyncSource* src, ResyncWithRateLimitCallback resyncWithRateLimitCallback,
                         InterceptVSyncsCallback interceptVSyncsCallback, const char* threadName)
      : mVSyncSource(src),
        mResyncWithRateLimitCallback(resyncWithRateLimitCallback),
        mInterceptVSyncsCallback(interceptVSyncsCallback) {
    for (auto& event : mVSyncEvent) {
        event.header.type = DisplayEventReceiver::DISPLAY_EVENT_VSYNC;
        event.header.id = 0;
        event.header.timestamp = 0;
        event.vsync.count = 0;
    }
    // 创建了一个线程, 其执行函数为 EventThread
    mThread = std::thread(&EventThread::threadMain, this);
    pthread_setname_np(mThread.native_handle(), threadName);
    pid_t tid = pthread_gettid_np(mThread.native_handle());
    .......
}
```
可以看到 EventThread 内部创建了一个线程, 这个线程会执行 threadMain 函数, 它的定义如下
```
// frameworks/native/services/surfaceflinger/EventThread.cpp
void EventThread::threadMain() NO_THREAD_SAFETY_ANALYSIS {
    std::unique_lock<std::mutex> lock(mMutex);
    while (mKeepRunning) {
        DisplayEventReceiver::Event event;
        Vector<sp<EventThread::Connection> > signalConnections;
        // 查找事件, 并找寻能够响应事件的连接
        signalConnections = waitForEventLocked(&lock, &event);
        // 分发事件给监听者
        const size_t count = signalConnections.size();
        for (size_t i = 0; i < count; i++) {
            const sp<Connection>& conn(signalConnections[i]);
            // 分发给事件的监听者
            status_t err = conn->postEvent(event);
            // 处理异常情况
            if (err == -EAGAIN || err == -EWOULDBLOCK) {
                ......// 抛弃事件
            } else if (err < 0) {
                // 出现致命错误, 清理这个连接
                removeDisplayEventConnectionLocked(signalConnections[i]);
            }
        }
    }
}
```
threadMain 中是一个 while 循环, 其内部的主要工作如下
- 查找并找寻能够响应事件的连接
- 调用 EventThread 连接者的 postEvent 函数处理事件

#### 1. 查找响应事件连接
```
// frameworks/native/services/surfaceflinger/EventThread.h
// 描述 VSync 事件数组
DisplayEventReceiver::Event mVSyncEvent[DisplayDevice::NUM_BUILTIN_DISPLAY_TYPES] GUARDED_BY(
            mMutex);
// 描述 Pending 事件集合
Vector<DisplayEventReceiver::Event> mPendingEvents GUARDED_BY(mMutex);

// frameworks/native/services/surfaceflinger/EventThread.cpp
Vector<sp<EventThread::Connection> > EventThread::waitForEventLocked(
        std::unique_lock<std::mutex>* lock, DisplayEventReceiver::Event* event) {
    Vector<sp<EventThread::Connection> > signalConnections;
    
    while (signalConnections.isEmpty() && mKeepRunning) {
        bool eventPending = false;
        bool waitForVSync = false;
        size_t vsyncCount = 0;
        nsecs_t timestamp = 0;
        
        // 1. 获取 VSyncEvent 事件
        for (int32_t i = 0; i < DisplayDevice::NUM_BUILTIN_DISPLAY_TYPES; i++) {
            timestamp = mVSyncEvent[i].header.timestamp;
            if (timestamp) {
                // 说明找到了一条 VSync 事件, 执行事件的分发
                if (mInterceptVSyncsCallback) {
                    // 让拦截器处理
                    mInterceptVSyncsCallback(timestamp);
                }
                // 保存到 event 中
                *event = mVSyncEvent[i];
                mVSyncEvent[i].header.timestamp = 0;
                // 记录事件的数量
                vsyncCount = mVSyncEvent[i].vsync.count;
                break;
            }
        }
        
        // 2. 没有 Vsync 事件, 尝试获取 Pending 事件
        if (!timestamp) {
              eventPending = !mPendingEvents.isEmpty();
               if (eventPending) {
                   // we have some other event to dispatch
                   *event = mPendingEvents[0];
                    mPendingEvents.removeAt(0);
               }
        }
        
        // 3. 查找能够接收事件的 Connection
        size_t count = mDisplayEventConnections.size();
        for (size_t i = 0; i < count;) {
            // 获取一个 Connection
            sp<Connection> connection(mDisplayEventConnections[i].promote());
            if (connection != nullptr) {
                bool added = false;
                // 3.1 找寻能够接收 Vsync 事件的 Connection
                if (connection->count >= 0) {
                    // we need vsync events because at least
                    // one connection is waiting for it
                    waitForVSync = true;
                    // 说明存在 Vsync 事件
                    if (timestamp) {
                        // 若为未触发的一次性事件
                        if (connection->count == 0) {
                            connection->count = -1;// 不允许再次接收事件了
                            signalConnections.add(connection);
                            added = true;
                        }
                        // 若为连续事件
                        else if (connection->count == 1 ||
                                   (vsyncCount % connection->count) == 0) {
                            // continuous event, and time to report it
                            signalConnections.add(connection);
                            added = true;
                        }
                    }
                }
                // 3.2 处理只存在 Pending 事件的情况
                if (eventPending && !timestamp && !added) {
                    signalConnections.add(connection);
                }
                ++i;
            } else {
                // 移除异常的 connection
                mDisplayEventConnections.removeAt(i);
                --count;
            }
        }
        // Here we figure out if we need to enable or disable vsyncs
        if (timestamp && !waitForVSync) {
            // 没有 Connection, 则调用 enableVSyncLocked
            disableVSyncLocked();
        } else if (!timestamp && waitForVSync) {
            // 至少有一个连接在等待 Vsync 信号, 因此调用 enableVSyncLocked 函数
            enableVSyncLocked();
        }
        // 4. 陷入等待
        if (!timestamp && !eventPending) {
            // 4.1 若存在 Connection 则等待 Vsync 信号输入
            if (waitForVSync) {
                bool softwareSync = mUseSoftwareVSync;
                auto timeout = softwareSync ? 16ms : 1000ms;
                // 陷入等待
                if (mCondition.wait_for(*lock, timeout) == std::cv_status::timeout) {
                    ......
                    mVSyncEvent[0].header.type = DisplayEventReceiver::DISPLAY_EVENT_VSYNC;
                    mVSyncEvent[0].header.id = DisplayDevice::DISPLAY_PRIMARY;
                    mVSyncEvent[0].header.timestamp = systemTime(SYSTEM_TIME_MONOTONIC);
                    mVSyncEvent[0].vsync.count++;
                }
            } 
            // 4.2 若无 Connection, 则陷入无超时机制的阻塞
            else {
                mCondition.wait(*lock);
            }
        }
    }
    
    return signalConnections;
}
```
找寻响应事件的 Connection 的流程如下所示
- 获取 VSync 事件
- 没有 VSync 事件, 尝试获取 Pending 事件
- 查找能够响应事件的 Connection
- 若无事件, 则陷入等待

接下来看看 Connection 的 postEvent 是如何处理事件的

#### 2. 事件的处理
```
// frameworks/native/services/surfaceflinger/EventThread.cpp
status_t EventThread::Connection::postEvent(const DisplayEventReceiver::Event& event) {
    // 调用了 sendEvents 发送事件
    ssize_t size = DisplayEventReceiver::sendEvents(&mChannel, &event, 1);
    return size < 0 ? status_t(size) : status_t(NO_ERROR);
}

// frameworks/native/libs/gui/DisplayEventReceiver.cpp
ssize_t DisplayEventReceiver::sendEvents(gui::BitTube* dataChannel,
        Event const* events, size_t count)
{   
    // 将事件发送到 BitTube 中
    return gui::BitTube::sendObjects(dataChannel, events, count);
}
```
好的, 可以看到 Connection 的最终会将 Vysor/Pending 事件发送到 BitTube 中去, 这到底是何用意呢?

我们带着疑问继续往下探究

### 二) 监听 EventThread
```
void MessageQueue::setEventThread(android::EventThread* eventThread) {
    if (mEventThread == eventThread) {
        return;
    }
    ......
    // 保存到成员变量
    mEventThread = eventThread;
    // 1. 创建事件的连接器
    mEvents = eventThread->createEventConnection();
    // 2. 获取 BitTube 对象
    mEvents->stealReceiveChannel(&mEventTube);
    // 3. 监听 BitTube，一旦有数据到来则调用 cb_eventReceiver
    mLooper->addFd(mEventTube.getFd(), 0, Looper::EVENT_INPUT, MessageQueue::cb_eventReceiver,
                   this);
}

```
可以看到 MessageQueue.setEventThread 中
- 首先会调用 EventThread 的 createEventConnection 函数, 创建一个 Connection 对象
- 然后调用 stealReceiveChannel 函数获取 Connection 中的 BitTube
- **最终通过 Looper 监听 BitTube 的文件描述符**

好的, 到这里我们就知晓之前 为什么 Connection.postEvent 仅仅是将事件发送到 BitTube 中了, 因为 Looper 会监听 BitTube 的文件名描述符, 一旦有数据写入, 则会回调 cb_eventReceiver 函数去进行处理

接下来看看 Connection 的构建流程

#### 1. Connection 的构建
```
// frameworks/native/services/surfaceflinger/EventThread.cpp
EventThread::Connection::Connection(EventThread* eventThread)
      : count(-1), mEventThread(eventThread), mChannel(gui::BitTube::DefaultSize) {}

sp<BnDisplayEventConnection> EventThread::createEventConnection() const {
    return new Connection(const_cast<EventThread*>(this));
}

void EventThread::Connection::onFirstRef() {
    mEventThread->registerDisplayEventConnection(this);
}

```
可以看到, Connection 创建之后会被调用 onFirstRef, 这个函数中调用了 registerDisplayEventConnection 将该 Connection 添加到了 EventThread 的缓存中
```
status_t EventThread::registerDisplayEventConnection(
       const sp<EventThread::Connection>& connection) {
    std::lock_guard<std::mutex> lock(mMutex);
    // 添加一个 Connection 到连接的缓存
    mDisplayEventConnections.add(connection);
    // 解除 waitForEventLocked 中的阻塞
    mCondition.notify_all();
    return NO_ERROR;
}
```
在上面的分析中我们知道, EventThread.waitForEventLocked 函数, 当没有 Connection 是会陷入无超时机制的阻塞中
- 在这里可以添加了一个 Connection 之后, 调用了 notify_all, 会解除 EventThread.waitForEventLocked 阻塞

接下来看看, 当 BitTube 发生变化时, 回调的 cb_eventReceiver 函数

#### 2. cb_eventReceiver 回调
```
int MessageQueue::cb_eventReceiver(int fd, int events, void* data) {
    MessageQueue* queue = reinterpret_cast<MessageQueue*>(data);
    return queue->eventReceiver(fd, events);
}

int MessageQueue::eventReceiver(int /*fd*/, int /*events*/) {
    ssize_t n;
    DisplayEventReceiver::Event buffer[8];
    // 获取事件
    while ((n = DisplayEventReceiver::getEvents(&mEventTube, buffer, 8)) > 0) {
        for (int i = 0; i < n; i++) {
            if (buffer[i].header.type == DisplayEventReceiver::DISPLAY_EVENT_VSYNC) 
                // 分发事件
                mHandler->dispatchInvalidate();
                break;
            }
        }
    }
    return 1;
}

void MessageQueue::Handler::dispatchInvalidate() {
    if ((android_atomic_or(eventMaskInvalidate, &mEventMask) & eventMaskInvalidate) == 0) {
        // 发送了一个 Type INVALIDATE 的消息, 最终会交由 SurfaceFlinger.onMessageReceived 处理
        mQueue.mLooper->sendMessage(this, Message(MessageQueue::INVALIDATE));
    }
}
```
MessageQueue.setEventThread 函数, 会让 Looper 监听 BitTube 的文件描述符
- 其中**若有数据**则回调 cb_eventReceiver 函数, **最终会交由 SurfaceFlinger 的 onMessageReceived 来处理**

如此以来这个 EventThread 就可以与 SurfaceFlinger 的主线程进行数据交互了

### 三) HWComposer 的初始化
[HWComposer](http://androidxref.com/9.0.0_r3/xref/frameworks/native/services/surfaceflinger/DisplayHardware/HWComposer.h) 是 SurfaceFlinger 操作硬件设备的入口, 它的构造方法如下
```
// frameworks/native/services/surfaceflinger/DisplayHardware/HWComposer.cpp
HWComposer::HWComposer(std::unique_ptr<android::Hwc2::Composer> composer)
      : mHwcDevice(std::make_unique<HWC2::Device>(std::move(composer))) {}
```
HWComposer 的构造函数中可以看到两个非常重要的对象
- 形参: Hwc2::Composer
- 成员变量: HWC2::Device

#### 1. Hwc2::Composer 的实例化
```
// frameworks/native/services/surfaceflinger/DisplayHardware/ComposerHal.cpp
Composer::Composer(const std::string& serviceName)
    : mWriter(kWriterInitialSize),
      mIsUsingVrComposer(serviceName == std::string("vr"))
{
    // 获取 IComposer 服务
    mComposer = V2_1::IComposer::getService(serviceName);

    ......

    // 创建客户端
    mComposer->createClient(
            [&](const auto& tmpError, const auto& tmpClient)
            {
                if (tmpError == Error::NONE) {
                    mClient = tmpClient;
                }
            });
    ......
}
```
Composer 中定义了诸多操作硬件的方法, 但是其均是有 IComposer 代理实现

#### 2. HWC2::Device 的实例化
```
// frameworks/native/services/surfaceflinger/DisplayHardware/HWC2.cpp
Device::Device(std::unique_ptr<android::Hwc2::Composer> composer) : mComposer(std::move(composer)) {
    loadCapabilities();
}

void Device::loadCapabilities()
{
    .....
    // 调用了 mComposer 的 getCapabilities
    auto capabilities = mComposer->getCapabilities();
    // 添加到 mCapabilities 集合中
    for (auto capability : capabilities) {
        mCapabilities.emplace(static_cast<Capability>(capability));
    }
}
```

#### 3. 回调的注册
```
// frameworks/native/services/surfaceflinger/DisplayHardware/HWComposer.cpp
void HWComposer::registerCallback(HWC2::ComposerCallback* callback,
                                  int32_t sequenceId) {
    mHwcDevice->registerCallback(callback, sequenceId);
}

// frameworks/native/services/surfaceflinger/DisplayHardware/HWC2.cpp
void Device::registerCallback(ComposerCallback* callback, int32_t sequenceId) {
    ......
    mRegisteredCallback = true;
    // 将参数包裹成 ComposerCallbackBridge 对象
    sp<ComposerCallbackBridge> callbackBridge(
            new ComposerCallbackBridge(callback, sequenceId));
    // 调用了 mComposer 的注册函数
    mComposer->registerCallback(callbackBridge);
}
```
好的, 可以看到在回调注册的过程中, 首先将传入的形参包裹成 ComposerCallbackBridge, 然后再调用 Composer 的注册方法, 接下来我们一一查看
```
class ComposerCallbackBridge : public Hwc2::IComposerCallback {
public:
    ComposerCallbackBridge(ComposerCallback* callback, int32_t sequenceId)
            : mCallback(callback), mSequenceId(sequenceId) {}

    Return<void> onHotplug(Hwc2::Display display,
                           IComposerCallback::Connection conn) override
    {
        HWC2::Connection connection = static_cast<HWC2::Connection>(conn);
        mCallback->onHotplugReceived(mSequenceId, display, connection);
        return Void();
    }

    Return<void> onRefresh(Hwc2::Display display) override
    {
        mCallback->onRefreshReceived(mSequenceId, display);
        return Void();
    }

    Return<void> onVsync(Hwc2::Display display, int64_t timestamp) override
    {
        mCallback->onVsyncReceived(mSequenceId, display, timestamp);
        return Void();
    }

private:
    ComposerCallback* mCallback;
    int32_t mSequenceId;
};
```
ComposerCallbackBridge 可以视为 ComposerCallback 的代理实现, 其内部定义了函数
- onHotplug
- onRefresh
- onVsync

从这里可以看出硬件驱动接受到的信号后, 最终会通过 ComposerCallbackBridge 回调到 mCallback 中, 而 mCallback 的实现即为 SurfaceFlinger

也就是说**最终的硬件信号会汇总到 SurfaceFlinger 中处理**

### 四) 初始化显示设备
```
// frameworks/native/services/surfaceflinger/DisplayHardware/HWComposer.cpp
void SurfaceFlinger::initializeDisplays() {
    class MessageScreenInitialized : public MessageBase {
        SurfaceFlinger* flinger;
    public:
        explicit MessageScreenInitialized(SurfaceFlinger* flinger) : flinger(flinger) { }
        virtual bool handler() {
            // 回调 onInitializeDisplays 函数
            flinger->onInitializeDisplays();
            return true;
        }
    };
    sp<MessageBase> msg = new MessageScreenInitialized(this);
    postMessageAsync(msg);  // we may be called from main thread, use async message
}

void SurfaceFlinger::onInitializeDisplays() {
    // reset screen orientation and use primary layer stack
    Vector<ComposerState> state;
    Vector<DisplayState> displays;
    DisplayState d;
    d.what = DisplayState::eDisplayProjectionChanged |
             DisplayState::eLayerStackChanged;
    d.token = mBuiltinDisplays[DisplayDevice::DISPLAY_PRIMARY];
    d.layerStack = 0;
    d.orientation = DisplayState::eOrientationDefault;
    d.frame.makeInvalid();
    d.viewport.makeInvalid();
    d.width = 0;
    d.height = 0;
    displays.add(d);
    setTransactionState(state, displays, 0);
    setPowerModeInternal(getDisplayDevice(d.token), HWC_POWER_MODE_NORMAL,
                         /*stateLockHeld*/ false);

    const auto& activeConfig = getBE().mHwc->getActiveConfig(HWC_DISPLAY_PRIMARY);
    const nsecs_t period = activeConfig->getVsyncPeriod();
    mAnimFrameTracker.setDisplayRefreshPeriod(period);

    setCompositorTimingSnapped(0, period, 0);
}
```

到这里 SurfaceFlinger 就初始化好了, 下面回顾一下整体的架构

### 回顾)
![SurfaceFlinger 初始化相关类](https://i.loli.net/2019/10/23/J7eSPCO3ndb9TYf.png)

## 三. SurfaceFlinger 的启动
```
// frameworks/native/services/surfaceflinger/SurfaceFlinger.cpp
void SurfaceFlinger::run() {
    do {
        waitForEvent();
    } while (true);
}


void SurfaceFlinger::waitForEvent() {
    mEventQueue->waitMessage();
}
```
好的, 可以看到 SurfaceFlinger 的 run 函数非常简单, 即不断等待 Event 回调的死循环

## 总结
![SurfaceFlinger 初始化相关类](https://i.loli.net/2019/10/23/ODuxcyfPS36snCQ.png)

这篇文章主要是从广义上了解 SurfaceFlinger 的启动流程, 这里做个简单的总结, Surface 的启动主要有如下几个步骤
- **SurfaceFlinger 的创建**
  - **创建 DispSync 对象**:  用于将硬件 hw-Vsync 信号转为软件 sw-Vysnc 信号
    - DispSync.init 会**启动 DisplaySyncThread 线程**, 该线程在 sw-Vysnc 到来时唤醒并分发给监听者
  - **onFirstRef 创建 MessageQueue 对象**
    - Native 层的 MessageQueue 与 Java 的不同, 它没有真正的队列, 而是配合 Looper 进行 IO 多路复用的监听
- **SurfaceFlinger 的初始化**
  - **创建 APP 的 Vsync 信号处理线程 EventThread-app**
    - 负责接收 sw-Vsync 并且发送信号给 App 进程的 Choreographer 执行 View 的绘制
  - **创建 SurfaceFlinger 的 Vysnc 信号处理线程 EventThread-sf**
    - 负责接收 sw-Vsync 并且发送给 SurfaceFlinger 触发图层的合成
  - EventThread 执行过程
    - 使用 Connection 描述 Vsync 信号的处理者
    - 有 Vsync 信号时, 会写入 Connection 的 BitTube
  - **MessageQueue 监听 EventThread-sf 线程**
    - 创建与 EventThread 的连接 Connection
    - 获取 Connection 内部的 BitTube 对象
    - **使用 mLooper 监听 BitTube，一旦有数据到来则调用 cb_eventReceiver**
  - **初始化 HWComposer**
    - onHotplugReceived: 处理热插拔信号
    - onRefreshReceived: 处理刷新信号
    - onVsyncReceived: 处理垂直同步信号
  - **创建 EventControlThread 用来开关 Vsync 信号**
- **SurfaceFlinger 的发布**
  - 将这个 Binder 本地对象发布到 ServiceManager 进程
- **SurfaceFlinger 的启动**
  - 死循环, 等待 Looper 唤醒处理任务

简单总结起来, SurfaceFlinger 创建了 4 个非常重要的线程
- **DispSyncThread**: 用于将 DispSync 处理好的 SW-VSYNC 分发给监听者 DisplaySyncSource
- **EventThread-app**: 维护了与 App 进程的 Connection 连接, 用于接收 DispSyncThread 发送的信号, 并且发送给 App 端, 会触发 Choreographer 执行 View 的绘制流程
- **EventThread-sf**: 维护了与 SurfaceFlinger 的 Connection, 接收到 VSYNC 后, 会调用 cb_eventReceiver 执行图层的合成操作
- **EventControlThread**: 用于控制硬件 Vsync 的开关


## 参考文献
- [Android Project Buffer](https://blog.csdn.net/innost/article/details/8272867)
- [Android Developer DispSync](https://source.android.google.cn/devices/graphics/implement-vsync)
- [DispSync 的 SW-Vsync 计算细节](https://www.jianshu.com/p/d3e4b1805c92)