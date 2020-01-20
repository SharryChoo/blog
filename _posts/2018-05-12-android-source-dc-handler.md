---
title: Android 系统架构 —— Looper 的创建与启动
permalink: android-source/dc-handler
key: android-source-dc-handler
tags: AndroidFramework
---

## 前言
我们知道, 应用进程主线程初始化的入口是在 ActivityThread.main() 中, 我们看看他是如何构建消息队列的
```
public class ActivityThread {

    static volatile Handler sMainThreadHandler;  // set once in main()

    public static void main(String[] args) {
        ......
        // 1. 做一些主线程消息循环的初始操作
        Looper.prepareMainLooper();
        
        ......
        
        // 2. 启动消息循环
        Looper.loop();
    }
    
}
```
好的, 可以看到 ActivityThread 中的消息循环构建过程如下
- 调用 Looper.prepareMainLooper, 做一些准备操作
- 调用 Looper.loop 真正的开启了消息循环

接下来我们先看看准备操作中做了些什么

## 一. 消息循环的准备
```
public final class Looper {

    private static Looper sMainLooper;  // guarded by Looper.class
    
    public static void prepareMainLooper() {
        // 1. 调用 prepare 方法真正执行主线程的准备操作
        prepare(false);
        
        synchronized (Looper.class) {
            if (sMainLooper != null) {
                throw new IllegalStateException("The main Looper has already been prepared.");
            }
            // 2. 调用了 myLooper 方法, 获取一个 Looper 对象给 sMainLooper 赋值
            sMainLooper = myLooper();
        }
    }

    static final ThreadLocal<Looper> sThreadLocal = new ThreadLocal<Looper>();

    private static void prepare(boolean quitAllowed) {
        if (sThreadLocal.get() != null) {
            throw new RuntimeException("Only one Looper may be created per thread");
        }
        // 1.1 new 了一个 Looper 对象
        // 1.2 将这个 Looper 对象写入 ThreadLocal 中
        sThreadLocal.set(new Looper(quitAllowed));
    }
    
    public static @Nullable Looper myLooper() {
        // 2.1 通过 ThreadLocal 获取这个线程中唯一的 Looper 对象
        return sThreadLocal.get();
    }
    
    final MessageQueue mQueue;
    final Thread mThread;
    
    private Looper(boolean quitAllowed) {
        // 创建了一个消息队列
        mQueue = new MessageQueue(quitAllowed);
        // 获取了当前的线程
        mThread = Thread.currentThread();
    }
    
}
```
可以看到 Looper.prepareMainLooper 中主要做了如下几件事情
- 调用 prepare 方法真正执行主线程的准备操作
  - 创建了一个 Looper 对象
    - **创建了 MessageQueue 这个消息队列, 保存到成员变量 mQueue 中**
    - 获取该对象创建线程, 保存到成员变量 mThread 中
  - 将这个 ThreadLocal 和 Looper 存入当前线程的 ThreadLocalMap 中
- 调用 myLooper 方法, **从当前线程的 ThreadLocalMap 中获取 ThreadLocal 对应的 Looper 对象**
- 将这个 Looper 对象, 保存到静态变量 sMainLooper 中, **表示这个是当前应用进程主线程的 Looper 对象**

好的, 可以看到在创建 Looper 对象的时候, 同时会创建一个 MessageQueue 对象, 将它保存到 Looper 对象的成员变量 mQueue 中, 因此每一个 Looper 对象都对应一个 MessageQueue 对象

我们接下来看看 MessageQueue 对象创建时, 做了哪些操作

### 一) MessageQueue 的创建
```
public final class MessageQueue {
    
    private final boolean mQuitAllowed; // true 表示这个消息队列是可停止的
    private long mPtr;                  // 描述一个 Native 的句柄
    
    MessageQueue(boolean quitAllowed) {
        mQuitAllowed = quitAllowed;
        // 获取一个 native 句柄
        mPtr = nativeInit();
    }
    
    private native static long nativeInit();

}
```
好的, 可以看到 MessageQueue 对象在创建非常简单, 内部会调用 nativeInit 来获取一个 native 的句柄

下面我们看看 native 层进行了怎样的初始化

