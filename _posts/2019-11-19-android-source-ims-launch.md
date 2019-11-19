---
layout: article
title: "Android 系统架构 —— IMS 的启动"
key:  "Android 系统架构 —— IMS 的启动"
tags: AndroidFramework
aside:
  toc: true
---

## 前言
前面我们根据 SystemServer 的启动流程逐一分析了 AMS 和 PMS 的启动, 这里我们来看看 IMS 的启动流程
```java
public final class SystemServer {
    
    private void startOtherServices() {
        ......
        InputManagerService inputManager = null;
        try {
            ......
            // 1. 创建 IMS
            inputManager = new InputManagerService(context);
            ......
            ServiceManager.addService(Context.INPUT_SERVICE, inputManager,
                    /* allowIsolated= */ false, DUMP_FLAG_PRIORITY_CRITICAL);
            ......
            // 2. 启动 IMS
            inputManager.start();
        }
        ......        
    }
    
}
```
InputManagerService 的启动比 AMS 简单明了很多, 我们这里主要从这两个方面来分析 IMS 的启动
- IMS 的创建
- IMS 的启动

<!--more-->

## 一. IMS 的创建
```java
public class InputManagerService extends IInputManager.Stub
        implements Watchdog.Monitor {
            
    public InputManagerService(Context context) {
        this.mContext = context;
        // 1. 构建 DisplayThread 线程 Looper 的 handler
        this.mHandler = new InputManagerHandler(DisplayThread.get().getLooper());
        ......
        // 2. 初始化 Native 层
        mPtr = nativeInit(this, mContext, mHandler.getLooper().getQueue());
        ......
    }
    
    private static native long nativeInit(InputManagerService service,
            Context context, MessageQueue messageQueue);
            
}
```
IMS 的构造函数中主要做了如下的实现
- 构建 DisplayThread 线程 Looper 的 handler
- 调用 nativeInit, 进行 native 层的初始化操作

接下来我们一一分析

### 一) DisplayThread 的创建
```java
/**
 * Shared singleton foreground thread for the system.  This is a thread for
 * operations that affect what's on the display, which needs to have a minimum
 * of latency.  This thread should pretty much only be used by the WindowManager,
 * DisplayManager, and InputManager to perform quick operations in real time.
 */
public final class DisplayThread extends ServiceThread {
    private static DisplayThread sInstance;
    private static Handler sHandler;

    private DisplayThread() {
        // DisplayThread runs important stuff, but these are not as important as things running in
        // AnimationThread. Thus, set the priority to one lower.
        super("android.display", Process.THREAD_PRIORITY_DISPLAY + 1, false /*allowIo*/);
    }

    private static void ensureThreadLocked() {
        if (sInstance == null) {
            sInstance = new DisplayThread();
            sInstance.start();
            sInstance.getLooper().setTraceTag(Trace.TRACE_TAG_SYSTEM_SERVER);
            sHandler = new Handler(sInstance.getLooper());
        }
    }

    public static DisplayThread get() {
        synchronized (DisplayThread.class) {
            ensureThreadLocked();
            return sInstance;
        }
    }

    public static Handler getHandler() {
        synchronized (DisplayThread.class) {
            ensureThreadLocked();
            return sHandler;
        }
    }
}
```
从相关注释可以知道, DisplayThread 只会被 WindowManager, DisplayManager 和 InputManager 使用, 它在系统服务进程中是一个单例的存在

下面我们看看 native 层的初始化操作

