---
layout: article
title: "Android 系统架构 —— 数据通信篇 概览"
key: "Android 系统架构 —— 数据通信篇 概览" 
tags: AndroidFramework
aside:
  toc: true
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

<!--more-->

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
    - 创建了 MessageQueue 这个消息队列, 保存到成员变量 mQueue 中
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
好的, 可以看到 MessageQueue 对象在创建的过程中, 会调用 nativeInit 来获取一个 native 的句柄, 我们看看这个 nativeInit 做了哪些操作

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
- 创建了一个 C++ 的 NativeMessageQueue 对象
  - 尝试通过 Looper.getForThread 获取一个 C++ 的 Looper 对象 
  - 若当前线程没有创建过 Looper
    - new 一个 C++ 的 Looper 对象
    - 给当前线程绑定 Looper 对象

可以看到这里的流程与 Java 的相反
- Java 是先创建 Looper, 然后在 Looper 内部创建 MessageQueue
- Native 是先创建 NativeMessageQueue, 在其创建的过程中创建 Looper 

接下来看看这个 C++ 的 Looper 对象在实例化的过程中做了哪些事情

### 二. Looper(Native) 的实例化
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
- 调用 eventfd 创建了一个事件描述符
  - eventfd 是 Linux 2.6.22 版本新特性
  - Android 6.0 之前使用的 pipe, 6.0 之后使用的 eventfd
- 创建了一个 epoll 对象
- 将事件描述符 mWakeEventFd 添加到 epoll 中, 让 epoll 对其进行监听
  - 有其他线程写入数据时, 唤醒睡眠在 mWakeEventFd 上的读线程

### 相互依赖的结构图
**到这里消息循环的准备工作就已经完成了, 这里分析一下它们的结构图**

