---
layout: article
title: "Android 系统架构 —— IMS 的事件分发"
key:  "Android 系统架构 —— IMS 的事件分发"
tags: AndroidFramework
aside:
  toc: true
---

## 前言
上一篇文章中我们留下了很多疑问, 这篇文章我们就对此进行揭秘, 对 Andorid 窗体有所了解的开发者应该知晓, 当 ViewRootImpl 会通过 WindowManagerSession 想 WMS 发起请求创建一个窗体, 对应的实现如下
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
            // 创建一个窗体的描述
            final WindowState win = new WindowState(this, session, client, token, parentWindow,
                    appOp[0], seq, attrs, viewVisibility, session.mUid,
                    session.mCanAddInternalSystemWindow);
            ......
            // 给这个窗体描述打开一个输入通道, 用于接收屏幕的点击事件(事件分发)
            final boolean openInputChannels = (outInputChannel != null
                    && (attrs.inputFeatures & INPUT_FEATURE_NO_INPUT_CHANNEL) == 0);
            if  (openInputChannels) {
                win.openInputChannel(outInputChannel);
            }
            ......
        }
        ......
        return res;
    }
}
```
IMS 能够将事件分发到窗体上以至于最后能够分发到 View 中就是从这里开始关联的, 我们先跳过 WMS 的启动, 看看添加 Window 的过程

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

### 回顾
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

### 流程回顾
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
        if (q.shouldSendToSynthesizer()) {
            ......
        } else {
            stage = q.shouldSkipIme() ? mFirstPostImeInputStage : mFirstInputStage;
        }
        if (stage != null) {
            ......
            // 调用 stage.deliver 分发事件, 它的对象在 setView 中构建
            stage.deliver(q);
        } else {
            ......
        }
    }
               
}
```
可以看到 doProcessInputEvents 最终会**调用 InputStage.deliver 进行 InputEvent 的分发**, ViewRootImpl 中的 InputState 对象是在 setView 时初始化的, **我们主要看看 ViewPostImeInputStage 的实现**

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

### 一) DecorView 的事件分发
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

### 二) Activity 的事件分发
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
好的, 兜兜转转最后还是回调了 View 的 dispatchTouchEvent 方法, 下面我们来分析 ViewGroup 的事件分发

