---
title: Android 系统架构 —— IMS 的 ANR 产生流程
permalink: android-source/ims-anr
key: android-source-ims-anr
tags: AndroidFramework
---
## 前言
前面我们学习了 IMS 的启动和 IMS 的事件分发, 了解整个事件的分发流程, 除了事件分发之外 ANR 也是与 IMS 息息相关的

这篇文章我们就着重了解一下 ANR 的产生原因

<!--more-->

## 一. ANR 的产生流程
我们在 IMS 启动篇中分析 InputDispatcherThread 启动的时候, 是直接分析无 ANR 的情况的, 这里我们再简单的走一遍流程, 看看 ANR 是怎么记录并且发出的

```
// frameworks/native/services/inputflinger/InputDispatcher.cpp
void InputDispatcher::dispatchOnceInnerLocked(nsecs_t* nextWakeupTime) {
    nsecs_t currentTime = now();
    ......
    // 1. 若 mPendingEvent 为 NULL, 则尝试获取事件
    if (! mPendingEvent) {
        if (mInboundQueue.isEmpty()) {
            ......
        } else {
            // 从 mInboundQueue 中取事件
            mPendingEvent = mInboundQueue.dequeueAtHead();
            traceInboundQueueLengthLocked();
        }

        // 重置 ANR 信息
        resetANRTimeoutsLocked();
    }
    switch (mPendingEvent->type) {
    ......
    case EventEntry::TYPE_KEY: {
        KeyEntry* typedEntry = static_cast<KeyEntry*>(mPendingEvent);
        ......
        // 2. 分发事件
        done = dispatchKeyLocked(currentTime, typedEntry, &dropReason, nextWakeupTime);
        break;
    }
    ......
    if (done) {
        // 3. 将 mPendingEvent 置空, 表示事件分发完毕了
        releasePendingEventLocked();
        *nextWakeupTime = LONG_LONG_MIN;  // force next poll to wake up immediately
    }
}
```
dispatchOnceInnerLocked 函数我们在 IMS 启动篇中已经分析过了, 这里我们看看不一样的代码
- mPendingEvent 为 NULL, 说明分发一个新的事件
- mPendingEvent 不为 NULL, 说明分发之前的事件

我们知道分发成功之后, mPendingEvent 会被置空, 若是 mPendingEvent 没有被置空, 则说明上一次分发事件失败了

ANR 是一种异常情况, 显然不可能在分发成功的状态下产生, 因此这里我们再看看 dispatchKeyLocked 的流程, 主要关注一下它是如何分发失败的

```
// frameworks/native/services/inputflinger/InputDispatcher.cpp
bool InputDispatcher::dispatchKeyLocked(nsecs_t currentTime, KeyEntry* entry,
        DropReason* dropReason, nsecs_t* nextWakeupTime) {
    ......

    // 1. 找寻焦点窗口
    Vector<InputTarget> inputTargets;
    int32_t injectionResult = findFocusedWindowTargetsLocked(currentTime,
            entry, inputTargets, nextWakeupTime);
    // 若无焦点窗口, 则会返回失败
    if (injectionResult == INPUT_EVENT_INJECTION_PENDING) {
        return false;
    }

    // 2. 交由焦点窗口分发
    dispatchEventLocked(currentTime, entry, inputTargets);
    return true;
}
```
可以看到在获取焦点窗口失败的情况下, 会返回失败, 下面我们看看什么情况下会找寻失败

