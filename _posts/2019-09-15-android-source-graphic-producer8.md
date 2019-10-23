---
layout: article
title: "Android 系统架构 —— 图形架构篇 之 View 的硬件渲染"
key: "Android 系统架构 —— 图形架构篇 之 View 的硬件渲染" 
tags: AndroidFramework
aside:
  toc: true
---
# 前言
上一篇文章中我们学习了软件绘制的过程, 知晓了它是如何通过 Canvas 将 View 绘制到 Surface 的 GraphicBuffer 中的

<!--more-->

这里我们聚焦于硬件绘制, 看看它的工作流程, 先回顾一下硬件加速绘制的时机
```
public final class ViewRootImpl implements ViewParent,
        View.AttachInfo.Callbacks, ThreadedRenderer.DrawCallbacks {
    
    ......
    
    private boolean draw(boolean fullRedrawNeeded) {
        ......
        final Rect dirty = mDirty;
        if (!dirty.isEmpty() || mIsAnimating || accessibilityFocusDirty) {
            // 若开启了硬件加速, 则使用 OpenGL 的 ThreadedRenderer 进行绘制
            if (mAttachInfo.mThreadedRenderer != null && mAttachInfo.mThreadedRenderer.isEnabled()) {
                ......
                mAttachInfo.mThreadedRenderer.draw(mView, mAttachInfo, this, callback);
            } else {
                // ......软件绘制
            }
        }
        .......
    }
}
```
从 ViewRootImpl.draw 的代码中, 我们看到硬件绘制只有在 mAttachInfo 的 mThreadedRenderer 有效的情况下才会执行

因此在想了解硬件绘制流程之前, 需要搞清楚 mThreadedRenderer 它是如何创建和初始化的

## 一. ThreadedRenderer 的创建
关于硬件绘制的开启, 这需要从 ViewRootImpl 的创建说起

ViewRootImpl 是用于管理 View 树与其依附 Window 的一个媒介, 当我们调用 WindowManager.addView 时便会创建一个 ViewRootImpl 来管理即将添加到 Window 中的 View
```
public final class WindowManagerGlobal {

    public void addView(View view, ViewGroup.LayoutParams params,
            Display display, Window parentWindow) {
        ViewRootImpl root;
        synchronized (mLock) {
            // 创建了 ViewRootImpl 的实例
            root = new ViewRootImpl(view.getContext(), display);
            view.setLayoutParams(wparams);
            // 添加到缓存中
            mViews.add(view);
            mRoots.add(root);
            mParams.add(wparams);
            try {
                // 将要添加的视图添加到 ViewRootImpl 中
                root.setView(view, wparams, panelParentView);
            } catch (RuntimeException e) {
                ......
            }
        }
    }
    
}

public final class ViewRootImpl implements ViewParent,
        View.AttachInfo.Callbacks, ThreadedRenderer.DrawCallbacks {
        
    final View.AttachInfo mAttachInfo;
    
    public ViewRootImpl(Context context, Display display) {
        ......
        // 构建了一个 View.AttachInfo 用于描述这个 View 树与其 Window 的依附关系
        mAttachInfo = new View.AttachInfo(mWindowSession, mWindow, display, this, mHandler, this,
                context);
        ......
    }
    
    public void setView(View view, WindowManager.LayoutParams attrs, View panelParentView) {
        synchronized (this) {
            if (mView == null) {
                 // DecorView 实现了 RootViewSurfaceTaker 接口
                 if (view instanceof RootViewSurfaceTaker) {
                    // 尝试获取 callback
                    mSurfaceHolderCallback =
                            ((RootViewSurfaceTaker)view).willYouTakeTheSurface();
                    // callback 不为 null, 则创建 SurfaceHolder
                    if (mSurfaceHolderCallback != null) {
                        mSurfaceHolder = new TakenSurfaceHolder();
                        mSurfaceHolder.setFormat(PixelFormat.UNKNOWN);
                        mSurfaceHolder.addCallback(mSurfaceHolderCallback);
                    }
                }
                // 若 mSurfaceHolder 为 null, 也就是说 DecorView.willYouTakeTheSurface 为 null, 则开启硬件加速
                if (mSurfaceHolder == null) {
                    enableHardwareAcceleration(attrs);
                }
            }
        }
    }
    
    private void enableHardwareAcceleration(WindowManager.LayoutParams attrs) {
        mAttachInfo.mHardwareAccelerated = false;
        mAttachInfo.mHardwareAccelerationRequested = false;
        ......
        final boolean hardwareAccelerated =
                (attrs.flags & WindowManager.LayoutParams.FLAG_HARDWARE_ACCELERATED) != 0;

        if (hardwareAccelerated) {
            ......
            if (fakeHwAccelerated) {
                ......
            } else if (!ThreadedRenderer.sRendererDisabled
                    || (ThreadedRenderer.sSystemRendererDisabled && forceHwAccelerated)) {
                ......
                // 创建硬件加速的渲染器
                mAttachInfo.mThreadedRenderer = ThreadedRenderer.create(mContext, translucent,
                        attrs.getTitle().toString());
                mAttachInfo.mThreadedRenderer.setWideGamut(wideGamut);
                if (mAttachInfo.mThreadedRenderer != null) {
                    mAttachInfo.mHardwareAccelerated =
                            mAttachInfo.mHardwareAccelerationRequested = true;
                }
            }
        }
    }
}
```
好的, 这个硬件加速渲染器是通过 **WindowManager.LayoutParams.FLAG_HARDWARE_ACCELERATED** 来决定的, 最终会调用 ThreadedRenderer.create 来创建一个 ThreadedRenderer 对象

