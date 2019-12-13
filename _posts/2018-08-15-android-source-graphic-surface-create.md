---
title: Android 系统架构 —— Surface 的创建
permalink: android-source/graphic-surface-create
key: android-source-graphic-surface-create
tags: AndroidFramework
---

## 前言
通过上面一篇的分析, 我们知道: **当测量数据发生改变时, 调用 relayoutWindow 方法, 重新请求窗体大小**, 这里我们就看看它是如何重置窗体大小的

<!--more-->

```
public final class ViewRootImpl implements ViewParent,
        View.AttachInfo.Callbacks, ThreadedRenderer.DrawCallbacks {
    
    int mWidth;                     // 描述窗体的宽度
    int mHeight;                    // 描述窗体的高度
    boolean mFirst;                 // 描述是否第一次进行 performTraversals
    boolean mFullRedrawNeeded;
    public final Surface mSurface = new Surface();// 描述当前窗体绘制的画布
      
    private void performTraversals() {
        ......
        // 在请求了 layout, 并且窗体大小有可能变化的情况下做如下的判断
        // 2.1.1 当前 ViewRootImpl 中保存的宽高比起 DecorView 重新测量的宽高是否有变化
        // 2.1.2 宽度布局参数为 WRAP_CONTENT 的情况下, 比起之前是否有变化
        // 2.1.3 高度布局参数为 WRAP_CONTENT 的情况下, 比起之前是否有变化
        boolean windowShouldResize = layoutRequested && windowSizeMayChange
            && (
                    (mWidth != host.getMeasuredWidth() || mHeight != host.getMeasuredHeight()) || 
                    (lp.width == ViewGroup.LayoutParams.WRAP_CONTENT && frame.width() < desiredWindowWidth && frame.width() != mWidth) || 
                    (lp.height == ViewGroup.LayoutParams.WRAP_CONTENT && frame.height() < desiredWindowHeight && frame.height() != mHeight)
              );
        ......
        // 2.2 满足下列条件, 进入下面的分支, 重置 ViewRootImpl 的 Surface 画布
        if (mFirst || windowShouldResize || insetsChanged ||
                viewVisibilityChanged || params != null || mForceNextWindowRelayout) {
            ......
            boolean hadSurface = mSurface.isValid();
            try {
                ......
                // 2.3 调用了 relayoutWindow 重置 Surface
                relayoutResult = relayoutWindow(params, viewVisibility, insetsPending);
                ......
                if (!hadSurface) {
                    if (mSurface.isValid()) {
                        ......
                        newSurface = true;
                        // 用来描述是否需要将一个应用程序窗口的全部区域都重新绘制
                        mFullRedrawNeeded = true;
                        ......
                    }
                }
            }
        }
        ......
    }
    
}

public class Surface implements Parcelable {
    
    final Object mLock = new Object(); 
    long mNativeObject; 
    
    public Surface() {
    }
    
    public boolean isValid() {
        synchronized (mLock) {
            // 可以看见, 该 Surface 没有绑定 Native 数据, 它就是无效的
            if (mNativeObject == 0) return false;
            return nativeIsValid(mNativeObject);
        }
    }
    
}
```
好的, 可以看到当 DecorView 测量之后, 会判断窗体大小是否变更了, 若改变则会调用 relayoutWindow 来重置窗体, 可以看到 ViewRootImpl 中有一个 Surface 对象, 这个 **Surface 对象并没有绑定 Native 层的 Surface 因此它是无效的**, 也就是 hadSurface 为 fasle

接下来我们就来分析如何通过 relayoutWindow 重置窗体

