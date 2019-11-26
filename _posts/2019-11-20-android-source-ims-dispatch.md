---
layout: article
title: "Android 系统架构 —— IMS 的事件分发"
permalink: android-source/ims-dispatch
key: android-source-ims-dispatch
tags: AndroidFramework
---

## 前言
上一篇文章中我们留下了很多疑问, 这篇文章我们就对此进行揭秘, 对 Android 窗体有所了解的开发者应该知晓, 当 ViewRootImpl 会通过 WindowManagerSession 向 WMS 发起请求创建一个窗体, 对应的实现如下
```java
public class WindowManagerService extends IWindowManager.Stub {

    final WindowManagerPolicy mPolicy;// 在 WMS 的构造函数中赋值, 其实例为 PhoneWindowManager
    final WindowHashMap mWindowMap = new WindowHashMap();

    public int addWindow(Session session, IWindow client, int seq,
            LayoutParams attrs, int viewVisibility, int displayId, Rect outFrame,
            Rect outContentInsets, Rect outStableInsets, Rect outOutsets,
            DisplayCutout.ParcelableWrapper outDisplayCutout, InputChannel outInputChannel) {
            
        .......
        synchronized(mWindowMap) {
            ......
            // 1. 创建一个窗体的描述
            final WindowState win = new WindowState(this, session, client, token, parentWindow,
                    appOp[0], seq, attrs, viewVisibility, session.mUid,
                    session.mCanAddInternalSystemWindow);
            ......
            // 2. 给这个窗体描述打开一个输入通道, 用于接收屏幕的点击事件(事件分发)
            final boolean openInputChannels = (outInputChannel != null && (attrs.inputFeatures & INPUT_FEATURE_NO_INPUT_CHANNEL) == 0);
            if  (openInputChannels) {
                win.openInputChannel(outInputChannel);
            }
            ......
            // 3. 将新创建添加的 WindowState 设置为 IMS 的焦点 Window, 即 native 层的 mFocusedWindowHandle
            if (win.canReceiveKeys()) {
                focusChanged = updateFocusedWindowLocked(UPDATE_FOCUS_WILL_ASSIGN_LAYERS,
                        false /*updateInputWindows*/);
                ......
            }
            .......
        }
        ......
        return res;
    }
}
```
上述代码即 WMS 添加一个 Window 的片段, 它主要做了如下几件事情
- 创建窗体描述 WindowState
- **打开窗体的输入通道 InputChannel**
- 更新 IMS 的焦点窗体
  - native 层的 mFocusedWindowHandle

IMS 能够将事件分发到窗体上 openInputChannel 输入通道的创建便是重中之重了, 这里我们主要对 openInputChannel 进行分析

<!--more-->

## 一. 系统服务进程创建 InputChannel 通道组
```java
class WindowState extends WindowContainer<WindowState> implements WindowManagerPolicy.WindowState {
   
    // Input channel and input window handle used by the input dispatcher.
    final InputWindowHandle mInputWindowHandle;
    InputChannel mInputChannel;
    private InputChannel mClientChannel;
    
     WindowState(WindowManagerService service, Session s, IWindow c, WindowToken token,
            WindowState parentWindow, int appOp, int seq, WindowManager.LayoutParams a,
            int viewVisibility, int ownerId, boolean ownerCanAddInternalSystemWindow,
            PowerManagerWrapper powerManagerWrapper) {
        ......
        // 描述当前 WindowState 的句柄值
        mInputWindowHandle = new InputWindowHandle(
                mAppToken != null ? mAppToken.mInputApplicationHandle : null, this, c,
                    getDisplayId());
    }
    
    void openInputChannel(InputChannel outInputChannel) {
        ......
        // 1. 创建一个 InputChannel 组
        String name = getName();
        InputChannel[] inputChannels = InputChannel.openInputChannelPair(name);
        // 2. 一个通道保存在当前 WindowState 中
        mInputChannel = inputChannels[0];
        // 2.1 保存到 WindowState 句柄值中
        mInputWindowHandle.inputChannel = inputChannels[0];
        // 3. 一个通道用于给对应的应用进程使用
        mClientChannel = inputChannels[1];
        if (outInputChannel != null) {
            // 3.1 将客户端的通道拷贝到 outInputChannel 对象中, 以便通过 Binder 驱动返回给客户端
            mClientChannel.transferTo(outInputChannel);
            mClientChannel.dispose();
            mClientChannel = null;
        } else {
            ......
        }
        // 4. 向 InputManager 注册这个窗体的输入通道
        mService.mInputManager.registerInputChannel(mInputChannel, mInputWindowHandle);
    }
    
}
```
可以看到窗体通道的注册主要有如下几步
- 调用 InputChannel.openInputChannelPair 获取一个输入通道组
- inputChannels[0] 保存在 WindowState 中
  - 保存在 WindowStateHandle 句柄值中 
- inputChannels[1] 通过 Binder 驱动返回给客户端
- 向 IMS 这个 InputChannel 通道组

这里我们主要关注, **InputChannel 输入通道组的创建** 和 **向 IMS 注册的过程**

### 一) InputChannel 的创建
还记得我们在 IMS 启动中残留的疑问吗, **这里可以看到在 WindowState 创建时, 会创建一个对 InputChannel, 这里我们就看看它的创建流程**