### 三) ViewGroup 的事件分发
Android 支持多指触控的, 因此一个事件序列存在多个焦点, 在 MotionEvent 中用 PointerId 描述, 好的了解了这个基础知识之后, 我们便看看 ViewGroup 的 dispatchTouchEvent
```java
public abstract class ViewGroup extends View implements ViewParent, ViewManager {
    
    // TouchTarget 描述一个事件序列的响应者 从 DOWN -> UP | POINTER_DOWN -> POINTER_UP
    // mFirstTouchTarget 为所有事件序列的链表头
    private TouchTarget mFirstTouchTarget;
    
    @Override
    public boolean dispatchTouchEvent(MotionEvent ev) {
        ......
        boolean handled = false;
        // 过滤不安全的事件
        if (onFilterTouchEventForSecurity(ev)) {
            // 获取事件的类型
            final int action = ev.getAction();
            final int actionMasked = action & MotionEvent.ACTION_MASK;
            // 若为 ACTION_DOWN, 则说明是一个全新的事件序列
            if (actionMasked == MotionEvent.ACTION_DOWN) {
                cancelAndClearTouchTargets(ev);   // 清空所有 TouchTarget
                resetTouchState();                // 清空所有触摸状态
            }
            // 1. 处理事件拦截
            final boolean intercepted;
            // 1.1 若为初始事件 或 当前容器存在子 View 正在消费事件, 则尝试拦截
            if (actionMasked == MotionEvent.ACTION_DOWN || mFirstTouchTarget != null) {
                // 1.1.1 判断当前 ViewGroup 的 Flag 中是否设置了 不允许拦截事件
                // 通过 ViewGroup.requestDisallowInterceptTouchEvent 进行设置, 常用于内部拦截法, 由子 View 控制容器的行为
                final boolean disallowIntercept = (mGroupFlags & FLAG_DISALLOW_INTERCEPT) != 0;
                if (!disallowIntercept) {
                    // 若允许拦截, 则调用 onInterceptTouchEvent, 尝试进行拦截
                    intercepted = onInterceptTouchEvent(ev);
                    ev.setAction(action);
                } else {
                    // 不允许拦截, 则直接将标记为置为 false
                    intercepted = false;
                }
            } else {
                // 1.2 若非初始事件, 并且没有子 View 响应事件中的焦点, 则自己拦截下来由自己处理
                // 自己处理不代表一定能消费, 这点要区分开来
                intercepted = true;
            }
            ......
            // 判断当前容器是否被取消了响应事件
            final boolean canceled = resetCancelNextUpFlag(this)
                    || actionMasked == MotionEvent.ACTION_CANCEL;
            ......
            // 判断是否需要拆分 MotionEvent
            final boolean split = (mGroupFlags & FLAG_SPLIT_MOTION_EVENTS) != 0;
            TouchTarget newTouchTarget = null;                // 描述响应的目标
            boolean alreadyDispatchedToNewTouchTarget = false;// 描述这个事件是否分发给了新的 TouchTarget
            if (!canceled && !intercepted) {
                ......
                // 2. 若为 ACTION_DOWN 或者 ACTION_POINTER_DOWN, 则说明当前事件序列出现了一个新的焦点, 则找寻该焦点的处理者
                if (actionMasked == MotionEvent.ACTION_DOWN
                        || (split && actionMasked == MotionEvent.ACTION_POINTER_DOWN)
                        || actionMasked == MotionEvent.ACTION_HOVER_MOVE) { 
                    // 2.1 获取焦点的索引(第一个手指的 down 为 0, 第二个手指的 down 为 1)
                    final int actionIndex = ev.getActionIndex(); // always 0 for down
                    // 将焦点的索引序号映射成二进制位, 用于后续保存在 TouchTarget 的 pointerIdBits 中
                    // 0 -> 1, 1 -> 10, 2 -> 100
                    final int idBitsToAssign = split ? 1 << ev.getPointerId(actionIndex) : TouchTarget.ALL_POINTER_IDS;
                    // 清理 TouchTarget 中对历史遗留的 idBitsToAssign 的缓存
                    removePointersFromTouchTargets(idBitsToAssign);
                    final int childrenCount = mChildrenCount;
                    if (newTouchTarget == null && childrenCount != 0) {
                        // 2.2 获取事件在当前容器中的相对位置
                        final float x = ev.getX(actionIndex);
                        final float y = ev.getY(actionIndex);
                        // 获取当前容器前序遍历的 View 序列(即层级从高到低排序)
                        final ArrayList<View> preorderedList = buildTouchDispatchChildList();
                        // 2.3 遍历子 View, 找寻可以响应事件的目标
                        final boolean customOrder = preorderedList == null
                                && isChildrenDrawingOrderEnabled();
                        final View[] children = mChildren;
                        for (int i = childrenCount - 1; i >= 0; i--) {
                            // 通过索引找到子孩子实例
                            final int childIndex = getAndVerifyPreorderedIndex(
                                    childrenCount, i, customOrder);
                            final View child = getAndVerifyPreorderedView(
                                    preorderedList, children, childIndex);
                            ......
                            // 2.3.1 判断这个子 View 是否在响应事件的区域
                            if (!canViewReceivePointerEvents(child)
                                    || !isTransformedTouchPointInView(x, y, child, null)) {
                                ......
                                // 若不在可响应的区域, 则继续遍历下一个子 View
                                continue;
                            }
                            // 2.3.2 判断获取这个子 View, 是否已经响应了序列中的一个焦点
                            newTouchTarget = getTouchTarget(child);
                            if (newTouchTarget != null) {
                                // 若这个 View 已经响应了一个焦点, 则在它的 TouchTarget.pointerIdBits 添加新焦点的索引
                                newTouchTarget.pointerIdBits |= idBitsToAssign;
                                // 则直接结束查找, 直接进行后续的分发操作
                                break;
                            }
                            ......
                            // 2.3.3 调用 dispatchTransformedTouchEvent 尝试将这个焦点分发给这个 View 处理
                            if (dispatchTransformedTouchEvent(ev, false, child, idBitsToAssign)) {
                                ......
                                // 子 View 成功的消费了这个事件, 则将这个 View 封装成 TouchTarget 链到表头
                                newTouchTarget = addTouchTarget(child, idBitsToAssign);
                                // 表示这个事件在找寻新的响应目标时已经消费了
                                alreadyDispatchedToNewTouchTarget = true;
                                break;
                            }
                            ......
                        }
                        ......
                    }
                    // 2.4 若没有找到可以响应的子 View, 则交由最早的 TouchTarget 处理
                    if (newTouchTarget == null && mFirstTouchTarget != null) {
                        newTouchTarget = mFirstTouchTarget;
                        while (newTouchTarget.next != null) {
                            newTouchTarget = newTouchTarget.next;
                        }
                        // 在其 pointerIdBits 保存焦点的 ID 
                        newTouchTarget.pointerIdBits |= idBitsToAssign;
                    }
                }
            }
            // 3. 执行事件分发
            // 3.1 没有任何子 View 可以响应事件序列, 则交由自己处理
            if (mFirstTouchTarget == null) {
                handled = dispatchTransformedTouchEvent(ev, canceled, null,
                        TouchTarget.ALL_POINTER_IDS);
            } else {
                // 3.2 若存在子 View 响应事件序列, 则将这个事件分发下去
                TouchTarget predecessor = null;
                TouchTarget target = mFirstTouchTarget;
                while (target != null) {
                    final TouchTarget next = target.next;
                    // 3.2.1 alreadyDispatchedToNewTouchTarget 为 true 并且 target 为我们上面新找到的响应目标时, 跳过这次分发
                    // 因为在查找焦点处理者的过程中, 已经分发给这个 View 了
                    if (alreadyDispatchedToNewTouchTarget && target == newTouchTarget) {
                        handled = true;
                    } else {
                        // 3.2.2 处理还未消耗这个事件的子 View
                        final boolean cancelChild = resetCancelNextUpFlag(target.child)
                                || intercepted;// 判断是否需要给 View 发送 CANCEL 事件, view 设置了 PFLAG_CANCEL_NEXT_UP_EVENT 这个 Flag, 或者这个事件被该容器拦截了, 那么将会给子 View 发送 Cancel 事件
                        // 3.2.2 调用 dispatchTransformedTouchEvent 将事件分发给子 View 
                        if (dispatchTransformedTouchEvent(ev, cancelChild,
                                target.child, target.pointerIdBits)) {
                            handled = true;
                        }
                        // 3.2.3 若给这个 View 分发了 cancel 事件, 则说明它已经无法响应事件序列的焦点了, 因此将它从响应链表中移除
                        if (cancelChild) {
                            if (predecessor == null) {
                                mFirstTouchTarget = next;
                            } else {
                                predecessor.next = next;
                            }
                            target.recycle();
                            target = next;
                            continue;
                        }
                    }
                    predecessor = target;
                    target = next;
                }
            } 
            // 4. 清理失效的 TouchTarget
            // 4.1 若事件序列取消 或 整个事件 UP 了, 则移除所有的 TouchTarget
            if (canceled || actionMasked == MotionEvent.ACTION_UP || actionMasked == MotionEvent.ACTION_HOVER_MOVE) {
                resetTouchState();
            } else if (split && actionMasked == MotionEvent.ACTION_POINTER_UP) {
                // 4.2 若事件序列的一个焦点 UP 了, 则移除响应这个焦点的 TouchTarget
                final int actionIndex = ev.getActionIndex();
                final int idBitsToRemove = 1 << ev.getPointerId(actionIndex);
                removePointersFromTouchTargets(idBitsToRemove);
            }
        }
        ......
        return handled;
    }
    
}
```
好的, 可以看到上述的步骤非常复杂的, 其注释已经非常详细了, 总结下来其实就三点
- 若为初始事件或该容器能够处理该事件, 则尝试拦截
  - 若为初始事件或事件被他处理了, 则尝试拦截, 当然还与是否设置了 FLAG_DISALLOW_INTERCEPT 有关
