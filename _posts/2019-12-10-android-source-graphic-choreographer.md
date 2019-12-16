---
title: Android 系统架构 —— Choreographer 的工作机制
permalink: android-source/graphic-choreographer
key: android-source-graphic-choreographer
tags: AndroidFramework
---

## 前言
在 ViewRootImpl 的创建过程中, 我们看到了它会通过 Choreographer.getInstance 获取到 Choreographer 对象

```java
public final class ViewRootImpl implements ViewParent,
        View.AttachInfo.Callbacks, ThreadedRenderer.DrawCallbacks {
    ......
    
    Choreographer mChoreographer;
     
    public ViewRootImpl(Context context, Display display) {
        ......
        mContext = context;
        ......
        // 获取渲染编舞者实例对象
        mChoreographer = Choreographer.getInstance();
    }

}
```
这里我们就深入的了解一下 Choreographer 的工作机制, 主要内容如下
- Choreographer 的创建
- VSYNC 信号的分发

接下来先看看它的创建流程

<!--more-->
 
## 一. 创建流程
```java
public final class Choreographer {
    
    // Thread local storage for the choreographer.
    private static final ThreadLocal<Choreographer> sThreadInstance =
            new ThreadLocal<Choreographer>() {
        @Override
        protected Choreographer initialValue() {
            Looper looper = Looper.myLooper();
            if (looper == null) {
                throw new IllegalStateException("The current thread must have a looper!");
            }
            // 创建 Choreographer 对象
            Choreographer choreographer = new Choreographer(looper, VSYNC_SOURCE_APP);
            if (looper == Looper.getMainLooper()) {
                mMainInstance = choreographer;
            }
            return choreographer;
        }
    };

    private static volatile Choreographer mMainInstance;

    public static Choreographer getInstance() {
        return sThreadInstance.get();
    }
    
}
```
从 Choreographer 的 getInstance 方法中, 我们可以看到它调用 ThreadLocal.get 方法, 从当前线程的 ThreadLocalMap 散列表中获取以这个 ThreadLocal 为 Key 的 Value 值, 若 Value 不存在, 则会调用 initialValue 创建 Choreographer 实例对象

**由此我们可以得出 Choreographer 是线程单例的, 并且它只能运行在存在 Looper 的线程中**, 下面看看它的构造函数

```java
public final class Choreographer {
    
    private static final boolean USE_VSYNC = SystemProperties.getBoolean(
            "debug.choreographer.vsync", true);
    
    
    // 触摸事件
    public static final int CALLBACK_INPUT = 0;
    // 动画的 Callback
    public static final int CALLBACK_ANIMATION = 1;
    // View 遍历的 Callback
    public static final int CALLBACK_TRAVERSAL = 2;
    // 在 Traversals 之后, 优先级最低的 Callback
    public static final int CALLBACK_COMMIT = 3;

    private static final int CALLBACK_LAST = CALLBACK_COMMIT;
    
    private Choreographer(Looper looper, int vsyncSource) {
        mLooper = looper;
        // 1. 用于处理 Frame 相关消息的 Handler
        mHandler = new FrameHandler(looper);
        // 2. 创建垂直同步信号的接收器
        mDisplayEventReceiver = USE_VSYNC
                ? new FrameDisplayEventReceiver(looper, vsyncSource)
                : null;
        // 记录最后绘制的时间
        mLastFrameTimeNanos = Long.MIN_VALUE;
        // 记录每帧间隔: 1s / 屏幕刷新率
        mFrameIntervalNanos = (long)(1000000000 / getRefreshRate());
        // 3. 创建 CallbackQueue 数组, 一共有 4 个 CallbackQueue
        mCallbackQueues = new CallbackQueue[CALLBACK_LAST + 1];
        for (int i = 0; i <= CALLBACK_LAST; i++) {
            mCallbackQueues[i] = new CallbackQueue();
        }
        // b/68769804: 用于 Low FPS 实验
        setFPSDivisor(SystemProperties.getInt(ThreadedRenderer.DEBUG_FPS_DIVISOR, 1));
    }
    
}
```
从 Choreographer 的构造中, 我们可以看到它主要做了三件事情
- 创建当前线程的 FrameHandler, 用于处理帧操作
- 创建 FrameDisplayEventReceiver, 用于接收硬件信号, 即 SurfaceFlinger 的 EventThread-app 发出的 VSYNC 信号
- 创建 CallbackQueue 数组, 每个数组的槽位如下
  - 0: 触摸事件的 Callback
  - 1: 动画的 Callback
  - 2: Traversals 的 Callback
  - 3: Commit 的 Callback