```
// frameworks/native/services/inputflinger/InputDispatcher.cpp
int32_t InputDispatcher::findFocusedWindowTargetsLocked(nsecs_t currentTime,
        const EventEntry* entry, Vector<InputTarget>& inputTargets, nsecs_t* nextWakeupTime) {
    int32_t injectionResult;
    std::string reason;

    ......
    // 1. 检查窗口是否可以接收输入流
    reason = checkWindowReadyForMoreInputLocked(currentTime,
            mFocusedWindowHandle, entry, "focused");
    // 2. 若 reason 非空, 则说明无法接收输入流, 则调用 handleTargetsNotReadyLocked 处理异常情况
    if (!reason.empty()) {
        injectionResult = handleTargetsNotReadyLocked(currentTime, entry,
                mFocusedApplicationHandle, mFocusedWindowHandle, nextWakeupTime, reason.c_str());
        goto Unresponsive;
    }

    // 2. 找寻成功
    injectionResult = INPUT_EVENT_INJECTION_SUCCEEDED;
    addWindowTargetLocked(mFocusedWindowHandle,
            InputTarget::FLAG_FOREGROUND | InputTarget::FLAG_DISPATCH_AS_IS, BitSet32(0),
            inputTargets);
    .......
    return injectionResult;
}
```
好的, 这里可以看到首先会调用 **checkWindowReadyForMoreInputLocked** 检查窗口是否可以响应输入流, 若是成功则调用 addWindowTargetLocked 进行正常分发, 若是失败则调用 **handleTargetsNotReadyLocked** 处理失败情况

关于 addWindowTargetLocked 的实现, 我们已经分析过了, 这里我们看看检查和处理失败的逻辑

### 一) 检查窗口是否可以接收输入流
```
// frameworks/native/services/inputflinger/InputDispatcher.cpp
std::string InputDispatcher::checkWindowReadyForMoreInputLocked(nsecs_t currentTime,
        const sp<InputWindowHandle>& windowHandle, const EventEntry* eventEntry,
        const char* targetType) {
    // 窗口处于 paused 状态, 则无法响应输入流
    if (windowHandle->getInfo()->paused) {
        return StringPrintf("Waiting because the %s window is paused.", targetType);
    }
    // 如果窗口的连接没有注册，则无法响应输入流
    ssize_t connectionIndex = getConnectionIndexLocked(windowHandle->getInputChannel());
    if (connectionIndex < 0) {
        return StringPrintf("Waiting because the %s window's input channel is not "
                "registered with the input dispatcher.  The window may be in the process "
                "of being removed.", targetType);
    }
    // 窗口连接失效, 则无法响应输入流
    sp<Connection> connection = mConnectionsByFd.valueAt(connectionIndex);
    if (connection->status != Connection::STATUS_NORMAL) {
        return StringPrintf("Waiting because the %s window's input connection is %s."
                "The window may be in the process of being removed.", targetType,
                connection->getStatusLabel());
    }
    // 窗口的事件待处理队列 outboundQueue 已满, 则无法响应输入流
    if (connection->inputPublisherBlocked) {
        return StringPrintf("Waiting because the %s window's input channel is full.  "
                "Outbound queue length: %d.  Wait queue length: %d.",
                targetType, connection->outboundQueue.count(), connection->waitQueue.count());
    }

    if (eventEntry->type == EventEntry::TYPE_KEY) {
        // 若为 KEY Event, 窗口的 outboundQueue 或 waitQueue 有数据, 则无法响应输入流
        if (!connection->outboundQueue.isEmpty() || !connection->waitQueue.isEmpty()) {
            return StringPrintf("Waiting to send key event because the %s window has not "
                    "finished processing all of the input events that were previously "
                    "delivered to it.  Outbound queue length: %d.  Wait queue length: %d.",
                    targetType, connection->outboundQueue.count(), connection->waitQueue.count());
        }
    } else {
        // 非 KEY Event, 窗口 waitQueue 非空 且 头事件分发超时 500ms, 则无法响应输入流
        if (!connection->waitQueue.isEmpty()
                && currentTime >= connection->waitQueue.head->deliveryTime
                        + STREAM_AHEAD_EVENT_TIMEOUT) {
            return StringPrintf("Waiting to send non-key event because the %s window has not "
                    "finished processing certain input events that were delivered to it over "
                    "%0.1fms ago.  Wait queue length: %d.  Wait queue head age: %0.1fms.",
                    targetType, STREAM_AHEAD_EVENT_TIMEOUT * 0.000001f,
                    connection->waitQueue.count(),
                    (currentTime - connection->waitQueue.head->deliveryTime) * 0.000001f);
        }
    }
    return "";
}
```
从 checkWindowReadyForMoreInputLocked 函数中我们可以看到引起事件分发失败的原因如下
- **窗口处于 paused 状态**
- **如果窗口的连接没有注册**
- **窗口连接失效**
- **窗口的事件待处理队列 outboundQueue 已满**
- 若为 Key Event, 窗口的 outboundQueue 或 waitQueue 有数据
- 若非 Key Event, 窗口 waitQueue 非空 且 头事件分发超时 500ms

