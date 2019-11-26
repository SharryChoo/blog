---
layout: article
title: "Android 系统架构 —— SurfaceFlinger 对 Vsync 的处理"
permalink: android-source/surfaceflinger-vsync
key: android-source-surfaceflinger-vsync
tags: AndroidFramework
---

## 前言
在前面的分析中, 我们了解了 SurfaceFlinger 的启动流程, 得知 SurfaceFlinger 对热插拔事件的处理
- onHotplugReceived
- onRefreshReceived
- onVsyncReceived

这里我们主要分析一下对 onVsyncReceived 函数对垂直同步信号 [Vsync](https://blog.csdn.net/zhaizu/article/details/51882768)(屏幕刷新一次之后便会产生一个 Vsync 信号) 的处理

<!--more-->

```
void SurfaceFlinger::onVsyncReceived(int32_t sequenceId,
        hwc2_display_t displayId, int64_t timestamp) {
    Mutex::Autolock lock(mStateLock);
    // Ignore any vsyncs from a previous hardware composer.
    if (sequenceId != getBE().mComposerSequenceId) {
        return;
    }
    // 1. 通知底层, 接收到了垂直同步信号
    int32_t type;
    if (!getBE().mHwc->onVsync(displayId, timestamp, &type)) {
        return;
    }
    
    bool needsHwVsync = false;

    {   // 将这个垂直同步的信号添加到 mPrimaryDispSync 中
        Mutex::Autolock _l(mHWVsyncLock);
        if (type == DisplayDevice::DISPLAY_PRIMARY && mPrimaryHWVsyncEnabled) {
            // 2. 添加垂直同步事件
            needsHwVsync = mPrimaryDispSync.addResyncSample(timestamp);
        }
    }

    .......
    
}
```
可以看到 SurfaceFlinger 收到垂直同步信号之后的操作如下
- 通知 HwComposer 收到了垂直同步信号
- 调用 mPrimaryDispSync.addResyncSample 添加一个重新同步的信号

关于 [DispSync](http://androidxref.com/9.0.0_r3/xref/frameworks/native/services/surfaceflinger/DispSync.cpp) 的创建, 我们在 SurfaceFlinger 启动的构造函数中已经分析了, 其内部持有一个 DispSyncThread 线程, 在 threadLoop 中持续的等待 Vsync 信号的到来并执行分发

关于 HAL 层的操作, 我们就不关心了, 这里主要看看后面两个操作, 我们先看看这个 mPrimaryDispSync 后续两个操作

## 一. 添加 Vysnc 事件
```
// frameworks/native/services/surfaceflinger/DispSync.cpp

bool DispSync::addResyncSample(nsecs_t timestamp) {
    Mutex::Autolock lock(mMutex);

    size_t idx = (mFirstResyncSample + mNumResyncSamples) % MAX_RESYNC_SAMPLES;
    mResyncSamples[idx] = timestamp;
    if (mNumResyncSamples == 0) {
        mPhase = 0;
        mReferenceTime = timestamp;
        ALOGV("[%s] First resync sample: mPeriod = %" PRId64 ", mPhase = 0, "
              "mReferenceTime = %" PRId64,
              mName, ns2us(mPeriod), ns2us(mReferenceTime));
        mThread->updateModel(mPeriod, mPhase, mReferenceTime);
    }

    if (mNumResyncSamples < MAX_RESYNC_SAMPLES) {
        mNumResyncSamples++;
    } else {
        mFirstResyncSample = (mFirstResyncSample + 1) % MAX_RESYNC_SAMPLES;
    }
    
    // 调用了 updateModelLocked 函数
    updateModelLocked();
    ......
    return !modelLocked;
}
```
可以看到 mPrimaryDispSync.addResyncSample 函数, 调用了 updateModelLocked 函数, 从字面意思上来看, 这个很有可能是解除 threadLoop 中管程 wait 的关键
```
void DispSync::updateModelLocked() {
    ......
    if (mNumResyncSamples >= MIN_RESYNC_SAMPLES_FOR_UPDATE) {
        
        nsecs_t durationSum = 0;
        nsecs_t minDuration = INT64_MAX;
        nsecs_t maxDuration = 0;
        for (size_t i = 1; i < mNumResyncSamples; i++) {
            size_t idx = (mFirstResyncSample + i) % MAX_RESYNC_SAMPLES;
            size_t prev = (idx + MAX_RESYNC_SAMPLES - 1) % MAX_RESYNC_SAMPLES;
            nsecs_t duration = mResyncSamples[idx] - mResyncSamples[prev];
            durationSum += duration;
            minDuration = min(minDuration, duration);
            maxDuration = max(maxDuration, duration);
        }

        // Exclude the min and max from the average
        durationSum -= minDuration + maxDuration;
        mPeriod = durationSum / (mNumResyncSamples - 3);

        ......
        
        // Artificially inflate the period if requested.
        mPeriod += mPeriod * mRefreshSkipCount;
        
        // 调用 DispSyncThread 的 updateModel
        mThread->updateModel(mPeriod, mPhase, mReferenceTime);
        mModelUpdated = true;
    }
}
```
DispSync::updateModelLocked 中主要是一些**周期的更新操作**, 笔者对其也不是非常了解, 我们主要是为了理清 Vsync 的走向, 因此我们直接看看 DispSyncThread 的 updateModel 操作
```
class DispSyncThread : public Thread {
public:
    
    void updateModel(nsecs_t period, nsecs_t phase, nsecs_t referenceTime) {
        ......
        mPeriod = period;
        mPhase = phase;
        mReferenceTime = referenceTime;
        // 唤醒阻塞的 threadLoop
        mCond.signal();
    }
    
}
```
到这里便可以看到它唤醒了阻塞 DispSyncThread 中阻塞的 threadLoop, 既然如此依赖 threadLoop 就可以执行垂直同步的分发了

接下来看看 threadLoop 如何分发这个 Vsync 事件

## 二. 分发 Vsync 事件
```
// frameworks/native/services/surfaceflinger/DispSync.cpp 
virtual bool threadLoop() {
    status_t err;
    nsecs_t now = systemTime(SYSTEM_TIME_MONOTONIC);
    while (true) {
        Vector<CallbackInvocation> callbackInvocations;
        nsecs_t targetTime = 0;
        { // Scope for lock
            // 1. 若 mPeriod 为 0, 则循环等待
            if (mPeriod == 0) {
                err = mCond.wait(mMutex);
                if (err != NO_ERROR) {
                    ALOGE("error waiting for new events: %s (%d)", strerror(-err), err);
                    return false;
                }
                continue;
            }
            ......
            // 2. 收集的所有事件处理方法
            callbackInvocations = gatherCallbackInvocationsLocked(now);
        }
        if (callbackInvocations.size() > 0) {
            // 3. 执行事件处理函数
            fireCallbackInvocations(callbackInvocations);
        }
    }
    return false;
}
```
我们知道, 通过 DispSyncThread.updateModel 的函数调用, threadLoop 的阻塞便解除了, 进而就可以执行后续的操作了

我们先看看收集回调方法的实现

### 一) 收集处理事件处理方法
```
// frameworks/native/services/surfaceflinger/DispSync.cpp 
Vector<CallbackInvocation> gatherCallbackInvocationsLocked(nsecs_t now) {
    if (kTraceDetailedInfo) ATRACE_CALL();
    .......
    Vector<CallbackInvocation> callbackInvocations;
    nsecs_t onePeriodAgo = now - mPeriod;
    for (size_t i = 0; i < mEventListeners.size(); i++) {
        nsecs_t t = computeListenerNextEventTimeLocked(mEventListeners[i], onePeriodAgo);
        if (t < now) {
            // 将 mEventListeners 中的 Callback 封装进 CallbackInvocation 
            CallbackInvocation ci;
            ci.mCallback = mEventListeners[i].mCallback;
            ci.mEventTime = t;
            .......
            callbackInvocations.push(ci);
            mEventListeners.editItemAt(i).mLastEventTime = t;
        }
    }
    return callbackInvocations;
}
```
收集回调的方式也比较简单, 它将 mEventListeners 中的 mCallback 封装进了 CallbackInvocation 中

下面我们看看 fireCallbackInvocations 

### 二) 执行事件处理函数
```
void fireCallbackInvocations(const Vector<CallbackInvocation>& callbacks) {
    if (kTraceDetailedInfo) ATRACE_CALL();
    for (size_t i = 0; i < callbacks.size(); i++) {
        // 调用了 mCallback 的 onDispSyncEvent 来消费这一个 Vsync 信号事件
        callbacks[i].mCallback->onDispSyncEvent(callbacks[i].mEventTime);
    }
}
```
事件处理函数简单的调用了 mCallback 的 onDispSyncEvent 来消费这一个 Vsync 信号事件

**这里我们有个疑问, 这个 EnventListener 中的 Callback 是什么时候注入的呢?**

### 三) 回调注入时机
我们在分析 SurfaceFlinger 的启动时, 有分析过 EventThread 的 waitForEventLocked 函数, 这里回顾一下
```
// frameworks/native/services/surfaceflinger/EventThread.cpp
Vector<sp<EventThread::Connection> > EventThread::waitForEventLocked(
        std::unique_lock<std::mutex>* lock, DisplayEventReceiver::Event* event) {
    Vector<sp<EventThread::Connection> > signalConnections;
    
    while (signalConnections.isEmpty() && mKeepRunning) {
        bool eventPending = false;
        bool waitForVSync = false;
        size_t vsyncCount = 0;
        nsecs_t timestamp = 0;
        ......
        
        // Here we figure out if we need to enable or disable vsyncs
        if (timestamp && !waitForVSync) {
            // 没有 Connection, 则调用 enableVSyncLocked
            disableVSyncLocked();
        } else if (!timestamp && waitForVSync) {
            // 至少有一个连接在等待 Vsync 信号, 因此调用 enableVSyncLocked 函数
            enableVSyncLocked();
        }
        ......
    }
    
    return signalConnections;
}
```
当 EventThread 存在 Connection 在等待 Vsync 信号时, 则会调用 enableVSyncLocked 函数
```
// frameworks/native/services/surfaceflinger/EventThread.cpp
void EventThread::enableVSyncLocked() {
    if (!mUseSoftwareVSync) {
        // never enable h/w VSYNC when screen is off
        if (!mVsyncEnabled) {
            mVsyncEnabled = true;
            // 调用了 DispSyncSource 的 Callback
            mVSyncSource->setCallback(this);
            // 调用了 DispSyncSource 的 setVSyncEnabled
            mVSyncSource->setVSyncEnabled(true);
        }
    }
    mDebugVsyncEnabled = true;
}
```
可以看到 EventThread 中调用了 DispSyncSource 的 setCallback 和 setVSyncEnabled, 他们的函数实现如下

```
// frameworks/native/services/surfaceflinger/SurfaceFlinger.cpp
class DispSyncSource final : public VSyncSource, private DispSync::Callback {
    
    void setCallback(VSyncSource::Callback* callback) override{
        Mutex::Autolock lock(mCallbackMutex);
        // 保存 EventThread 传递过来的 Callback
        mCallback = callback;
    }
    
    void setVSyncEnabled(bool enable) override {
        Mutex::Autolock lock(mVsyncMutex);
        if (enable) {
            // 调用了 DispSync.addEventListener 来添加一个 EventListener, 将其自身作为 Callback 传入
            status_t err = mDispSync->addEventListener(mName, mPhaseOffset,
                    static_cast<DispSync::Callback*>(this));
            .......
        } else {
            // 移除事件回调 
            status_t err = mDispSync->removeEventListener(
                    static_cast<DispSync::Callback*>(this));
            ......
        }
        mEnabled = enable;
    }
    
}
```
好的, 到这里我们就知晓了, **其实 DispSync 中 mEventListerer 的回调 mCallback 其实就是 DispSyncSource**

下面我们便来分析一下 DispSyncSource 的 onDispSyncEvent 是如何响应 Vsync 事件的

## 三. Vsync 事件的响应
```
// frameworks/native/services/surfaceflinger/SurfaceFlinger.cpp
class DispSyncSource final : public VSyncSource, private DispSync::Callback {
    
private:
    virtual void onDispSyncEvent(nsecs_t when) {
        VSyncSource::Callback* callback;
        {
            Mutex::Autolock lock(mCallbackMutex);
            callback = mCallback;
            ......
        }
        // 调用了 mCallback 的 onVSyncEvent
        if (callback != nullptr) {
            callback->onVSyncEvent(when);
        }
    }    
    
}
```
好的, 从这里可以看出 DispSyncSource 其实是作为 EventThread 与 DispSyncThread 的中转站, 有些类似 MVP 中的 Presenter

它将事件分发给了 EventThread 的 onVSyncEvent 去处理, 我们继续往下分析

```
// frameworks/native/services/surfaceflinger/EventThread.cpp

void EventThread::onVSyncEvent(nsecs_t timestamp) {
    std::lock_guard<std::mutex> lock(mMutex);
    mVSyncEvent[0].header.type = DisplayEventReceiver::DISPLAY_EVENT_VSYNC;
    mVSyncEvent[0].header.id = 0;
    mVSyncEvent[0].header.timestamp = timestamp;
    mVSyncEvent[0].vsync.count++;
    // 好的, 这里调用了 notify_all, 解除了 EventThread 中 waitForEventLocked 的阻塞
    mCondition.notify_all();
}
```
在 SurfaceFlinger 启动的过程中, 我们知道 EventThread 的 threadMain 在解除了阻塞之后, 会调用将这个事件写入 BitTube 中, 从而**唤醒 SurfaceFlinger 主线程的 Looper 去执行 onMessageReceived 函数, Message 类型为 INVALIDATE**

接下来我们看看 onMessageReceived 的函数到底是如何处理这个垂直同步信号的

```
void SurfaceFlinger::onMessageReceived(int32_t what) {
    ATRACE_CALL();
    switch (what) {
        case MessageQueue::INVALIDATE: {
            ......
            bool refreshNeeded = handleMessageTransaction();
            // 1. 调用了 handleMessageInvalidate
            refreshNeeded |= handleMessageInvalidate();
            refreshNeeded |= mRepaintEverything;
            if (refreshNeeded) {
                // 2. 调用了 signalRefresh 函数
                signalRefresh();
            }
            break;
        }
        ......
    }
}
```
好的可以看到 INVALIDATE 消息, 首先会调用 handleMessageInvalidate, 判断是否需要重新刷新界面, 若需要则会调用 signalRefresh 进行数据重绘

自此之后这便是 SurfaceFlinger 重要的视图合成操作了, 到下一篇文章中再继续分析

## 总结
![Vsync 处理流程](https://i.loli.net/2019/10/23/3zNfX5i2ExRu9cM.png)

- DispSync: 负责处理 Vsync 信号周期相关的操作
- DispSyncThread: 负责 Vsync 事件的分发
- EventThread: 负责 Vsync 事件的响应

最终通过 Native 层的 Handler 交付给主线程的 onMessageReceivced 去处理