```
// frameworks/base/core/jni/android_os_MessageQueue.cpp
static jlong android_os_MessageQueue_nativeInit(JNIEnv* env, jclass clazz) {
    // 1. 创建了一个 NativeMessageQueue 对象
    NativeMessageQueue* nativeMessageQueue = new NativeMessageQueue();
    ......
    // 增加 env 对它的强引用计数
    nativeMessageQueue->incStrong(env);
    // 2. 将这个 NativeMessageQueue 对象强转成了一个句柄返回 Java 层
    return reinterpret_cast<jlong>(nativeMessageQueue);
}


class NativeMessageQueue : public MessageQueue, public LooperCallback {
public:
    NativeMessageQueue();
    ......
private:
    JNIEnv* mPollEnv;
    jobject mPollObj;
    jthrowable mExceptionObj;
};

NativeMessageQueue::NativeMessageQueue() :
        mPollEnv(NULL), mPollObj(NULL), mExceptionObj(NULL) {
    // 1.1 尝试调用 Looper.getForThread 获取一个 C++ 的 Looper 对象
    mLooper = Looper::getForThread();
    // 1.2 若当前线程没有创建过 Looper, 则走这个分支
    if (mLooper == NULL) {
        // 创建 Looper 对象
        mLooper = new Looper(false);
        // 给当前线程绑定这个 Looper 对象
        Looper::setForThread(mLooper);
    }
}
```
好的可以看到 nativeInit 方法主要做了如下的操作
- 创建线程单例的 NativeMessageQueue 对象
  - 因为 Java 的 MessageQueue 是线程单例的, 因此 NativeMessageQueue 对象也是线程单例的 
- 创建线程单例的 Looper(Native)

可以看到这里的流程与 Java 的相反
- Java 是先创建 Looper, 然后在 Looper 内部创建 MessageQueue
- Native 是先创建 NativeMessageQueue, 在其创建的过程中创建 Looper 

接下来看看这个 C++ 的 Looper 对象在实例化的过程中做了哪些事情

```
// system/core/libutils/Looper.cpp
Looper::Looper(bool allowNonCallbacks) :
        mAllowNonCallbacks(allowNonCallbacks), mSendingMessage(false),
        mPolling(false), mEpollFd(-1), mEpollRebuildRequired(false),
        mNextRequestSeq(0), mResponseIndex(0), mNextMessageUptime(LLONG_MAX) {
    // 1. 通过 Linux 创建 eventfd, 进行线程间通信
    // Android 6.0 之前为 pipe, 6.0 之后为 eventfd
    mWakeEventFd = eventfd(0, EFD_NONBLOCK | EFD_CLOEXEC);
    AutoMutex _l(mLock);
    // 2. 使用 epoll 监听 eventfd
    rebuildEpollLocked();
}

void Looper::rebuildEpollLocked() {
    ......
    // 2.1 创建一个 epoll 对象, 将其文件描述符保存在成员变量 mEpollFd 中
    mEpollFd = epoll_create(EPOLL_SIZE_HINT);
    ......
    // 2.2 使用 epoll 监听 mWakeEventFd, 用于线程唤醒
    int result = epoll_ctl(mEpollFd, EPOLL_CTL_ADD, mWakeEventFd, & eventItem);
    ......
}
```
好的, 可以看到 Looper 实例化时做了如下的操作
- 创建了一个名为 mWakeEventFd 的 eventfd 
  - Android 6.0 之前使用的 pipe
  - Android 6.0 之后使用的 eventfd
    - 比起管道, 它只有一个文件描述符, 更加轻量级
- 创建了一个 IO 多路复用的 epoll 对象
- 让 epoll 对其进行监听 mWakeEventFd

### 二) 回顾
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


## 二. 消息循环的启动
```
public final class Looper {
    
    public static void loop() {
        // 获取调用线程的 Looper 对象
        final Looper me = myLooper();
        ......
        // 获取 Looper 对应的消息队列
        final MessageQueue queue = me.mQueue;
        // 死循环, 不断的处理消息队列中的消息
        for (;;) {
            // 1. 获取消息队列中的消息, 取不到消息会阻塞
            Message msg = queue.next(); // might block
            if (msg == null) {
                // 若取到的消息为 null, 这个 Looper 就结束了
                return;
            }
            ......
            try {
                // 2. 通过 Message 的 Handler 处理消息
                msg.target.dispatchMessage(msg);
            } finally {
                ......
            }
            ......
        }
    }
    
}
```
好的, 可以看到 Looper 的 loop 内处理的事务非常清晰
- 通过 MessageQueue.next() 获取下一条要处理的 Message
- 通过 Message 的 Handler 处理这条消息