- 若为初始事件或事件序列的新焦点, 则在容器内部找寻能够响应它的子 View
  - 通过 canViewReceivePointerEvents 和 isTransformedTouchPointInView 判断是否落在了 View 的区域内
- 若无响应目标, 则自行处理, 若存在响应目标, 则将事件分发给响应目标

以上过程有两个疑问
- **当事件的新焦点到来时, 它是如何找寻能够响应它的子 View 的, 仅仅根据坐标这么简单吗?**
- 在事件分发时, 我们看到 ViewGroup 遍历了所有的 TouchTarget 对它们逐一进行事件分发, **直接分发给这事件对应焦点的处理者不就可以了吗, 为什么要分发给事件序列所有焦点的处理者呢?**

我们先看看第一个问题

#### 1. 找寻事件的响应目标
从 ViewGroup 的 dispatchTouchEvent 中, 我们知道判断一个 View 是否可以响应事件主要有两个方法分别是 canViewReceivePointerEvents 和 isTransformedTouchPointInView, 这里我们一个一个探究
```java
public abstract class ViewGroup extends View implements ViewParent, ViewManager {
    
    private static boolean canViewReceivePointerEvents(@NonNull View child) {
        // Condition1: View 为 Visible
        // Condition2: 当前 View 设置了动画
        return (child.mViewFlags & VISIBILITY_MASK) == VISIBLE
                || child.getAnimation() != null;
    }
    
}
```
可以看到 canViewReceivePointerEvents 中, 若 View 是可见的 或 View 存在动画, 则说明它可以接收事件, 可以看到这个方法是根据 View 的状态来判断是否可以接收事件的, 具体是否能够接收事件还是得看后面一个方法