由此我们可以猜测, Choreographer 主要职责是通过 FrameDisplayEventReceiver 接收 VSYNC 信号, 然后调用 FramHandler 处理 CallbackQueue 中的任务

这里来我们先看看 FrameDisplayEventReceiver 的创建

### 一) FrameDisplayEventReceiver 的创建
```java
public final class Choreographer {
    
    private final class FrameDisplayEventReceiver extends DisplayEventReceiver
            implements Runnable {
        private boolean mHavePendingVsync;
        private long mTimestampNanos;
        private int mFrame;

        public FrameDisplayEventReceiver(Looper looper, int vsyncSource) {
            // 调用了父类构造方法
            super(looper, vsyncSource);
        }
        
    }
    
}

public abstract class DisplayEventReceiver {
    
    // 描述 APP-VSYNC 信好
    public static final int VSYNC_SOURCE_APP = 0;
    
    private MessageQueue mMessageQueue;
    private long mReceiverPtr;

    public DisplayEventReceiver(Looper looper, int vsyncSource) {
        ......
        mMessageQueue = looper.getQueue();
        // 调用了进行 Native 初始化 
        mReceiverPtr = nativeInit(new WeakReference<DisplayEventReceiver>(this), mMessageQueue,
                vsyncSource);
    }
    
    private static native long nativeInit(WeakReference<DisplayEventReceiver> receiver,
            MessageQueue messageQueue, int vsyncSource);
    
}
```
FrameDisplayEventReceiver 的构造的核心操作即调用 nativeInit 执行 Native 初始化, 