好的, 可以看到获取消息的方式是通过 MessageQueue.next 拿到的, 我们接下来看看它是如何获取的

### 一) MessageQueue.next 获取下一条消息
```
public final class MessageQueue {
    
    Message next() {
        // 获取 NativeMessageQueue 的句柄
        final long ptr = mPtr;
        if (ptr == 0) {
            return null;
        }
        // 描述空闲事件处理者的数量, 初始值为 -1 
        int pendingIdleHandlerCount = -1; 
        // 描述当前线程没有新消息处理时, 可睡眠的时间
        int nextPollTimeoutMillis = 0;
        // 死循环, 获取可执行的 Message 对象
        for (;;) {
            ......
            // 1. 这里调用了 nativePollOnce, 则执行线程的睡眠, 时间由 nextPollTimeoutMillis 决定
            nativePollOnce(ptr, nextPollTimeoutMillis);
            // 2. 获取队列中下一条要执行的消息, 返回给 Looper
            synchronized (this) {
                final long now = SystemClock.uptimeMillis(); // 获取当前时刻
                Message prevMsg = null;
                Message msg = mMessages;
                // 2.1 若有同步屏障消息, 找寻第一个异步的消息执行
                // msg.target 为 null, 则说明队首是同步屏障消息
                if (msg != null && msg.target == null) {
                    // 找寻第一个异步消息执行
                    do {
                        prevMsg = msg;
                        msg = msg.next;
                    } while (msg != null && !msg.isAsynchronous());
                }
                // 2.2 直接找寻队首消息执行
                if (msg != null) {
                    // 2.2.1 若当前的时刻 早于 消息的执行时刻
                    if (now < msg.when) {
                        // 给 nextPollTimeoutMillis 赋值, 表示当前线程, 可睡眠的时间
                        nextPollTimeoutMillis = (int) Math.min(msg.when - now, Integer.MAX_VALUE);
                    } else {
                        // 2.2.2 若当前的时刻 不早于 消息的执行时刻, 则将该消息从队列中移除, 返回给 Looper 执行
                        mBlocked = false;// 更改标记位置
                        if (prevMsg != null) {
                            prevMsg.next = msg.next;
                        } else {
                            mMessages = msg.next;
                        }
                        msg.next = null;
                        // 将队首消息返回出去
                        return msg;
                    }
                } else {
                    // 2.3 若未找到可执行消息, 则给 nextPollTimeoutMillis 置为 -1
                    // 表示可以无限睡眠, 直至消息队列中有同步消息可读
                    nextPollTimeoutMillis = -1;
                }
                
                .......// 4. 收集空闲处理者
            }

            ......// 4.1 处理空闲事件
            
            // 走到这里, 说明所有的空闲事件都已经处理好了
            // 将需要处理的空闲事件,置为 0 
            pendingIdleHandlerCount = 0;

            // 5. 因为处理空闲事件是耗时操作, 期间可能有新的 Message 入队列, 因此将可睡眠时长置为 0, 表示需要再次检查
            nextPollTimeoutMillis = 0;
        }
    }
    
    // native 方法
    private native void nativePollOnce(long ptr, int timeoutMillis); /*non-static for callbacks*/
 
}
```
可以看到 MessageQueue.next 内部维护了一个死循环, 用于获取下一条 msg, 这个 for 循环做了如下的操作
- **调用 nativePollOnce 根据 nextPollTimeoutMillis 时长, 执行当前线程的睡眠操作**
- **睡眠结束后, 取队列中可执行的 Msg 返回给 Looper 分发**
  - 若有同步屏障消息, 找寻第一个异步的消息执行
  - 若未找到, 则找寻队首消息直接执行
    - 若当前的时刻 < 第一个同步的消息的执行时刻
      - 则更新 nextPollTimeoutMillis, 下次进行 for 循环时, 会进行睡眠操作
    - 若当前的时刻 >= 第一个同步的消息的执行时刻
      - 则将第一个同步消息的返回给 Looper 执行
  - 若未找到可执行消息
    - 将 nextPollTimeoutMillis 置为 -1, 下次 for 循环时可以无限睡眠, 直至消息队列中有新的消息为止
- **未找到可执行 Message, 进行空闲消息的收集和处理**: 到后面分析