## 一. 请求重置窗体
```
public final class ViewRootImpl implements ViewParent,
        View.AttachInfo.Callbacks, ThreadedRenderer.DrawCallbacks {
        
    final IWindowSession mWindowSession;
    public final Surface mSurface = new Surface();

    private int relayoutWindow(WindowManager.LayoutParams params, int viewVisibility,
            boolean insetsPending) throws RemoteException {

        ......
        // 通过 mWindowSession 请求 WMS 进行 relayout 操作
        int relayoutResult = mWindowSession.relayout(mWindow, mSeq, params,
                (int) (mView.getMeasuredWidth() * appScale + 0.5f),
                (int) (mView.getMeasuredHeight() * appScale + 0.5f), viewVisibility,
                insetsPending ? WindowManagerGlobal.RELAYOUT_INSETS_PENDING : 0, frameNumber,
                mWinFrame, mPendingOverscanInsets, mPendingContentInsets, mPendingVisibleInsets,
                mPendingStableInsets, mPendingOutsets, mPendingBackDropFrame, mPendingDisplayCutout,
                mPendingMergedConfiguration, mSurface);
        ......
        return relayoutResult;
    }
    
}
```
通过 ViewRootImpl 与 WMS 的关系中, 我们可知, 这个 mWindowSession 对应 Binder 实体对象为系统服务进程的 Session 类, 因此我去去系统服务进程中查看
```
class Session extends IWindowSession.Stub implements IBinder.DeathRecipient {
    
    @Override
    public int relayout(IWindow window, int seq, WindowManager.LayoutParams attrs,
            int requestedWidth, int requestedHeight, int viewFlags, int flags, long frameNumber,
            Rect outFrame, Rect outOverscanInsets, Rect outContentInsets, Rect outVisibleInsets,
            Rect outStableInsets, Rect outsets, Rect outBackdropFrame,
            DisplayCutout.ParcelableWrapper cutout, MergedConfiguration mergedConfiguration,
            Surface outSurface) {
        ......
        // 通过 WMS 重置窗体大小
        int res = mService.relayoutWindow(this, window, seq, attrs,
                requestedWidth, requestedHeight, viewFlags, flags, frameNumber,
                outFrame, outOverscanInsets, outContentInsets, outVisibleInsets,
                outStableInsets, outsets, outBackdropFrame, cutout,
                mergedConfiguration, outSurface);
        ......
        return res;
    }
    
}

public class WindowManagerService extends IWindowManager.Stub
        implements Watchdog.Monitor, WindowManagerPolicy.WindowManagerFuncs {
     
    public int relayoutWindow(......Surface outSurface) {
        int result = 0;
        boolean configChanged;
        synchronized(mWindowMap) {
            // 1. 获取窗体描述, 在 ViewRootImpl 与 WMS 的关系中, 分析它是如何创建的
            WindowState win = windowForClientLocked(session, client, false);
            if (win == null) {
                return 0;
            }
            // 2. 获取 WindowState 的成员变量 mWinAnimator
            // 它用于跟踪 WindowState 的动画和 Surface 画布操作
            WindowStateAnimator winAnimator = win.mWinAnimator;
             ......
            // 判断是否需要重置 Window 大小
            final boolean shouldRelayout = viewVisibility == View.VISIBLE &&
                    (win.mAppToken == null || win.mAttrs.type == TYPE_APPLICATION_STARTING
                            || !win.mAppToken.isClientHidden());
            ......
            if (shouldRelayout) {
                try {
                    // 3. 创建画布, 并且输出到 outSurface 引用中
                    result = createSurfaceControl(outSurface, result, win, winAnimator);
                } catch (Exception e) {
                    ......
                    return 0;
                }
            }
            ......
        }
        return result;     
    }
    
    private int createSurfaceControl(Surface outSurface, int result, WindowState win,
            WindowStateAnimator winAnimator) {
        ......
        WindowSurfaceController surfaceController;
        try {
            ......
            // 3.1 创建通过 winAnimator 来创建一个画布控制器
            surfaceController = winAnimator.createSurfaceLocked(win.mAttrs.type, win.mOwnerUid);
        } finally {
            ......
        }
        if (surfaceController != null) {
            // 3.2 使用画布控制器将画布输出到 outSurface 中
            surfaceController.getSurface(outSurface);
            ......
        } else {
            ......
        }
        return result;
    }
                
}
```
好的, 可以看到 Session.relayout 最终会调用到 WindowManagerService.relayoutWindow 中, 它内部的实现非常复杂, 我们只关注如何重置画布大小
- 首先获取了我们客户端在 WMS 中窗体的描述 WindowState
- 获取 WindowState 的成员变量 mWinAnimator
  - WindowStateAnimator 用于监控 Window 的动画和 Surface 画布变化