![消息循环相互依赖关系图](https://i.loli.net/2019/10/19/pF21xQiYhkE85As.png)

## 三. 消息循环的启动
```
public final class Looper {
    
    public static void loop() {
        // 1. 获取调用线程的 Looper 对象
        final Looper me = myLooper();
        ......
        // 2. 获取 Looper 对应的消息队列
        final MessageQueue queue = me.mQueue;
        // 3. 死循环, 不断的处理消息队列中的消息
        for (;;) {
            // 3.1 获取消息队列中的消息, 取不到消息会阻塞
            Message msg = queue.next(); // might block
            if (msg == null) {
                // 若取到的消息为 null, 这个 Looper 就结束了
                return;
            }
            ......
            try {
                // 3.2 处理消息
                msg.target.dispatchMessage(msg);
            } finally {
                ......
            }
            ......
        }
    }
    
}
```
好的, 可以看到 Looper 的 loop 方法主要做了如下几件事情
- 从方法调用的线程中取 Looper 对象
- 获取这个 Looper 对应的消息队列
- 通过死循环, 不断地处理消息队列中的数据
  - 通过 MessageQueue.next() 获取下一条要处理的 Msg
  - 通过 msg.target.dispatchMessage(msg); 分发处理消息(本次不细究)
 
好的, 可以看到获取消息的方式是通过 MessageQueue.next 拿到的, 我们接下来看看它是如何获取的

### 一) 从队列中取消息
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
            // 1. 若 nextPollTimeoutMillis 不为 0, 则说明距离下一个 Message 执行, 有一定的时间间隔
            if (nextPollTimeoutMillis != 0) {
                // 在空闲期间, 执行 Binder 通信相关的指令
                Binder.flushPendingCommands();
            }
            // 2. 这里调用了 nativePollOnce, 在 native 层检查消息队列中是否有 msg 可读, 若无新消息的到来, 则执行线程的睡眠, 时间由 nextPollTimeoutMillis 决定
            nativePollOnce(ptr, nextPollTimeoutMillis);
            // 3. 获取队列中下一条要执行的消息, 返回给 Looper
            synchronized (this) {
                final long now = SystemClock.uptimeMillis(); // 获取当前时刻
                Message prevMsg = null;
                Message msg = mMessages;
                // 3.1 msg.target 为 null, 则说明队首是同步屏障消息
                if (msg != null && msg.target == null) {
                    // 找寻第一个异步消息执行
                    do {
                        prevMsg = msg;
                        msg = msg.next;
                    } while (msg != null && !msg.isAsynchronous());
                }
                // 3.2 若找到可执行消息
                if (msg != null) {
                    // 3.2.1 若当前的时刻 早于 消息的执行时刻
                    if (now < msg.when) {
                        // 给 nextPollTimeoutMillis 赋值, 表示当前线程, 可睡眠的时间
                        nextPollTimeoutMillis = (int) Math.min(msg.when - now, Integer.MAX_VALUE);
                    } else {
                        // 3.2.2 若当前的时刻 不早于 消息的执行时刻, 则将该消息从队列中移除, 返回给 Looper 执行
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
                    // 3.3 若未找到可执行消息, 则给 nextPollTimeoutMillis 置为 -1
                    // 表示可以无限睡眠, 直至消息队列中有同步消息可读
                    nextPollTimeoutMillis = -1;
                }
                // 4. 获取一些空闲事件的处理者
                if (pendingIdleHandlerCount < 0
                        && (mMessages == null || now < mMessages.when)) {
                    pendingIdleHandlerCount = mIdleHandlers.size();
                }
                // 若无空闲事件, 则进行下一次 for 循环
                if (pendingIdleHandlerCount <= 0) {
                    mBlocked = true;
                    continue;
                }
                .......
            }

            // 4.1 处理空闲事件
            ......
            
            // 4.2 走到这里, 说明所有的空闲事件都已经处理好了
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
- 若 nextPollTimeoutMillis 不为 0, 则说明距离下一个 Message 执行, 有一定的时间间隔
  - **在空闲期间, 执行 Binder 通信相关的指令**
- **调用 nativePollOnce 根据 nextPollTimeoutMillis 时长, 执行当前线程的睡眠操作**
- **睡眠结束后, 取队列中可执行的 Msg 返回给 Looper 分发**
  - **若存在同步屏障消息**(同步屏障的知识在后面进行解析)
    - 找寻队列中第一个异步的消息执行
  - **若找到可执行消息**
    -  若当前的时刻 < 第一个同步的消息的执行时刻
       - 则更新 nextPollTimeoutMillis, 下次进行 for 循环时, 会进行睡眠操作
    -  若当前的时刻 >= 第一个同步的消息的执行时刻
       - 则将第一个同步消息的返回给 Looper 执行
  - **若未找到可执行消息**
    - 将 nextPollTimeoutMillis 置为 -1, 下次 for 循环时可以无限睡眠, 直至消息队列中有新的消息为止
- **空闲消息的获取和处理**(本次不细究)
   - 获取空闲事件的处理者
  - 若无空闲事件, 则进行下一次 for 循环 
  - 若存在空闲事件, 则处理空闲事件
    - 将 pendingIdleHandlerCount 置为 0, 表示空闲事件都已经处理完成了
    - 将 nextPollTimeoutMillis 置为 0, 因为处理空闲事件是耗时操作, 期间可能有新的 Message 入队列, 因此将可睡眠时长置为 0, 表示需要再次检查

至此一次 for 循环就结束了, 可以看到 Message.next() 中其他的逻辑都非常的清晰, 但其睡眠是一个 native 方法, 我们继续看看它的内部实现

### 二) 睡眠操作
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
    // 3. 调用 epoll_wait 来监听 mEpollFd 中的 IO 事件, 若无事件, 则睡眠在文件读操作上, 时长由 timeoutMillis 决定
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
- 调用 epoll_wait 来监听 mEpollFd 中的 IO 事件, 若无事件, **则睡眠在 mWakeEventFd 文件读操作上**, 时长由 timeoutMillis 决定
- 睡眠结束后调用 awoken 唤醒

**好的, 至此线程是睡眠的机制也明了了**<br> 

**最后通过 UML 图总结一下, 线程消息队列的创建与循环**

![调用时序图](https://i.loli.net/2019/10/19/agmFRuUArN38yfs.png)

## 总结
线程消息队列的创建主要有两步
- 调用 Looper.prepare 为当前线程创建唯一的 Looper 对象, 保存在 ThreadLocalMap 中
  - **Looper 构建时, 会创建一个消息对象 MessageQueue**
- 调用 Looper.loop(), 它是一个死循环, 会调用 MessageQueue.next() 来获取一个可执行的消息
  - MessageQueue.next() 中的操作非常重要
    - 无消息时会进行睡眠操作, 让出 CPU 资源
    - 有消息时, 从队列中获取可执行的消息返回给 Looper.loop 方法执行

其睡眠操作是睡眠在 eventfd 的读操作中

### 思考
**为什么 Android 6.0 之后, 将 pipe 更新为 eventfd 呢?**
- pipe: 管道是 Linux 内核提供的最基础的 IPC 方式, 管道是由内核管理的一个缓冲区, 创建时会产生两个文件描述符, 一端用于读, 一端用于写
- eventfd: eventfd 是 Linux 内核在 2.6.22 版本新增的轻量级 IPC 方式, 它只会产生一个文件描述符

通过 pipe 和 eventfd 的比较, 我们知道 eventfd 比其 pipe 是要更加轻量级, 并且都是支持与 select, poll, epoll 配合 IO 多路使用, 实现 IO 监听的