上面的流程非常清晰, 这里我们主要看看 **nativePollOnce Native 休眠机制** 和 **空闲消息处理** 的流程

#### 1. nativePollOnce 休眠
```
// frameworks/base/core/jni/android_os_MessageQueue.cpp
static void android_os_MessageQueue_nativePollOnce(JNIEnv* env, jobject obj,
        jlong ptr, jint timeoutMillis) {
    NativeMessageQueue* nativeMessageQueue = reinterpret_cast<NativeMessageQueue*>(ptr);
    // 调用了 NativeMessageQueue 的 pollOnce
    nativeMessageQueue->pollOnce(env, obj, timeoutMillis);
}

void NativeMessageQueue::pollOnce(JNIEnv* env, jobject pollObj, int timeoutMillis) {
    ......
    // 1. 调用了 Looper 的 pollOne
    mLooper->pollOnce(timeoutMillis);
    ......
}

// system/core/libutils/Looper.cpp
int Looper::pollOnce(int timeoutMillis, int* outFd, int* outEvents, void** outData) {
    int result = 0;
    for (;;) {
        ......
        // result 不为 0 说明读到了消息
        if (result != 0) {
            ......
            return result;
        }
        // 2. 若未读到消息, 则调用 pollInner
        result = pollInner(timeoutMillis);
    }
}


int Looper::pollInner(int timeoutMillis) {
    ......
    int result = POLL_WAKE;
    ......
    struct epoll_event eventItems[EPOLL_MAX_EVENTS];
    // 3. 调用 epoll_wait 阻塞来监听 mEpollFd 中的 IO 事件, 超时机制由 timeoutMillis 决定
    int eventCount = epoll_wait(mEpollFd, eventItems, EPOLL_MAX_EVENTS, timeoutMillis);
    // 4. 走到这里, 说明睡眠结束了
    for (int i = 0; i < eventCount; i++) {
        int fd = eventItems[i].data.fd;              // 获取文件描述
        uint32_t epollEvents = eventItems[i].events;
        // 5. 若存在唤醒事件的文件描述, 则执行唤醒操作
        if (fd == mWakeEventFd) {
            if (epollEvents & EPOLLIN) {
                awoken();// 唤醒
            } else {
                ......
            }
        } else {
           ......
        }
    }
    ......
    return result;
}
```
好的可以看到 JNI 方法 nativePollOnce, 其内部流程如下
- 将 Poll 操作转发给了 Looper.pollOne
- 若未读到消息, 则调用 Looper.pollInner
- 调用 epoll_wait 阻塞监听 mEpollFd 中的 IO 事件
  - 若无事件, **则睡眠在 mWakeEventFd 文件读操作上**, 时长由 timeoutMillis 决定
- 睡眠结束后调用 awoken 唤醒

好的, 至此线程是睡眠的机制也明了了, 下面看看空闲消息处理机制

#### 2. 空闲处理者执行
通过上面的分析我们知道, 当我们的队列中没有消息, 在进行下一次 for 循环进行睡眠之前会执行空闲消息, 这里我们就详细剖析一下
```
public final class MessageQueue {

    // 空闲消息集合
    private final ArrayList<IdleHandler> mIdleHandlers = new ArrayList<IdleHandler>();
    // 空闲消息处理者的数组
    private IdleHandler[] mPendingIdleHandlers;
    
    Message next() {
        ...... 
        for (;;) {
            ......
            synchronized (this) {
                // 省略获取 msg 的代码
                ......
                // 1. 获取空闲消息
                // 1.1 从空闲消息集合 mIdleHandlers 中获取 空闲处理者 数量
                if (pendingIdleHandlerCount < 0
                        && (mMessages == null || now < mMessages.when)) {
                    pendingIdleHandlerCount = mIdleHandlers.size();
                }
                // 1.2 若无空闲处理者, 则进行下一次 for 循环
                if (pendingIdleHandlerCount <= 0) {
                    mBlocked = true;
                    continue;
                }
                ......
                // 1.3 将空闲消息处理者集合转为数组
                mPendingIdleHandlers = mIdleHandlers.toArray(mPendingIdleHandlers);
            }

            // 2. 处理空闲消息
            for (int i = 0; i < pendingIdleHandlerCount; i++) {
                // 获取第 i 给位置的空闲处理者
                final IdleHandler idler = mPendingIdleHandlers[i];
                mPendingIdleHandlers[i] = null; // 置空
                boolean keep = false;        
                try {
                    // 2.1 处理空闲消息
                    keep = idler.queueIdle(); 
                } catch (Throwable t) {
                    ......
                }
                if (!keep) {
                    synchronized (this) {
                        // 2.2 走到这里表示它是一次性的处理者, 从 mIdleHandlers 移除
                        mIdleHandlers.remove(idler);
                    }
                }
            }
            ......
        }
    }
    
}
```
好的, 可以看到 MessageQueue.next 在获取不到 msg 时, 会进行一些空闲消息的处理
- 收集空闲消息
  - 从空闲消息集合 mIdleHandlers 中获取 空闲处理者 数量
  - 若无空闲处理者, 则进行下一次 for 循环
  - 若存在空闲处理者, 则空闲消息处理者集合转为数组 mPendingIdleHandlers 
