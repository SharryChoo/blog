---
layout: article
title: Android 系统架构 —— MessageQueue 的同步屏障
permalink: android-source/dc-handler3
key: android-source-dc-handler3
tags: AndroidFramework
---

<!--more-->
## 一. 什么是同步屏障
Handler 是与线程的 Looper 绑定的, 其发送和处理的消息都是针对其绑定 MessageQueue 的, 在同一个线程里肯定不可能异步执行(Java 没有协程), **因此它的同步屏障并非是指简单的线程同步, 而是指屏蔽同步类型的消息, 优先处理异步的消息**

## 二. 什么场景下会使用同步屏障
我们知道, 使用 Handler 发送消息时, 会按照时间的顺序添加到 MessageQueue 中, 这就说明了我们的消息一定会排在队列后面的执行, **但我们的消息又非常的重要, 想让他尽快的执行**, 这个时候就可以采用同步屏障的方式

我们可以给 MessageQueue 设置一个同步屏障, 让它的 MessageQueue.next 跳过屏障后所有的同步消息, 然后将我们的消息设置为异步消息, 如此一来就可以预约的提前执行了, **Android 为了保证 UI 渲染的速度, 因此这一点在 ViewRootImpl 的 requestLayout 中就着有很好的体现**

<!--more-->

## 三. 场景分析
```
public final class ViewRootImpl implements ViewParent,
        View.AttachInfo.Callbacks, ThreadedRenderer.DrawCallbacks {
    
    void scheduleTraversals() {
        if (!mTraversalScheduled) {
            mTraversalScheduled = true;
            // 1. 调用 MessageQueue.postSyncBarrier 方法, 为当前的 MessageQueue 设置一个同步屏障
            mTraversalBarrier = mHandler.getLooper().getQueue().postSyncBarrier();
            // 1.1 发送一个异步消息
            mChoreographer.postCallback(
                    Choreographer.CALLBACK_TRAVERSAL, mTraversalRunnable, null);
            ......
        }
    }

    void unscheduleTraversals() {
        if (mTraversalScheduled) {
            mTraversalScheduled = false;
            // 2. 移除同步屏障
            mHandler.getLooper().getQueue().removeSyncBarrier(mTraversalBarrier);
            // 2.1 移除异步消息
            mChoreographer.removeCallbacks(
                    Choreographer.CALLBACK_TRAVERSAL, mTraversalRunnable, null);
        }
    }
                
}
```
好的, 可以看到在 
- ViewRootImpl.scheduleTraversals 将 mTraversalRunnable 封装成 Msg 投递到 MessageQueue 之前, 它先调用了 MessageQueue.postSyncBarrier 添加了一个同步屏障
- ViewRootImpl.unscheduleTraversals 调用 MessageQueue.removeSyncBarrier 移除同步屏障

我们先看看它是如何实现的同步屏障的添加过程

### 一) 添加同步屏障
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

### 二) 同步屏障的实现
```
public final class MessageQueue {
    
    Message next() {
        ......
        for (;;) {
            ......
            synchronized (this) {
                final long now = SystemClock.uptimeMillis();
                Message prevMsg = null;
                Message msg = mMessages;
                // 可以看到, 当一个 msg 的 target 为 null 时, 会进入下面的分支
                if (msg != null && msg.target == null) {
                    // 这个 do...while 循环, 是找寻第一个非同步的消息, 即异步的消息
                    do {
                        prevMsg = msg;
                        msg = msg.next;
                    }
                    while (msg != null && !msg.isAsynchronous());
                }
                // 若进入了上面的分支, 此时这里的 msg 一定是异步的
                if (msg != null) {
                    ......// 返回找寻当前时间点可执行的 MSG
                } else {
                    // 进入无限睡眠
                    nextPollTimeoutMillis = -1;
                }
                ......
            }
        }
    }
    
}
```
关于 MessageQueue 的 next 方法在之前已经详细的分析过了, 因此这里就不赘述了, 我们主要看看代码中的 if 判断, **正是这个 target 为 null 的消息起到了同步屏障的作用, 它会找寻这个 msg 之后的第一个异步消息执行, 否则就进入等待, 直至同步屏障移除**

了解了同步屏障的实现原理, 接下来我们看看同步屏障的移除

### 三) 移除同步屏障
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

## 总结
至此 Android 的消息机制的同步屏障的实现就阐述完毕了, 这里总结一下, 若想使用同步屏障主要有以下三个步骤
- 添加同步屏障
- 发送异步消息
- 移除同步屏障

同步屏障的添加一移除一定要成对出现, 若不移除同步屏障, 会造成同步屏障之后 MessageQueue 的同步消息一直处于阻塞状态