### 二) native 初始化
InputManagerService 的 Native 文件定义在 [com_android_server_input_InputManagerService.cpp](http://androidxref.com/9.0.0_r3/xref/frameworks/base/services/core/jni/com_android_server_input_InputManagerService.cpp) 中, 我们看看它的实现
```
// frameworks/base/services/core/jni/com_android_server_input_InputManagerService.cpp
static jlong nativeInit(JNIEnv* env, jclass /* clazz */,
        jobject serviceObj, jobject contextObj, jobject messageQueueObj) {
    // 根据 Java 的 MessageQueue, 获取 Native 的 MessageQueue
    sp<MessageQueue> messageQueue = android_os_MessageQueue_getMessageQueue(env, messageQueueObj);
    if (messageQueue == NULL) {
        jniThrowRuntimeException(env, "MessageQueue is not initialized.");
        return 0;
    }
    // 创建一个 NativeInputManager 对象
    NativeInputManager* im = new NativeInputManager(contextObj, serviceObj,
            messageQueue->getLooper());
    // 增加一个强引用计数
    im->incStrong(0);
    // 返回句柄值
    return reinterpret_cast<jlong>(im);
}
```
这里可以看到 nativeInit 中, 首先获取了当前线程的 MessageQueue, 然后构建了一个与 Java 层对应的 **NativeInputManager** 对象, 我们看看它的构建流程

```
// frameworks/base/services/core/jni/com_android_server_input_InputManagerService.cpp
NativeInputManager::NativeInputManager(jobject contextObj,
        jobject serviceObj, const sp<Looper>& looper) :
        mLooper(looper), mInteractive(true) {
    JNIEnv* env = jniEnv();

    mContextObj = env->NewGlobalRef(contextObj);
    mServiceObj = env->NewGlobalRef(serviceObj);
    ......
    // 创建了一个 EventHub
    sp<EventHub> eventHub = new EventHub();
    // 创建了 InputManager 对象
    mInputManager = new InputManager(eventHub, this, this);
}
```
这里就比较重要了 NativeInputManager 只是一个与 Java 层交互的媒介, 其具体的实现是由 EventHub 和 InputManager 完成, 接下来看看它们实例化的过程

#### 1. [EventHub](http://androidxref.com/9.0.0_r3/xref/frameworks/native/services/inputflinger/EventHub.cpp) 的创建
```
// frameworks/native/services/inputflinger/EventHub.cpp
EventHub::EventHub(void) :
        mBuiltInKeyboardId(NO_BUILT_IN_KEYBOARD), mNextDeviceId(1), mControllerNumbers(),
        mOpeningDevices(0), mClosingDevices(0),
        mNeedToSendFinishedDeviceScan(false),
        mNeedToReopenDevices(false), mNeedToScanDevices(true),
        mPendingEventCount(0), mPendingEventIndex(0), mPendingINotify(false) {
    ......
    // 1. 创建了一个 Epoll, 用于监听 mINotifyFd 和 mWakeReadPipeFd 中的数据
    mEpollFd = epoll_create(EPOLL_SIZE_HINT);
    ......
    // 2. Epoll 监听 input 驱动设备文件的监听者
    // 2.1 创建输入驱动设备文件路径的监听者 mINotifyFd
    mINotifyFd = inotify_init()
    // DEVICE_PATH 为 "/dev/input" 表示监听输入驱动设备文件
    int result = inotify_add_watch(mINotifyFd, DEVICE_PATH, IN_DELETE | IN_CREATE);
    ......
    // 2.3 调用 epoll 文件的 ctl 系统调用, 监听 mINotifyFd 文件描述符
    result = epoll_ctl(mEpollFd, EPOLL_CTL_ADD, mINotifyFd, &eventItem);
    ......
    // 3. 创建管道
    int wakeFds[2];
    result = pipe(wakeFds);
    // 记录读端
    mWakeReadPipeFd = wakeFds[0];
    // 记录写端
    mWakeWritePipeFd = wakeFds[1];
    // 将管道的读写两端设置为非阻塞的模式
    result = fcntl(mWakeReadPipeFd, F_SETFL, O_NONBLOCK);
    result = fcntl(mWakeWritePipeFd, F_SETFL, O_NONBLOCK);
    ......
    // 4. Epoll 监听管道的读端 mWakeReadPipeFd 的数据变化
    eventItem.data.u32 = EPOLL_ID_WAKE;
    result = epoll_ctl(mEpollFd, EPOLL_CTL_ADD, mWakeReadPipeFd, &eventItem);
    ......
}
```
EventHub 中主要做了如下的操作
- **创建用于 IO 多路复用对象 Epoll**
- **Epoll 监听 mINotifyFd**
  - 创建输入驱动设备文件的监听者 mINotifyFd
  - Epoll 监听 mINotifyFd
- 创建非阻塞读写的管道
- **Epoll 监听管道的读端文件描述符**

在 EventHub 的构造中我们并没有看到对数据变更后的处理和分发, 我们继续往下探索, 看看 InputManager 的创建过程

#### 2. [InputManager](http://androidxref.com/9.0.0_r3/xref/frameworks/native/services/inputflinger/InputManager.cpp) 的创建
```
// frameworks/native/services/inputflinger/InputManager.cpp
InputManager::InputManager(
        const sp<EventHubInterface>& eventHub,
        const sp<InputReaderPolicyInterface>& readerPolicy,
        const sp<InputDispatcherPolicyInterface>& dispatcherPolicy) {
    // 1. 创建 InputDispatcher
    mDispatcher = new InputDispatcher(dispatcherPolicy);
    // 2. 创建 InputReader
    mReader = new InputReader(eventHub, readerPolicy, mDispatcher);
    // 3. 执行初始化操作
    initialize();
}

void InputManager::initialize() {
    // 3.1 创建读取输入流的线程
    mReaderThread = new InputReaderThread(mReader);
    // 3.2 创建分发输入流的线程
    mDispatcherThread = new InputDispatcherThread(mDispatcher);
}
```
非常有趣 InputManager 的构造函数中, 创建了四个对象, 分别如下
- InputDispatcher: 输入流的分发者
- InputReader: 输入流的读取者
- InputReaderThread: 输入流的分发线程
- InputDispatcherThread: 输入流的读取线程

InputManager 整个事件的读取和分发使用的是**生产者-消费者模型**, 接下来我们一一查看他们的创建过程

##### 1) [InputDispatcher](http://androidxref.com/9.0.0_r3/xref/frameworks/native/services/inputflinger/InputDispatcher.cpp)
```
// frameworks/native/services/inputflinger/InputDispatcher.cpp
InputDispatcher::InputDispatcher(const sp<InputDispatcherPolicyInterface>& policy) :
    // 将 NativeInputMangaer 保存到成员变量
    mPolicy(policy),
    mPendingEvent(NULL),
    mLastDropReason(DROP_REASON_NOT_DROPPED),
    mAppSwitchSawKeyDown(false), 
    mAppSwitchDueTime(LONG_LONG_MAX),
    mNextUnblockedEvent(NULL),
    mDispatchEnabled(false), 
    mDispatchFrozen(false), 
    mInputFilterEnabled(false),
    mInputTargetWaitCause(INPUT_TARGET_WAIT_CAUSE_NONE) {
    // 创建了一个 Looper, 先置为了 false
    mLooper = new Looper(false);
    mKeyRepeatState.lastKeyEntry = NULL;
    policy->getDispatcherConfiguration(&mConfig);
}
```

##### 2) [InputReader](http://androidxref.com/9.0.0_r3/xref/frameworks/native/services/inputflinger/InputReader.cpp)
```
// frameworks/native/services/inputflinger/InputReader.cpp
InputReader::InputReader(const sp<EventHubInterface>& eventHub,
        const sp<InputReaderPolicyInterface>& policy,
        const sp<InputListenerInterface>& listener) :
        mContext(this), 
        // 将 EventHub 保存到成员变量
        mEventHub(eventHub), 
        // policy 实现类为 NativeInputManager
        mPolicy(policy),
        mGlobalMetaState(0), 
        mGeneration(1),
        mDisableVirtualKeysTimeout(LLONG_MIN),
        mNextTimeout(LLONG_MAX),
        mConfigurationChangesToRefresh(0) {
    // 创建了一个事件队列, listener 为 InputDispatcher
    mQueuedListener = new QueuedInputListener(listener);
    { // acquire lock
        ......
    } // release lock
}
```
可以看到 InputReader 持有了一个事件队列 QueuedInputListener, 它用于存储接收到的事件, 并且通知 InputDispatcher 进行后续的分发操作

##### 3) InputReaderThread
```
// frameworks/native/services/inputflinger/InputReader.cpp
InputReaderThread::InputReaderThread(const sp<InputReaderInterface>& reader) :
        // 继承自一个 Native 的 Thread 封装类
        Thread(/*canCallJava*/ true),
        // 保存了 InputReader 到成员变量中
        mReader(reader) {}
```

##### 4) InputDispatcherThread
```
// frameworks/native/services/inputflinger/InputDispatcher.cpp
InputDispatcherThread::InputDispatcherThread(const sp<InputDispatcherInterface>& dispatcher) :
        // 继承自 Native 的 Thread 封装类
        Thread(/*canCallJava*/ true), 
        // 保存了 InputDispatcher 到成员变量
        mDispatcher(dispatcher) {
}
```

### 三) 回顾
IMS 的构造有两个部分
- 在 Java 层创建一个 **DisplayThread** 消息队列的 Handler
- 将 DisplayThread 的 **MessageQueue** 传入 Native 层, 执行 Native 层 IMS 的初始化操作
- Native 层创建了一个与 Java 层 InputManagerService 对应的 **NativeInputManager** 对象
  - 创建 **EventHub**, 监听输入设备驱动的事件流变化
    - **创建用于 IO 多路复用对象 Epoll**
    - **Epoll 监听 mINotifyFd 文件描述符, 进而监听输入设备集合(Android 手机支持连接多个输入设备)**
      - 创建输入驱动设备文件的监听者 mINotifyFd
      - Epoll 监听 mINotifyFd
    - **创建非阻塞读写的管道**
    - **Epoll 监听管道的读端文件描述符**
  - **创建 InputManager 来 读取和分发 事件序列**
    - 创建线程 InputReaderThread
      - 使用 InputReader 读取输入设备产生事件到 QueuedInputListener
    - 创建线程 InputDispatcherThread
      - 使用 InputDispatcher 分发数据

好的, 现在 IMS 的创建流程就结束了, 下面看看 IMS 的启动过程

## 二. IMS 的启动
```java
public class InputManagerService extends IInputManager.Stub
        implements Watchdog.Monitor {
    
    public void start() {
        // 调用了 Native 的 start 函数
        nativeStart(mPtr);
        // 注册一些事件的观察者
        registerPointerSpeedSettingObserver();
        registerShowTouchesSettingObserver();
        registerAccessibilityLargePointerSettingObserver();
        // 注册广播
        mContext.registerReceiver(new BroadcastReceiver() {
            @Override
            public void onReceive(Context context, Intent intent) {
                // 更新事件
                updatePointerSpeedFromSettings();
                updateShowTouchesFromSettings();
                updateAccessibilityLargePointerFromSettings();
            }
        }, new IntentFilter(Intent.ACTION_USER_SWITCHED), null, mHandler);
        // 更新触摸点的速度
        updatePointerSpeedFromSettings();
        // 更新屏幕上的触摸点
        updateShowTouchesFromSettings();
        updateAccessibilityLargePointerFromSettings();
    }

            
}
```
Java 的 IMS 的 start 中调用 nativeStart, 然后注册了一些事件的观察者, 更新了相关数据, 最重要的还是 native 层的 nativeStart 的实现, 下面我们就去 native 层探索一下, NativeInputManager 的启动做了哪些操作

```
// frameworks/base/services/core/jni/com_android_server_input_InputManagerService.cpp
static void nativeStart(JNIEnv* env, jclass /* clazz */, jlong ptr) {
    NativeInputManager* im = reinterpret_cast<NativeInputManager*>(ptr);
    // 直接调用了 InputManager 的 start 函数
    status_t result = im->getInputManager()->start();
    ......
}

// frameworks/native/services/inputflinger/InputManager.cpp
status_t InputManager::start() {
    // 启动 InputDispatcherThread 线程
    status_t result = mDispatcherThread->run("InputDispatcher", PRIORITY_URGENT_DISPLAY);
    ......
    // 启动 InputReaderThread 线程
    result = mReaderThread->run("InputReader", PRIORITY_URGENT_DISPLAY);
    ......
    return OK;
}
```
好的, Native 层的 start 操作, 启动了我们上面创建的 InputDispatcherThread 和 InputReaderThread 线程

接下来我们分别看看他们启动后执行了哪些操作

### 一) InputReaderThread 线程的启动
```
// frameworks/native/services/inputflinger/InputReader.cpp
bool InputReaderThread::threadLoop() {
    // 调用了 InputReader 的 loopOnce 读取输入流
    mReader->loopOnce();
    return true;
}
```
InputReaderThread 启动之后, 会不断的调用 threadLoop 执行事件的读取

每次调用均会触发 InputReader 的 loopOnce 执行一次读取输入流的操作, 这是整个输入系统的核心之一

下面我们看看它的具体实现
```
// frameworks/native/services/inputflinger/InputReader.cpp
void InputReader::loopOnce() {
    ......
    int32_t timeoutMillis;
    ......
    // 1. 调用了 EventHub 的 getEvents, 将输入事件写入到 mEventBuffer 中
    // EVENT_BUFFER_SIZE 为 256
    size_t count = mEventHub->getEvents(timeoutMillis, mEventBuffer, EVENT_BUFFER_SIZE);

    { // acquire lock
        AutoMutex _l(mLock);
        ......
        if (count) {
            // 2. 处理接收到的事件
            processEventsLocked(mEventBuffer, count);
        }
        ......
    } // release lock
    ......
    // 3. 调用生产队列的 flush 函数
    mQueuedListener->flush();
}
```
InputReader 的 loopOnce 中的操作如下
- 调用 EventHub 的 getEvents 来获取输入事件流, 写入 mEventBuffer 数组
- 调用 processEventsLocked 处理事件
- 调用生产队列的 flush 函数

接下来我们一一分析

#### 1. Eventhub 获取事件输入流
```
// // frameworks/native/services/inputflinger/EventHub.h
// 描述 Epoll 最多能够接收到的事件为 16 个
static const int EPOLL_MAX_EVENTS = 16;
// 描述 Epoll 接收到的事件集合
struct epoll_event mPendingEventItems[EPOLL_MAX_EVENTS];

size_t mPendingEventCount;
size_t mPendingEventIndex;

// frameworks/native/services/inputflinger/EventHub.cpp
size_t EventHub::getEvents(int timeoutMillis, RawEvent* buffer, size_t bufferSize) {
    ......
    AutoMutex _l(mLock);
    // 创建一个 input_event 的数组, 最大能够接收 256 个事件
    struct input_event readBuffer[bufferSize];
    // 描述一个存储事件数组的首地址
    RawEvent* event = buffer;
    // 记录需要接收的事件数量
    size_t capacity = bufferSize;
    // 维护一个死循环
    for (;;) {
        nsecs_t now = systemTime(SYSTEM_TIME_MONOTONIC);
        
        ...... // 处理多个输入设备的打开和关闭 
        
        // 获取下一个输入事件
        bool deviceChanged = false;
        
        // 遍历 mPendingEventItems 暂存的待分发事件
        while (mPendingEventIndex < mPendingEventCount) {
        
            // 1. 取出 epoll 的一个事件 epoll_event
            const struct epoll_event& eventItem = mPendingEventItems[mPendingEventIndex++];
            
            ......
            
            // 2. 获取 epoll 事件对应的输入设备 Device
            ssize_t deviceIndex = mDevices.indexOfKey(eventItem.data.u32);
            ......
            Device* device = mDevices.valueAt(deviceIndex);
            if (eventItem.events & EPOLLIN) {
                // 3. 从该设备中读取事件序列, 写入到 readBuffer
                int32_t readSize = read(device->fd, readBuffer, sizeof(struct input_event) * capacity);
                
                if (readSize == 0 || (readSize < 0 && errno == ENODEV)) {
                    ......
                } else if (readSize < 0) {
                    ......
                } else if ((readSize % sizeof(struct input_event)) != 0) {
                    ......
                } else {
                    int32_t deviceId = device->id == mBuiltInKeyboardId ? 0 : device->id;
                    size_t count = size_t(readSize) / sizeof(struct input_event);
                    // 4. 遍历 readBuffer 读取到的输入事件序列
                    for (size_t i = 0; i < count; i++) {
                        // 4.1 获取事件描述 input_event
                        struct input_event& iev = readBuffer[i];
                        ......
                        // 4.2 将 input_event 中的数据注入到 RawEvent 数组中
                        event->when = nsecs_t(iev.time.tv_sec) * 1000000000LL
                                + nsecs_t(iev.time.tv_usec) * 1000LL;
                        ......
                        event->deviceId = deviceId;
                        event->type = iev.type;
                        event->code = iev.code;
                        event->value = iev.value;
                        // 数组移动到下一个位置
                        event += 1;
                        capacity -= 1;
                    }
                    // 5. capacity 为 0, 表示任务完成了
                    if (capacity == 0) {
                        // 取走了一个事件序列, 将 mPendingEventIndex - 1
                        mPendingEventIndex -= 1;
                        break;
                    }
                }
            } else if (eventItem.events & EPOLLHUP) {
                ......
            } else {
                ......
            }
        }
        ......
        // 事件取尽
        mPendingEventIndex = 0;
        ......
        // 使用 epoll_wait 等待它所监听的文件描述符中事件的到来
        int pollResult = epoll_wait(mEpollFd, mPendingEventItems, EPOLL_MAX_EVENTS, timeoutMillis);
        ......
        // 在 timeoutMillis 内没有设备产生输入事件, 则退出 looper
        if (pollResult == 0) {
            mPendingEventCount = 0;
            break;
        }
        if (pollResult < 0) {
            .....// 出现了额一些异常情况
        } else {
            // 记录事件的数量
            mPendingEventCount = size_t(pollResult);
        }
    }
    // 返回所读取的事件的个数
    return event - buffer;
}
```
EventHub 的 getEvents 函数中维护了一个 looper, 内部操作如下
- 若 mPendingEventItems 存在产生事件的输入设备, 则执行遍历操作
  - 从 mPendingEventItems 中取出 epoll 的一个事件 epoll_event
  - 从 epoll_event 对应的输入设备中读取事件序列, 写入到 readBuffer
  - 遍历 readBuffer 读取到的输入事件序列
    - 获取事件描述 input_event
    - 将 input_event 中的数据注入到 RawEvent 数组中
  - 直至取满 256 个事件, 结束本次 getEvents 任务
- 若 mPendingEventItems 不存在产生事件的输入设备
  - 调用 epoll_wait 阻塞式读取 mINotifyFd 产生的事件到 mPendingEventItems 中
    - 若操作超时, 则退出 looper
  - 若存在输入设备产生了事件, 则重复上面的 looper 从 mPendingEventItems 中获取事件序列

**EventHub 的 getEvents 执行结束之后, 其 InputReader 的 mEventBuffer 中就填充了输入事件序列了, 我们回到 InputReader::loopOnce 中, 看看它是如何处理接收到的事件序列的**

#### 2. processEventsLocked 处理事件
```
// frameworks/native/services/inputflinger/InputReader.cpp
void InputReader::processEventsLocked(const RawEvent* rawEvents, size_t count) {
    // 遍历读取到的事件集合
    for (const RawEvent* rawEvent = rawEvents; count;) {
        int32_t type = rawEvent->type;
        size_t batchSize = 1;
        // 1. 处理输入事件
        if (type < EventHubInterface::FIRST_SYNTHETIC_EVENT) {
            // 1.1 对同一个设备的事件打包
            int32_t deviceId = rawEvent->deviceId;
            while (batchSize < count) {
                if (rawEvent[batchSize].type >= EventHubInterface::FIRST_SYNTHETIC_EVENT
                        || rawEvent[batchSize].deviceId != deviceId) {
                    break;
                }
                batchSize += 1;
            }
            ......
            // 1.2 处理一个设备产生的事件序列
            processEventsForDeviceLocked(deviceId, rawEvent, batchSize);
        } else {
            ......
            // 2. 处理输入设备发生变更的事件
        }
        count -= batchSize;
        rawEvent += batchSize;
    }
}
```
InputReader::processEventsLocked 中主要处理两个方面的事务
- 处理输入事件
  - 将同一个设备的输入事件打包, 发送给 processEventsForDeviceLocked 函数处理
- 处理设备变更事件

这里我们看看他是如何处理一个设备的事件序列的

```
// frameworks/native/services/inputflinger/InputReader.cpp
void InputReader::processEventsForDeviceLocked(int32_t deviceId,
        const RawEvent* rawEvents, size_t count) {
    // 获取设备索引
    ssize_t deviceIndex = mDevices.indexOfKey(deviceId);
    ......
    // 获取输入设备的描述
    InputDevice* device = mDevices.valueAt(deviceIndex);
    // 调用 InputDevice 的 process 函数, 来处理接收到的事件
    device->process(rawEvents, count);
}
```
接下来我们看看 InputDevice 如何处理接收到的事件序列

##### InputDevice 处理事件序列
```
// frameworks/native/services/inputflinger/InputReader.cpp
void InputDevice::process(const RawEvent* rawEvents, size_t count) {
    size_t numMappers = mMappers.size();
    for (const RawEvent* rawEvent = rawEvents; count != 0; rawEvent++) {
        if (mDropUntilNextSync) {
            ......
        } else if (rawEvent->type == EV_SYN && rawEvent->code == SYN_DROPPED) {
            ......
        } else {
            // 调动 InputMapper 处理事件
            for (size_t i = 0; i < numMappers; i++) {
                InputMapper* mapper = mMappers[i];
                mapper->process(rawEvent);
            }
        }
        --count;
    }
}
```
可以看到这里它调用了 InputMapper 来处理接收到的输入事件, 从 InputManager 的定义我们就知道, 它主要负责处理事件的映射, 将底层输出的事件映射成上层的具体描述

我们这里以 KeyboardInputMapper 为例, 看看将 RawEvent 转化为键盘输入事件的过程
```
// frameworks/native/services/inputflinger/InputReader.cpp
void KeyboardInputMapper::process(const RawEvent* rawEvent) {
    switch (rawEvent->type) {
    case EV_KEY: {
        ......
        if (isKeyboardOrGamepadKey(scanCode)) {
            // 调用了 processKey 执行事件的映射
            processKey(rawEvent->when, rawEvent->value != 0, scanCode, usageCode);
        }
        break;
    }
    ......
    }
}

void KeyboardInputMapper::processKey(nsecs_t when, bool down, int32_t scanCode,
        int32_t usageCode) {
    ......
    // 添加一个 down 事件
    if (down) {
        ssize_t keyDownIndex = findKeyDown(scanCode);
        if (keyDownIndex >= 0) {
            // key repeat, be sure to use same keycode as before in case of rotation
            keyCode = mKeyDowns.itemAt(keyDownIndex).keyCode;
        } else {
            ......
            // 压入栈顶
            mKeyDowns.push();
            KeyDown& keyDown = mKeyDowns.editTop();
            keyDown.keyCode = keyCode;
            keyDown.scanCode = scanCode;
        }
        // 记录按下时间
        mDownTime = when;
    }
    // 移除按下事件
    else {
        ssize_t keyDownIndex = findKeyDown(scanCode);
        if (keyDownIndex >= 0) {
            // 按键抬起, 移除按下的操作
            keyCode = mKeyDowns.itemAt(keyDownIndex).keyCode;
            mKeyDowns.removeAt(size_t(keyDownIndex));
        }
        ......
    }
    ......
    // 构建该事件的参数描述
    NotifyKeyArgs args(when, getDeviceId(), mSource, policyFlags,
            down ? AKEY_EVENT_ACTION_DOWN : AKEY_EVENT_ACTION_UP,
            AKEY_EVENT_FLAG_FROM_SYSTEM, keyCode, scanCode, keyMetaState, downTime);
    // 调用 mQueuedListener 的 notifyKey
    getListener()->notifyKey(&args);
}
```
KeyboardInputMapper 的 process 函数的主要操作如下
- 将一个 rawEvent 事件转化为 down 事件保存在 mKeyDowns 中
- 构建事件参数描述 NotifyKeyArgs 
- 调用 mQueuedListener 的 notifyKey 通知一个新事件的到来

mQueuedListener 的 notifyKey 操作如下
```
// frameworks/native/services/inputflinger/InputListener.cpp 
void QueuedInputListener::notifyKey(const NotifyKeyArgs* args) {
    // 添加到事件的参数中
    mArgsQueue.push(new NotifyKeyArgs(*args));
}
```
好的, 如此一来一个映射好的事件便缓存到 QueuedInputListener 的 mArgsQueue 中了, 下面我们回到  InputReader::loopOnce 中, 看看它是 QueuedInputListener.flush 做了什么操作

#### 3. QueuedInputListener.flush
```
// frameworks/native/services/inputflinger/InputListener.cpp
void QueuedInputListener::flush() {
    size_t count = mArgsQueue.size();
    for (size_t i = 0; i < count; i++) {
        NotifyArgs* args = mArgsQueue[i];
        // 调用了 NotifyArgs 的 notify
        args->notify(mInnerListener);
        delete args;
    }
    mArgsQueue.clear();
}

void NotifyKeyArgs::notify(const sp<InputListenerInterface>& listener) const {
    // 调用了 InputDispatcher 消费事件
    listener->notifyKey(this);
}
```
可以看到 QueuedInputListener::flush 操作, 遍历了队列中的事件, 最终会调用 InputDispatcher 来消费这个事件

##### InputDispatcher.notifyKey
```
// frameworks/native/services/inputflinger/InputDispatcher.cpp
void InputDispatcher::notifyKey(const NotifyKeyArgs* args) {
    .......
    bool needWake;
    { // acquire lock
        mLock.lock();
        ......
        int32_t repeatCount = 0;
        // 1. 构建一个 KeyEntry 对象
        KeyEntry* newEntry = new KeyEntry(args->eventTime,
                args->deviceId, args->source, policyFlags,
                args->action, flags, keyCode, args->scanCode,
                metaState, repeatCount, args->downTime);
        // 2. 将这个对象入队列
        needWake = enqueueInboundEventLocked(newEntry);
        mLock.unlock();
    } // release lock
    // 3. 唤醒 InputDispatcherThread 的睡眠
    if (needWake) {
        mLooper->wake();
    }
}


bool InputDispatcher::enqueueInboundEventLocked(EventEntry* entry) {
    bool needWake = mInboundQueue.isEmpty();
    // 2.1 将 EventEntry 添加到 mInboundQueue 队列
    mInboundQueue.enqueueAtTail(entry);
    ......
    return needWake;
}
```
可以看到 InputDispatcher.notifyKey 主要操作是构建了一个 KeyEntry 对象, 然后将它添加到 mInboundQueue 中缓存, 最后通过 mLooper.wake 来唤醒阻塞的 InputDispatcher 的线程

因为 InputDispatcherThread 的启动我们还没有分析, 等后面分析之后整个流程就更加清晰了, 下面做个简单的回顾

#### 回顾
整个事件读取操作还是非常漫长的, 这里对其进行一个简单的梳理

![InputReaderThread 启动过程](https://i.loli.net/2019/11/19/aAsoZhDIuNMTqCi.png)

InputReaderThread 启动之后会不断的调用 InputReader 的 loopOnce 来读取输入事件, 其处理事务如下
- 调用 EventHub 的 getEvents 来获取输入事件流, 写入 mEventBuffer 数组
  - 使用 epoll 阻塞式的获取 epoll_event 事件
  - 根据 epoll_event 获取具体的输入设备
  - 从输入设备中读取事件序列 input_event 存到 RawEvent 数组中并返回给上层
- 调用 processEventsLocked 处理事件
  - 调用了 InputDevice 的 process 处理该设备产生的事件
  - 调用 InputMapper, 将抽象的 RawEvent 事件映射成具体的事件类型
    - 映射完毕之后, 使用 NotifyArgs 描述
  - 将 NotifyArgs 投入 QueuedInputListener 队列
- 调用 QueuedInputListener 的 flush 函数
  - 通知 InputDispatcher.notify 消费事件
  - 将 NotifyArgs 构建成 KeyEntry 添加到 InputDispatcher 的 mInboundQueue 队列
  - 唤醒 InputDispatcherThread 执行事件分发

整个事件的转变过程如下所示
```
epoll_event -> input_event -> RawEvent -> NotifyArgs -> KeyEntry
```

事件的生产线程的工作机制我们已经理清了, 下面我们探究一下事件消费线程 InputDispatcherThread 的启动

### 二) InputDispatcherThread 线程的启动
```
// frameworks/native/services/inputflinger/InputDispatcher.cpp
bool InputDispatcherThread::threadLoop() {
    mDispatcher->dispatchOnce();
    return true;
}
```
InputDispatcherThread 启动之后, 会调用 InputDispatcher 的 dispatchOnce来执行一次事件分发, 这是整个输入系统的核心之一

下面我们就看看它的具体实现
```
// frameworks/native/services/inputflinger/InputDispatcher.cpp
void InputDispatcher::dispatchOnce() {
    nsecs_t nextWakeupTime = LONG_LONG_MAX;
    { // acquire lock
        .......
        // 1. 若存在等待执行的指令, 则调用 dispatchOnceInnerLocked 执行分发任务
        if (!haveCommandsLocked()) {
            // 1.2 调用 dispatchOnceInnerLocked 执行分发任务
            dispatchOnceInnerLocked(&nextWakeupTime);
        }
        ......
    } // release lock
    // 2. 通过 Looper 的 pollOnce 函数执行睡眠操作
    nsecs_t currentTime = now();
    int timeoutMillis = toMillisecondTimeoutDelay(currentTime, nextWakeupTime);
    mLooper->pollOnce(timeoutMillis);
}

bool InputDispatcher::haveCommandsLocked() const {
    // 1.1 指令队列不为空, 则说明需要执行事件分发
    return !mCommandQueue.isEmpty();
}
```
从 InputDispatcher::dispatchOnce 中可知它内部主要有两步操作
- 若存在指令, 则调用 dispatchOnceInnerLocked 执行事件分发操作
- 计算需要睡眠的时间, 通过 Looper.pollOnce 执行睡眠操作

Looper.pollOnce 最终会睡眠在 epoll_wait 上, 发生超时或者有数据写入时会被唤醒, **也就是说上面我们分析的 InputDispatcher.notify 函数, 最终会唤醒阻塞的 InputDispatcherThread 继续进行事件分发**

下面我们看看 dispatchOnceInnerLocked 的操作
```
// frameworks/native/services/inputflinger/InputDispatcher.cpp
void InputDispatcher::dispatchOnceInnerLocked(nsecs_t* nextWakeupTime) {
    // 记录当前时间
    nsecs_t currentTime = now();
    ......

    // 1. 尝试从 mInboundQueue 中获取事件, 进行分发
    if (!mPendingEvent) {
        if (mInboundQueue.isEmpty()) {
            ......
            // 没有要分发的事件, 直接返回
            if (!mPendingEvent) {
                return;
            }
        } else {
            // 从 mInboundQueue 中获取事件 KeyEntry 保存到 mPendingEvent
            mPendingEvent = mInboundQueue.dequeueAtHead();
            traceInboundQueueLengthLocked();
        }
        ......
        // 取到了一个新的事件, 则重置上一次 ANR 超时信息
        resetANRTimeoutsLocked();
    }
    ......
    // 2. 执行事件分发
    switch (mPendingEvent->type) {
    ......
    // 根据事件类型执行不同的事件分发操作
    case EventEntry::TYPE_KEY: {
        .......
        // 执行事件 KEY 事件的分发
        done = dispatchKeyLocked(currentTime, typedEntry, &dropReason, nextWakeupTime);
        break;
    }
    ......
    // 3. 处理后续事务
    if (done) {
        ......
        // 释放已分发的事件
        releasePendingEventLocked();
        // 分发成功之后, 将 nextWakeupTime 置为 LONG_LONG_MIN, 表示不进行睡眠, 立即进行后续的事件分发
        *nextWakeupTime = LONG_LONG_MIN;  // force next poll to wake up immediately
    }
}
```
整个 dispatchOnceInnerLocked 的流程比较清晰
- 从 mInboundQueue 中获取队首事件 KeyEntry 保存到 mPendingEvent 进行分发处理
  - 重置 ANR 的超时信息 
- 根据 mPendingEvent 的事件类型, 执行对应的事件分发操作
  - KEY 事件通过 **dispatchKeyLocked** 进行分发处理
- 分发完成之后调用 releasePendingEventLocked 释放已分发的事件

这里我们主要看看 dispatchKeyLocked 的分发流程
```
// frameworks/native/services/inputflinger/InputDispatcher.cpp
bool InputDispatcher::dispatchKeyLocked(nsecs_t currentTime, KeyEntry* entry,
        DropReason* dropReason, nsecs_t* nextWakeupTime) {
    // 1. 找寻当前的事件响应者, 保存到 inputTargets 中
    Vector<InputTarget> inputTargets;
    bool conflictingPointerActions = false;
    int32_t injectionResult;
    if (isPointerEvent) {
        injectionResult = findTouchedWindowTargetsLocked(currentTime,
                entry, inputTargets, nextWakeupTime, &conflictingPointerActions);
    } else {
        injectionResult = findFocusedWindowTargetsLocked(currentTime,
                entry, inputTargets, nextWakeupTime);
    }
    // 若没有焦点, 则返回 false
    if (injectionResult == INPUT_EVENT_INJECTION_PENDING) {
        return false;
    }
    ......
    // 将这个事件分发给响应者集合
    dispatchEventLocked(currentTime, entry, inputTargets);
    return true;
}

void InputDispatcher::dispatchEventLocked(nsecs_t currentTime,
        EventEntry* eventEntry, const Vector<InputTarget>& inputTargets) {
    ......
    // 遍历 inputTargets 序列, 将事件分发给响应者
    for (size_t i = 0; i < inputTargets.size(); i++) {
        const InputTarget& inputTarget = inputTargets.itemAt(i);
        ssize_t connectionIndex = getConnectionIndexLocked(inputTarget.inputChannel);
        if (connectionIndex >= 0) {
            // 获取响应者连接
            sp<Connection> connection = mConnectionsByFd.valueAt(connectionIndex);
            // 2. 调用 prepareDispatchCycleLocked 分发给响应者连接
            prepareDispatchCycleLocked(currentTime, connection, eventEntry, &inputTarget);
        }
    }
}

void InputDispatcher::prepareDispatchCycleLocked(nsecs_t currentTime,
        const sp<Connection>& connection, EventEntry* eventEntry, const InputTarget* inputTarget) {
    .....
    //  Not splitting.  Enqueue dispatch entries for the event as is.
    enqueueDispatchEntriesLocked(currentTime, connection, eventEntry, inputTarget);
}
```
dispatchKeyLocked 函数中主要进行了如下的操作
- 通过 findFocusedWindowTargetsLocked 找寻当前的焦点 Window 作为事件的响应者, 添加到 inputTargets 中
- 遍历事件响应者集合 inputTargets
  - 调用 enqueueDispatchEntriesLocked 将事件发送给响应者

我们先看看查找响应窗体集合的过程

### 一) 找寻事件响应窗体
```
// frameworks/native/services/inputflinger/InputDispatcher.cpp
int32_t InputDispatcher::findFocusedWindowTargetsLocked(nsecs_t currentTime,
        const EventEntry* entry, Vector<InputTarget>& inputTargets, nsecs_t* nextWakeupTime) {
    int32_t injectionResult;
    std::string reason;
    ......
    // Success!  Output targets.
    injectionResult = INPUT_EVENT_INJECTION_SUCCEEDED;
    // 将 mFocusedWindowHandle 作为响应窗体添加到 inputTargets 中
    addWindowTargetLocked(mFocusedWindowHandle,
            InputTarget::FLAG_FOREGROUND | InputTarget::FLAG_DISPATCH_AS_IS, BitSet32(0),
            inputTargets);
    ......
    return injectionResult;
}
```
这里调用了 addWindowTargetLocked 将 mFocusedWindowHandle 焦点窗体添加到了 inputTargets 中

关于焦点窗体的查找过程, 我们到后面的章节中进行分析, 这里先看看 addWindowTargetLocked 的操作
```
// frameworks/native/services/inputflinger/InputDispatcher.cpp
void InputDispatcher::addWindowTargetLocked(const sp<InputWindowHandle>& windowHandle,
        int32_t targetFlags, BitSet32 pointerIds, Vector<InputTarget>& inputTargets) {
    inputTargets.push();
    // 获取窗体信息
    const InputWindowInfo* windowInfo = windowHandle->getInfo();
    // 将数据注入 InputTarget
    InputTarget& target = inputTargets.editTop();
    target.inputChannel = windowInfo->inputChannel;
    target.flags = targetFlags;
    target.xOffset = - windowInfo->frameLeft;
    target.yOffset = - windowInfo->frameTop;
    target.scaleFactor = windowInfo->scaleFactor;
    target.pointerIds = pointerIds;
}
```
很简单, 首先获取了窗体信息, 然后将这个 InputWindowInfo 构建成 InputTarget 存入 inputTargets

下面我们看看找到 InputTarget 之后如何分发后续的事件

### 二) 将事件发送给响应窗体
```
// frameworks/native/services/inputflinger/InputDispatcher.cpp
void InputDispatcher::enqueueDispatchEntriesLocked(nsecs_t currentTime,
        const sp<Connection>& connection, EventEntry* eventEntry, const InputTarget* inputTarget) {
    bool wasEmpty = connection->outboundQueue.isEmpty();
    // 根据 dispatchMode 来分别执行 DispatchEntry 事件加入队列的操作
    enqueueDispatchEntryLocked(connection, eventEntry, inputTarget,
            InputTarget::FLAG_DISPATCH_AS_HOVER_EXIT);
    enqueueDispatchEntryLocked(connection, eventEntry, inputTarget,
            InputTarget::FLAG_DISPATCH_AS_OUTSIDE);
    enqueueDispatchEntryLocked(connection, eventEntry, inputTarget,
            InputTarget::FLAG_DISPATCH_AS_HOVER_ENTER);
    enqueueDispatchEntryLocked(connection, eventEntry, inputTarget,
            InputTarget::FLAG_DISPATCH_AS_IS);
    enqueueDispatchEntryLocked(connection, eventEntry, inputTarget,
            InputTarget::FLAG_DISPATCH_AS_SLIPPERY_EXIT);
    enqueueDispatchEntryLocked(connection, eventEntry, inputTarget,
            InputTarget::FLAG_DISPATCH_AS_SLIPPERY_ENTER);

    // 若入队列之后, Connection 的 outboundQueue 不为空了, 则执行后续的事件分发
    if (wasEmpty && !connection->outboundQueue.isEmpty()) {
        startDispatchCycleLocked(currentTime, connection);
    }
}
```
将事件发送给响应者的过程如下
- 调用 enqueueDispatchEntryLocked 将事件添加到响应者的 outboundQueue 队列
- 调用 startDispatchCycleLocked 分发整个事件

#### 1. 添加到 Connection.outboundQueue 队列
```
void InputDispatcher::enqueueDispatchEntryLocked(
        const sp<Connection>& connection, EventEntry* eventEntry, const InputTarget* inputTarget,
        int32_t dispatchMode) {
    ......
    // 1. 将 EventEntry(KeyEvent) 构建成 DispatchEntry
    DispatchEntry* dispatchEntry = new DispatchEntry(eventEntry, // increments ref
            inputTargetFlags, inputTarget->xOffset, inputTarget->yOffset,
            inputTarget->scaleFactor);
    ......
    // 2. 添加到响应者连接队列的尾部
    connection->outboundQueue.enqueueAtTail(dispatchEntry);
    ......
}
```
可以看到在添加到 Connection 的 outboundQueue 队列之前, 首先将 KeyEvent 构建成了 DispatchEntry 对象, 然后再添加到 outboundQueue 队列中
- 这里又进行了一次事件的转换, 将 KeyEvent 转换成 DispatchEntry

#### 2. 执行响应者的事件分发
```
void InputDispatcher::startDispatchCycleLocked(nsecs_t currentTime,
        const sp<Connection>& connection) {
    while (connection->status == Connection::STATUS_NORMAL
            && !connection->outboundQueue.isEmpty()) {
        DispatchEntry* dispatchEntry = connection->outboundQueue.head;
        dispatchEntry->deliveryTime = currentTime;
        // Publish the event.
        status_t status;
        EventEntry* eventEntry = dispatchEntry->eventEntry;
        switch (eventEntry->type) {
        case EventEntry::TYPE_KEY: {
            KeyEntry* keyEntry = static_cast<KeyEntry*>(eventEntry);
            // 1. 发布这个事件
            status = connection->inputPublisher.publishKeyEvent(dispatchEntry->seq,
                    keyEntry->deviceId, keyEntry->source,
                    dispatchEntry->resolvedAction, dispatchEntry->resolvedFlags,
                    keyEntry->keyCode, keyEntry->scanCode,
                    keyEntry->metaState, keyEntry->repeatCount, keyEntry->downTime,
                    keyEntry->eventTime);
            break;
        }
        ......
        default:
            return;
        }
        ......
        // 2. 从 outboundQueue 队列中移除
        connection->outboundQueue.dequeue(dispatchEntry);
        ......
        // 3. 放入 connection 的 waitQueue 中
        connection->waitQueue.enqueueAtTail(dispatchEntry);
        ......
    }
}
```
可以看到 startDispatchCycleLocked 函数最终会调用 connection->inputPublisher.publishKeyEvent 来发布事件
```
// frameworks/native/libs/input/InputTransport.cpp
status_t InputPublisher::publishKeyEvent(
        uint32_t seq,
        int32_t deviceId,
        int32_t source,
        int32_t action,
        int32_t flags,
        int32_t keyCode,
        int32_t scanCode,
        int32_t metaState,
        int32_t repeatCount,
        nsecs_t downTime,
        nsecs_t eventTime) {
    ......
    // 创建了一个 InputMessage
    InputMessage msg;
    msg.header.type = InputMessage::TYPE_KEY;
    msg.body.key.seq = seq;
    msg.body.key.deviceId = deviceId;
    msg.body.key.source = source;
    msg.body.key.action = action;
    msg.body.key.flags = flags;
    msg.body.key.keyCode = keyCode;
    msg.body.key.scanCode = scanCode;
    msg.body.key.metaState = metaState;
    msg.body.key.repeatCount = repeatCount;
    msg.body.key.downTime = downTime;
    msg.body.key.eventTime = eventTime;
    // 通过 InputChannel 发布这个输入信息
    return mChannel->sendMessage(&msg);
}
```
可以看到 InputPublisher 的 publishKeyEvent 函数汇总, 最终调用了 mChannel 的 sendMessage 来发布事件

**这个 mChannel 是一个 InputChannel 对象, 它是将输入事件流传递到应用进程窗体的关键所在**, 其创建的流程和 sendMessage 操作我们到后面章节再进行分析

#### 回顾

![InputDispatcherThread 启动过程](https://i.loli.net/2019/11/19/CLIpQHBuXSdortj.png)

InputDispatcherThread 启动之后会不断的调用 InputDispatcher 的 dispatchOnce 来执行事件分发, 其处理流程如下
- 从 mInboundQueue 中获取队首事件 KeyEntry 保存到 mPendingEvent 进行分发处理
- 找寻事件的响应窗体保存到 inputTargets 集合
  - 找寻焦点窗体 mFocusedWindowHandle, 将它的数据注入 inputTarget
- 遍历 inputTargets 集合进行事件分发
  - 获取响应者连接 Connection 
  - 将 KeyEvent 构建成了 DispatchEntry 对象, 添加到 Connection 的 outboundQueue 队列中
  - Connection 内部的 InputPublisher 获取队列中的元素进行发布
  - 将 KeyEvent 构建成 InputMessage, 通过 InputChannel.sendMessage 发布

```
// 事件转换流程
KeyEntry -> DispatchEntry -> InputMessage
```


## 总结
IMS 的启动主要分为两个部分, 分别是构建和启动

### 创建流程
- 在 Java 层创建一个 **DisplayThread** 消息队列的 Handler
- 将 DisplayThread 的 **MessageQueue** 传入 Native 层, 执行 Native 层 IMS 的初始化操作
- Native 层创建了一个与 Java 层 InputManagerService 对应的 **NativeInputManager** 对象
  - 创建 **EventHub**, 监听输入设备驱动的事件流变化
    - **创建用于 IO 多路复用对象 Epoll**
    - **Epoll 监听 mINotifyFd 文件描述符, 进而监听输入设备集合(Android 手机支持连接多个输入设备)**
      - 创建输入驱动设备文件的监听者 mINotifyFd
      - Epoll 监听 mINotifyFd
    - **创建非阻塞读写的管道**
    - **Epoll 监听管道的读端文件描述符**
  - **创建 InputManager 来 读取和分发 事件序列**
    - 创建线程 InputReaderThread
      - 使用 InputReader 读取输入设备产生事件到 QueuedInputListener
    - 创建线程 InputDispatcherThread
      - 使用 InputDispatcher 分发数据

### 启动流程

![IMS 启动流程](https://i.loli.net/2019/11/19/yldG7qvI4H6Wf5R.png)

#### InputReaderThread 的启动
InputReaderThread 启动之后会不断的调用 InputReader 的 loopOnce 来读取输入事件, 其处理事务如下
- 调用 EventHub 的 getEvents 来获取输入事件流, 写入 mEventBuffer 数组
  - 使用 epoll 阻塞式的获取 epoll_event 事件
  - 根据 epoll_event 获取具体的输入设备
  - 从输入设备中读取事件序列 input_event 存到 RawEvent 数组中并返回给上层
- 调用 processEventsLocked 处理事件
  - 调用了 InputDevice 的 process 处理该设备产生的事件
  - 调用 InputMapper, 将抽象的 RawEvent 事件映射成具体的事件类型
    - 映射完毕之后, 使用 NotifyArgs 描述
  - 将 NotifyArgs 投入 QueuedInputListener 队列
- 调用 QueuedInputListener 的 flush 函数
  - 通知 InputDispatcher.notify 消费事件
  - 将 NotifyArgs 构建成 KeyEntry 添加到 InputDispatcher 的 mInboundQueue 队列
  - 唤醒 InputDispatcherThread 执行事件分发

```
// 事件转换流程
epoll_event -> input_event -> RawEvent -> NotifyArgs -> KeyEntry
```

#### InputDispatcherThread 的启动
InputDispatcherThread 启动之后会不断的调用 InputDispatcher 的 dispatchOnce 来执行事件分发, 其处理流程如下
- 从 mInboundQueue 中获取队首事件 KeyEntry 保存到 mPendingEvent 进行分发处理
- 找寻事件的响应窗体保存到 inputTargets 集合
  - 找寻焦点窗体 mFocusedWindowHandle, 将它的数据注入 inputTarget
- 遍历 inputTargets 集合进行事件分发
  - 获取响应者连接 Connection 
  - 将 KeyEvent 构建成了 DispatchEntry 对象, 添加到 Connection 的 outboundQueue 队列中
  - Connection 内部的 InputPublisher 获取队列中的元素进行发布
  - 将 KeyEvent 构建成 InputMessage, 通过 InputChannel.sendMessage 发布

```
// 事件转换流程
KeyEntry -> DispatchEntry -> InputMessage
```

## 后记
这次 IMS 的启动我们分析了很多, 但是仍然留下了几个疑问
- **mFocusedWindowHandle 焦点窗体是如何创建的?**
- **InputChannel 是怎么创建的?**
- **InputChannel 的 sendMessage 将这个事件发向了何方?**
- **应用进程是如何接收到输入事件的?**

这些疑问点, 我们到 WMS 的文章中再进行揭秘

## 参考文献
http://gityuan.com/2016/12/17/input-dispatcher/