接下来我们看看 isTransformedTouchPointInView 这个方法做了哪些判断
```java
public abstract class ViewGroup extends View implements ViewParent, ViewManager {
    
    protected boolean isTransformedTouchPointInView(float x, float y, View child,
            PointF outLocalPoint) {
        // 1. 获取一个坐标数组, 数据为事件在当前 ViewGroup 中的相对坐标 x, y 值
        final float[] point = getTempPoint();
        point[0] = x;
        point[1] = y;
        // 2. 调用了 transformPointToViewLocal, 将坐标转为 Child 的相对坐标
        transformPointToViewLocal(point, child);
        // 3. 调用了 View.pointInView 判断坐标是否落在了 View 中
        final boolean isInView = child.pointInView(point[0], point[1]);
        // 若在子 View 中, 则尝试输出到 outLocalPoint 中, dispatchToucEvent 中传入的为 null
        if (isInView && outLocalPoint != null) {
            outLocalPoint.set(point[0], point[1]);
        }
        return isInView;
    }
    
     public void transformPointToViewLocal(float[] point, View child) {
        // 2.1 将 point 转为 View 的相对坐标
        point[0] += mScrollX - child.mLeft;
        point[1] += mScrollY - child.mTop;
        // 2.2 若 View 设置了 Matrix 变化, 则通过 Matrix 来映射这个坐标
        if (!child.hasIdentityMatrix()) {
            child.getInverseMatrix().mapPoints(point);
        }
    }
    
}

public class View implements Drawable.Callback, KeyEvent.Callback,
        AccessibilityEventSource {
    
    final boolean pointInView(float localX, float localY) {
        return pointInView(localX, localY, 0);
    }
    
    public boolean pointInView(float localX, float localY, float slop) {
        // 很简单, 即判断是否落在 View 的区域内, 小于 View 的宽度和高度
        // slop 为当前 Android 系统能够识别出来的手指区域的大小
        return localX >= -slop && localY >= -slop && localX < ((mRight - mLeft) + slop) &&
                localY < ((mBottom - mTop) + slop);
    }
    
}
```
好的, 可以看到 isTransformedTouchPointInView 中判断其是否可以响应事件的依据是判断事件的相对坐标是否落在了子 View 的宽高之内