- 调用 WindowManagerService.createSurfaceControl 将创建的 Surface 输出到 outSurface 中
  - **调用 WindowStateAnimator.createSurfaceLocked 获取 surfaceController**
  - **通过控制器的 WindowSurfaceController.getSurface 将数据写入到 outSurface 中** 

接下来我们一步一步分析, 先看看如何 SurfaceControl 的创建流程

## 二.  创建 SurfaceControl
先看看通过 WindowStateAnimator.createSurfaceLocked 获取画布控制器的流程
```
class WindowStateAnimator {

    final WindowState mWin;
    WindowSurfaceController mSurfaceController;
    
    WindowStateAnimator(final WindowState win) {
        ......
        mWin = win;
        ......
    }
    
    WindowSurfaceController createSurfaceLocked(int windowType, int ownerUid) {
        
        final WindowState w = mWin;
        if (mSurfaceController != null) {
            return mSurfaceController;
        }
        try {
            ......
            mSurfaceController = new WindowSurfaceController(mSession.mSurfaceSession,
                    attrs.getTitle().toString(), width, height, format, flags, this,
                    windowType, ownerUid);
        } catch(...) {
            ......
        }
        return mSurfaceController;
    }
    
}
```
可以看到, 内部 new 了一个 WindowSurfaceController 对象, 看看它内部做了些什么
```
class WindowSurfaceController {
    
    final WindowStateAnimator mAnimator;

    SurfaceControl mSurfaceControl;
    
    public WindowSurfaceController(SurfaceSession s, String name, int w, int h, int format,
            int flags, WindowStateAnimator animator, int windowType, int ownerUid) {
        ......
        // 使用 SurfaceControl.Builder 对象将 Surface 相关参数传入, 构建了一个 SurfaceControl 对象
        final WindowState win = animator.mWin;
        final SurfaceControl.Builder b = win.makeSurface()
                .setParent(win.getSurfaceControl())
                .setName(name)
                .setSize(w, h)
                .setFormat(format)
                .setFlags(flags)
                .setMetadata(windowType, ownerUid);
        mSurfaceControl = b.build();
    }
    
}
```
可以看到 WindowSurfaceController 内部维护了一个 SurfaceControl, 它才是真正的画布控制器

接下来我们看看这个 SurfaceControl.Builder.build() 在构建 SurfaceControl 对象时, 做了哪些操作

### 一) SurfaceControl 的创建
```
public class SurfaceControl implements Parcelable {
    
    public static class Builder {
      
        ......
        
        public SurfaceControl build() {
            ......
            return new SurfaceControl(mSession, mName, mWidth, mHeight, mFormat,
                    mFlags, mParent, mWindowType, mOwnerUid);
        }
    }
    
    long mNativeObject; // package visibility only for Surface.java access

    private SurfaceControl(SurfaceSession session, String name, int w, int h, int format, int flags,
            SurfaceControl parent, int windowType, int ownerUid)
                    throws OutOfResourcesException, IllegalArgumentException {
        // 将 Session 等参数传入, 构建 native 层对应的 SurfaceControl
        mNativeObject = nativeCreate(session, name, w, h, format, flags,
            parent != null ? parent.mNativeObject : 0, windowType, ownerUid);
        ......
    }
    
    private static native long nativeCreate(SurfaceSession session, String name,
            int w, int h, int format, int flags, long parentObject, int windowType, int ownerUid)
            throws OutOfResourcesException;
            
}
```
好的, 可以看到 java 层的 SurfaceControl 是一层薄封装, 我们去看看它 native 层对应的 SurfaceControl 的构建过程