**上述是引起事件流分发失败的原因, 但分发失败并不意味着会直接生成 ANR, 只有失败超过一定的时长才会产生 ANR**

因此还需要一个超时机制, 下面我们继续探索

### 二) 处理响应失败的情况
```
// frameworks/native/services/inputflinger/InputDispatcher.cpp
constexpr nsecs_t DEFAULT_INPUT_DISPATCHING_TIMEOUT = 5000 * 1000000LL; // 5 秒钟
int32_t InputDispatcher::handleTargetsNotReadyLocked(nsecs_t currentTime,
        const EventEntry* entry,
        const sp<InputApplicationHandle>& applicationHandle,
        const sp<InputWindowHandle>& windowHandle,
        nsecs_t* nextWakeupTime, const char* reason) {
    if (applicationHandle == NULL && windowHandle == NULL) {
        ......
    } else {
        // 1. 若当前事件还未设置超时监听, 则将当前事件的最迟分发事件记录在 mInputTargetWaitTimeoutTime 中
        if (mInputTargetWaitCause != INPUT_TARGET_WAIT_CAUSE_APPLICATION_NOT_READY) {
            // 1.1 获取分发的等待时长
            nsecs_t timeout;
            if (windowHandle != NULL) {
                timeout = windowHandle->getDispatchingTimeout(DEFAULT_INPUT_DISPATCHING_TIMEOUT);
            } else if (applicationHandle != NULL) {
                timeout = applicationHandle->getDispatchingTimeout(
                        DEFAULT_INPUT_DISPATCHING_TIMEOUT);
            } else {
                timeout = DEFAULT_INPUT_DISPATCHING_TIMEOUT;
            }
            // 1.2 更新等待原因
            mInputTargetWaitCause = INPUT_TARGET_WAIT_CAUSE_APPLICATION_NOT_READY;
            mInputTargetWaitStartTime = currentTime;
            // 1.3 记录最迟分发时间
            mInputTargetWaitTimeoutTime = currentTime + timeout;
            mInputTargetWaitTimeoutExpired = false;
            mInputTargetWaitApplicationHandle.clear();
            // 1.4 记录等待的窗体信息
            if (windowHandle != NULL) {
                mInputTargetWaitApplicationHandle = windowHandle->inputApplicationHandle;
            }
            if (mInputTargetWaitApplicationHandle == NULL && applicationHandle != NULL) {
                mInputTargetWaitApplicationHandle = applicationHandle;
            }
        }
    }
    ......
    // 3. 若当前时间已经超过了唤醒时间, 但还是分发失败了, 则调用 onANRLocked 抛出 ANR
    if (currentTime >= mInputTargetWaitTimeoutTime) {
        // 抛出 ANR
        onANRLocked(currentTime, applicationHandle, windowHandle,
                entry->eventTime, mInputTargetWaitStartTime, reason);
        // 表明立即唤醒
        *nextWakeupTime = LONG_LONG_MIN;
        return INPUT_EVENT_INJECTION_PENDING;
    } else {
        // 2. 将下一次唤醒的时间记为 mInputTargetWaitTimeoutTime 的值, 延时 5s 唤醒
        if (mInputTargetWaitTimeoutTime < *nextWakeupTime) {
            *nextWakeupTime = mInputTargetWaitTimeoutTime;
        }
        return INPUT_EVENT_INJECTION_PENDING;
    }
}
```
可以看到 handleTargetsNotReadyLocked 中便是安放定时器的逻辑
- 若当前事件还未设置超时监听, 则将当前事件的最迟分发时间记录在 mInputTargetWaitTimeoutTime 中
  - 默认延时为 5 秒钟
  - 将 nextWakeupTime 记录为 mInputTargetWaitTimeoutTime
    - 表示让 InputDispatcher 的 Looper 睡眠到 mInputTargetWaitTimeoutTime 时刻再继续进行这个事件的分发