不过**这里有一个细微的操作, 它会调用 child.getInverseMatrix().mapPoints(point); 来映射一次坐标, 这是为什么呢?**
- 因为我们在执行属性动画的时候, 有的时候会进行 View 的 transition, scale 等操作, 这些操作并不会改变 View 的原始坐标, 但会改变它内部的 RenderNode 的 Matrix, **进行坐标映射, 是为了让 View 在变化后的区域依旧可以响应事件流, 这就是为什么属性动画作用后的 View 依旧可以响应点击事件的原因**

好的, 分析完了如何判断子 View 是否可以响应事件之后, 接下来看一看 ViewGroup 事件流是如何分发的

#### 2. 将事件分发给子 View
在 ViewGroup.dispatchTouchEvent 中, 我们知道最后它会遍历 TouchTarget 链表, 逐个调用 dispatchTransformedTouchEvent 方法, 将一个事件, 分发给该事件所在序列的所有焦点处理者, 接下来我们就看看它是如何做的
```java
public abstract class ViewGroup extends View implements ViewParent, ViewManager {
    
    private boolean dispatchTransformedTouchEvent(MotionEvent event, boolean cancel,
            View child, int desiredPointerIdBits) {
        final boolean handled;
        // 获取事件的动作
        final int oldAction = event.getAction();
        // 1. 处理 Cancel 操作
        if (cancel || oldAction == MotionEvent.ACTION_CANCEL) {
            // 1.1 将事件的 Action 强行改为 ACTION_CANCEL
            event.setAction(MotionEvent.ACTION_CANCEL);
            // 1.2 ACTION_CANCEL 的分发操作
            if (child == null) {
                // 1.2.1 自己处理
                handled = super.dispatchTouchEvent(event);
            } else {
                // 1.2.2 分发给子 View 
                handled = child.dispatchTouchEvent(event);
            }
            event.setAction(oldAction);
            return handled;
        }
        
        // 2. 判断这个 child 是否可以响应该事件
        // 2.1 获取这个事件所在序列的所有焦点
        final int oldPointerIdBits = event.getPointerIdBits();
        // 2.2 desiredPointerIdBits 描述这个 child 能够处理的焦点
        final int newPointerIdBits = oldPointerIdBits & desiredPointerIdBits; 
        // 2.3 若他们之间没有交集, 则说明出现了异常情况, 这个 child 所响应的并非是这个事件序列中的焦点
        if (newPointerIdBits == 0) {
            return false;
        }
        
        // 3. 将事件分发给子 View 
        final MotionEvent transformedEvent;
        // 3.1 若这个子 View 能够处理这个事件序列的所有焦点, 则直接进行分发操作
        if (newPointerIdBits == oldPointerIdBits) {
            // 若不存子 View, 或者子 View 设置 Matrix 变幻, 则走下面的分支
            if (child == null || child.hasIdentityMatrix()) {
                if (child == null) {
                    // 3.1.1 自己处理
                    handled = super.dispatchTouchEvent(event);
                } else {
                    // 3.1.2 分发给子 View
                    ......
                    handled = child.dispatchTouchEvent(event);
                    ......
                }
                return handled;
            }
            transformedEvent = MotionEvent.obtain(event);
        } else {
            // 3.2 若这个子 View 只能够处理事件序列中部分的焦点, 则调用 MotionEvent.split 进行焦点分割
            transformedEvent = event.split(newPointerIdBits);
        }

        if (child == null) {
            // 3.2.1 不存在子 View 则自己处理
            handled = super.dispatchTouchEvent(transformedEvent);
        } else {
            // 3.2.2 将进行焦点分割后的事件, 分发给子 View
            ......
            handled = child.dispatchTouchEvent(transformedEvent);
        }
        ......
        return handled;
    }
    
}
```
可以看到, **只要 View 能够处理当前 MotionEvent 所在事件序列中的一个焦点, 便会分发给它**, 分发之前会调用 MotionEvent.split 将事件分割成为 View 能够处理的焦点事件, 并将其分发下去