- 处理空闲消息
  - 调用 IdleHandler.queueIdle 处理空闲消息
    - 返回 true, 下次再 MessageQueue.next 获取不到 msg 的空闲时间会继续处理 
    - 返回 false 表示它是一次性的处理者, 从 mIdleHandlers 移除

空闲处理者的添加方式如下
```
public final class MessageQueue {
    
    public static interface IdleHandler {
        /**
         * 处理空闲消息
         */
        boolean queueIdle();
    }
    
    // 空闲消息集合
    private final ArrayList<IdleHandler> mIdleHandlers = new ArrayList<IdleHandler>();
    
    public void addIdleHandler(@NonNull IdleHandler handler) {
        synchronized (this) {
            mIdleHandlers.add(handler);
        }
    }
    
}
```
通过上述代码可以得到以下的信息
- 空闲处理者使用 IdleHandler 接口描述
- 空闲处理者通过 MessageQueue.addIdleHandler() 添加
- 空闲处理者使用 MessageQueue.mIdleHandlers 维护

好的, 到这里我们的消息获取的流程就完成了, 下面看看 Looper 获取到 Message 之后是如何执行的

### 二) 消息的处理
当 MessageQueue.next 返回一个 Message 时, Looper 中的 loop 方法便会处理消息的执行, 先回顾一下代码
```
public final class Looper {
    
    public static void loop() {
        final Looper me = myLooper();
        ......
        final MessageQueue queue = me.mQueue;
        for (;;) {
            Message msg = queue.next(); // might block
            if (msg == null) {
                // 若取到的消息为 null, 这个 Looper 就结束了
                return;
            }
            ......
            try {
                // 处理消息
                msg.target.dispatchMessage(msg);
            } finally {
                ......
            }
            ......
        }
    }
    
}
```
好的, 可以看到当 MessageQueue.next 返回一个 Message 时, 便会调用 msg.target.dispatchMessage(msg) 去处理这个 msg
- msg.target 为一个 Handler 对象, 在上面的分析中可知, 在通过 Handler 发送消息的 enqueueMessage 方法中, 会将 msg.target 设置为当前的 Handler
- 可以看到, 这个 msg 正是由将它投递到消息队列的 Handler 处理了, 它们是一一对应的

接下来我们看看 Handler 处理消息的流程
```
 public class Handler {
     
    public void dispatchMessage(Message msg) {
        // 1. 若 msg 对象中存在 callback, 则调用 handleCallback
        if (msg.callback != null) {
            handleCallback(msg);
        } else {
            // 2. 若当前 Handler 设值了 Callback, 进入这个分支
            if (mCallback != null) {
                // 2.1 若这个 Callback 的 handleMessage 返回 true, 则不会将消息继续向下分发
                if (mCallback.handleMessage(msg)) {
                    return;
                }
            }
            // 3. 若消息没有被 mCallback 拦截, 则会调用 handleMessage 进行最后的处理
            handleMessage(msg);
        }
    } 
    
    /**
     * 方式一: 优先级最高
     */
    private static void handleCallback(Message message) {
        message.callback.run();
    }
    
    public interface Callback {
        /**
         * 方式二: 优先级次高
         * @return True if no further handling is desired
         */
        public boolean handleMessage(Message msg);
    }
     
    public Handler(Callback callback, boolean async) {
        ......
        // Callback 由构造函数传入
        mCallback = callback;
        ......
    } 
    
    /**
     * 方式三: 这个处理消息的方式, 由子类重写, 优先级最低
     */
    public void handleMessage(Message msg) {
    
    }
    
 }
```
可以看到 Handler 的 dispatchMessage 处理消息主要有三种方式
- 若是 Message 对象内部设置了 callback, 则**调用 handleCallback 方法直接处理, 不会再往下分发**
- 若 Handler 设置 Callback, 则会**调用 Callback.handleMessage 方法**
  - Callback.handleMessage **返回 false, 则会将消息处理继续分发给 Handler.handleMessage**