```java
public final class InputChannel implements Parcelable {
    
    public static InputChannel[] openInputChannelPair(String name) {
        ......
        return nativeOpenInputChannelPair(name);
    }
    
    private static native InputChannel[] nativeOpenInputChannelPair(String name);

}
```
nativeOpenInputChannelPair 的实现定义在 [android_view_InputChannel.cpp](http://androidxref.com/9.0.0_r3/xref/frameworks/base/core/jni/android_view_InputChannel.cpp) 中, 下面看看它的实现
```
// frameworks/base/core/jni/android_view_InputChannel.cpp
static jobjectArray android_view_InputChannel_nativeOpenInputChannelPair(JNIEnv* env,
        jclass clazz, jstring nameObj) {
    // 1. 获取 name
    const char* nameChars = env->GetStringUTFChars(nameObj, NULL);
    std::string name = nameChars;
    env->ReleaseStringUTFChars(nameObj, nameChars);
    
    sp<InputChannel> serverChannel;
    sp<InputChannel> clientChannel;
    // 2. 调用了 InputChannel 的 openInputChannelPair 创建两个 InputChannel
    status_t result = InputChannel::openInputChannelPair(name, serverChannel, clientChannel);
    ......
    // 3. 通过 JNI 创建一个 InputChannel 数组, 将 serverChannelObj 保存
    jobjectArray channelPair = env->NewObjectArray(2, gInputChannelClassInfo.clazz, NULL);
    ......
    env->SetObjectArrayElement(channelPair, 0, serverChannelObj);
    env->SetObjectArrayElement(channelPair, 1, clientChannelObj);
    return channelPair;
}
```
Native 的实现中, 调用了 [InputChannel::openInputChannelPair](http://androidxref.com/9.0.0_r3/xref/frameworks/native/libs/input/InputTransport.cpp) 实现通道的创建
```
// frameworks/native/libs/input/InputTransport.cpp
static const size_t SOCKET_BUFFER_SIZE = 32 * 1024;

status_t InputChannel::openInputChannelPair(const std::string& name,
        sp<InputChannel>& outServerChannel, sp<InputChannel>& outClientChannel) {
    int sockets[2];
    // 1. 创建一个 Socket
    if (socketpair(AF_UNIX, SOCK_SEQPACKET, 0, sockets)) {
        ......
        return result;
    }
    // 2. 配置 Socket 端口数据
    // Socket 传输数据大小为 32 KB
    int bufferSize = SOCKET_BUFFER_SIZE;
    // 2.1 配置服务端端口
    setsockopt(sockets[0], SOL_SOCKET, SO_SNDBUF, &bufferSize, sizeof(bufferSize));
    setsockopt(sockets[0], SOL_SOCKET, SO_RCVBUF, &bufferSize, sizeof(bufferSize));
    // 2.2 配置客户端端口
    setsockopt(sockets[1], SOL_SOCKET, SO_SNDBUF, &bufferSize, sizeof(bufferSize));
    setsockopt(sockets[1], SOL_SOCKET, SO_RCVBUF, &bufferSize, sizeof(bufferSize));

    // 3. 创建 InputChannel
    // 3.1 创建服务端的 InputChannel
    std::string serverChannelName = name;
    serverChannelName += " (server)";
    outServerChannel = new InputChannel(serverChannelName, sockets[0]);

    // 3.2 创建客户端的 InputChannel
    std::string clientChannelName = name;
    clientChannelName += " (client)";
    outClientChannel = new InputChannel(clientChannelName, sockets[1]);
    return OK;
}
```
InputChannel 的 openInputChannelPair 中的信息量非常令人惊喜
- 创建了一个 Socket 
- 配置端口信息
  - **buffer 上限为 32 KB**
- 创建 InputChannel

看到这里这个方法我们可以确定的是, **服务端的 IMS 与应用进程的数据交互是通过 Socket 进行跨进程通信的了**, 下面我们看看 InputChannel 的实例化过程

```
// frameworks/native/libs/input/InputTransport.cpp
InputChannel::InputChannel(const std::string& name, int fd) :
        mName(name), 
        // 保存 socket 端口的文件描述符
        mFd(fd) {
    ......
    // O_NONBLOCK 意为设置为非阻塞式 Socket 通信
    int result = fcntl(mFd, F_SETFL, O_NONBLOCK);
    ......
}
```
InputChannel 的创建非常简单, 它保存了 Socket 的端口文件描述符, 然后将其设置为**非阻塞的 Socket**

到这里 InputChannel 就创建好了, 下面我们看看 IMS 注册 InputChannel 的过程

### 二) IMS 注册 InputChannel
```java
public class InputManagerService extends IInputManager.Stub
        implements Watchdog.Monitor {
    
    public void registerInputChannel(InputChannel inputChannel,
            InputWindowHandle inputWindowHandle) {
        // 调用了 nativeRegisterInputChannel
        nativeRegisterInputChannel(mPtr, inputChannel, inputWindowHandle, false);
    }
    
}
```
我们看看 [nativeRegisterInputChannel](http://androidxref.com/9.0.0_r3/xref/frameworks/base/services/core/jni/com_android_server_input_InputManagerService.cpp) 中做了什么操作
```
// frameworks/base/services/core/jni/com_android_server_input_InputManagerService.cpp
static void nativeRegisterInputChannel(JNIEnv* env, jclass /* clazz */,
        jlong ptr, jobject inputChannelObj, jobject inputWindowHandleObj, jboolean monitor) {
    
    NativeInputManager* im = reinterpret_cast<NativeInputManager*>(ptr);
    // 获取 InputChannel
    sp<InputChannel> inputChannel = android_view_InputChannel_getInputChannel(env,
            inputChannelObj);
    ......
    // 获取 WindowStateHandle
    sp<InputWindowHandle> inputWindowHandle =
            android_server_InputWindowHandle_getHandle(env, inputWindowHandleObj);
    // 调用了 InputManager 的 registerInputChannel 注册这个 WindowStateHandle 和 InputChannel
    status_t status = im->registerInputChannel(
            env, inputChannel, inputWindowHandle, monitor);
    ......
}

status_t NativeInputManager::registerInputChannel(JNIEnv* /* env */,
        const sp<InputChannel>& inputChannel,
        const sp<InputWindowHandle>& inputWindowHandle, bool monitor) {
    // 将注册操作转发到 InputDispatcher 中
    return mInputManager->getDispatcher()->registerInputChannel(
            inputChannel, inputWindowHandle, monitor);
}
```
可以看到nativeRegisterInputChannel 最终会调用 InputDispatcher 的 registerInputChannel 函数注册这个新创建的 WindowStateHandle 和 InputChannel
```
// frameworks/native/services/inputflinger/InputDispatcher.cpp
status_t InputDispatcher::registerInputChannel(const sp<InputChannel>& inputChannel,
        const sp<InputWindowHandle>& inputWindowHandle, bool monitor) {

    { // acquire lock
        AutoMutex _l(mLock);
        // 1. 创建一个 Connection 描述 IMS 和窗体的连接
        sp<Connection> connection = new Connection(inputChannel, inputWindowHandle, monitor);
        // 2. 将 InputChannel 的 socket 文件描述符, 添加到 mConnectionsByFd 中缓存
        int fd = inputChannel->getFd();
        mConnectionsByFd.add(fd, connection);
        // 3. 让 Looper 中的 epool 监听这个服务端的 socket 端口
        mLooper->addFd(fd, 0, ALOOPER_EVENT_INPUT, handleReceiveCallback, this);
    } // release lock
    // 4. 一个新 Window 添加进来了, 唤醒阻塞在 Looper.pollOnce 上的 InputDispatcherThread 线程
    mLooper->wake();
    return OK;
}
```
可以看到这里创建了一个 描述 IMS 和 WindowState 连接的 Connection 对象, 然后让 InputDispatcher 的成员变量 mLooper 来监听 InputChannel 中的 Socket 端口

这里我们可能会有疑问, 服务端的端口不是用来向应用进程发送数据的吗? 监听它的意义何在呢? 
- Socket 与匿名管道不同, 它是支持双端读写的, 当应用进程读取了事件, 消费之后是需要通知 IMS 所在系统服务进程事件已经被消费了的, 这里的监听操作就是为了接收客户端的回执信息

到这里我们在 IMS 中留下的疑问就全部明了了, InputChannel 是在 WMS 新建窗体时创建的, InputChannel 的 sendMessage 最终会发送到应用进程的 Socket 端口

### 三) 回顾
- 创建 InputChannel 输入通道组
  - 创建最大缓冲为 32 kb 的 Socket
  - 服务端的 InputChannel 保留 Socket 服务端的文件描述符
  - 客户端的 InputChannel 保留 Socket 客户端的文件描述符
- 将服务端的新建的 WindowState 和 InputChannel 注册到 IMS 中
  - InputDipatcher 将 WindowState 和 InputChannel 封装到 Connection 中用于后续的事件分发

到这里我们在 IMS 中留下的疑问就全部明了了
- InputChannel 是在 WMS 新建窗体时创建的
- InputChannel 的 sendMessage 最终会发送到应用进程的 Socket 端口

IMS 和 WMS 的关联形式如下

![IMS 与 WMS 的关联](https://i.loli.net/2019/11/20/3UCtHrxJ1OqYjai.png)

接下来应用进程监听 InputChannel 过程

## 二. 应用进程监听 InputChannel
```java
public final class ViewRootImpl implements ViewParent,
        View.AttachInfo.Callbacks, ThreadedRenderer.DrawCallbacks {
            
    InputChannel mInputChannel;
    
    public void setView(View view, WindowManager.LayoutParams attrs, View panelParentView) {
        synchronized (this) {
            if (mView == null) {
                // 这里是我们的请求布局
                requestLayout(); 
                // 1. 创建了一个 InputChannel 对象, 此时它是没有意义的
                if ((mWindowAttributes.inputFeatures
                        & WindowManager.LayoutParams.INPUT_FEATURE_NO_INPUT_CHANNEL) == 0) {
                    mInputChannel = new InputChannel();
                }
    
                try {
                    // 2. 调用 addToDisplay, 进而调用 WMS.addWindow 进行窗体添加操作
                    // mInputChannel 作为传出参数
                    res = mWindowSession.addToDisplay(......, /*将 mInputChannel 传入*/mInputChannel);
                } catch (RemoteException e) {
                   ......
                } finally {
                   ......
                }
                ......
                // 3. WMS 的添加窗体完成后, mInputChannel 便通过 Binder 驱动拷贝 WMS 中传递过来的值
                if (mInputChannel != null) {
                    ......
                    // 3.1 创建了输入事件接收器
                    mInputEventReceiver = new WindowInputEventReceiver(mInputChannel,
                            Looper.myLooper());
                }
                ......
            }
        }
    }        
            
}
```
当 mWindowSession.addToDisplay 的跨进程操作完成之后, mInputChannel 便会被 Binder 驱动赋值了, 正是我们在系统服务进程创建的 InputChannel 组的另一端的值

**熟悉 Linux 内核机制的同学可能会就有疑问了, InputChannel 中持有的是 Socket 一端的文件描述符, fd 不是只在当前进程有效吗? 从系统服务进程跨进程传输到应用进程不就无效了吗?**
- 这是因为 Binder 驱动对 FD 的传输做了特殊处理, 它不仅仅只是传递一个句柄的 int 值, 而是会在另一个进程找到一个空闲的句柄, 然后让它指向对应的文件, **因此客户端 mInputChannel 中的 mFd 与在系统服务进程中创建的 InputChannel 的 mFd 不一定是相同的, 但是他们指向的却是同一个 socket 端口文件, 这也是 Binder 驱动设计的精妙之处**

获取到了 mInputChannel 被赋予意义之后, 会创建一个 WindowInputEventReceiver, 从名字就可以看出, 它描述一个事件的接收者, 我们看看它的构建过程

### 一) 事件接收器的初始化
```java
public final class ViewRootImpl implements ViewParent,
        View.AttachInfo.Callbacks, ThreadedRenderer.DrawCallbacks {
        
    final class WindowInputEventReceiver extends InputEventReceiver {
        public WindowInputEventReceiver(InputChannel inputChannel, Looper looper) {
            super(inputChannel, looper);
        }
    }
    
}

public abstract class InputEventReceiver {

    private long mReceiverPtr;
    private InputChannel mInputChannel;
    private MessageQueue mMessageQueue;

    public InputEventReceiver(InputChannel inputChannel, Looper looper) {
        .......
        mInputChannel = inputChannel;
        // 这个 mMessageQueue 是主线程的消息队列
        mMessageQueue = looper.getQueue();
        // 执行 Native 初始化
        mReceiverPtr = nativeInit(new WeakReference<InputEventReceiver>(this),
                inputChannel, mMessageQueue);
    }
    
    private static native long nativeInit(WeakReference<InputEventReceiver> receiver,
            InputChannel inputChannel, MessageQueue messageQueue);
}
```
好的, 这里它调用了 nativeInit 获取了一个 Native 的句柄, 我们看看它在 native 层做了些什么
```
// frameworks/base/core/jni/android_view_InputEventReceiver.cpp
static jlong nativeInit(JNIEnv* env, jclass clazz, jobject receiverWeak,
        jobject inputChannelObj, jobject messageQueueObj) {
    // 1. 获取 Native 的 InputChannel
    sp<InputChannel> inputChannel = android_view_InputChannel_getInputChannel(env,
            inputChannelObj);
    // 2. 获取 MessageQueue 对应的 native 的对象
    sp<MessageQueue> messageQueue = android_os_MessageQueue_getMessageQueue(env, messageQueueObj);
    // 3. 创建了一个 native 的 NativeInputEventReceiver 对象, 用于监听 InputChannel 中的事件
    sp<NativeInputEventReceiver> receiver = new NativeInputEventReceiver(env,
            receiverWeak, inputChannel, messageQueue);
    // 3.1 初始化这个 receiver 对象
    receiver->initialize();
    ......
    // 将句柄返回给 Java 层
    return reinterpret_cast<jlong>(receiver.get());
}
```
可以看到 Native 层的处理如下
- 先是获取 InputChannel 和 MessageQueue 对应的 native 对象
- 然后利用这两个对象创建了一个 NativeInputEventReceiver 对象
- 调用 NativeInputEventReceiver.initialize 进行初始化

接下来我们看看它的初始化操作
```
// frameworks/base/core/jni/android_view_InputEventReceiver.cpp
NativeInputEventReceiver::NativeInputEventReceiver(JNIEnv* env,
        jobject receiverWeak, const sp<InputChannel>& inputChannel,
        const sp<MessageQueue>& messageQueue) :
        mReceiverWeakGlobal(env->NewGlobalRef(receiverWeak)),
        // 保存服务端传来的 InputChannel
        mInputConsumer(inputChannel),
        // 保存主线程的 MessageQueue
        mMessageQueue(messageQueue),
        mBatchedInputEventPending(false), 
        mFdEvents(0) {
    
}

status_t NativeInputEventReceiver::initialize() {
    setFdEvents(ALOOPER_EVENT_INPUT);
    return OK;
}

void NativeInputEventReceiver::setFdEvents(int events) {
    if (mFdEvents != events) {
        mFdEvents = events;
        // 获取 Socket 的端口文件描述符
        int fd = mInputConsumer.getChannel()->getFd();
        if (events) {
            // 让主线程的 Looper 监听这个 Socket 端口的文件描述符
            // this 指的是 fd 中有数据流入时的回调处理函数, 它由 NativeInputEventReceiver 的 handleEvent 实现
            mMessageQueue->getLooper()->addFd(fd, 0, events, this, NULL);
        }
        ......
    }
}

```
NativeInputEventReceiver 初始化操作, 即**使用主线程的 Looper(Native) 对 InputChannel 中保存的 Socket 端口文件描述符进行了监听操作, 当 Socket 端口的 fd 中有数据流入时, 会回调 NativeInputEventReceiver 的 handleEvent 处理事件**

接下来我们便看看 handleEvent 事件处理的过程

### 二) 事件流的处理
接下来我们看看它是如何处理事件的
```
// frameworks/base/core/jni/android_view_InputEventReceiver.cpp
int NativeInputEventReceiver::handleEvent(int receiveFd, int events, void* data) {
    ......
    if (events & ALOOPER_EVENT_INPUT) {
        ......
        status_t status = consumeEvents(env, false /*consumeBatches*/, -1, NULL);
        ......
    }
    .....
    return 1;
}
int register_android_view_InputEventReceiver(JNIEnv* env) {
    ......
    gInputEventReceiverClassInfo.dispatchInputEvent = GetMethodIDOrDie(env,
            gInputEventReceiverClassInfo.clazz,
            "dispatchInputEvent", "(ILandroid/view/InputEvent;I)V");
    ......
}
status_t NativeInputEventReceiver::consumeEvents(JNIEnv* env,
        bool consumeBatches, nsecs_t frameTime, bool* outConsumedBatch) {
    ......
    for (;;) {
        ......
        if (!skipCallbacks) {
            jobject inputEventObj;
            ...... // 构建 InputEvent 实例
            if (inputEventObj) {
                ......
                // 回调了 Java 的 InputEventReceiver.dispatchInputEvent 方法
                env->CallVoidMethod(receiverObj.get(),
                        gInputEventReceiverClassInfo.dispatchInputEvent, seq, inputEventObj,
                        displayId);
                ......
            } else {
               ......
            }
        }
        ......
    }
}
```
好的, 可以看到当有事件产生时, 便会通过 JNI 回调 InputEventReceiver.dispatchInputEvent 回调 Java 层去进行分发事件, 如此一来 Native 层的工作就分析完了

### 三) 回顾
![应用进程事件的处理](https://i.loli.net/2019/11/20/pBJIRLyGa6mHFe4.png)

## 三. 事件流的分发
```java
public abstract class InputEventReceiver {
    
    // Called from native code.
    private void dispatchInputEvent(int seq, InputEvent event, int displayId) {
        onInputEvent(event, displayId);
    }
    
}

public final class ViewRootImpl implements ViewParent,
        View.AttachInfo.Callbacks, ThreadedRenderer.DrawCallbacks {
        
    final class WindowInputEventReceiver extends InputEventReceiver {
        @Override
        public void onInputEvent(InputEvent event, int displayId) {
            enqueueInputEvent(event, this, 0, true);
        }
    }
    
}
```
InputEventReceiver.dispatchInputEvent 中回调 onInputEvent 方法, 这个方法被子类重写了, 因此会回调 WindowInputEventReceiver.enqueueInputEvent 真正处理事件的分发

接下来我们看看 enqueueInputEvent 的实现
```java
public final class ViewRootImpl implements ViewParent,
        View.AttachInfo.Callbacks, ThreadedRenderer.DrawCallbacks {
    
    // ViewRootImpl 中维护了一个 InputEvent 链表, 用来描述待处理的输入事件
    QueuedInputEvent mPendingInputEventHead;        
    QueuedInputEvent mPendingInputEventTail;
    
    void enqueueInputEvent(InputEvent event,
            InputEventReceiver receiver, int flags, boolean processImmediately) {
        // 1. 将 InputEvent 包装成一个 QueuedInputEvent, 添加到 ViewRootImpl 中维护的事件队列尾部
        QueuedInputEvent q = obtainQueuedInputEvent(event, receiver, flags);
        QueuedInputEvent last = mPendingInputEventTail;
        if (last == null) {
            mPendingInputEventHead = q;
            mPendingInputEventTail = q;
        } else {
            last.mNext = q;
            mPendingInputEventTail = q;
        }
        mPendingInputEventCount += 1;
        
        if (processImmediately) {
            // 2.1 是立即处理
            doProcessInputEvents();
        } else {
            // 2.2 添加到 MessageQueue 处理
            scheduleProcessInputEvents();
        }
    }
    
}
```
可以看到 ViewRootImpl 中维护了一个输出事件队列, enqueueInputEvent 方法先是将它添加到队列尾部, 然后执行处理操作, 添加到 MessageQueue 中处理最后也会回调到 doProcessInputEvents 这个方法, 我们这里直接看看它的实现
```java
public final class ViewRootImpl implements ViewParent,
        View.AttachInfo.Callbacks, ThreadedRenderer.DrawCallbacks {
            
    void doProcessInputEvents() {
        while (mPendingInputEventHead != null) {
            QueuedInputEvent q = mPendingInputEventHead;
            ......
            // 分发这个待处理的事件
            deliverInputEvent(q);
        }
        ......
    }
    
    InputStage mFirstInputStage;
    InputStage mFirstPostImeInputStage;

    public void setView(View view, WindowManager.LayoutParams attrs, View panelParentView) {
        synchronized (this) {
            if (mView == null) {
                ......
                InputStage viewPostImeStage = new ViewPostImeInputStage(mSyntheticInputStage);
                InputStage nativePostImeStage = new NativePostImeInputStage(viewPostImeStage,
                        "aq:native-post-ime:" + counterSuffix);
                InputStage earlyPostImeStage = new EarlyPostImeInputStage(nativePostImeStage);
                InputStage imeStage = new ImeInputStage(earlyPostImeStage,
                        "aq:ime:" + counterSuffix);
                InputStage viewPreImeStage = new ViewPreImeInputStage(imeStage);
                InputStage nativePreImeStage = new NativePreImeInputStage(viewPreImeStage,
                        "aq:native-pre-ime:" + counterSuffix);
                mFirstInputStage = nativePreImeStage;
                mFirstPostImeInputStage = earlyPostImeStage;
            }
        }
    }

    private void deliverInputEvent(QueuedInputEvent q) {
        ......
        InputStage stage;
        if (stage != null) {
            ......
            // 调用 stage.deliver 分发事件, 它的对象在 setView 中构建
            stage.deliver(q);
        } else {
            // 描述事件分发完毕
            finishInputEvent(q);
        }
    }
               
}
```
可以看到 doProcessInputEvents 会调用 deliverInputEvnet 来分发这个事件
- 调用 InputStage 启动事件分发
- 调用 finishInputEvent 结束事件分发

接下来我们就从这两个方面看看应用层事件分发的流程

### 一) 事件分发的启动
**调用 InputStage.deliver 进行 InputEvent 的分发**, ViewRootImpl 中的 InputState 对象是在 setView 时初始化的, **我们主要看看 ViewPostImeInputStage 的实现**

```java
public final class ViewRootImpl implements ViewParent,
        View.AttachInfo.Callbacks, ThreadedRenderer.DrawCallbacks {
    
    abstract class InputStage {
        
        public final void deliver(QueuedInputEvent q) {
            if ((q.mFlags & QueuedInputEvent.FLAG_FINISHED) != 0) {
                forward(q);
            } else if (shouldDropInputEvent(q)) {
                finish(q, false);
            } else {
                // 1. 调用子类重写的 onProcess 方法
                apply(q, onProcess(q));
            }
        }
        
    }
    
    final class ViewPostImeInputStage extends InputStage {
        
        @Override
        protected int onProcess(QueuedInputEvent q) {
            if (q.mEvent instanceof KeyEvent) {
                ......
            } else {
                final int source = q.mEvent.getSource();
                if ((source & InputDevice.SOURCE_CLASS_POINTER) != 0) {
                    // 调用了 processPointerEvent 
                    return processPointerEvent(q);
                } 
                ......
            }
        }
        
        private int processPointerEvent(QueuedInputEvent q) {
            final MotionEvent event = (MotionEvent)q.mEvent;
            // 调用了 DecorView 的 dispatchPointerEvent
            boolean handled = mView.dispatchPointerEvent(event);
            ......
            return handled ? FINISH_HANDLED : FORWARD;
        }
        
    }
                
}
```
好的, 可以看到 InputState.deliver 中回调了子类重写的 onProcess, ViewPostImeInputStage 中调用 processPointerEvent 方法, **最终会回调 DecorView 的 dispatchPointerEvent 方法**

#### 1. DecorView 的事件分发
```java
public class View implements Drawable.Callback, KeyEvent.Callback,
        AccessibilityEventSource {
            
    public final boolean dispatchPointerEvent(MotionEvent event) {
        // 若为触摸事件, 则调用 dispatchTouchEvent 进行分发
        if (event.isTouchEvent()) {
            return dispatchTouchEvent(event);
        } else {
            // 处理通用事件
            return dispatchGenericMotionEvent(event);
        }
    }
                
}
```
好的, 可以看到这里调用我们非常属性的 dispatchTouchEvent, DecorView 中重写了这个方法, 我们看看它是怎么做的
```java
public class DecorView extends FrameLayout implements RootViewSurfaceTaker, WindowCallbacks {
    
    public boolean dispatchTouchEvent(MotionEvent ev) {
        // 1. 获取 Window 的 Callback
        final Window.Callback cb = mWindow.getCallback();
        // 2. 若 Callback 有效, 则调用 Callback.dispatchTouchEvent 进行事件分发否则执行 View 的事件分发
        return cb != null && !mWindow.isDestroyed() && mFeatureId < 0
                ? cb.dispatchTouchEvent(ev) : super.dispatchTouchEvent(ev);
    }
    
}
```
这里很有趣, DecorView 会优先调用 Window.Callback 的 dispatchTouchEvent 方法, 这个 Callback 是谁呢? 

这里不妨回顾一个应用进程中 Activity 的 Window 的创建过程
```java
public class Activity extends ContextThemeWrapper {

    final void attach(Context context, ActivityThread aThread,
            Instrumentation instr, IBinder token, int ident,
            Application application, Intent intent, ActivityInfo info,
            CharSequence title, Activity parent, String id,
            NonConfigurationInstances lastNonConfigurationInstances,
            Configuration config, String referrer, IVoiceInteractor voiceInteractor,
            Window window, ActivityConfigCallback activityConfigCallback) {
        ......
        mWindow = new PhoneWindow(this, window, activityConfigCallback);
        ......
        mWindow.setCallback(this);
        ......
    }
}
```
原来**若是在 Activity 中创建的 Window, 那么这个 Window 的 Callback 就为 Activity**

#### 2. Activity 的事件分发
```java
public class Activity extends ContextThemeWrapper {
    
    public boolean dispatchTouchEvent(MotionEvent ev) {
        if (ev.getAction() == MotionEvent.ACTION_DOWN) {
            onUserInteraction();
        }
        // 还是会优先调用 Window 的 superDispatchTouchEvent
        if (getWindow().superDispatchTouchEvent(ev)) {
            return true;
        }
        // 若是没有被消费, 则自行处理
        return onTouchEvent(ev);
    }
    
}
```
可以看到 Activity 中对于事件分发的处理方式为 
- 首先调用了 Window 的 superDispatchTouchEvent 进行分发处理
- 若事件没有被消费, 则调用 onTouchEvent 自行处理

Window 的实例为 PhoneWindow, 我们看看 PhoneWindow 的 superDispatchTouchEvent 方法如何分发事件
```java
public class PhoneWindow extends Window implements MenuBuilder.Callback {
    
    @Override
    public boolean superDispatchTouchEvent(MotionEvent event) {
        // 调用了 DecorView 的 superDispatchTouchEvent 方法
        return mDecor.superDispatchTouchEvent(event);
    }
    
}

public class DecorView extends FrameLayout implements RootViewSurfaceTaker, WindowCallbacks {
    
     public boolean superDispatchTouchEvent(MotionEvent event) {
        return super.dispatchTouchEvent(event);
    }
    
}
```
好的, 兜兜转转最后还是回调了 View 的 dispatchTouchEvent 方法, 从此便进入的 View 的事件分发流程, 我们到下一篇文章中再进程阐述

接下来看看 finishInputEvent 结束事件分发执行了哪些操作

### 二) 事件分发的结束
```java
public final class ViewRootImpl implements ViewParent,
        View.AttachInfo.Callbacks, ThreadedRenderer.DrawCallbacks {
        
    private void finishInputEvent(QueuedInputEvent q) {
        ......
        if (q.mReceiver != null) {
            boolean handled = (q.mFlags & QueuedInputEvent.FLAG_FINISHED_HANDLED) != 0;
            // 调用了 InputEventReceiver 的 finishInputEvent 方法
            q.mReceiver.finishInputEvent(q.mEvent, handled);
        } else {
            ......
        }
    }
    
}

public abstract class InputEventReceiver {
    
    public final void finishInputEvent(InputEvent event, boolean handled) {
        ......
        if (mReceiverPtr == 0) {
            ......
        } else {
            ......
            if (index < 0) {
                ......
            } else {
                ......
                // 到了 Native 层结束事件分发
                nativeFinishInputEvent(mReceiverPtr, seq, handled);
            }
        }
        ......
    }
    
}
```
可以看到 finishInputEvent 最终会调用到 InputEventReceiver 的 nativeFinishInputEvent, 它主要是负责向 IMS 发送回执信息, 下面看看它的实现

#### 1. 客户发送回执信息
```
// frameworks/base/core/jni/android_view_InputEventReceiver.cpp
static void nativeFinishInputEvent(JNIEnv* env, jclass clazz, jlong receiverPtr,
        jint seq, jboolean handled) {
    // 调用了 NativeInputEventReceiver 的 finishInputEvent
    sp<NativeInputEventReceiver> receiver =
            reinterpret_cast<NativeInputEventReceiver*>(receiverPtr);
    status_t status = receiver->finishInputEvent(seq, handled);
    ......
}

status_t NativeInputEventReceiver::finishInputEvent(uint32_t seq, bool handled) {
    ......
    // 调用了 InputChannel 的 sendFinishedSignal 向服务端发送了一个执行结束的指令
    status_t status = mInputConsumer.sendFinishedSignal(seq, handled);
    ......
    return status;
}
```
好的, 最终调用了 InputChannel 的 sendFinishedSignal 通知服务端应用进程的事件分发完毕了

还记得我们上面 IMS 注册 InputChannel 的过程吗, 它使用 InputDispatcher 内部的 mLooper 监听了服务端的 socket 端口, 有数据会回调 handleReceiveCallback, 下面我们看看服务端收到了回执消息之后的处理

#### 2. 服务端处理回执信息
```
// frameworks/native/services/inputflinger/InputDispatcher.cpp
int InputDispatcher::handleReceiveCallback(int fd, int events, void* data) {
    InputDispatcher* d = static_cast<InputDispatcher*>(data);

    { // acquire lock
        AutoMutex _l(d->mLock);
        // 1. 找到分发的窗体连接
        ssize_t connectionIndex = d->mConnectionsByFd.indexOfKey(fd);
        ......
        bool notify;
        sp<Connection> connection = d->mConnectionsByFd.valueAt(connectionIndex);
        if (!(events & (ALOOPER_EVENT_ERROR | ALOOPER_EVENT_HANGUP))) {
            ......
            nsecs_t currentTime = now();
            bool gotOne = false;
            status_t status;
            for (;;) {
                uint32_t seq;
                bool handled;
                // 2. 接收 Socket 发送过来的回执信息
                status = connection->inputPublisher.receiveFinishedSignal(&seq, &handled);
                if (status) {
                    break;
                }
                // 3. 处理回执数据
                d->finishDispatchCycleLocked(currentTime, connection, seq, handled);
                gotOne = true;
            }
            if (gotOne) {
                // 4. 执行 Commands
                d->runCommandsLockedInterruptible();
                if (status == WOULD_BLOCK) {
                    return 1;
                }
            }
            ......
        } else {
            ......
            // InputChannel 被关闭或者发生错误
        }
        ......
    } // release lock
}
```
IMS 处理回执消息流程如下
- 找到分发事件的窗体连接 Connection
- 通过 Socket 获取回执数据
- 调用 finishDispatchCycleLocked 添加分发结束的指令
- 调用 runCommandsLockedInterruptible 执行分发结束指令

接下来我们从指令的添加和执行来看看 InputDispatcher 对回执信息的处理

##### 1) 添加结束指令
```
// frameworks/native/services/inputflinger/InputDispatcher.cpp
void InputDispatcher::finishDispatchCycleLocked(nsecs_t currentTime,
        const sp<Connection>& connection, uint32_t seq, bool handled) {
    connection->inputPublisherBlocked = false;
    if (connection->status == Connection::STATUS_BROKEN
            || connection->status == Connection::STATUS_ZOMBIE) {
        return;
    }
    // 1. 通知其他组件分发下一个事件序列
    onDispatchCycleFinishedLocked(currentTime, connection, seq, handled);
}

void InputDispatcher::onDispatchCycleFinishedLocked(
        nsecs_t currentTime, const sp<Connection>& connection, uint32_t seq, bool handled) {
    // 2. 创建了一个指令信息实体类 CommandEntry
    CommandEntry* commandEntry = postCommandLocked(
            & InputDispatcher::doDispatchCycleFinishedLockedInterruptible);
    // 3. 填充数据
    commandEntry->connection = connection;
    commandEntry->eventTime = currentTime;
    commandEntry->seq = seq;
    commandEntry->handled = handled;
}

InputDispatcher::CommandEntry* InputDispatcher::postCommandLocked(Command command) {
    // 2.1 创建指令信息 CommandEntry 对象, 指令为 doDispatchCycleFinishedLockedInterruptible
    CommandEntry* commandEntry = new CommandEntry(command);
    // 2.2 添加到指令队列尾部
    mCommandQueue.enqueueAtTail(commandEntry);
    return commandEntry;
}
```
finishDispatchCycleLocked 处理回执数据的过程非常简单
- 它**创建了一个 doDispatchCycleFinishedLockedInterruptible 执行指令对象 CommandEntry**
- 将它添加到了指令队列的尾部, 等待后续的执行

接下来我们看看 runCommandsLockedInterruptible 执行指令的过程

##### 2) 执行结束指令
```
// frameworks/native/services/inputflinger/InputDispatcher.cpp
bool InputDispatcher::runCommandsLockedInterruptible() {
    if (mCommandQueue.isEmpty()) {
        return false;
    }
    do {
        // 取出指令
        CommandEntry* commandEntry = mCommandQueue.dequeueAtHead();
        // 执行指令
        Command command = commandEntry->command;
        (this->*command)(commandEntry);
        // 删除指令
        commandEntry->connection.clear();
        delete commandEntry;
    } while (! mCommandQueue.isEmpty());
    return true;
}
```
InputDispatcher 执行指令的过程即遍历指令队列, 执行具体的指令

由上面的分析可知, 分发结束的执行执行函数为 
doDispatchCycleFinishedLockedInterruptible, 下面看看它的具体是实现
```
// frameworks/native/services/inputflinger/InputDispatcher.cpp
void InputDispatcher::doDispatchCycleFinishedLockedInterruptible(
        CommandEntry* commandEntry) {
    sp<Connection> connection = commandEntry->connection;
    nsecs_t finishTime = commandEntry->eventTime;
    uint32_t seq = commandEntry->seq;
    bool handled = commandEntry->handled;
    // 1. 获取分发的事件
    DispatchEntry* dispatchEntry = connection->findWaitQueueEntry(seq);
    if (dispatchEntry) {
        ......
        if (dispatchEntry == connection->findWaitQueueEntry(seq)) {
            // 2. 将事件从等待分发完毕的队列中移除
            connection->waitQueue.dequeue(dispatchEntry);
            ......
        }
        // 3. 让当前 connection 去处理其 outboundQueue 中的后续事件
        startDispatchCycleLocked(now(), connection);
    }
}
```
这里我们可以看到处理事件分发结束的操作主要如下
- 获取本次分发的事件描述 DispatchEntry
- 从 connection 待回执的事件队列 waitQueue 中移除
- 让 connection 继续处理其待分发队列 outboundQueue 的后续事件

### 三) 回顾
这里就与 IMS 启动时 connection 分发事件首尾呼应了, Connection 的整个流程如下
- 服务端 Connection 分发事件
  - 从 outboundQueue 取出一个事件, 交由 InputChannel 的 Socket 端口发到客户端
  - 将这个事件添加到等待队列 waitQueue
- 客户端处理完毕后通过 InputChannel 的 Socket 端口发到服务端
- 服务 Connection 的 InputChannel 接收到回执信息
  - 将这个事件从 waitQueue 中移除
  - 从 outboundQueue 获取下一个事件重复上述操作进行分发

## 总结
![Android 系统事件分发流程图](https://i.loli.net/2019/11/21/OVAPohwv2H8t9CW.png)

到这里我们整个 IMS 事件分发在宏观上就分析完毕了, 这里回顾一下整个事件分发的流程
- 创建 InputChannel 输入通道组
  - 创建最大缓冲为 32 kb 的 Socket
  - 服务端的 InputChannel 保留 Socket 服务端的文件描述符
  - 客户端的 InputChannel 保留 Socket 客户端的文件描述符
- 将服务端的新建的 WindowState 和 InputChannel 注册到 IMS 中
  - InputDipatcher 将 WindowState 和 InputChannel 封装到 Connection 中用于后续的事件分发
- 客户端监听 InputChannel
  - 应用进程拿到 WMS 传递过来的 InputChannel 后便会构建一个 InputChannelReceived
  - **InputChannelReceived 在 Native 层监听了 InputChannel 的 Socket 端口**
  - 有数据写入便会通过 JNI 调用 dispatchInputEvent 分发到 Java 层 
- 执行事件分发
  - 事件分发的启动
    - ViewRootImpl 中维护了一个输入事件队列, 这个事件最终会交给 InputStage 消费
    - InputStage 中会将事件传递给 DecorView
    - DecorView 首先会尝试将事件呈递给当前 Window 的 Callback 对象处理
      - 在 Activity 中创建的 Window, Callback 为 Activity 本身
    - Activity 将事件再次呈递给 DecorView, 最终调用 dispatchTouchEvent 进入 ViewGroup 的事件分发流程
  - 事件分发的结束
    - 客户端发送回执信息
    - 服务端接收回执信息并处理
      - 创建结束指令到 mCommandQueue 指令队列
      - mCommandQueue 执行的结束指令
        - 从 Connection 待回执的事件队列 waitQueue 中移除
        - 让 Connection 继续处理其待分发队列 outboundQueue 的后续事件 

IMS 的事件分发, 是比较困难但又非常重要的知识点, 通过本次的分析, 我们对事件流从哪儿来, 从哪儿获取, 以及如何分发处理, 就有了一个宏观上的的认知
 
关于更加微观的 View 的事件分发我们在前面已经分析过了, 请[查阅这篇文章](https://sharrychoo.github.io/blog/2018/10/15/android-source-view-dispatch.html)