```
// frameworks/base/core/jni/android_view_SurfaceControl.cpp

static jlong nativeCreate(JNIEnv* env, jclass clazz, jobject sessionObj,
        jstring nameStr, jint w, jint h, jint format, jint flags, jlong parentObject,
        jint windowType, jint ownerUid) {
    .......
    // 1. 从 SurfaceSession 对象中取出之前创建的那个 SurfaceComposerClient 对象
    sp<SurfaceComposerClient> client(android_view_SurfaceSession_getClient(env, sessionObj));
    ......
    // 2. 通过 SurfaceComposerClient.createSurfaceChecked 创建一个新的 SurfaceControl 对象, 保存在强引用 surface 中
    sp<SurfaceControl> surface;
    status_t err = client->createSurfaceChecked(
            String8(name.c_str()), w, h, format, &surface, flags, parent, windowType, ownerUid);
    ......
    surface->incStrong((void *)nativeCreate);// 增加强引用计数
    // 3. 将对象转成句柄返回给 Java 层
    return reinterpret_cast<jlong>(surface.get());
}
```
其中的操作主要如下
- 首先通过 Java 中的 SurfaceSession 获取一个 Native 层的 SurfaceComposerClient 对象
  -  SurfaceComposerClient 对象是 SurfaceSession 在 Native 的映射, 承担着与 SurfaceFlinger 交互的作用
  -  在 ViewRootImpl 与 WMS 交互中已经分析过了
- 调用 SurfaceComposerClient.createSurfaceChecked 创建了一个 SurfaceControl 对象
- 将对象转为句柄返回给 Java 层

好的, 可以看到重点在 SurfaceComposerClient.createSurfaceChecked 这个方法, 我们继续往下分析
```
class SurfaceComposerClient : public RefBase
{
public:
    SurfaceComposerClient(const sp<ISurfaceComposerClient>& client);
                
private:
    sp<ISurfaceComposerClient>  mClient;
};

// frameworks/native/libs/gui/SurfaceComposerClient.cpp
SurfaceComposerClient::SurfaceComposerClient(const sp<ISurfaceComposerClient>& client)
    : mStatus(NO_ERROR), mClient(client)
{}

status_t SurfaceComposerClient::createSurfaceChecked(......, sp<SurfaceControl>* outSurface, ......)
{
    sp<SurfaceControl> sur;
    status_t err = mStatus;

    if (mStatus == NO_ERROR) {
        sp<IBinder> handle;
        sp<IBinder> parentHandle;
        // 1. 调用 BpSurfaceComposerClient.createSurface, 请求 SurfaceFlinger 进程创建一个 IGraphicBufferProducer 对象
        sp<IGraphicBufferProducer> gbp;
        err = mClient->createSurface(name, w, h, format, flags, parentHandle,
                windowType, ownerUid, &handle, &gbp);
        if (err == NO_ERROR) {
            // 2. 将 IGraphicBufferProducer 传入 SurfaceControl 的构造函数, 构建一个 SurfaceControl 对象
            *outSurface = new SurfaceControl(this, handle, gbp, true /* owned */);
        }
    }
    return err;
}
```
好的, 可见构造 SurfaceControl 对象主要有两步操作
- 调用 BpSurfaceComposerClient.createSurface, 通过 SurfaceFlinger 进程获取一个 **IGraphicBufferProducer** 代理对象
- 创建一个 SurfaceControl 对象, 并且将 IGraphicBufferProducer 保存在 mGraphicBufferProducer 中

**这个 IGraphicBufferProducer 代理对象非常的重要, 它描述一个 GraphicBuffer 的生产者, 只有持有 IGraphicBufferProducer 才具有生产 Buffer 的能力**

因此我们看看 SurfaceFlinger 进程 createSurface 是如何创建 IGraphicBufferProducer 对象的