### 三) 回顾
Looper.loop 主要是通过死循环, 不断地处理消息队列中的数据, 流程如下
- **通过 MessageQueue.next() 获取下一条要处理的 Msg**
  - **调用 nativePollOnce 根据 nextPollTimeoutMillis 时长, 执行当前线程的睡眠操作**
    - 调用 epoll_wait 来监听 mEpollFd 中的 IO 事件, 若无事件, **则睡眠在 mWakeEventFd 文件读操作上**, 时长由 timeoutMillis 决定
    - 睡眠结束后调用 awoken 唤醒
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
      - 从空闲消息集合 mIdleHandlers 中获取 空闲处理者 数量
      - 若无空闲处理者, 则进行下一次 for 循环
      - 若存在空闲处理者, 则空闲消息处理者集合转为数组 mPendingIdleHandlers 
    - 处理空闲消息
      - 调用 IdleHandler.queueIdle 处理空闲消息
      - 返回 true, 下次再 MessageQueue.next 获取不到 msg 的空闲时间会继续处理 
      - 返回 false 表示它是一次性的处理者, 从 mIdleHandlers 移除
- **通过 msg.target.dispatchMessage(msg); 分发处理消息**
  - 若是 Message 对象内部设置了 callback, 则**调用 handleCallback 方法直接处理, 不会再往下分发**
  - 若 Handler 设置 Callback, 则会**调用 Callback.handleMessage 方法**
    - Callback.handleMessage **返回 false, 则会将消息处理继续分发给 Handler.handleMessage** 

## 三. 消息的发送
Handler 主要有三种类型的消息
- 同步消息
- Android 4.1 配合 Choreographer 引入
  - 同步屏障: 从同步屏障起的时刻跳过同步消息, 优先执行异步消息
  - 异步消息: 用于和同步区分, 并非多线程异步执行的意思

### 一) 消息的发送
```
public class Handler {
    
    ......
    public final boolean sendMessage(Message msg) {
        // 回调了发送延时消息的方法
        return sendMessageDelayed(msg, 0);
    }
    
    public final boolean sendMessageDelayed(Message msg, long delayMillis) {
        if (delayMillis < 0) {
            delayMillis = 0;
        }
        // 回调了发送指定时刻消息的方法
        return sendMessageAtTime(msg, /*指定时刻 = 当前时刻 + 延时时间*/SystemClock.uptimeMillis() + delayMillis);
    }
    
    public boolean sendMessageAtTime(Message msg, long uptimeMillis) {
        MessageQueue queue = mQueue;
        ......
        // 回调了入消息队列的方法
        return enqueueMessage(queue, msg, uptimeMillis);
    }
    
    private boolean enqueueMessage(MessageQueue queue, Message msg, long uptimeMillis) {
        // 1. 将这个消息的处理者, 设置为其自身
        msg.target = this;
        // 2. 若 mAsynchronous 为 true
        if (mAsynchronous) {
            // 在投递到消息队列之前, 将其设置为异步消息
            msg.setAsynchronous(true);
        }
        // 3. 调用 MessageQueue 的 enqueueMessage 执行入队列的操作
        return queue.enqueueMessage(msg, uptimeMillis);
    }
 }
```
可以看到发送消息的操作, 进过了一系列方法的调用, 会走到 sendMessageAtTime 中, 表示**发送指定时刻的消息**, 然后会调用 enqueueMessage 执行入消息队列操作
- **将这个消息的 target 赋值为自身, 表示这个消息到时候会被当前 Handler 对象处理**
- **mAsynchronous 在 Handler 构造时传入, 若为 true, 则说明通过这个 Handler 发送的消息为异步消息**
- **调用了 MessageQueue.enqueueMessage 将消息投递到消息队列中去**