```
E/TAG: --------------------dispatch begin----------------------
E/TAG: ViewGroup PointerId is: 0, eventRawX 531.7157, eventRawY 747.5201
E/TAG: Child PointerId is: 1, eventRawX 467.7539, eventRawY 1387.9688
E/TAG: Child PointerId is: 0, eventRawX 531.7157, eventRawY 747.5201
E/TAG: --------------------dispatch begin----------------------
E/TAG: ViewGroup PointerId is: 0, eventRawX 534.09283, eventRawY 744.18854
E/TAG: Child PointerId is: 1, eventRawX 467.7539, eventRawY 1387.9688
E/TAG: Child PointerId is: 0, eventRawX 534.09283, eventRawY 744.18854
```
从测试的打印日志可以看出, **ViewGroup 中的一个事件的确会分发给这个事件序列所有的焦点处理者, 可以看到 MotionEvent.split 方法, 不仅仅分割了焦点, 并且还巧妙的将触摸事件转换到了焦点处理 View 对应的区域, 这样一来就没有任何违和感了, Google 如此处理的原因可能是为了保证触摸过程中事件的连续性, 以实现更好的交互效果**

好的, 接下来我们看看 View 中是如何处理事件的

### 四) View 的事件分发
```java
public class View implements Drawable.Callback, KeyEvent.Callback,
        AccessibilityEventSource {
    
    public boolean dispatchTouchEvent(MotionEvent event) {
        ......
        boolean result = false;
        ......
        // 获取 Mask 后的 action
        final int actionMasked = event.getActionMasked();
        // 若为 DOWN, 则停止 Scroll 操作
        if (actionMasked == MotionEvent.ACTION_DOWN) {
            // Defensive cleanup for new gesture
            stopNestedScroll();
        }
        // 处理事件
        if (onFilterTouchEventForSecurity(event)) {
            // 1. 若是拖拽滚动条, 则优先将事件交给 handleScrollBarDragging 方法消耗
            if ((mViewFlags & ENABLED_MASK) == ENABLED && handleScrollBarDragging(event)) {
                result = true;
            }
            // 2. 若 ListenerInfo 中设置了 OnTouchListener, 则尝试将其交给 mOnTouchListener 消耗
            ListenerInfo li = mListenerInfo;
            if (li != null && li.mOnTouchListener != null
                    && (mViewFlags & ENABLED_MASK) == ENABLED
                    && li.mOnTouchListener.onTouch(this, event)) {
                result = true;
            }
            // 3. 若上面的操作没有将 result 位置 true, 则交给 onTouchEvent 消耗
            if (!result && onTouchEvent(event)) {
                result = true;
            }
        }
        ......
        return result;
    }
               
}
```
好的, 可以看到主要有三个操作
- 优先将事件交给 ScrollBar 处理
  - 成功消费则将 result 置为 true
- 次优先将事件交给 OnTouchListener 处理
  - 成功消费则将 result 置为 true
- 最后将事件交给 onTouchEvent 处理

好的, 可以看到这里 View 的事件分发比 ViewGroup 的要简单太多太多, 接下来我们看看 View 的事件处理