###  二) 创建 IGraphicBufferProducer
```
// frameworks/native/services/surfaceflinger/Client.cpp
 status_t Client::createSurface(
        const String8& name,
        uint32_t w, uint32_t h, PixelFormat format, uint32_t flags,
        const sp<IBinder>& parentHandle, int32_t windowType, int32_t ownerUid,
        sp<IBinder>* handle,
        sp<IGraphicBufferProducer>* gbp)
{
    ......
    class MessageCreateLayer : public MessageBase {
        ......
    public:
        ......
        virtual bool handler() {
            // 调用了 SurfaceFlinger 的 createLayer 方法
            result = flinger->createLayer(name, client, w, h, format, flags,
                    windowType, ownerUid, handle, gbp, parent);
            return true;
        }
    };
    // 投递到 SurfaceFlinger 的消息队列中执行
    sp<MessageBase> msg = new MessageCreateLayer(mFlinger.get(),
            name, this, w, h, format, flags, handle,
            windowType, ownerUid, gbp, &parent);
    mFlinger->postMessageSync(msg);
    return static_cast<MessageCreateLayer*>( msg.get() )->getResult();
}
```
很简单, 它将创建操作转发到了 SurfaceFlinger 的 createLayer 方法, 因此我们可以猜想, 我们的一个 SurfaceControl 对应 SurfaceFlinger 端的一个 Layer 对象

下面看看 Layer 的创建流程
```
// frameworks/native/services/surfaceflinger/SurfaceFlinger.cpp
status_t SurfaceFlinger::createLayer(
        const String8& name,
        const sp<Client>& client,
        uint32_t w, uint32_t h, PixelFormat format, uint32_t flags,
        int32_t windowType, int32_t ownerUid, sp<IBinder>* handle,
        sp<IGraphicBufferProducer>* gbp, sp<Layer>* parent)
{
    status_t result = NO_ERROR;
    sp<Layer> layer;
    String8 uniqueName = getUniqueLayerName(name);
    // 根据 Type 创建 Layer
    switch (flags & ISurfaceComposerClient::eFXSurfaceMask) {
        // 创建一个 Buffer 的 Layer
        case ISurfaceComposerClient::eFXSurfaceNormal:
            result = createBufferLayer(client,
                    uniqueName, w, h, flags, format,
                    handle, gbp, &layer);
            break;
        // 创建一个 Color 的 Layer
        case ISurfaceComposerClient::eFXSurfaceColor:
            result = createColorLayer(client,
                    uniqueName, w, h, flags,
                    handle, &layer);
            break;
        default:
            result = BAD_VALUE;
            break;
    }
    ......
    // 添加到 ClientLayer
    result = addClientLayer(client, *handle, *gbp, layer, *parent);
    if (result != NO_ERROR) {
        return result;
    }
    ......
    return result;
}
```
可以看到 SurfaceFlinger 支持两种类型的 Layer, 一种是 Buffer 类型的, 一种是 Color 类型, 创建完成之后会调用 addClientLayer 添加到缓存

这里我们先看看 Buffer 类型的 Layer 创建