接下来看看 MessageQueue.enqueueMessage 做了哪些操作
```
public final class MessageQueue {
    
    Message mMessages;
    
    boolean enqueueMessage(Message msg, long when) {
        ......
        synchronized (this) {
            ......
            msg.when = when;           // 将消息要执行的时刻写入 msg 成员变量
            Message p = mMessages;     // 获取当前队首的消息
            boolean needWake;          // 描述是否要唤醒该 MessageQueue 所绑定的线程
            // 1. 队列中无消息 | 新消息要立即执行 | 新消息要执行的时间比队首的早
            if (p == null || when == 0 || when < p.when) {
                // 1.1 将该消息放置到队首
                msg.next = p;
                mMessages = msg;
                // 1.2 若该 MessageQueue 之前为阻塞态, 则说明需要唤醒
                needWake = mBlocked;  
            } else {
                // 2. 新消息执行的时间比队首的晚
                // 2.1 判断是否需要唤醒, 队首是同步屏障, 并且该消息异步的, 则有机会唤醒队列
                needWake = mBlocked && p.target == null && msg.isAsynchronous();
                // 2.2 将该消息插入到消息队列合适的位置
                Message prev;
                for (;;) {
                    prev = p;
                    p = p.next;
                    if (p == null || when < p.when) {
                        break;
                    }
                    // 若该消息并非第一个异步消息, 则不允许唤醒
                    if (needWake && p.isAsynchronous()) {
                        needWake = false;
                    }
                }
                msg.next = p;
                prev.next = msg;
            }
            // 3. 处理该消息队列绑定线程的唤醒操作
            if (needWake) {
                nativeWake(mPtr);
            }
        }
        return true;
    }
    
    private native static void nativeWake(long ptr);

}
```
可以看到 MessageQueue 在执行消息入队列时, 做了如下操作
- **队列中无消息 | 新消息要立即执行 | 新消息要执行的时间比队首的早** 时
  - 将该消息放置到队首
  - 若该 MessageQueue 之前为阻塞态, 则说明需要唤醒
- 新消息执行的时间比队首的晚
  - 判断是否需要唤醒队列
    - 队列前有一个同步屏障, 并且该消息异步的, 则有机会唤醒队列
  - 将该消息插入到消息队列合适的位置
    - 若该消息并非第一个异步消息, 则不允许唤醒
- 通过 **needWake** 处理该消息队列绑定线程的唤醒操作

这里我们可以看到, 新消息入队列只有两种情况才会将阻塞的队列唤醒
- 它插入到了队首, 需要立即执行
- **队首存在同步屏障并且它是第一个异步的消息**

#### nativeWake 唤醒
```
// frameworks/base/core/jni/android_os_MessageQueue.cpp
static void android_os_MessageQueue_nativeWake(JNIEnv* env, jclass clazz, jlong ptr) {
    NativeMessageQueue* nativeMessageQueue = reinterpret_cast<NativeMessageQueue*>(ptr);
    nativeMessageQueue->wake();
}

void NativeMessageQueue::wake() {
    mLooper->wake();
}

// system/core/libutils/Looper.cpp
void Looper::wake() {
    // 向 Looper 绑定的线程 mWakeEventFd 管道中写入一个新的数据
    uint64_t inc = 1;
    ssize_t nWrite = TEMP_FAILURE_RETRY(write(mWakeEventFd, &inc, sizeof(uint64_t)));
    ....
}
```
可以看到, nativeWake 是通过向 Looper 绑定的线程的 mWakeEventFd 写入一个数据的来唤醒目标线程
- 通过上一篇的分析可知, 此时 Looper::pollInner 会睡眠在 mWakeEventFd 的读操作上

### 二) 同步屏障的发送
我们知道, 使用 Handler 发送消息时, 会按照时间的顺序添加到 MessageQueue 中, 这就说明了我们的消息一定会排在队列后面的执行, **但我们的消息又非常的重要, 想让他尽快的执行**, 这个时候就可以采用同步屏障的方式

它能够优先执行的原因, 我们在 MessageQueue.next 中已经分析过了, 下面我们看看它的发送和移除