- 若当前事件已经设置过了超时监听, 并且当前时刻 >= mInputTargetWaitTimeoutTime 
  - 则说明这个事件已经超时了, 则调用 onANRLocked 扔出 ANR

### 三) 回顾
好的, 到这里我们就清晰的知道 ANR 产生的流程了

![ANR 产生流程](https://i.loli.net/2019/11/21/MUv8GWXHtjQglF5.png)

- **判断窗口是否可以分发事件**
  - 窗口处于 paused 状态, 则无法分发事件
  - 如果窗口的连接没有注册, 则无法分发事件
  - 窗口连接失效, 则无法分发事件
  - 窗口的事件待处理队列 outboundQueue 已满, 则无法分发事件
  - 若为 Key Event, 窗口的 outboundQueue 或 waitQueue 有数据, 则无法分发事件
  - 若非 Key Event, 窗口 waitQueue 非空 且 头事件分发超时 500ms, 则无法分发事件
- **若不能分发, 则安放定时器, 默认 5s 之后重试**
  - 在此期间若是分发成功之后会重置定时器
- **重试的时候, 若仍然分发失败, 则调用 onANRLocked 弹出 ANR 弹窗**

## 二. ANR 的重置时机
我们在 InputDispatcher 事件分发的起始函数 dispatchOnceInnerLocked 中就层看到若是 mPendingEvent 为 NULL, 则会从 mInboundQueue 中获取一个新的事件, 与此同时会调用 **resetANRTimeoutsLocked** 重置超时数据
```
// frameworks/native/services/inputflinger/InputDispatcher.cpp
void InputDispatcher::dispatchOnceInnerLocked(nsecs_t* nextWakeupTime) {
    nsecs_t currentTime = now();
    ......
    // 若 mPendingEvent 为 NULL, 则尝试获取事件
    if (! mPendingEvent) {
        if (mInboundQueue.isEmpty()) {
            ......
        } else {
            // 从 mInboundQueue 中取事件
            mPendingEvent = mInboundQueue.dequeueAtHead();
            ......
        }

        // 重置 ANR 信息
        resetANRTimeoutsLocked();
    }
    ......
}
```
这里也不难理解, mPendingEvent 为 NULL, 说明上一次的事件已经分发结束了, 这时的超时信息已经无效了, 下面看看 resetANRTimeoutsLocked 重置超时信息函数的实现
```
// frameworks/native/services/inputflinger/InputDispatcher.cpp
void InputDispatcher::resetANRTimeoutsLocked() {
    // 清空超时原因
    mInputTargetWaitCause = INPUT_TARGET_WAIT_CAUSE_NONE;
    // 清空等待的窗体信息
    mInputTargetWaitApplicationHandle.clear();
}
```
好的, 可以看到 resetANRTimeoutsLocked 是重置 ANR 的关键, 也就是说只要调用了 resetANRTimeoutsLocked 这个函数, 就能够找到清空 ANR 的时机了

```
// frameworks/native/services/inputflinger/InputDispatcher.h
class InputDispatcherInterface : public virtual RefBase, public InputListenerInterface {
protected:
    InputDispatcherInterface() { }
    virtual ~InputDispatcherInterface() { }

public:

    void dispatchOnceInnerLocked(nsecs_t* nextWakeupTime);
    
    // Reset and drop everything the dispatcher is doing.
    void resetAndDropEverythingLocked(const char* reason);
    
    void releasePendingEventLocked();
    
    /* Sets the focused application.
     *
     * This method may be called on any thread (usually by the input manager).
     */
    virtual void setFocusedApplication(
            const sp<InputApplicationHandle>& inputApplicationHandle) = 0;
            
    virtual void setInputDispatchMode(bool enabled, bool frozen);
    
}
```
调用 resetANRTimeoutsLocked 函数的时机如上述函数所示, 这里笔者就没有一个一个看了, 从 [gityuan 的博文](http://gityuan.com/2017/01/01/input-anr/)中了解到, 具体的时机如下
- 再次分发事件的过程(dispatchOnceInnerLocked)
- 解冻屏幕, 系统开/关机的时刻点 (thawInputDispatchingLw, setEventDispatchingLw)
- wms 聚焦 app 的改变 (WMS.setFocusedApp, WMS.removeAppToken)
- 设置 input filter 的过程 (IMS.setInputFilter)

关于 ANR 的重置时机就介绍到这里, 下面我们看看 ANR 的弹出流程

## 三. ANR 的弹出
从上面的分析我们知道, ANR 的弹出是由 onANRLocked 发起的, 下面看看它的实现流程
```
// frameworks/native/services/inputflinger/InputDispatcher.cpp
void InputDispatcher::onANRLocked(
        nsecs_t currentTime, const sp<InputApplicationHandle>& applicationHandle,
        const sp<InputWindowHandle>& windowHandle,
        nsecs_t eventTime, nsecs_t waitStartTime, const char* reason) {
    ......
    // 提交 doNotifyANRLockedInterruptible 执行的指令到 mCommandQueue 指令队列
    CommandEntry* commandEntry = postCommandLocked(
            & InputDispatcher::doNotifyANRLockedInterruptible);
    commandEntry->inputApplicationHandle = applicationHandle;
    commandEntry->inputWindowHandle = windowHandle;
    commandEntry->reason = reason;
}
```
可以看到这里创建一条 doNotifyANRLockedInterruptible 的执行指令到指令队列中, 下面看看这个函数的实现
```
// frameworks/native/services/inputflinger/InputDispatcher.cpp
void InputDispatcher::doNotifyANRLockedInterruptible(
        CommandEntry* commandEntry) {
    ......
    // 交由 mPolicy 的 notifyANR 函数执行
    nsecs_t newTimeout = mPolicy->notifyANR(
            commandEntry->inputApplicationHandle, commandEntry->inputWindowHandle,
            commandEntry->reason);
    ......
}
```
我们在 IMS 启动时分析过 InputDispatcher 的构造函数, 这个 mPolicy 即 NativeInputMangaer 对象, 下面我们看看它的 notifyANR 的实现
```
// frameworks/base/services/core/jni/com_android_server_input_InputManagerService.cpp
nsecs_t NativeInputManager::notifyANR(const sp<InputApplicationHandle>& inputApplicationHandle,
        const sp<InputWindowHandle>& inputWindowHandle, const std::string& reason) {
    ......
    JNIEnv* env = jniEnv();
    jobject inputApplicationHandleObj =
            getInputApplicationHandleObjLocalRef(env, inputApplicationHandle);
    jobject inputWindowHandleObj =
            getInputWindowHandleObjLocalRef(env, inputWindowHandle);
    jstring reasonObj = env->NewStringUTF(reason.c_str());
    // 通过 JNIEnv 回调到 Java 层处理
    jlong newTimeout = env->CallLongMethod(mServiceObj,
                gServiceClassInfo.notifyANR, inputApplicationHandleObj, inputWindowHandleObj,
                reasonObj);
    ......
    return newTimeout;
}
```
至此 Native 层就将 ANR 的指令传到了 Java 层的 IMS 中, 下面看看它的事件

### 一) IMS 处理 ANR
```
public class InputManagerService extends IInputManager.Stub
        implements Watchdog.Monitor {
        
    private WindowManagerCallbacks mWindowManagerCallbacks;

    // Native callback.
    private long notifyANR(InputApplicationHandle inputApplicationHandle,
            InputWindowHandle inputWindowHandle, String reason) {
        // 交由 WMS 处理
        return mWindowManagerCallbacks.notifyANR(
                inputApplicationHandle, inputWindowHandle, reason);
    }
    
}
```
这个 mWindowManagerCallbacks 的实现为 InputMonitor, 下面看看它的实现
```
final class InputMonitor implements InputManagerService.WindowManagerCallbacks {

    public InputMonitor(WindowManagerService service) {
        mService = service;
    }
    
     @Override
    public long notifyANR(InputApplicationHandle inputApplicationHandle,
            InputWindowHandle inputWindowHandle, String reason) {
        AppWindowToken appWindowToken = null;
        WindowState windowState = null;
        boolean aboveSystem = false;
        synchronized (mService.mWindowMap) {
            // 获取窗体信息
            if (inputWindowHandle != null) {
                windowState = (WindowState) inputWindowHandle.windowState;
                if (windowState != null) {
                    appWindowToken = windowState.mAppToken;
                }
            }
            if (appWindowToken == null && inputApplicationHandle != null) {
                appWindowToken = (AppWindowToken)inputApplicationHandle.appWindowToken;
            }
            ......
        }
        ......
        if (appWindowToken != null && appWindowToken.appToken != null) {
            // 调用 AppWindowContainerController.keyDispatchingTimedOut 处理超时
            final AppWindowContainerController controller = appWindowToken.getController();
            final boolean abort = controller != null && controller.keyDispatchingTimedOut(reason,
                            (windowState != null) ? windowState.mSession.mPid : -1);
           ......
        } else if (windowState != null) {
            try {
                // 调用 AMS 的 inputDispatchingTimedOut 处理窗体超时
                long timeout = ActivityManager.getService().inputDispatchingTimedOut(
                        windowState.mSession.mPid, aboveSystem, reason);
                .......
            } catch (RemoteException ex) {
            }
        }
        return 0;
    }
    
}
```
可以看到在 InputMonitor 执行 notifyANR 方法最终会**交由 AMS 的 inputDispatchingTimedOut 方法去处理**, 下面看看 AMS 的实现

### 二) AMS 处理 ANR
```
public class ActivityManagerService extends IActivityManager.Stub
        implements Watchdog.Monitor, BatteryStatsImpl.BatteryCallback {
    
    @Override
    public long inputDispatchingTimedOut(int pid, final boolean aboveSystem, String reason) {
        ......
        if (inputDispatchingTimedOut(proc, null, null, aboveSystem, reason)) {
            return -1;
        }

        return timeout;
    }

    /**
     * Handle input dispatching timeouts.
     * Returns whether input dispatching should be aborted or not.
     */
    public boolean inputDispatchingTimedOut(final ProcessRecord proc,
            final ActivityRecord activity, final ActivityRecord parent,
            final boolean aboveSystem, String reason) {
        ......
        if (proc != null) {
            ......
            // 投递到 ServiceThread 中, 交由 AppErros 处理 ANR
            mHandler.post(new Runnable() {
                @Override
                public void run() {
                    mAppErrors.appNotResponding(proc, activity, parent, aboveSystem, annotation);
                }
            });
        }

        return true;
    }
    
}
```
这个 mHandler 我们在 AMS 启动篇中分析过, 它是 ServiceThread 的 Handler, 也就是说 AMS 最终会在 ServiceThread 中交由 AppErros 处理 ANR, 下面看看 AppErros 的 appNotResponding 实现

#### 1. 在 ServiceThread 中执行 AppErrors.appNotResponding
```
class AppErrors {

    private final ActivityManagerService mService;
    private final Context mContext;
    
    final void appNotResponding(ProcessRecord app, ActivityRecord activity,
            ActivityRecord parent, boolean aboveSystem, final String annotation) {
            
        ......// dump ANR 信息
        
        // 弹出 ANR 弹窗
        synchronized (mService) {
            mService.mBatteryStatsService.noteProcessAnr(app.processName, app.uid);

            if (isSilentANR) {
                app.kill("bg anr", true);
                return;
            }
            
            // 将消息投递到 UIThread 展示 Dialog
            Message msg = Message.obtain();
            msg.what = ActivityManagerService.SHOW_NOT_RESPONDING_UI_MSG;
            msg.obj = new AppNotRespondingDialog.Data(app, activity, aboveSystem);
            mService.mUiHandler.sendMessage(msg);
        }
    }
    
}
```
在 AppErrors.appNotResponding 中可以看到, 它投放了一个 SHOW_NOT_RESPONDING_UI_MSG 消息到 UIThread 展示 ANR 弹窗

下面看看 ANR 弹出的过程
```
public class ActivityManagerService extends IActivityManager.Stub
        implements Watchdog.Monitor, BatteryStatsImpl.BatteryCallback {
        
    final class UiHandler extends Handler {
        public UiHandler() {
            super(com.android.server.UiThread.get().getLooper(), null, true);
        }

        @Override
        public void handleMessage(Message msg) {
            switch (msg.what) {
                case SHOW_ERROR_UI_MSG: {
                    mAppErrors.handleShowAppErrorUi(msg);
                    ensureBootCompleted();
                }
                break;
            }
        }
    }
    
}
```
这次是在 UIThread 中调用了 AppErrors 的 handleShowAppErrorUi, 下面看看 UIThread 的弹窗过程

#### 2. 在 UIThread 弹出 ANR 弹窗
```
class AppErrors {
    
    void handleShowAppErrorUi(Message msg) {
        AppErrorDialog.Data data = (AppErrorDialog.Data) msg.obj;
        boolean showBackground = Settings.Secure.getInt(mContext.getContentResolver(),
                Settings.Secure.ANR_SHOW_BACKGROUND, 0) != 0;

        AppErrorDialog dialogToShow = null;
        final String packageName;
        final int userId;
        synchronized (mService) {
            final ProcessRecord proc = data.proc;
            final AppErrorResult res = data.result;
            ......
            // 构造 ERROR 数据
            if ((mService.canShowErrorDialogs() || showBackground) && !crashSilenced
                    && (showFirstCrash || showFirstCrashDevOption || data.repeating)) {
                // 创建 ANR 弹窗
                proc.crashDialog = dialogToShow = new AppErrorDialog(mContext, mService, data);
            } else {
                ......
            }
        }
        // 弹出 ANR 弹窗
        if (dialogToShow != null) {
            dialogToShow.show();
        }
    }
    
}
```
可以看到最终构建了一个 AppErrorDialog 并且展示出来, 到这里 ANR 的弹出流程就分析清楚了

## 总结
### ANR 产生流程
![ANR 产生流程](https://i.loli.net/2019/11/21/MUv8GWXHtjQglF5.png)

- **判断窗口是否可以分发事件**
  - 窗口处于 paused 状态, 则无法分发事件
  - 如果窗口的连接没有注册, 则无法分发事件
  - 窗口连接失效, 则无法分发事件
  - 窗口的事件待处理队列 outboundQueue 已满, 则无法分发事件
  - 若为 Key Event, 窗口的 outboundQueue 或 waitQueue 有数据, 则无法分发事件
  - 若非 Key Event, 窗口 waitQueue 非空 且 头事件分发超时 500ms, 则无法分发事件
- **若不能分发, 则安放定时器, 默认 5s 之后重试**
  - 在此期间若是分发成功之后会重置定时器
- **重试的时候, 若仍然分发失败, 则调用 onANRLocked 弹出 ANR 弹窗**

### ANR 的重置时机
- 解冻屏幕, 系统开/关机的时刻点 (thawInputDispatchingLw, setEventDispatchingLw)
- wms聚焦app的改变 (WMS.setFocusedApp, WMS.removeAppToken)
- 设置input filter的过程 (IMS.setInputFilter)
- 再次分发事件的过程(dispatchOnceInnerLocked)

### ANR 弹出流程
- onANRLocked 从 Native 层回溯到 Java 层的 IMS
- IMS 交由 AMS 处理 ANR
- AMS 处理 ANR
  - 在 ServiceThread 线程 dump ANR 信息
  - 在 UIThread 线程弹出 AppErrorDialog

## 参考资料
- http://gityuan.com/2017/01/01/input-anr/