#### onTouchEvent 处理事件
```java
public class View implements Drawable.Callback, KeyEvent.Callback,
        AccessibilityEventSource {
            
    public boolean onTouchEvent(MotionEvent event) {
        final float x = event.getX();
        final float y = event.getY();
        final int viewFlags = mViewFlags;
        final int action = event.getAction();
        // 判断当前 View 是否是可点击的
        final boolean clickable = ((viewFlags & CLICKABLE) == CLICKABLE
                || (viewFlags & LONG_CLICKABLE) == LONG_CLICKABLE)
                || (viewFlags & CONTEXT_CLICKABLE) == CONTEXT_CLICKABLE;

        ......
        // 若设置了代理, 则交由代理处理
        if (mTouchDelegate != null) {
            if (mTouchDelegate.onTouchEvent(event)) {
                return true;
            }
        }
        // 执行事件处理
        if (clickable || (viewFlags & TOOLTIP) == TOOLTIP) {
            switch (action) {
                // 1. 处理按压事件
                case MotionEvent.ACTION_DOWN:
                    ......
                    // 处理 View 在可滑动容器中的按压事件
                    if (isInScrollingContainer) {
                        ......
                    } else {
                        // 处理在非滑动容器中的按压事件
                        // 1.1 设置为 Pressed 状态
                        setPressed(true, x, y);
                        // 1.2 尝试添加一个长按事件
                        checkForLongClick(0, x, y);
                    }
                    break;
                // 2. 处理移动事件
                case MotionEvent.ACTION_MOVE:
                    ......
                    // 处理若移动到了 View 之外的情况
                    if (!pointInView(x, y, mTouchSlop)) {
                        // 移除 TapCallback
                        removeTapCallback();
                        // 移除长按的 Callback
                        removeLongPressCallback();
                        // 清除按压状态
                        if ((mPrivateFlags & PFLAG_PRESSED) != 0) {
                            setPressed(false);
                        }
                        ......
                    }
                    break;
                // 3. 处理 UP 事件
                case MotionEvent.ACTION_UP:
                    // 3.1 若置为不可点击了, 则移除相关回调, 重置相关 Flag
                    if (!clickable) {
                        removeTapCallback();
                        removeLongPressCallback();
                        ......
                        break;
                    }
                    // 3.2 判断 UP 时, Viwe 是否处于按压状态
                    boolean prepressed = (mPrivateFlags & PFLAG_PREPRESSED) != 0;
                    if ((mPrivateFlags & PFLAG_PRESSED) != 0 || prepressed) {
                        ......
                        // 3.2.1 若没有触发长按事件, 则处理点击事件
                        if (!mHasPerformedLongPress && !mIgnoreNextUpEvent) {
                            removeLongPressCallback();// 移除长按回调
                            // 处理点击事件
                            if (!focusTaken) {
                                // 创建一个事件处理器
                                if (mPerformClick == null) {
                                    mPerformClick = new PerformClick();
                                }
                                // 发送到 MessageQueue 中执行
                                if (!post(mPerformClick)) {
                                    performClickInternal();
                                }
                            }
                        }
                        // 清除按压状态
                        ......
                    }
                    ......
                    break;
                // 4. 处理取消事件
                case MotionEvent.ACTION_CANCEL:
                    // 移除回调, 重置标记位
                    if (clickable) {
                        setPressed(false);
                    }
                    removeTapCallback();
                    removeLongPressCallback();
                    mInContextButtonPress = false;
                    mHasPerformedLongPress = false;
                    mIgnoreNextUpEvent = false;
                    break;
            }
            return true;
        }
        return false;
    }
                
}
```
好的, 可以看到 onTouchEvent 中也做了非常多的操作, 这里省略了一些代码, 我们主要看看它如何处理不同事件的
- ACTION_DOWN
  - 将 View 设置为按压状态 
  - 添加了一个长按监听器 
- ACTION_MOVE
  - 若坐标在 View 的范围之外了, 则移除相关回调, 清除按压的状态 
- ACTION_UP
  - 若长按事件没有响应, 则处理 View 的点击事件
  - 移除按压状态
- ACTION_CANCEL
  - 移除相关回调, 清除按压状态

好的, 至此 View 对事件的处理就分析完了

## 总结
至此窗体事件流的分发就到这里了, 这里回顾一下整个事件分发的流程
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
  - ViewRootImpl 中维护了一个输入事件队列, 这个事件最终会交给 InputStage 消费
  - InputStage 中会将事件传递给 DecorView
  - DecorView 首先会尝试将事件呈递给当前 Window 的 Callback 对象处理
  - 在 Activity 创建的 Window 则会先将事件发送给 Activity
  - 最终调用 View.dispatchTouchEvent 进入 View 的事件分发流程
    - ViewGroup 的事件分发
    - View 的事件分发

View 的事件分发, 是比较基础也是比较重要的一个知识点, 通过本次的分析, 我们理清了事件是如何从 ViewGroup 中分发到 View 的, 以及 View 中是如何处理事件的, 再加上之前 IMS 的知识, 我们对事件流从哪儿来, 从哪儿获取, 以及如何分发处理, 就有了一个比较全面的认知

到这里整个 Android 系统的事件分发也完成了, 它的具体流程图如下

![Android 系统事件分发流程图](https://i.loli.net/2019/11/20/1aHfUsijBbtQXOw.png)