#### 1. 创建 BufferLayer
```
// frameworks/native/services/surfaceflinger/SurfaceFlinger.cpp
status_t SurfaceFlinger::createBufferLayer(const sp<Client>& client,
        const String8& name, uint32_t w, uint32_t h, uint32_t flags, PixelFormat& format,
        sp<IBinder>* handle, sp<IGraphicBufferProducer>* gbp, sp<Layer>* outLayer)
{
    // initialize the surfaces
    switch (format) {
    case PIXEL_FORMAT_TRANSPARENT:
    case PIXEL_FORMAT_TRANSLUCENT:
        format = PIXEL_FORMAT_RGBA_8888;
        break;
    case PIXEL_FORMAT_OPAQUE:
        format = PIXEL_FORMAT_RGBX_8888;
        break;
    }
    // 创建了一个 BufferLayer
    sp<BufferLayer> layer = new BufferLayer(this, client, name, w, h, flags);
    // 注入相关数据
    status_t err = layer->setBuffers(w, h, format, flags);
    if (err == NO_ERROR) {
        // 为传出参数赋值
        *handle = layer->getHandle();
        *gbp = layer->getProducer();
        *outLayer = layer;
    }
    ......
    return err;
}
```
可以看到这里创建了一个 [BufferLayer](http://androidxref.com/9.0.0_r3/xref/frameworks/native/services/surfaceflinger/BufferLayer.cpp) 对象
```
// frameworks/native/services/surfaceflinger/BufferLayer.cpp
BufferLayer::BufferLayer(SurfaceFlinger* flinger, const sp<Client>& client, const String8& name,
                         uint32_t w, uint32_t h, uint32_t flags)
      : Layer(flinger, client, name, w, h, flags),
        mConsumer(nullptr),
        mTextureName(UINT32_MAX),
        mFormat(PIXEL_FORMAT_NONE),
        mCurrentScalingMode(NATIVE_WINDOW_SCALING_MODE_FREEZE),
        mBufferLatched(false),
        mPreviousFrameNumber(0),
        mUpdateTexImageFailed(false),
        mRefreshPending(false) {
    // 初始化 GL 的外部纹理
    mFlinger->getRenderEngine().genTextures(1, &mTextureName);
    mTexture.init(Texture::TEXTURE_EXTERNAL, mTextureName);
    ......
}
```
BufferLayer 的创建非常简单, 可以看到其内部持有一个 GL 的外部纹理, 用于承载当前 Layer 的渲染数据

BufferLayer 是支持智能指针的, 因此它被引用时, 会立即回调 onFirstRef 函数, 其实现如下
```
// frameworks/native/services/surfaceflinger/BufferLayer.cpp
void BufferLayer::onFirstRef() {
    // Creates a custom BufferQueue for SurfaceFlingerConsumer to use
    sp<IGraphicBufferProducer> producer;
    sp<IGraphicBufferConsumer> consumer;
    // 1. 创建 BufferQueue
    BufferQueue::createBufferQueue(&producer, &consumer, true);
    // 2. 创建 IGraphicBufferProducer 的包装类
    mProducer = new MonitoredProducer(producer, mFlinger, this);
    // 3. 创建 IGraphicBufferConsumer 的包装类
    mConsumer = new BufferLayerConsumer(consumer,
            mFlinger->getRenderEngine(), mTextureName, this);
    mConsumer->setConsumerUsageBits(getEffectiveUsage(0));
    mConsumer->setContentsChangedListener(this);
    mConsumer->setName(mName);
    // 4. 判断是否支持 3 缓存机制
    if (mFlinger->isLayerTripleBufferingDisabled()) {
        // 将应用进程的最大缓存获取置为 2 个
        mProducer->setMaxDequeuedBufferCount(2);
    }
    ......
}
```
onFirstRef 中的实现是重中之重了
- 创建生产者消费者队列 BufferQueue
- 创建 IGraphicBufferProducer 包装类 MonitoredProducer, 返回给客户端
  - 默认为三级缓冲, 不支持则置为二级缓冲 
- 创建 IGraphicBufferConsumer 包装类 BufferLayerConsumer

从这里我们就可以清楚的看到生产者消费者模型了, BufferLayer 即 SurfaceFlinger 进程的消费者, 而我们的 Surface 为其他进程的生产者, 而且代码中很好的体现了 3 缓冲机制, 它是 Android 4.1 升级的重点, 解决了生产端 Buffer 不足引发的 Jank 问题, 这里就不展开叙述了

接下来我们看看这个 Layer 的缓存操作

#### 2. 添加 Layer 到缓存
```
// frameworks/native/services/surfaceflinger/SurfaceFlinger.h
static const size_t MAX_LAYERS = 4096; 

// frameworks/native/services/surfaceflinger/SurfaceFlinger.cpp
status_t SurfaceFlinger::addClientLayer(const sp<Client>& client,
        const sp<IBinder>& handle,
        const sp<IGraphicBufferProducer>& gbc,
        const sp<Layer>& lbc,
        const sp<Layer>& parent)
{
    // add this layer to the current state list
    {
        Mutex::Autolock _l(mStateLock);
        if (mNumLayers >= MAX_LAYERS) {
            ......
            return NO_MEMORY;
        }
        if (parent == nullptr) {
            // 按照 Z 层排序, 添加到 mCurrentState 中缓存
            mCurrentState.layersSortedByZ.add(lbc);
        } else {
            ......
            // 若存在父 Layer, 则添加到父 Layer 的中缓存
            parent->addChild(lbc);
        }

        if (gbc != nullptr) {
            // 将生产者 Binder 对象缓存在 mGraphicBufferProducerList 中
            mGraphicBufferProducerList.insert(IInterface::asBinder(gbc).get());
            ......
        }
        mLayersAdded = true;
        // 更新 Layer 数量
        mNumLayers++;
    }
    // 缓存到 Client 中
    client->attachLayer(handle, lbc);
    return NO_ERROR;
}
```
Layer 的缓存流程也比较简单, SurfaceFlinger 主要做了两方面的缓存
- 缓存 Layer
  - 不存在父 Layer, 按照 Z 轴排序, 添加到 mCurrentState 中 
  - 存在父 Layer, 添加到父 Layer 中缓存
- 缓存 Layer 的生产者到 mGraphicBufferProducerList

而且我们看到 **SurfaceFlinger 支持的最大 Layer 数量为 4096 个**, 超过会返回内存不足的错误, 当然我们很难达到这个数值

### 三) 回顾
![SurfaceControl 与 Layer](https://i.loli.net/2019/12/13/Ji2y1VgDeG38Yvu.png)

- 系统服务进程创建 SurfaceControl
- SurfaceFlinger 创建 GraphicBufferProducer
  - 创建 BufferLayer
    - 创建 GraphicBuffer 队列
    - 创建队列生产者 mProducer
    - 创建队列消费者 mConsumer
      - Android 4.1 之后默认支持获取 3 个 GraphicBuffer 缓冲
  - 缓存 BufferLayer
    - 最大 Layer 数量为 4096 
- 将 mProducer 的 Binder 代理对象保存到 SurfaceControl 中


到这里 SurfaceControl 就准备好了, 接下来我们看看 WindowSurfaceController.getSurface 如何将构建好的 SurfaceControl 数据输出到 outSurface 中

## 三. 将数据写入 outSurface
```
public class WindowManagerService extends IWindowManager.Stub
        implements Watchdog.Monitor, WindowManagerPolicy.WindowManagerFuncs {
     
    ......
    
    private int createSurfaceControl(Surface outSurface, int result, WindowState win,
            WindowStateAnimator winAnimator) {
        ......
        WindowSurfaceController surfaceController;
        try {
            ......
            // 上面详细分析过如何构建画布控制器
            surfaceController = winAnimator.createSurfaceLocked(win.mAttrs.type, win.mOwnerUid);
        } finally {
            ......
        }
        if (surfaceController != null) {
            // 使用画布控制器将画布输出到 outSurface 中
            surfaceController.getSurface(outSurface);
            ......
        } else {
            ......
        }
        return result;
    }
                
}

class WindowSurfaceController {

    void getSurface(Surface outSurface) {
        outSurface.copyFrom(mSurfaceControl);
    }
    
}
```
很简单, WindowSurfaceController.getSurface 方法内部调用了 Surface.copyFrom
```
public class Surface implements Parcelable {

    final Object mLock = new Object(); // protects the native state
    private String mName;
    long mNativeObject;                // package scope only for SurfaceControl access

    public void copyFrom(SurfaceControl other) {
        ......
        // 1. 获取画布控制器 native 对应的句柄值
        long surfaceControlPtr = other.mNativeObject;
        ......
        // 2. 通过这个 surfaceControlPtr 句柄指代的 SurfaceControl(Native) 构建一个新的 Surface 对象(Native)句柄
        long newNativeObject = nativeGetFromSurfaceControl(surfaceControlPtr);

        synchronized (mLock) {
            // 3. 释放当前 Surface 内部的数据
            if (mNativeObject != 0) {
                nativeRelease(mNativeObject);
            }
            // 4. 链接上 native 层新创建的数据
            setNativeObjectLocked(newNativeObject);
        }
    }
    
    private void setNativeObjectLocked(long ptr) {
        if (mNativeObject != ptr) {
            ......
            mNativeObject = ptr;
            ......
        }
    }
    
}
```
可见 Surface.copyFrom 中的操作也非常的简单, 主要有以下几步
- 获取 SurfaceControl 控制器中数据的句柄值 SurfaceControl.mNativeObject
- 将 SurfaceControl.mNativeObject 传入 nativeGetFromSurfaceControl 方法, 创建 Surface 的 Native 对象
- 若该 Surface 之前存在 Native 对象, 则释放之前的 mNativeObject
- 链接新创建的数据对象

我们这里主要关心一下如何通过 nativeGetFromSurfaceControl 创建 Native 的 Surface 对象

### 创建 Surface Native 对象
```
// frameworks/base/core/jni/android_view_Surface.cpp
static jlong nativeGetFromSurfaceControl(JNIEnv* env, jclass clazz,
        jlong surfaceControlNativeObj) {
    // 获取 SurfaceControl 对象
    sp<SurfaceControl> ctrl(reinterpret_cast<SurfaceControl *>(surfaceControlNativeObj));
    // 通过 SurfaceControl.getSurface 获取一个 Surface
    sp<Surface> surface(ctrl->getSurface());
    if (surface != NULL) {
        // 增加一个强引用计数
        surface->incStrong(&sRefBaseOwner);
    }
    // 将这个 surface 转为句柄值返回给 java 端
    return reinterpret_cast<jlong>(surface.get());
}

// frameworks/native/libs/gui/SurfaceControl.cpp
sp<Surface> SurfaceControl::getSurface() const
{
    Mutex::Autolock _l(mLock);
    if (mSurfaceData == 0) {
        return generateSurfaceLocked();
    }
    return mSurfaceData;
}

sp<Surface> SurfaceControl::generateSurfaceLocked() const
{
    // 可以看到这里 new 了一个 Surface 对象
    mSurfaceData = new Surface(mGraphicBufferProducer, false);
    return mSurfaceData;
}

// frameworks/native/libs/gui/Surface.cpp
Surface::Surface(const sp<IGraphicBufferProducer>& bufferProducer, bool controlledByApp)
      : mGraphicBufferProducer(bufferProducer)// 保存了 SurfaceFlinger 分配的数据缓存区
       ...... {}
```
Native 层的 nativeGetFromSurfaceControl 最终会调用到 SurfaceControl.generateSurfaceLocked 函数, 这里创建了一个 Native 的 Surface, **其 mGraphicBufferProducer 变量也保存了数据缓冲的描述**

至此 Java 层的 outSurface 中就可以获取到创建的句柄值了, 之后便通过 Binder 驱动将 outSurface 拷贝回应用进程了

## 总结
![Surface 创建示意图](https://i.loli.net/2019/12/13/Vz5iMlC8NQvFG6W.png)

当客户端 View 测量的数据导致窗体大小变更时, 会调用 relayout 通知 WMS 中窗体的尺寸变更了, 其整个流程如下
- 系统服务进程创建 SurfaceControl 对象
- SurfaceFlinger 创建 GraphicBufferProducer
  - 创建 BufferLayer
    - 创建 GraphicBuffer 队列
    - 创建队列生产者 mProducer
    - 创建队列消费者 mConsumer
      - Android 4.1 之后默认支持获取 3 个 GraphicBuffer 缓冲
  - 缓存 BufferLayer
    - 最大 Layer 数量为 4096 
- 将 mProducer 的 Binder 代理对象保存到 SurfaceControl 中
- 系统服务进程将 SurfaceControl 中的 IGraphicBufferProducer 拷贝会应用进程的 Surface 中

至此我们应用进程的 Surface 就有存储渲染数据, 以及将渲染数据推送到 SurfaceFlinger 的能力