他会给 mAttachInfo 的 mThreadedRenderer 属性创建一个渲染器线程的描述

```
public final class ThreadedRenderer {

    private long mNativeProxy;   // 描述渲染代理对象 Native 句柄值
    private RenderNode mRootNode;// 描述当前窗体的根渲染器

    ThreadedRenderer(Context context, boolean translucent, String name) {
        ......
        // 构建了一个根 RenderNode, 它表示当前窗体的根渲染器
        long rootNodePtr = nCreateRootRenderNode();
        mRootNode = RenderNode.adopt(rootNodePtr);
        // 创建了一个 RenderProxy 渲染代理对象
        mNativeProxy = nCreateProxy(translucent, rootNodePtr);
   }
    
}

public class RenderNode {

    public static RenderNode adopt(long nativePtr) {
        return new RenderNode(nativePtr);
    }
    
}
```
ThreadedRenderer 的构造函数中, 有两个非常重要的操作
- 通过 nCreateRootRenderNode 创建渲染根结点
- 通过 nCreateProxy 创建渲染代理对象

#### 创建渲染根结点
```
// frameworks/base/core/jni/android_view_ThreadedRenderer.cpp
static jlong android_view_ThreadedRenderer_createRootRenderNode(JNIEnv* env, jobject clazz) {
    // 创建了一个 RootRenderNode 对象
    RootRenderNode* node = new RootRenderNode(env);
    node->incStrong(0);
    node->setName("RootRenderNode");
    return reinterpret_cast<jlong>(node);
}


class RootRenderNode : public RenderNode, ErrorHandler {
public:
    explicit RootRenderNode(JNIEnv* env) : RenderNode() {
        mLooper = Looper::getForThread();
        env->GetJavaVM(&mVm);
    }
}

// frameworks/base/libs/hwui/RenderNode.cpp
RenderNode::RenderNode()
        : mDirtyPropertyFields(0)
        , mNeedsDisplayListSync(false)
        , mDisplayList(nullptr)
        , mStagingDisplayList(nullptr)
        , mAnimatorManager(*this)
        , mParentCount(0) {}
```
接下来看看渲染代理对象的创建 

