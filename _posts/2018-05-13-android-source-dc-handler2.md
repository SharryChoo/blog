---
layout: article
title: Android 系统架构 —— 线程消息的发送与处理
permalink: android-source/dc-handler2
key: android-source-dc-handler2
tags: AndroidFramework
---

<!--more-->

## 前言
我们都知道, Android 系统提供了 Handler 类, 描述一个消息的处理者, 它**负责消息的发送与处理**
 
 ```
 public class Handler {
    
    final Looper mLooper;
    final MessageQueue mQueue;
    final Callback mCallback;
    final boolean mAsynchronous;

    public Handler() {
        this(null, false);
    }
    
    public Handler(Callback callback, boolean async) {
        // 调用该方法的线程的 Looper
        mLooper = Looper.myLooper();
        // 这里扔出了 Runtime 异常, 因此 Handler 是无法在没有 Looper 的线程中执行的
        if (mLooper == null) {
            throw new RuntimeException(
                "Can't create handler inside thread " + Thread.currentThread()
                        + " that has not called Looper.prepare()");
        }
        // 获取消息队列
        mQueue = mLooper.mQueue;
        // 给 Callback 赋值
        mCallback = callback;
        // 设置该 Handler 是否发送异步消息
        mAsynchronous = async;
    }
    
    public final boolean sendMessage(Message msg){
        ......
    }
    
    public void handleMessage(Message msg) {
    
    }
    
 }
 ```
 好的, 可以看到 Handler 的结构如上述代码所示, 本次只 focus 以下三个方法
 - 构造方法
   - 获取当前线程的 Looper
   - 获取当前线程的 MessageQueue
   - 给 mCallback 赋值(这个 Callback 的作用, 在后面描述)
   - 给 mAsynchronous 赋值, 表示通过 Handler 发送消息时, 会将消息置为异步消息
 - 发送消息的方法
   - sendMessage
 - 处理消息的方法
   - handleMessage

这里的异步与同步并非线程的同步与异步, **不论是同步消息还是异步消息它们都顺序的组织在 MessageQueue 中, 只不过异步的消息不会受到同步屏障的阻碍**, 其具体的实现会在下一篇文章中分析

好的, 接下来我们先看看如何发送消息的
 
 ## 一. 消息的发送
 我们先看看, Android 中的消息是如何发送的
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
                // 2.1 判断是否需要唤醒, 队列前有一个同步屏障, 并且该消息异步的, 则有机会唤醒队列
                needWake = mBlocked && p.target == null && msg.isAsynchronous();
                // 2.2 将该消息插入到消息队列合适的位置
                Message prev;
                for (;;) {
                    prev = p;
                    p = p.next;
                    if (p == null || when < p.when) {
                        break;
                    }
                    // 若该消息非最早的异步消息, 则不允许唤醒
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
    - 若该消息非最早的异步消息, 则不允许唤醒
- 根据 needWake 处理该消息队列绑定线程的唤醒操作

好的, 这里我们可以看到, **新消息入队列只有两种情况才会将阻塞的队列唤醒**
- **它插入到了队首**
- **队首存在同步屏障并且它是第一个异步的消息**

消息入队列的过程还是很清晰明了的, 从上一篇文章的分析中我们知道, 若 MessageQueue 在当前时刻没有要执行的消息时, 会睡眠在 MessageQueue.next() 方法上, 接下来看看它是如何通过 nativeWake 唤醒的 
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

## 二. 消息的处理
通过上一篇的分析可知, 当 MessageQueue.next 返回一个 Message 时, Looper 中的 loop 方法便会处理消息的执行, 先回顾一下代码
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

好的, 消息的发送和处理到这里就分析结束了, 最后再了解一下 MessageQueue 中空闲处理者的相关知识

## 三. 空闲处理者的添加与处理
### 一) 什么是空闲处理者
通过上一篇文章的分析可知 MessageQueue 通过 next 方法通过死循环获取下一个要处理的 Message, 若当前时刻不存在要处理的消息, 下次循环会进行睡眠操作
- **在没有取到可执行消息 ---> 下次 for 循环进行睡眠** 之间的时间间隔, 称之为**空闲时间**
- **在空闲时间处理事务的对象, 称之为空闲处理者**
```
public final class MessageQueue {

   Message next() {
        for (;;) {
            // 睡眠操作
            nativePollOnce(ptr, nextPollTimeoutMillis);
            synchronized (this) {
                Message prevMsg = null;
                Message msg = mMessages;
                ......
                if (msg != null) {
                    // ...... 
                    return msg;
                }
                
                // 空闲时间
                ....... 
            }

            // 空闲时间
            ......
            
        }
    }
    
}
```

### 二) 空闲处理者的添加
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

好的, 结下来看看处理细节

### 三) 空闲消息的处理
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
                // 1. 从空闲消息集合 mIdleHandlers 中获取 空闲处理者 数量
                if (pendingIdleHandlerCount < 0
                        && (mMessages == null || now < mMessages.when)) {
                    pendingIdleHandlerCount = mIdleHandlers.size();
                }
                // 2 若无空闲处理者, 则进行下一次 for 循环
                if (pendingIdleHandlerCount <= 0) {
                    mBlocked = true;
                    continue;
                }
                ......
                // 3. 将空闲消息处理者集合转为数组
                mPendingIdleHandlers = mIdleHandlers.toArray(mPendingIdleHandlers);
            }

            // 4. 处理空闲消息
            for (int i = 0; i < pendingIdleHandlerCount; i++) {
                final IdleHandler idler = mPendingIdleHandlers[i];// 获取第 i 给位置的空闲处理者
                mPendingIdleHandlers[i] = null; // 置空
                boolean keep = false;        
                try {
                    // 4.1 处理空闲消息
                    keep = idler.queueIdle(); 
                } catch (Throwable t) {
                    ......
                }
                if (!keep) {
                    synchronized (this) {
                        // 4.2 走到这里表示它是一次性的处理者, 从 mIdleHandlers 移除
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
- 从空闲消息集合 mIdleHandlers 中获取 空闲处理者 数量
- 若无空闲处理者, 则进行下一次 for 循环
- 若存在空闲处理者, 则空闲消息处理者集合转为数组 mPendingIdleHandlers 
- for 循环处理空闲消息
  - 调用 IdleHandler.queueIdle 处理空闲消息
    - 返回 true, 下次再 MessageQueue.next 获取不到 msg 的空闲时间会继续处理 
    - 返回 false 表示它是一次性的处理者, 从 mIdleHandlers 移除


## 总结
好的, 本次的分析主要有两个重点
- 一个消息是通过 Handler 投递到其绑定的 MessageQueue 中的
- 当 Looper.loop 中通过 MessageQueue.next 获取到一个消息时, 便会使用发送该消息的 Handler.dispatchMessage 进行分发执行

除此之外还介绍了空闲消息处理者, 当 MessageQueue.next() 中无可用消息时不会立即进入睡眠, 而是会利用这段时间处理添加的空闲消息