DisplayEventReceiver 的 nativeInit 方法定义在 [android_view_DisplayEventReceiver.cpp](http://androidxref.com/9.0.0_r3/xref/frameworks/base/core/jni/android_view_DisplayEventReceiver.cpp) 中

我们追踪下去看看

```
// frameworks/base/core/jni/android_view_DisplayEventReceiver.cpp
static jlong nativeInit(JNIEnv* env, jclass clazz, jobject receiverWeak,
        jobject messageQueueObj, jint vsyncSource) {
    // 获取 Native 的 MessageQueue 对象
    sp<MessageQueue> messageQueue = android_os_MessageQueue_getMessageQueue(env, messageQueueObj);
    if (messageQueue == NULL) {
        jniThrowRuntimeException(env, "MessageQueue is not initialized.");
        return 0;
    }
    // 1. 创建 NativeDisplayEventReceiver 对象
    sp<NativeDisplayEventReceiver> receiver = new NativeDisplayEventReceiver(env,
            receiverWeak, messageQueue, vsyncSource);
    // 2. 调用 NativeDisplayEventReceiver 的 initialze 方法执行初始化操作
    status_t status = receiver->initialize();
    ......
    return reinterpret_cast<jlong>(receiver.get());
}
```
nativeInit 中先是创建了 **NativeDisplayEventReceiver** 对象, 紧接着调用了 **initialize** 函数, 下面我们逐一看看

### 二) 创建 NativeDisplayEventReceiver
```
// frameworks/base/core/jni/android_view_DisplayEventReceiver.cpp
NativeDisplayEventReceiver::NativeDisplayEventReceiver(JNIEnv* env,
        jobject receiverWeak, const sp<MessageQueue>& messageQueue, jint vsyncSource) :
        // 1. 父类构造函数
        DisplayEventDispatcher(
            messageQueue->getLooper(),
            static_cast<ISurfaceComposer::VsyncSource>(vsyncSource)
        ),
        // Java 的 DisplayEventReceiver 对象
        mReceiverWeakGlobal(env->NewGlobalRef(receiverWeak)),
        // 当前线程的 MessageQueue
        mMessageQueue(messageQueue) {
    ......
}

// frameworks/base/libs/androidfw/DisplayEventDispatcher.cpp
DisplayEventDispatcher::DisplayEventDispatcher(const sp<Looper>& looper,
        ISurfaceComposer::VsyncSource vsyncSource) :
        mLooper(looper), 
        // 创建了 DisplayEventReceiver 对象
        mReceiver(vsyncSource),
        mWaitingForVsync(false) {
    ......
}
```
好的, 可以看到 NativeDisplayEventReceiver 继承自 DisplayEventDispatcher 对象, 在其父类构造函数调用期间, 创建 [DisplayEventReceiver](http://androidxref.com/9.0.0_r3/xref/frameworks/native/libs/gui/DisplayEventReceiver.cpp) 对象来接收 VSYNC 信号

因此 NativeDisplayEventReceiver 通过 mReceiver 成员变量实现 VSYNC 信号的接收, 通过继承父类 DisplayEventDispatcher, 实现对 VSYNC 的分发功能


关于其分发的实现, 我们到后面再进行查看, 这里先看看 VSYNC 信号接收器 DisplayEventReceiver 的创建流程

下面看看它的实现
```
// frameworks/native/libs/gui/DisplayEventReceiver.cpp
DisplayEventReceiver::DisplayEventReceiver(ISurfaceComposer::VsyncSource vsyncSource) {
    // 获取 SurfaceFlinger 的代理对象
    sp<ISurfaceComposer> sf(ComposerService::getComposerService());
    if (sf != NULL) {
        // 1. 创建与 SurfaceFlinger 进程中 EventTread-app 线程的一个 vsyncSource 信号连接
        mEventConnection = sf->createDisplayEventConnection(vsyncSource);
        if (mEventConnection != NULL) {
            // 2. 获取这个连接的 BitTube, 保存在 mDataChannel 中
            mDataChannel = std::make_unique<gui::BitTube>();
            mEventConnection->stealReceiveChannel(mDataChannel.get());
        }
    }
}
```
可以看到 DisplayEventReceiver 为了能够获取 VSYNC 信号, 它创建了与 SurfaceFlinger 进程中 EventThread-app 的一个 Connection 的 Binder 代理对象, 然后获取这个 Connection 读端的文件描述符保存到 mDataChannel 中

下面我们看看 SurfaceFlinger 进程获取的具体实现

#### 1. 创建 Connection 对象
```
// frameworks/native/services/surfaceflinger/SurfaceFlinger.cpp
sp<IDisplayEventConnection> SurfaceFlinger::createDisplayEventConnection(
        ISurfaceComposer::VsyncSource vsyncSource) {
    if (vsyncSource == eVsyncSourceSurfaceFlinger) {
        return mSFEventThread->createEventConnection();
    }
    // 很显然我们这里是 App 端的 VsyncSource
    else {
        return mEventThread->createEventConnection();
    }
}


// frameworks/native/services/surfaceflinger/EventThread.cpp
sp<BnDisplayEventConnection> EventThread::createEventConnection() const {
    // 很简单, 创建了一个 Connection 对象
    return new Connection(const_cast<EventThread*>(this));
}

EventThread::Connection::Connection(EventThread* eventThread)
      : count(-1), mEventThread(eventThread),
      // 创建了 BitTube mChannel, 用于 EventThread 与接收端进行消息传递
      mChannel(gui::BitTube::DefaultSize) {}
```
可以看到 Connection 创建的过程中, 创建了一个 [BitTube](http://androidxref.com/9.0.0_r3/xref/frameworks/native/libs/gui/BitTube.cpp) 对象保存在 mChannel 中, 下面看看 BitTube 的构建

```
// frameworks/native/libs/gui/BitTube.cpp
// Socket buffer 默认为 128kb, VSYCN 信息量较少, 这里改为 4KB
static const size_t DEFAULT_SOCKET_BUFFER_SIZE = 4 * 1024;

BitTube::BitTube(size_t bufsize) {
    init(bufsize, bufsize);
}

BitTube::BitTube(DefaultSizeType) : BitTube(DEFAULT_SOCKET_BUFFER_SIZE) {}

void BitTube::init(size_t rcvbuf, size_t sndbuf) {
    int sockets[2];
    if (socketpair(AF_UNIX, SOCK_SEQPACKET, 0, sockets) == 0) {
        size_t size = DEFAULT_SOCKET_BUFFER_SIZE;
        setsockopt(sockets[0], SOL_SOCKET, SO_RCVBUF, &rcvbuf, sizeof(rcvbuf));
        setsockopt(sockets[1], SOL_SOCKET, SO_SNDBUF, &sndbuf, sizeof(sndbuf));
        // since we don't use the "return channel", we keep it small...
        setsockopt(sockets[0], SOL_SOCKET, SO_SNDBUF, &size, sizeof(size));
        setsockopt(sockets[1], SOL_SOCKET, SO_RCVBUF, &size, sizeof(size));
        fcntl(sockets[0], F_SETFL, O_NONBLOCK);
        fcntl(sockets[1], F_SETFL, O_NONBLOCK);
        // 接收端保存 0 号端口
        mReceiveFd.reset(sockets[0]);
        // 发送端保存 1 号端口
        mSendFd.reset(sockets[1]);
    } else {
        ......
    }
}
```
好的, 可以看到 BitTube 是一个 Socket 的封装类, 因此 SurfaceFlinger 进程的 VSYNC 信号, 是由 Socket 发送到客户端的

下面看看 获取 Connection 读端文件描述符的

#### 2. 获取 Connection 读端文件描述符
```
// frameworks/native/services/surfaceflinger/EventThread.cpp
status_t EventThread::Connection::stealReceiveChannel(gui::BitTube* outChannel) {
    // 通过 mChannel 的 moveReceiveFd 函数获取读端文件描述符
    outChannel->setReceiveFd(mChannel.moveReceiveFd());
    return NO_ERROR;
}
```
Connection::stealReceiveChannel 最终通过 mChannel.moveReceiveFd() 获取了接收端的文件描述符

```
// frameworks/native/libs/gui/BitTube.cpp
base::unique_fd BitTube::moveReceiveFd() {
    return std::move(mReceiveFd);
}
```
可以看到, 这里简单的返回了 Socket 读端的文件描述符

最终会作为传出参数, 保存在 outChannel 中, 通过 Binder 驱动返回到我们的客户端进程

到这里我们的 DisplayEventReceiver 就创建好了, 其内部的 mDataChannel 保存了与 SurfaceFlinger 进程 EventThread-app 线程的 Connection 中读端的文件描述符

接下来我们看看 NativeDisplayEventReceiver 的初始化操作

### 三) 初始化 DisplayEventDispatcher
NativeDisplayEventReceiver 的 initialize 函数由父类 DisplayEventDispatcher 实现, 它的实现如下
```
// frameworks/base/libs/androidfw/DisplayEventDispatcher.cpp
status_t DisplayEventDispatcher::initialize() {
    ......
    // 通过 Looper 监听了 Socket 读端描述符
    // 唤醒时的事件类型为 EVENT_INPUT
    // 这个 this 指代 DisplayEventDispatcher 中的 handleEvent 函数
    int rc = mLooper->addFd(mReceiver.getFd(), 0, Looper::EVENT_INPUT,
            this, NULL);
    ......
    return OK;
}
```
可以看到这里其初始化操作, 即让 Looper 监听了 Socket 读端的文件描述符, 当有 VSYNC 信号时, 会调用发送 EVENT_INPUT 到 handleEvent 函数中处理

### 四) 回顾
![VSYNC 信号接收流程图](https://i.loli.net/2019/12/16/rda5z1IBSY9M4Cu.png)

整个 Choreographer 的创建流程如下所示
- **创建 FrameHandler 处理任务**
- **创建 FrameDisplayEventReceiver 接收 VSYNC 信号**
  - **创建 NativeDisplayEventReceiver**
    - 父类 DisplayEventDispatcher 负责信号分发
    - 成员变量 DisplayEventReceiver mReceiver 负责信号接收
  - **创建 DisplayEventReceiver**
    - 创建一个与 SurfaceFlinger 进程中 EventThread-app 的 Connection
      - 创建一个 Socket 链接, 缓冲区为 4 kb
    - 获取 Connection 中 Socket 读端的文件描述符保存到 mDataChannel 中
  - **初始化 DisplayEventDispatcher**
    - Looper 监听 Socket 读端, 有数据时回调 handleEvent 函数处理
- **创建 CallbackQueue 数组, 描述各个不同任务的队列**
  - 0: 触摸事件的 Callback
  - 1: 动画的 Callback
  - 2: Traversals 的 Callback
  - 3: Commit 的 Callback

好的, 接下来我们看看一个 VSYNC 信号到来时, handleEvent 是如何进行分发处理的

## 二. VSYNC 的分发
```
// frameworks/base/libs/androidfw/DisplayEventDispatcher.cpp
int DisplayEventDispatcher::handleEvent(int, int events, void*) {
    ......
    nsecs_t vsyncTimestamp;
    int32_t vsyncDisplayId;
    uint32_t vsyncCount;
    // 1. 从 Socket 端口中读取数据
    if (processPendingEvents(&vsyncTimestamp, &vsyncDisplayId, &vsyncCount)) {
        mWaitingForVsync = false;
        // 2. 分发这个 VSYNC 信号
        dispatchVsync(vsyncTimestamp, vsyncDisplayId, vsyncCount);
    }
    return 1; // keep the callback
}
```
handleEvent 内部实现非常清晰, 首先通过 processPendingEvents 从 Socket 端口中读取信号保存到传出参数中, 然后调用 dispatchVsync 对读取到的 VSYNC 信号进行分发处理, 下面我们逐一查看

### 一) 从 Socket 端口读取信号
```
// frameworks/base/libs/androidfw/DisplayEventDispatcher.cpp

// Number of events to read at a time from the DisplayEventDispatcher pipe.
// The value should be large enough that we can quickly drain the pipe
// using just a few large reads.
static const size_t EVENT_BUFFER_SIZE = 100;

bool DisplayEventDispatcher::processPendingEvents(
        nsecs_t* outTimestamp, int32_t* outId, uint32_t* outCount) {
    bool gotVsync = false;
    // 创建一个大小为 100 的事件数据
    DisplayEventReceiver::Event buf[EVENT_BUFFER_SIZE];
    ssize_t n;
    // 1. 从 Socket 读端获取数据
    while ((n = mReceiver.getEvents(buf, EVENT_BUFFER_SIZE)) > 0) {
        ......
        for (ssize_t i = 0; i < n; i++) {
            const DisplayEventReceiver::Event& ev = buf[i];
            switch (ev.header.type) {
            // 读取到了一个 VSYNC 信号
            case DisplayEventReceiver::DISPLAY_EVENT_VSYNC:
                ......
                gotVsync = true;
                // 2. 为传出参数赋值
                // 获取信号时间戳
                *outTimestamp = ev.header.timestamp;
                // 获取事件 ID
                *outId = ev.header.id;
                // 获取信号数量
                *outCount = ev.vsync.count;
                break;
            case DisplayEventReceiver::DISPLAY_EVENT_HOTPLUG:
                // 对热插拔信号直接进行分发操作
                dispatchHotplug(ev.header.timestamp, ev.header.id, ev.hotplug.connected);
                break;
            default:
                ......
                break;
            }
        }
    }
    ......
    return gotVsync;
}
```
从这里可以看出, 它首先将事件读入 buf 数组中, 然后遍历数组中的事件, SurfaceFlinger 进程可能发送的事件可能有 VSYNC 信号和 HOTPLUG 信号, 对于 HOTPLUG 操作会直接进行分发, 可见显示设备的热插拔优先级处理比 VSYNC 要高

这里我们主要一下 VSYNC 信号, 可以看到它会给传出参数中赋值, 也就是说最终传出参数会保留读取到的最后一个 VSYNC 信号值

既然获取到了 VSYNC 信号, 下面我们看看 dispatchVsync 的分发实现

### 二) VSYNC 的分发
```
// frameworks/base/libs/androidfw/DisplayEventDispatcher.cpp
void NativeDisplayEventReceiver::dispatchVsync(nsecs_t timestamp, int32_t id, uint32_t count) {
    JNIEnv* env = AndroidRuntime::getJNIEnv();
    // 1. 这里获取到了 Java 层的 DisplayEventReceiver 对象
    ScopedLocalRef<jobject> receiverObj(env, jniGetReferent(env, mReceiverWeakGlobal));
    if (receiverObj.get()) {
        // 2. 通过 JNIEnv 调用 dispatchVsync 方法
        env->CallVoidMethod(receiverObj.get(),
                gDisplayEventReceiverClassInfo.dispatchVsync, timestamp, id, count);
        ......
    }
}
```
好的, 可以看到至此 VSYNC 信号就流入了 Java 层, 下面我们看看 dispatchVsync 的实现

```java
public abstract class DisplayEventReceiver {


    // Called from native code.
    @SuppressWarnings("unused")
    private void dispatchVsync(long timestampNanos, int builtInDisplayId, int frame) {
        // 回调实现类 FrameDisplayEventReceiver 的 onVsync 方法
        onVsync(timestampNanos, builtInDisplayId, frame);
    }

}
```
DisplayEventReceiver 的 dispatchVsync 实现也非常简单, 它回调了 onVsync 方法, 交由实现类 FrameDisplayEventReceiver 处理这个 VSYNC 信号

```java
public final class Choreographer {

    private final class FrameDisplayEventReceiver extends DisplayEventReceiver
            implements Runnable {
   
        @Override
        public void onVsync(long timestampNanos, int builtInDisplayId, int frame) {
            // Choreographer 目前不支持处理第二个显示屏的 VSYNC 信号
            if (builtInDisplayId != SurfaceControl.BUILT_IN_DISPLAY_ID_MAIN) {
                ......
                return;
            }

            // 矫正 VSYNC 信号的时间
            long now = System.nanoTime();
            if (timestampNanos > now) {
                timestampNanos = now;
            }
            
            if (mHavePendingVsync) {
                
            } else {
                mHavePendingVsync = true;
            }
            
            mTimestampNanos = timestampNanos;
            // 记录垂直同步信号的数量
            mFrame = frame;
            // 发送一个异步消息, 保证 MessageQueue 的睡眠被立即唤醒
            Message msg = Message.obtain(mHandler, this);
            msg.setAsynchronous(true);
            mHandler.sendMessageAtTime(msg, timestampNanos / TimeUtils.NANOS_PER_MS);
        }
        
        @Override
        public void run() {
            mHavePendingVsync = false;
            // 执行异步消息
            doFrame(mTimestampNanos, mFrame);
        }
        
    }

}
```
好的从 onVsync 方法中还是能看到很多有价值的信息, Choreographer 目前仅支持处理主屏幕的 VSYNC 信号, **最终会通过向当前线程的 MessageQueue 中发送一条异步消息, 来强制唤醒 MessageQueue Native 层在 EventFd 上的睡眠操作**

MessageQueue 被异步消息唤醒之后, 会调用 doFrame 继续执行应用进程的帧的准备操作

#### 准备帧数据
```
public final class Choreographer {
    
    // 丢帧上报上限为 30 帧
    private static final int SKIPPED_FRAME_WARNING_LIMIT = SystemProperties.getInt(
            "debug.choreographer.skipwarning", 30);
    
    void doFrame(long frameTimeNanos, int frame) {
        final long startNanos;
        // 1. 更新待准备帧信息
        synchronized (mLock) {
            if (!mFrameScheduled) {
                return; // no work to do
            }
            ......

            long intendedFrameTimeNanos = frameTimeNanos;
            startNanos = System.nanoTime();
            
            // 1.1 计算丢帧数量
            // 计算这个 VSYNC 到来与处理的时间差
            final long jitterNanos = startNanos - frameTimeNanos;
            if (jitterNanos >= mFrameIntervalNanos) {
                // 1.2 计算丢帧数量
                final long skippedFrames = jitterNanos / mFrameIntervalNanos;
                if (skippedFrames >= SKIPPED_FRAME_WARNING_LIMIT) {
                    // 若我们在主线程处理耗时操作是, 控制台便会打印这条日志
                    Log.i(TAG, "Skipped " + skippedFrames + " frames!  "
                            + "The application may be doing too much work on its main thread.");
                }
                // 1.3 矫正 VSYNC 发布时间
                final long lastFrameOffset = jitterNanos % mFrameIntervalNanos;
                ......
                frameTimeNanos = startNanos - lastFrameOffset;
            }
            ......
            
            // 1.3 更新当前要准备的帧信息
            mFrameInfo.setVsync(intendedFrameTimeNanos, frameTimeNanos);
            mFrameScheduled = false;
            mLastFrameTimeNanos = frameTimeNanos;
        }
        // 2. 执行回调队列, 准备帧信息
        try {
            Trace.traceBegin(Trace.TRACE_TAG_VIEW, "Choreographer#doFrame");
            AnimationUtils.lockAnimationClock(frameTimeNanos / TimeUtils.NANOS_PER_MS);
            
            // 2.1 执行输入流的 Callback 队列
            mFrameInfo.markInputHandlingStart();
            doCallbacks(Choreographer.CALLBACK_INPUT, frameTimeNanos);
            
            // 2.2 执行动画的 Callback 队列
            mFrameInfo.markAnimationsStart();
            doCallbacks(Choreographer.CALLBACK_ANIMATION, frameTimeNanos);
            
            // 2.3 执行 View 绘制流程的 Callback 队列
            mFrameInfo.markPerformTraversalsStart();
            doCallbacks(Choreographer.CALLBACK_TRAVERSAL, frameTimeNanos);
            
            // 2.4 执行 COMMIT 的 Callback 队列
            doCallbacks(Choreographer.CALLBACK_COMMIT, frameTimeNanos);
        } finally {
            AnimationUtils.unlockAnimationClock();
            Trace.traceEnd(Trace.TRACE_TAG_VIEW);
        }
        ......
    }
    
}
```
doFrame 方法非常有趣, 它主要处理两个方面的事务
- 更新待准备帧信息
  - 计算与上一帧的差值
  - 丢帧数 = 与上一帧的差值 / 帧间隔
- 根据优先级执行 CallbackQueue
  - 输入流事件最优先执行
  - 动画事件次优先
  - View 绘制的流程滞后
  - Commit 最后

**从更新待准备帧信息中我们可以看到丢帧了数的计算方式, 单位时间的丢帧数是用来衡量页面流畅度的指标, 这种方式在监控主线程卡顿中常被用到**

CallbackQueue 的执行执行如下所示
```
public final class Choreographer {

    void doCallbacks(int callbackType, long frameTimeNanos) {
        CallbackRecord callbacks;
        synchronized (mLock) {
            ......
            final long now = System.nanoTime();
            // 1. 获取指定 Type 的回调队列
            callbacks = mCallbackQueues[callbackType].extractDueCallbacksLocked(
                    now / TimeUtils.NANOS_PER_MS);
            if (callbacks == null) {
                return;
            }
            mCallbacksRunning = true;
            ......
        }
        try {
            // 2. 执行所有 CallbackRecord 的 run 方法
            for (CallbackRecord c = callbacks; c != null; c = c.next) {
                c.run(frameTimeNanos);
            }
        } finally {
            synchronized (mLock) {
                mCallbacksRunning = false;
                // 3. 释放 CallbackRecord 资源
                do {
                    final CallbackRecord next = callbacks.next;
                    recycleCallbackLocked(callbacks);
                    callbacks = next;
                } while (callbacks != null);
            }
            ......
        }
    }
    
}
```
关于 CallbackQueue 的执行还是非常简单的
- 根据 Type 获取 CallbackQueue
- 执行 CallbackQueue 中的每一个 CallbackRecord 的 run 方法
- 释放 CallbackRecord 资源

### 三) 回顾
VSYNC 的分发主要如下
- 事件的获取
  - 从 Socket 端口读取事件, 一次性最多读取 100 个事件
  - 对于 HOTPLUG 事件, 立即调用 dispatchHotplug 处理
  - 对于 VSYNC 事件, 保留最后一个调用 dispatchVsync 处理
- 事件的分发
  - FrameDisplayEventReceiver 在 onVsync 中通过发送异步消息唤醒 MessageQueue 
  - 执行 doFrame 准备帧数据
    - 计算丢帧数
    - 按照优先级执行渲染操作
      - 输入流事件最优先执行
      - 动画事件次优先
      - View 绘制的流程滞后
      - Commit 事件优先级最低

帧数据准备完毕之后会推入当前 ViewRootImpl 中 Surface 的 GraphicBuffer 中, SurfaceFlinger 也会根据 VSYNC 信号对将 Buffer 合成后输出到屏幕上, 至此 Choreographer 的工作机制就分析完毕了

## 总结
整个 Choreographer 的创建流程如下所示
- **创建 FrameHandler 处理任务**
- **创建 FrameDisplayEventReceiver 接收 VSYNC 信号**
  - **创建 NativeDisplayEventReceiver**
    - 父类 DisplayEventDispatcher 负责信号分发
    - 成员变量 DisplayEventReceiver mReceiver 负责信号接收
  - **创建 DisplayEventReceiver**
    - 创建一个与 SurfaceFlinger 进程中 EventThread-app 的 Connection
      - 创建一个 Socket 链接, 缓冲区为 4 kb
    - 获取 Connection 中 Socket 读端的文件描述符保存到 mDataChannel 中
  - **初始化 DisplayEventDispatcher**
    - Looper 监听 Socket 读端, 有数据时回调 handleEvent 函数处理
- **创建 CallbackQueue 数组, 描述各个不同任务的队列**
  - 0: 触摸事件的 Callback
  - 1: 动画的 Callback
  - 2: Traversals 的 Callback
  - 3: Commit 的 Callback

VSYNC 的分发主要如下
- **事件的获取**
  - 从 Socket 端口读取事件, 一次性最多读取 100 个事件
  - 对于 HOTPLUG 事件, 立即调用 dispatchHotplug 处理
  - 对于 VSYNC 事件, 保留最后一个调用 dispatchVsync 处理
- **事件的分发**
  - FrameDisplayEventReceiver 在 onVsync 中通过发送异步消息唤醒 MessageQueue 
  - 执行 doFrame 准备帧数据
    - 计算丢帧数
    - 按照优先级执行渲染操作
      - 输入流事件最优先执行
      - 动画事件次优先
      - View 绘制的流程滞后
      - Commit 事件优先级最低

分析了 Choreographer 源码之后, 不禁感叹其实现真是太巧妙了, 它配合 VSYNC 很大程度上解决了 Android 系统的卡顿问题, 笔者总结下来有如下三点
- **通过接收 SurfaceFlinger 发出的 VSYNC 信号, 将 UI 渲染任务同步到 VSYNC 信号的时间线上, 更加有规律的调度 CPU 进行 UI 渲染**
- **帧准备的操作通过异步消息来强制唤醒 MessageQueue Native 层在 EventFd 上的睡眠操作, 保证帧准备操作的能够及时响应**
- **CallbackQueue 的执行顺序优先照顾了触摸和动画, 将最笨重的 View 绘制三大流程放到了后面, 优先照顾了用户操作的跟手性和动画的流畅度**