#### 创建渲染代理对象
```
// frameworks/base/core/jni/android_view_ThreadedRenderer.cpp
static jlong android_view_ThreadedRenderer_createProxy(JNIEnv* env, jobject clazz,
        jboolean translucent, jlong rootRenderNodePtr) {
    RootRenderNode* rootRenderNode = reinterpret_cast<RootRenderNode*>(rootRenderNodePtr);
    // 使用 RootRenderNode 创建了一个 ContextFactoryImpl
    ContextFactoryImpl factory(rootRenderNode);
    // 创建了一个渲染的代理对象 RenderProxy
    return (jlong) new RenderProxy(translucent, rootRenderNode, &factory);
}

RenderProxy::RenderProxy(bool translucent, RenderNode* rootRenderNode,
                         IContextFactory* contextFactory)
        // 获取 RenderThread 对象
        : mRenderThread(RenderThread::getInstance())
        , mContext(nullptr) {
    // 可以看到这里创建了一个 CanvasContext
    mContext = mRenderThread.queue().runSync([&]() -> CanvasContext* {
        return CanvasContext::create(mRenderThread, translucent, rootRenderNode, contextFactory);
    });
    mDrawFrameTask.setContext(&mRenderThread, mContext, rootRenderNode);
}
```
RenderProxy 中创建了一个 CanvasContext 上下文, 用于管理 Canvas 绘制时所需要的上下文数据
```
// frameworks/base/libs/hwui/renderthread/CanvasContext.cpp
CanvasContext* CanvasContext::create(RenderThread& thread, bool translucent,
                                     RenderNode* rootRenderNode, IContextFactory* contextFactory) {
    auto renderType = Properties::getRenderPipelineType();
    switch (renderType) {
        case RenderPipelineType::OpenGL:
            // 使用 OpenGL 的渲染管线 
            return new CanvasContext(thread, translucent, rootRenderNode, contextFactory,
                                     std::make_unique<OpenGLPipeline>(thread));
        case RenderPipelineType::SkiaGL:
            // 使用 SkiaOpenGLPipeline 的渲染管线
            return new CanvasContext(thread, translucent, rootRenderNode, contextFactory,
                                     std::make_unique<skiapipeline::SkiaOpenGLPipeline>(thread));
        case RenderPipelineType::SkiaVulkan:
            // 使用 SkiaVulkanPipeline 的渲染管线
            return new CanvasContext(thread, translucent, rootRenderNode, contextFactory,
                                     std::make_unique<skiapipeline::SkiaVulkanPipeline>(thread));
        default:
            LOG_ALWAYS_FATAL("canvas context type %d not supported", (int32_t)renderType);
            break;
    }
    return nullptr;
}

CanvasContext::CanvasContext(RenderThread& thread, bool translucent, RenderNode* rootRenderNode,
                             IContextFactory* contextFactory,
                             std::unique_ptr<IRenderPipeline> renderPipeline)
        : mRenderThread(thread)
        , mGenerationID(0)
        , mOpaque(!translucent)
        , mAnimationContext(contextFactory->createAnimationContext(mRenderThread.timeLord()))
        , mJankTracker(&thread.globalProfileData(), thread.mainDisplayInfo())
        , mProfiler(mJankTracker.frames())
        , mContentDrawBounds(0, 0, 0, 0)
        , mRenderPipeline(std::move(renderPipeline)) {
    // 标记为渲染根结点
    rootRenderNode->makeRoot();
    mRenderNodes.emplace_back(rootRenderNode);
    // 注册该对象为渲染的上下文
    mRenderThread.renderState().registerCanvasContext(this);
    mProfiler.setDensity(mRenderThread.mainDisplayInfo().density);
}
```

从这里可以看到, CanvasContext 的工厂方法的 create 中根据不同的 Type, 注入了不同的渲染管线, **通过这种方式这样子就可以将上层抽象的 API 与具体的实现分开了, 其中还可以看到最新的 Vulkan 渲染管线**, 这里我们主要探究 OpenGLPipeline 的渲染管线

### 回顾
到这里 ThreadRenderer 的初始化便完成了, 这里回顾一下各个对象的依赖关系