#### 1. 添加同步屏障
```
public final class MessageQueue {
    
    
    public int postSyncBarrier() {
        return postSyncBarrier(SystemClock.uptimeMillis());
    }

    private int postSyncBarrier(long when) {
        synchronized (this) {
            // 1. 为这个同步屏障的生成一个 token
            final int token = mNextBarrierToken++;
            // 2. 创建一个 Message
            final Message msg = Message.obtain();
            msg.markInUse();
            msg.when = when;
            // 2.1 将 token 保存到 arg1 中
            msg.arg1 = token;
            
            // 3. 找寻插入点
            Message prev = null;
            Message p = mMessages;
            if (when != 0) {
                while (p != null && p.when <= when) {
                    prev = p;
                    p = p.next;
                }
            }
            // 4. 链入队列
            if (prev != null) { 
                msg.next = p;
                prev.next = msg;
            } else {
                msg.next = p;
                mMessages = msg;
            }
            // 5. 返回 token
            return token;
        }
    }
    
}
```
好的, 可以看到这个添加同步屏障的过程非常简单, 可以说与将一个正常的消息添加到队列中做法一致, **那它是如何起到同步消息的屏障的作用的呢?**

关键点在于这个 msg.target 为 null, 也就是说他没有绑定处理它的 Handler, 这就能够实现同步屏障的效果吗? 我们去  MessageQueue.next 中验证我们的想法

#### 2. 移除同步屏障
```
public final class MessageQueue {
    
    public void removeSyncBarrier(int token) {
        synchronized (this) {
            Message prev = null;
            Message p = mMessages;
            // 1. 遍历消息队列, 找寻同步屏障及其前驱结点的 msg
            while (p != null && (p.target != null || p.arg1 != token)) {
                prev = p;
                p = p.next;
            }
           
            // 2.1 若存在前驱, 则说明这个同步屏障暂未被执行
            if (prev != null) {
                prev.next = p.next;
                needWake = false;// 不需要唤醒
            } else {
                // 2.2 若同步屏障生效了, 则根据其后继判断是否需要唤醒
                mMessages = p.next;
                needWake = mMessages == null || mMessages.target != null;
            }
            // 3. 回收消息
            p.recycleUnchecked();
            // 4. 判读是否需要唤醒在 Message.next 上的睡眠
            if (needWake && !mQuitting) {
                nativeWake(mPtr);
            }
        }
    }
    
}
```
好的, 可以看到移除同步屏障的过程也非常的简单, 即通过 token 移除这个 msg, 注释的也非常详细

### 三) 回顾
Android 的消息机制, 主要有三种类型的消息
- 同步消息
  - 正常的消息, 我们一般都使用这个
- Android 4.1 配合 Choreographer 引入
  - 同步屏障: 从同步屏障起的时刻跳过同步消息, 优先执行异步消息
  - 异步消息: 用于和同步区分, 并非多线程异步执行的意思

## 总结
### Looper 的创建
Looper 的 prepare 操作主要是创建当前线程单例的 MessageQueue 对象, 保存在 mQueue 中
- *创建线程单例的 NativeMessageQueue 对象*
  - 因为 Java 的 MessageQueue 是线程单例的, 因此 NativeMessageQueue 对象也是线程单例的 
- **创建线程单例的 Looper(Native)**
  - 创建了一个名为 mWakeEventFd 的 eventfd 
    - Android 6.0 之前使用的 pipe
    - Android 6.0 之后使用的 eventfd
      - 比起管道, 它只有一个文件描述符, 更加轻量级
  - 创建了一个 IO 多路复用的 epoll 对象 mEpollFd
  - 让 epoll 对其进行监听 mWakeEventFd

### Looper 循环的启动
Looper.loop 主要是通过死循环, 不断地处理消息队列中的数据, 流程如下
- **通过 MessageQueue.next() 获取下一条要处理的 Msg**
  - **调用 nativePollOnce 根据 nextPollTimeoutMillis 时长, 执行当前线程的睡眠操作**
    - 调用 epoll_wait 来监听 mEpollFd 中的 IO 事件, 若无事件, **则睡眠在 mWakeEventFd 文件读操作上**, 时长由 timeoutMillis 决定
    - 睡眠结束后调用 awoken 唤醒
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
      - 从空闲消息集合 mIdleHandlers 中获取 空闲处理者 数量
      - 若无空闲处理者, 则进行下一次 for 循环
      - 若存在空闲处理者, 则空闲消息处理者集合转为数组 mPendingIdleHandlers 
    - 处理空闲消息
      - 调用 IdleHandler.queueIdle 处理空闲消息
      - 返回 true, 下次再 MessageQueue.next 获取不到 msg 的空闲时间会继续处理 
      - 返回 false 表示它是一次性的处理者, 从 mIdleHandlers 移除
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