![依赖关系](https://i.loli.net/2019/10/23/GCLwjWkEJUOSZYu.png)

在 ThreadRenderer 创建的过程中, 我们并没有看到它与 Surface 进行绑定的过程, 这意为着**此时只有画笔却没有画布, 所以还需要一个与 Surface 绑定的过程**

接下来看看初始化操作

## 二. ThreadedRenderer 的初始化
```
public final class ViewRootImpl implements ViewParent,
        View.AttachInfo.Callbacks, ThreadedRenderer.DrawCallbacks {
            
    private void performTraversals() {
        
        try {
            
            ......
            // relayoutWindow 之后 Surface 便被填充了 IGraphicBufferProducer, 即有效了
            relayoutResult = relayoutWindow(params, viewVisibility, insetsPending);
            if (!hadSurface) {
                if (mSurface.isValid()) {
                    ......
                    if (mAttachInfo.mThreadedRenderer != null) {
                        try {
                            // 执行初始化操作
                            hwInitialized = mAttachInfo.mThreadedRenderer.initialize(
                                    mSurface);
                            ......
                        } catch (OutOfResourcesException e) {
                            ......
                        }
                    }
                }
            } 
        }
        ......
    }            
            
}
```
可以看到 ThreadedRenderer 初始化的时机在 Surface 可用之后进行的, 接下来看看初始化的流程
```
public final class ThreadedRenderer {

    private boolean mInitialized = false;

    boolean initialize(Surface surface) throws OutOfResourcesException {
        boolean status = !mInitialized;
        mInitialized = true;
        // 标记为 Enable
        updateEnabledState(surface);
        nInitialize(mNativeProxy, surface);
        return status;
    }
    
    private static native void nInitialize(long nativeProxy, Surface window);

}
```

#### nInitialize
```
// frameworks/base/core/jni/android_view_ThreadedRenderer.cpp
static void android_view_ThreadedRenderer_initialize(JNIEnv* env, jobject clazz,
        jlong proxyPtr, jobject jsurface) {
    RenderProxy* proxy = reinterpret_cast<RenderProxy*>(proxyPtr);
    // 调用了 RenderProxy 的 initialize 方法
    sp<Surface> surface = android_view_Surface_getSurface(env, jsurface);
    proxy->initialize(surface);
}

void RenderProxy::initialize(const sp<Surface>& surface) {
    mRenderThread.queue().post(
        [ this, surf = surface ]() mutable { 
            // 调用了 CanvasContext 的 setSurface 函数, 将 surface 保存在 Canvas 的上下文中
            mContext->setSurface(std::move(surf)); 
        }
    );
}
```
好的, 可以看到 initialize 方法, 会调用到 CanvasContext.setSurface 函数, 接下来继续探究
```
void CanvasContext::setSurface(sp<Surface>&& surface) {
    ATRACE_CALL();
    // 将 surface 保存到 mNativeSurface 成员变量中
    mNativeSurface = std::move(surface);
    ColorMode colorMode = mWideColorGamut ? ColorMode::WideColorGamut : ColorMode::Srgb;
    // 为 OpenGL ES 渲染管线绑定输出 Surface
    bool hasSurface = mRenderPipeline->setSurface(mNativeSurface.get(), mSwapBehavior, colorMode);
    ......
}


```
这里为 mRenderPipeline 注入了 Surface, 其具体操作如下
```
bool OpenGLPipeline::setSurface(Surface* surface, SwapBehavior swapBehavior, ColorMode colorMode) {
    // 销毁之前的 EGLSurfcae
    if (mEglSurface != EGL_NO_SURFACE) {
        mEglManager.destroySurface(mEglSurface);
        mEglSurface = EGL_NO_SURFACE;
    }
    // 创建 EGLSurface, 作为帧缓冲
    if (surface) {
        const bool wideColorGamut = colorMode == ColorMode::WideColorGamut;
        mEglSurface = mEglManager.createSurface(surface, wideColorGamut);
    }
    if (mEglSurface != EGL_NO_SURFACE) {
        const bool preserveBuffer = (swapBehavior != SwapBehavior::kSwap_discardBuffer);
        mBufferPreserved = mEglManager.setPreserveBuffer(mEglSurface, preserveBuffer);
        return true;
    }
    return false;
}
```
可以看到这里通过 Surface, 创建了一个 EGLSurface, 我们知道 EGLSurface 表示 OpenGL 的缓冲区, 有了缓冲区便可以进行渲染管线的操作了

到这里 ThreadedRenderer 的初始化工作便结束了, 接下来就可以**探索 mThreadedRenderer.draw 是如何进行硬件绘制的**了

## 三. ThreadedRenderer.draw
硬件渲染开启之后, 在 ViewRootImpl.draw 中就会执行 mAttachInfo.mThreadedRenderer.draw(mView, mAttachInfo, this, callback); 进行 View 的绘制
```
public final class ThreadedRenderer {

    private RenderNode mRootNode;// 描述当前窗体的渲染器

    ThreadedRenderer(Context context, boolean translucent, String name) {
        ......
        // 构建了一个根 RenderNode, 它表示当前窗体的根渲染器
        long rootNodePtr = nCreateRootRenderNode();
        mRootNode = RenderNode.adopt(rootNodePtr);
    }
    
    void draw(View view, AttachInfo attachInfo, DrawCallbacks callbacks,
            FrameDrawingCallback frameDrawingCallback) {
        ......    
        // 1. 更新当前窗体的渲染数据
        updateRootDisplayList(view, callbacks);
        ......
        // 2. 同步渲染帧, 并且推入 SurfaceFlinger
        int syncResult = nSyncAndDrawFrame(mNativeProxy, frameInfo, frameInfo.length);
        ......
    }
    
}
```
好的, 可以看到 ThreadedRenderer 主要进行了两个操作
- 构建窗体的渲染数据
- 调用 nSyncAndDrawFrame 将数据推入 SurfaceFlinger

可以看到当前窗体的渲染数据是从 View 树中获取的, 因此我们这里要着重分析一下, **如何构建 View 树的渲染数据**

### 一) 构建窗体的渲染数据
```
public final class ThreadedRenderer {
    
    private void updateRootDisplayList(View view, DrawCallbacks callbacks) {
        ......
        // 视图需要更新 || 当前的根渲染器中的数据已经无效了, 会进入如下分支
        if (mRootNodeNeedsUpdate || !mRootNode.isValid()) {
            // 1. 构建根结点的 DisplayListCanvas, 开始采集 View 绘制操作
            DisplayListCanvas canvas = mRootNode.start(mSurfaceWidth, mSurfaceHeight);
            try {
                ......
                // 2. 将 DecorView 中绘制的操作渲染数据, 保存到根结点 DisplayListCanvas 中
                canvas.drawRenderNode(view.updateDisplayListIfDirty());
                ......
                // 3. 表示当前窗体的视图数据更新完毕, 不需要更新了
                mRootNodeNeedsUpdate = false;
            } finally {
                // 3. 通知根结点数据采集完毕
                mRootNode.end(canvas);
            }
        }
    }
}
```
好的, 可以看到渲染的操作主要有三步
- 调用 RootNode.start: 构建根结点的 DisplayListCanvas
- 调用 DecorView.updateDisplayListIfDirty: 采集 View 的绘制操作
- 调用 RootNode.end: 通知数据采集完毕

#### 1. 开启数据的采集
```
public final class DisplayListCanvas extends RecordingCanvas {
    static DisplayListCanvas obtain(@NonNull RenderNode node, int width, int height) {
        ......
        DisplayListCanvas canvas = sPool.acquire();
        if (canvas == null) {
            canvas = new DisplayListCanvas(node, width, height);
        } else {
            ......
        }
        canvas.mNode = node;
        canvas.mWidth = width;
        canvas.mHeight = height;
        return canvas;
    }
    
    private DisplayListCanvas(@NonNull RenderNode node, int width, int height) {
        super(nCreateDisplayListCanvas(node.mNativeRenderNode, width, height));
        mDensity = 0; 
    }
    
    private static native long nCreateDisplayListCanvas(long node, int width, int height);

}
```
DisplayListCanvas 的创建, 会调用 nCreateDisplayListCanvas 方法, 接下来看看它的 native 实现
```
// frameworks/base/core/jni/android_view_DisplayListCanvas.cpp
static jlong android_view_DisplayListCanvas_createDisplayListCanvas(jlong renderNodePtr,
        jint width, jint height) {
    // 获取 RendererNode
    RenderNode* renderNode = reinterpret_cast<RenderNode*>(renderNodePtr);
    // 调用 create_recording_canvas
    return reinterpret_cast<jlong>(Canvas::create_recording_canvas(width, height, renderNode));
}

// frameworks/base/libs/hwui/hwui/Canvas.cpp
Canvas* Canvas::create_recording_canvas(int width, int height, uirenderer::RenderNode* renderNode) {
    if (uirenderer::Properties::isSkiaEnabled()) {
        // 若是允许使用 skia, 则创建 SkiaRecordingCanvas
        return new uirenderer::skiapipeline::SkiaRecordingCanvas(renderNode, width, height);
    }
    // 反之, 则创建 RecordingCanvas
    return new uirenderer::RecordingCanvas(width, height);
}
```
也就是说 DisplayListCanvas 它会关联一个 SkiaRecordingCanvas/RecordingCanvas

这里主要看看 [RecordingCanvas](http://androidxref.com/9.0.0_r3/xref/frameworks/base/libs/hwui/RecordingCanvas.cpp) 的操作
```
// frameworks/base/libs/hwui/RecordingCanvas.cpp
RecordingCanvas::RecordingCanvas(size_t width, size_t height)
        : mState(*this), mResourceCache(ResourceCache::getInstance()) {
    resetRecording(width, height);
}

void RecordingCanvas::resetRecording(int width, int height, RenderNode* node) {
    // 创建了 DisplayList
    mDisplayList = new DisplayList();
    // 初始化记录的栈
    mState.initializeRecordingSaveStack(width, height);
    mDeferredBarrierType = DeferredBarrierType::InOrder;
}

// frameworks/base/libs/hwui/DisplayList.cpp
DisplayList::DisplayList()
        : projectionReceiveIndex(-1)
        , stdAllocator(allocator)
        , chunks(stdAllocator)
        , ops(stdAllocator)
        , children(stdAllocator)
        , bitmapResources(stdAllocator)
        , pathResources(stdAllocator)
        , patchResources(stdAllocator)
        , paints(stdAllocator)
        , regions(stdAllocator)
        , referenceHolders(stdAllocator)
        , functors(stdAllocator)
        , vectorDrawables(stdAllocator) {}

// frameworks/base/libs/hwui/CanvasState.cpp      
void CanvasState::initializeRecordingSaveStack(int viewportWidth, int viewportHeight) {
    if (mWidth != viewportWidth || mHeight != viewportHeight) {
        mWidth = viewportWidth;
        mHeight = viewportHeight;
        mFirstSnapshot.initializeViewport(viewportWidth, viewportHeight);
        mCanvas.onViewportInitialized();
    }
    freeAllSnapshots();
    mSnapshot = allocSnapshot(&mFirstSnapshot, SaveFlags::MatrixClip);
    mSnapshot->setRelativeLightCenter(Vector3());
    mSaveCount = 1;
}
```
RecordingCanvas 在构建之后, 会创建一个 DisplayList, 同时初始化 CanvasState 用于记录每一步的绘制动作

好的, 到这里 DisplayListCanvas 的创建便完成了, 从这里可以看到 DisplayListCanvas 使用的 Native 对象为 RecordingCanvas

**RecordingCanvas 这个 Canvas 与 SkiaCanvas 不同, 它持有一个 DisplayList 和 CanvasState, 并没有没有发现 SkiaCanvas 的 SkBitmap 作为缓冲区, 这意味着它的 draw 操作并不会绘制到缓冲区, 而是记录了每一步的操作**, 也就是说后面的 updateDisplayListIfDirty 并不是绘制流程, 而是对绘制步骤的采集了

#### 2. 采集子 View 的数据
```
public class View {
    
    final RenderNode mRenderNode;
    
    public View(Context context) {
        ......
        // 可见在硬件绘制中, 每一个 View 对应着一个渲染器中的结点
        mRenderNode = RenderNode.create(getClass().getName(), this);
        ......    
    }
    
     public RenderNode updateDisplayListIfDirty() {
        final RenderNode renderNode = mRenderNode;
        .....
        
        if ((mPrivateFlags & PFLAG_DRAWING_CACHE_VALID) == 0
                || !renderNode.isValid() || (mRecreateDisplayList)) {
            // 当前 View 渲染器中数据依旧是有效的 && 没有要求重绘
            if (renderNode.isValid() && !mRecreateDisplayList) {
                // 将更新 Render 的操作分发给子 View, 该方法会在 ViewGroup 中重写
                dispatchGetDisplayList();
                // 直接跳过使用 Canvas 绘制的操作
                return renderNode;
            }

            // 走到这里说明当前 View 的渲染数据已经失效了, 需要重新构建渲染数据
            mRecreateDisplayList = true;

            int width = mRight - mLeft;
            int height = mBottom - mTop;
            int layerType = getLayerType();
            
            // 1. 通过渲染器构建一个 Canvas 画笔, 画笔的可作用的区域为 width, height
            final DisplayListCanvas canvas = renderNode.start(width, height);
            try {
                if (layerType == LAYER_TYPE_SOFTWARE) {
                    ......
                } else {
                    ......
                    // 2.2 当前 View 的 Draw 可跳过, 直接分发去构建子 View 渲染数据
                    if ((mPrivateFlags & PFLAG_SKIP_DRAW) == PFLAG_SKIP_DRAW) {
                        dispatchDraw(canvas);
                    } else {
                        // 2.3 绘制自身
                        draw(canvas);
                    }
                }
            } finally {
                // 3. 表示当前 View 的渲染数据已经保存在 Canvas 中了
                // 将它的数据传递到渲染器
                renderNode.end(canvas);
            }
        } else {
            ......
        }
        return renderNode;
    }
    
}
```
DecorView.updateDisplayListIfDirty 方法从名字上来理解是若 View 展示列表无效了则更新它, 事实上它做的也是如此, 它主要做了如下操作
- 当前 View 渲染数据依旧是有效的 并且没有要求重绘
  - 将更新渲染数据的操作分发给子 View
  - 遍历结束之后直接返回 当前现有的渲染器结点对象
- 当前 View 渲染数据无效了
  - 通过渲染器构建可渲染区域的画笔 DisplayListCanvas
  - 调用 view.draw 进行渲染数据的构建
    - **这个方法留在软件渲染的时候再分析**
  - 当前 View 的渲染数据重新构建好了, 则将它保存在渲染器 renderNode 中

可以看到 DecorView 也使用了 RenderNode 构建了 DisplayListCanvas, 也就是说绘制的操作会保存 DecorView 的 DisplayListCanvas 中

绘制结束之后同样会调用 renderNode.end, 表示数据采集结束, 接下来我们看看这个操作

#### 3. 结束采集操作
```
public class RenderNode {

    public void end(DisplayListCanvas canvas) {
        // 1. 结束 Canvas 数据的采集
        long displayList = canvas.finishRecording();
        // 2. 将采集到的 DisplayList 保存到 Native 的结点中
        nSetDisplayList(mNativeRenderNode, displayList);
        canvas.recycle();
    }
    
}
```
##### 1) 结束 Canvas 数据的采集
```
public final class DisplayListCanvas extends RecordingCanvas {

    long finishRecording() {
        return nFinishRecording(mNativeCanvasWrapper);
    } 
    
    private static native long nFinishRecording(long renderer);

}

// frameworks/base/core/jni/android_view_DisplayListCanvas.cpp
static jlong android_view_DisplayListCanvas_finishRecording(jlong canvasPtr) {
    // 调用了 RecordingCanvas 的 finishRecording
    Canvas* canvas = reinterpret_cast<Canvas*>(canvasPtr);
    return reinterpret_cast<jlong>(canvas->finishRecording());
}

// frameworks/base/libs/hwui/RecordingCanvas.cpp
DisplayList* RecordingCanvas::finishRecording() {
    restoreToCount(1);
    mPaintMap.clear();
    mRegionMap.clear();
    mPathMap.clear();
    DisplayList* displayList = mDisplayList;
    mDisplayList = nullptr;
    mSkiaCanvasProxy.reset(nullptr);
    return displayList;
}
```
从这里可以看出, finishRecording 的操作会将绘制过程中的 DisplayList 返回, 接下来看看如何将 DisplayList 返回给 RenderNode

##### 2) 将采集的数据保存到 RenderNode
```
// frameworks/base/core/jni/android_view_RenderNode.cpp
static void android_view_RenderNode_setDisplayList(JNIEnv* env,
        jobject clazz, jlong renderNodePtr, jlong displayListPtr) {
    RenderNode* renderNode = reinterpret_cast<RenderNode*>(renderNodePtr);
    DisplayList* newData = reinterpret_cast<DisplayList*>(displayListPtr);
    renderNode->setStagingDisplayList(newData);
}

// frameworks/base/libs/hwui/RenderNode.cpp
void RenderNode::setStagingDisplayList(DisplayList* displayList) {
    mValid = (displayList != nullptr);
    mNeedsDisplayListSync = true;
    delete mStagingDisplayList;
    // 将 Canvas 的数据保存到 mStagingDisplayList 中
    mStagingDisplayList = displayList;
}
```
好的, 到这里我们 Canvas 记录的 DisplayList 数据就保存到了 mStagingDisplayList 变量中了

#### 回顾

![View 硬件绘制结构图](https://i.loli.net/2019/10/23/RseqzM8aAkVt4GX.png)

接下来看看, 数据推送到 SurfaceFlinger 的过程

### 二) nSyncAndDrawFrame
ThreadedRenderer 通过 nSyncAndDrawFrame 将数据发送到 SurfaceFlinger, 这里看看它的具体操作
```
// frameworks/base/core/jni/android_view_ThreadedRenderer.cpp
static int android_view_ThreadedRenderer_syncAndDrawFrame(JNIEnv* env, jobject clazz,
        jlong proxyPtr, jlongArray frameInfo, jint frameInfoSize) {
    .....
    // 获取渲染器代理
    RenderProxy* proxy = reinterpret_cast<RenderProxy*>(proxyPtr);
    env->GetLongArrayRegion(frameInfo, 0, frameInfoSize, proxy->frameInfo());
    // 调用 syncAndDrawFrame 函数
    return proxy->syncAndDrawFrame();
}

// frameworks/base/libs/hwui/renderthread/RenderProxy.cpp
int RenderProxy::syncAndDrawFrame() {
    return mDrawFrameTask.drawFrame();
}

// frameworks/base/libs/hwui/renderthread/DrawFrameTask.cpp
int DrawFrameTask::drawFrame() {
    ......
    mSyncResult = SyncResult::OK;
    mSyncQueued = systemTime(CLOCK_MONOTONIC);
    postAndWait();
    return mSyncResult;
}

void DrawFrameTask::postAndWait() {
    AutoMutex _lock(mLock);
    // 交由渲染线程去执行这个 run 任务
    mRenderThread->queue().post([this]() { run(); });
    mSignal.wait(mLock);
}
```
可以看到, 这里调用了 DrawFrameTask 的 postAndWait 函数, 并且这个**绘制任务投放到 RenderThread 的队列中执行**, 这里我们看看执行的具体内容
```
// frameworks/base/libs/hwui/renderthread/DrawFrameTask.cpp
void DrawFrameTask::run() {
   ......
    if (CC_LIKELY(canDrawThisFrame)) {
        // 请求 Context 绘制当前帧
        context->draw();
    } else {
        ......
    }
    if (!canUnblockUiThread) {
        unblockUiThread();
    }
}

void CanvasContext::draw() {
    SkRect dirty;
    mDamageAccumulator.finish(&dirty);
    mCurrentFrameInfo->markIssueDrawCommandsStart();
    Frame frame = mRenderPipeline->getFrame();
    SkRect windowDirty = computeDirtyRect(frame, &dirty);
    // 1. 通知渲染管线绘制数据
    bool drew = mRenderPipeline->draw(frame, windowDirty, dirty, mLightGeometry, &mLayerUpdateQueue,
                                      mContentDrawBounds, mOpaque, mWideColorGamut, mLightInfo,
                                      mRenderNodes, &(profiler()));
    int64_t frameCompleteNr = mFrameCompleteCallbacks.size() ? getFrameNumber() : -1;
    waitOnFences();
    bool requireSwap = false;
    // 2. 通知渲染管线将数据, 交换到 Surface 的缓冲区, 并且推送给 SurfaceFlinger
    bool didSwap =
            mRenderPipeline->swapBuffers(frame, drew, windowDirty, mCurrentFrameInfo, &requireSwap);
    mIsDirty = false;
    ......
}
```
可以看到最终的绘制操作是在 CanvasContext 中交由渲染管线去执行的, 这里主要有两个步骤
- 通过 mRenderPipeline->draw, 将 RenderNode 中的 DisplayList 记录的数据绘制到 Surface 的缓冲区
- 通过 mRenderPipeline->swapBuffers 将缓冲区的数据推送到 SurfaceFlinger 的函数

至此, 一次硬件绘制就完成了

## 总结
可以看到 View 的硬件渲染比起软件渲染要复杂的多, 笔者也有很多内容没有看到, 旨在理清硬件渲染的流程, 这里做个简单的回顾
- ThreadRenderer 的创建
  - 会创建一个根结点 RenderNode 用于记录所有的绘制动作
  - 创建 RenderProxy 执行渲染操作
- ThreadRenderer 的初始化
  - 将 Surface 的缓冲区绑定到 RenderProxy 的渲染管线 RenderPipeline 中
- **ThreadRenderer 的绘制**
  - RenderNode.start 开始捕获绘制动作
  - 调用 DecorView.updateDisplayListIfDirty 捕获 View 绘制动作
  - RenderNode.end 结束绘制动作捕获
  - nSyncAndDrawFrame(在 RenderThread 线程执行)
    - 使用渲染管线绘制捕获到的绘制动作 DisplayList
    - 使用渲染管线将绘制的数据推到 SurfaceFlinger 中

###  硬件绘制与软件绘制差异
#### 1. 从渲染机制
- 硬件绘制使用的是 OpenGL/ Vulkan, 支持 3D 高性能图形绘制
- 软件绘制使用的是 Skia, 仅支持 2D 图形绘制

#### 2. 渲染效率上
- 硬件绘制
  - 在 Android 5.0 之后引入了 RendererThread, 它将 OpenGL 图形栅格化的操作全部投递到了这个线程
  - 硬件绘制会跳过渲染数据无变更的 View, 直接分发给子视图

- 软件绘制
  - 在将数据投入 SurfaceFlinger 之前, 所有的操作均在主线程执行
  - 不会跳过无变化的 View
 
因此硬件绘制较之软件绘制会更加流畅

#### 2. 从内存消耗上
硬件绘制消耗的内存要高于软件绘制, 但在当下大内存手机时代, 用空间去换时间还是非常值得的

#### 3. 从兼容性上
- 硬件绘制的 OpenGL 在各个 Android 版本的支持上会有一些不同, 常有因为兼容性出现的系统 bug
- 软件绘制的 Skia 库从 Android 1.0 便屹立不倒, 因此它的兼容性要好于硬件绘制