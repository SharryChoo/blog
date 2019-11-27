---
layout: article
title: Android 系统架构 —— SurfaceFlinger 渲染图层
permalink: android-source/surfaceflinger-composer
key: android-source-surfaceflinger-composer
tags: AndroidFramework
---

## 前言
在前面的分析中, 我们分别了解了 SurfaceFlinger 显示设备的创建和 Vsync 垂直同步的处理

这篇文章我们主要探究一下 SurfaceFlinger 对渲染数据的处理, 正好可以将前面所学的知识进行一次汇总

<!--more-->

在一个 Vsync 信号到来之后, 最终会触发 SurfaceFlinger 的 onMessageReceived 的函数

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
这里出现了两个非常重要的函数
- handleMessageInvalidate
- signalRefresh

这里我们便从这两个方面入手, 看看 SurfaceFlinger 到底是如何进行 Surface 数据合成的

## 一. handleMessageInvalidate
```
bool SurfaceFlinger::handleMessageInvalidate() {
    return handlePageFlip();
}

bool SurfaceFlinger::handlePageFlip()
{
    bool visibleRegions = false;
    bool frameQueued = false;
    bool newDataLatched = false;

    ......
    
    // 遍历所有的 Layer, 若需要展示, 则将他们添加到 mLayersWithQueuedFrames 中, 表示待重绘
    mDrawingState.traverseInZOrder([&](Layer* layer) {
        if (layer->hasQueuedFrame()) {
            frameQueued = true;
            if (layer->shouldPresentNow(mPrimaryDispSync)) {
                mLayersWithQueuedFrames.push_back(layer);
            } else {
                layer->useEmptyDamage();
            }
        } else {
            layer->useEmptyDamage();
        }
    });
    
    // 遍历 mLayersWithQueuedFrames 中的 Layer
    for (auto& layer : mLayersWithQueuedFrames) {
        // 调用 Layer 的 latchBuffer 函数, 去队列中获取一个缓冲区
        const Region dirty(layer->latchBuffer(visibleRegions, latchTime));
        layer->useSurfaceDamage();
        invalidateLayerStack(layer, dirty);
        // 若是成功锁定了一个缓冲, 则说明 GraphicBuffer 中, 存在该 Layer 的新渲染数据
        if (layer->isBufferLatched()) {
            newDataLatched = true;
        }
    }
    
    ......

    // Only continue with the refresh if there is actually new work to do
    return !mLayersWithQueuedFrames.empty() && newDataLatched;
}
```
从这里我们可以看到, 当 invalidate 消息到来时, 会**调用 Layer.latchBuffer 函数, 从队列中获取一个新的缓冲区**, 若缓冲队列中存在, 则说明需要进行重绘

下面我们就看看 Layer 是如何获取缓冲区的

### 一) Layer.latchBuffer
我们主要关心 Layer 的实现类 [BufferLayer](http://androidxref.com/9.0.0_r3/xref/frameworks/native/services/surfaceflinger/BufferLayer.cpp)
```
// frameworks/native/services/surfaceflinger/BufferLayer.cpp
Region BufferLayer::latchBuffer(bool& recomputeVisibleRegions, nsecs_t latchTime) {
    .....

    // 调用 mConsumer 的 updateTexImage 从队列中获取数据
    status_t updateResult =
            mConsumer->updateTexImage(&r, mFlinger->mPrimaryDispSync,
                                                    &mAutoRefresh, &queuedBuffer,
                                                    mLastFrameNumberReceived);
    ......
}
```
从这里可以看到, 它调用了 [BufferLayerConsumer](http://androidxref.com/9.0.0_r3/xref/frameworks/native/services/surfaceflinger/BufferLayerConsumer.h) 的 updateTexImage 从 GraphicBuffer 的队列中获取这个 Layer 的新缓冲数据
```
// frameworks/native/services/surfaceflinger/BufferLayerConsumer.cpp
status_t BufferLayerConsumer::updateTexImage(BufferRejecter* rejecter, const DispSync& dispSync,
                                             bool* autoRefresh, bool* queuedBuffer,
                                             uint64_t maxFrameNumber) {
    ......
    
    // 调用了 acquireBufferLocked 获取一个新的图像缓冲
    BufferItem item;
    status_t err = acquireBufferLocked(&item, computeExpectedPresent(dispSync), maxFrameNumber);
    ......
    if (queuedBuffer) {
        *queuedBuffer = item.mQueuedBuffer;
    }
    ......
    return err;
}
```
updateTexImage 的操作比较复杂, 这里省略了大量的代码, 主要关注一下 acquireBufferLocked 获取缓冲区的操作

### 二) acquireBufferLocked
```
// frameworks/native/services/surfaceflinger/BufferLayerConsumer.cpp
status_t BufferLayerConsumer::acquireBufferLocked(BufferItem* item, nsecs_t presentWhen,
                                                  uint64_t maxFrameNumber) {
    // 取 Buffer
    status_t err = ConsumerBase::acquireBufferLocked(item, presentWhen, maxFrameNumber);
    if (err != NO_ERROR) {
        return err;
    }
    // 若取到了新的 GrapicBuffer, 则创建一个新的 Image 缓存起来
    if (item->mGraphicBuffer != nullptr) {
        mImages[item->mSlot] = new Image(item->mGraphicBuffer, mRE);
    }
    return NO_ERROR;
}
```
这里会调用 [ConsumerBase](http://androidxref.com/9.0.0_r3/xref/frameworks/native/libs/gui/ConsumerBase.cpp) 的 acquireBufferLocked, 在热插拔信号处理的时候, 创建 DisplayDevice 之前, 会创建 FramebufferSurface 继承了这个 ConsumerBase, 因此它也也是 GraphicBufferConsumer 的直接操作者

```
// frameworks/native/libs/gui/ConsumerBase.cpp
status_t ConsumerBase::acquireBufferLocked(BufferItem *item,
        nsecs_t presentWhen, uint64_t maxFrameNumber) {
    ......
    // 调用了 BnGraphicBufferConsumer 的 acquireBuffer
    status_t err = mConsumer->acquireBuffer(item, presentWhen, maxFrameNumber);
    if (err != NO_ERROR) {
        return err;
    }
    // 保存在 mSlots 中
    if (item->mGraphicBuffer != NULL) {
        if (mSlots[item->mSlot].mGraphicBuffer != NULL) {
            freeBufferLocked(item->mSlot);
        }
        mSlots[item->mSlot].mGraphicBuffer = item->mGraphicBuffer;
    }
    mSlots[item->mSlot].mFrameNumber = item->mFrameNumber;
    mSlots[item->mSlot].mFence = item->mFence;
    ......
    return OK;
}
```
可以看到, 这里调用了 mConsumer 的 acquireBuffer 来请求一个 GraphicBuffer, 这个 mConsumer 即为 IGraphicBufferConsumer 的 Binder 本地实现对象 [GraphicBufferConsumer](http://androidxref.com/9.0.0_r3/xref/frameworks/native/libs/gui/BufferQueueConsumer.cpp) 这里就不再赘述了

### 三) 回顾
![Layer 获取 GraphicBuffer](https://i.loli.net/2019/10/23/EcntGaTeF36JsxZ.png)

当 handleMessageInvalidate 调用时, 会遍历所有的 Layer, 并且调用 Layer 的 latchBuffer, 尝试从缓冲区中获取 GraphicBuffer, 若取到了新的 GraphicBuffer, 则说明需要重新绘制

好的, 当 Layer 从 BufferQueueCore 中获取了新的 GraphicBuffer 之后, 下面需要做的便是将 Layer 进行合成了

## 二. signalRefresh
```
void SurfaceFlinger::signalRefresh() {
    mRefreshPending = true;
    mEventQueue->refresh();
}

// frameworks/native/services/surfaceflinger/MessageQueue.cpp
void MessageQueue::refresh() {
    mHandler->dispatchRefresh();
}

void MessageQueue::Handler::dispatchRefresh() {
    if ((android_atomic_or(eventMaskRefresh, &mEventMask) & eventMaskRefresh) == 0) {
        // 发送 MessageQueue::REFRESH 类型的 Message
        mQueue.mLooper->sendMessage(this, Message(MessageQueue::REFRESH));
    }
}
```
好的, 可以看到这发送了一条 MessageQueue::REFRESH 消息, 下面我们看看 onMessageReceived 对 MessageQueue::REFRESH 的处理
```
void SurfaceFlinger::onMessageReceived(int32_t what) {
    switch (what) {
        ......
        case MessageQueue::REFRESH: {
            handleMessageRefresh();
            break;
        }
    }
}

void SurfaceFlinger::handleMessageRefresh() {
    ......
    // 1. 将 Layer 按照 Z 轴排序
    rebuildLayerStacks();
    
    ......
    
    // 2. 执行 Layer 的合成并且展示到屏幕
    doComposition();
    ......
}
```
好的, 可以看到 SurfaceFlinger 的 handleMessageRefresh 中我们需要关心的操作主要有如上所示
- rebuildLayerStacks 负责 Layzer 的排序
- doComposition 负责合并 Layer, 并且输出到屏幕

下面我们逐一查看

### 一) 将 Layer 按照 Z 轴排序
```
// frameworks/native/services/surfaceflinger/SurfaceFlinger.cpp
void SurfaceFlinger::rebuildLayerStacks() {
    ......
    // rebuild the visible layer list per screen
    if (CC_UNLIKELY(mVisibleRegionsDirty)) {
        ......
        // 遍历每一个显示设备
        for (size_t dpy=0 ; dpy<mDisplays.size() ; dpy++) {
            Region opaqueRegion;
            Region dirtyRegion;
            
            // 用来存储按照 Z 轴排序好的 Layer
            Vector<sp<Layer>> layersSortedByZ;
            Vector<sp<Layer>> layersNeedingFences;
            
            // 获取显示设备描述
            const sp<DisplayDevice>& displayDevice(mDisplays[dpy]);
            const Transform& tr(displayDevice->getTransform());
            const Rect bounds(displayDevice->getBounds());
            
            if (displayDevice->isDisplayOn()) {
                // 计算可视的区域
                computeVisibleRegions(displayDevice, dirtyRegion, opaqueRegion);
                // 遍历每一个 Layers
                mDrawingState.traverseInZOrder([&](Layer* layer) {
                    bool hwcLayerDestroyed = false;
                    // 若这个 Layer 属于该显示设备
                    if (layer->belongsToDisplay(displayDevice->getLayerStack(),
                                displayDevice->isPrimary())) {
                        // 获取 Layer 绘制的区域
                        Region drawRegion(tr.transform(
                                layer->visibleNonTransparentRegion));
                        drawRegion.andSelf(bounds);
                        if (!drawRegion.isEmpty()) {
                            // 添加到集合汇总
                            layersSortedByZ.add(layer);
                        } else {
                            // 清除 HWC 层的 Layer, 让其不可见
                            hwcLayerDestroyed = layer->destroyHwcLayer(
                                    displayDevice->getHwcDisplayId());
                        }
                    } else {
                        // 清除 HWC 层的 Layer, 让其不可见
                        hwcLayerDestroyed = layer->destroyHwcLayer(
                                displayDevice->getHwcDisplayId());
                    }

                    // 若这个 Layer 因为 Fence 操作没有被 GPU 释放, 则添加到 layersNeedingFences 集合中
                    if (hwcLayerDestroyed) {
                        auto found = std::find(mLayersWithQueuedFrames.cbegin(),
                                mLayersWithQueuedFrames.cend(), layer);
                        if (found != mLayersWithQueuedFrames.cend()) {
                            layersNeedingFences.add(layer);
                        }
                    }
                });
            }
            // 为显示设备注入, 按照 Z 轴排序好的 Layers
            displayDevice->setVisibleLayersSortedByZ(layersSortedByZ);
            // 为显示设备注入, 需要 Fences 等的 GPU 释放缓冲的 Layers
            displayDevice->setLayersNeedingFences(layersNeedingFences);
            ......
        }
    }
}
```
可以看到 rebuildLayerStacks 的主要职责是将每个显示设备的 Layer 按照 Z 轴进行排序, 保存到其显示器上下文 DisplayDevice 中

到这里一个**显示屏的 Layer 便按照 Z 轴排序好了, 下面我们看看最重要的数据和合成**

### 二) 合成 Layer 展示到屏幕
```
void SurfaceFlinger::doComposition() {
    ......
    // 遍历每个显示屏
    for (size_t dpy=0 ; dpy<mDisplays.size() ; dpy++) {
        const sp<DisplayDevice>& hw(mDisplays[dpy]);
        if (hw->isDisplayOn()) {
            const Region dirtyRegion(hw->getDirtyRegion(repaintEverything));

            // 合成 Layer 输出到屏幕
            doDisplayComposition(hw, dirtyRegion);

            hw->dirtyRegion.clear();
            hw->flip();
        }
    }
}
```
好的 SurfaceFlinger::doComposition 函数中是真正的负责每个显示屏 Layer 合成的操作, 下面我们就分析一下 doDisplayComposition 函数中做了什么

```
// frameworks/native/services/surfaceflinger/SurfaceFlinger.cpp
void SurfaceFlinger::doDisplayComposition(
        const sp<const DisplayDevice>& displayDevice,
        const Region& inDirtyRegion)
{
    ......
    // 1. 调用 doComposeSurfaces
    if (!doComposeSurfaces(displayDevice)) return;
    // 2. 置换缓冲
    displayDevice->swapBuffers(getHwComposer());
}

bool SurfaceFlinger::doComposeSurfaces(const sp<const DisplayDevice>& displayDevice)
{
    const Region bounds(displayDevice->bounds());
    const DisplayRenderArea renderArea(displayDevice);
    const auto hwcId = displayDevice->getHwcDisplayId();
    const bool hasClientComposition = getBE().mHwc->hasClientComposition(hwcId);
    .......
    
    if (hasClientComposition) {
        ......
        // 1.1 切换 EGL 上下文
        if (!displayDevice->makeCurrent()) {
            .......
            return false;
        }
        ......
    }

    const Transform& displayTransform = displayDevice->getTransform();
    // 变量 Display 的 Layer 执行绘制操作
    for (auto& layer : displayDevice->getVisibleLayersSortedByZ()) {
        const Region clip(bounds.intersect(
                displayTransform.transform(layer->visibleRegion)));
        if (!clip.isEmpty()) {
            switch (layer->getCompositionType(hwcId)) {
                ......
                case HWC2::Composition::Client: {
                    ......
                    // 1.2 执行绘制操作
                    layer->draw(renderArea, clip);
                    break;
                }
                default:
                    break;
            }
        }
        ......
    }
    ......
    return true;
}
```
好的从 doDisplayComposition 函数中我们看到了 EGL 三个基本操作
- 切换 EGL 上下文
- 执行 Layer 绘制操作
- 置换缓冲到 EGLSurface

接下来我们一一查看

#### 1. 切换 EGL 上下文
```
// frameworks/native/services/surfaceflinger/DisplayDevice.cpp
bool DisplayDevice::makeCurrent() const {
    // 
    bool success = mFlinger->getRenderEngine().setCurrentSurface(*mSurface);
    ......
    return success;
}

// frameworks/native/services/surfaceflinger/RenderEngine/RenderEngine.cpp
bool RenderEngine::setCurrentSurface(const android::RE::Surface& surface) {
    ......
    return setCurrentSurface(static_cast<const android::RE::impl::Surface&>(surface));
}
bool RenderEngine::setCurrentSurface(const android::RE::impl::Surface& surface) {
    bool success = true;
    EGLSurface eglSurface = surface.getEGLSurface();
    if (eglSurface != eglGetCurrentSurface(EGL_DRAW)) {
        // 更新上下文
        success = eglMakeCurrent(mEGLDisplay, eglSurface, eglSurface, mEGLContext) == EGL_TRUE;
        if (success && surface.getAsync()) {
            eglSwapInterval(mEGLDisplay, 0);
        }
    }
    return success;
}
```
可以看到这里调用了 eglMakeCurrent, 也就是说**接下来 Layer.draw 所有的操作, 均会绘制到 RE::Surface 的 EGLSurface 上**

#### 2. Layer.draw
```
// frameworks/native/services/surfaceflinger/Layer.cpp
void Layer::draw(const RenderArea& renderArea, const Region& clip) const {
    onDraw(renderArea, clip, false);
}
```
这里的 onDraw 会调用其子类的实现, 这里我们看看 BufferLayer 的操作
```
// frameworks/native/services/surfaceflinger/BufferLayer.cpp
void BufferLayer::onDraw(const RenderArea& renderArea, const Region& clip,
                         bool useIdentityTransform) const {
    ......
    // 1. 绑定当前 Layer GraphicBuffer 到 OpenGL 的纹理上
    status_t err = mConsumer->bindTextureImage();
    ......
    // 2. 使用 OpenGL 进行绘制
    drawWithOpenGL(renderArea, useIdentityTransform);
    
    // 3. 解绑纹理
    engine.disableTexturing();
}
```
好的, 从这里可以看到 Layer 的 draw 操作起始就是 OpenGL 的渲染管线的相关操作

我们先看看 mConsumer->bindTextureImage 操作
```
// frameworks/native/services/surfaceflinger/BufferLayerConsumer.cpp
status_t BufferLayerConsumer::bindTextureImageLocked() {
    ......
    // 调用了 mRE 的 bindExternalTextureImage 操作
    mRE.bindExternalTextureImage(mTexName, mCurrentTextureImage->image());
    // Wait for the new buffer to be ready.
    return doFenceWaitLocked();
}
```

绑定了纹理之后, 还需要注入纹理的顶点坐标, 其相关操作如下
```
// frameworks/native/services/surfaceflinger/BufferLayer.cpp
void BufferLayer::drawWithOpenGL(const RenderArea& renderArea, bool useIdentityTransform) const {
    const State& s(getDrawingState());
    
    ......
    float left = float(win.left) / float(s.active.w);
    float top = float(win.top) / float(s.active.h);
    float right = float(win.right) / float(s.active.w);
    float bottom = float(win.bottom) / float(s.active.h);

    // 填充 mMesh 的纹理坐标
    Mesh::VertexArray<vec2> texCoords(getBE().mMesh.getTexCoordArray<vec2>());
    texCoords[0] = vec2(left, 1.0f - top);
    texCoords[1] = vec2(left, 1.0f - bottom);
    texCoords[2] = vec2(right, 1.0f - bottom);
    texCoords[3] = vec2(right, 1.0f - top);

    // 获取渲染引擎
    auto& engine(mFlinger->getRenderEngine());
    // 绑定
    engine.setupLayerBlending(mPremultipliedAlpha, isOpaque(s), false /* disableTexture */,
                              getColor());
    // 绘制这个 mMesh
    engine.drawMesh(getBE().mMesh);
    // 解绑
    engine.disableBlending();
    ......
}
```
好的, 到这里关于一个 Layer 的 EGL 渲染代码就完成了, 接下来只需要调用 swapBuffers 就可以将 GPU 的渲染数据同步到 EGLSurface 中了

#### 3. 将数据 swap 到 EGLSurface 缓冲区
```
// frameworks/native/services/surfaceflinger/DisplayDevice.cpp
void DisplayDevice::swapBuffers(HWComposer& hwc) const {
    if (hwc.hasClientComposition(mHwcDisplayId) || hwc.hasFlipClientTargetRequest(mHwcDisplayId)) {
        // 调用了 RE::Surface 的 swapBuffers
        mSurface->swapBuffers();
    }
    // 调用了 FramebufferSurface 的 advanceFrame
    status_t result = mDisplaySurface->advanceFrame();
    ......
}

// frameworks/native/services/surfaceflinger/RenderEngine/Surface.cpp
void Surface::swapBuffers() const {
    // 将缓冲区置换到 mEGLSurface 的 GraphicBuffer 中
    if (!eglSwapBuffers(mEGLDisplay, mEGLSurface)) {
       ......
    }
}
```
可以看到 DisplayDevice 的 swapBuffers 的主要操作如下
- **调用 RE::Surface 的 swapBuffers, 将 EGLSurface 中的数据通过 IGraphicBufferProducer 推入渲染队列**
- **调用了 FramebufferSurface 的 advanceFrame, 获取下一个屏幕的缓冲区**

关于 swapBuffers 的操作, 这里就不再赘述了, 可以[查看 EGL 的专栏](https://sharrychoo.github.io/blog/2019/08/13/opengl-es-2.0-egl.html), 下面看看 FramebufferSurface.advanceFrame 操作

##### FramebufferSurface.advanceFrame
```
// frameworks/native/services/surfaceflinger/DisplayHardware/FramebufferSurface.cpp
status_t FramebufferSurface::advanceFrame() {
    uint32_t slot = 0;
    sp<GraphicBuffer> buf;
    sp<Fence> acquireFence(Fence::NO_FENCE);
    Dataspace dataspace = Dataspace::UNKNOWN;
    // 获取一个屏幕的缓冲
    status_t result = nextBuffer(slot, buf, acquireFence, dataspace);
    mDataSpace = dataspace;
    ......
    return result;
}

status_t FramebufferSurface::nextBuffer(uint32_t& outSlot,
        sp<GraphicBuffer>& outBuffer, sp<Fence>& outFence,
        Dataspace& outDataspace) {
    Mutex::Autolock lock(mMutex);
    BufferItem item;
    // 1. 获取一个 GraphicBuffer, 这个 GraphicBuffer 便是 RE::Surface 中 eglSwapBuffers 推入的
    status_t err = acquireBufferLocked(&item, 0);
    if (err == BufferQueue::NO_BUFFER_AVAILABLE) {
        // 1.2 若队列中没有数据, 则将当前帧推入 HAL 层缓冲区
        mHwcBufferCache.getHwcBuffer(mCurrentBufferSlot, mCurrentBuffer,
                &outSlot, &outBuffer);
        return NO_ERROR;
    } else if (err != NO_ERROR) {
        ......
    }
    ......
    
    // 2. 将当前的帧, 置为上一帧
    if (mCurrentBufferSlot != BufferQueue::INVALID_BUFFER_SLOT &&
        item.mSlot != mCurrentBufferSlot) {
        mHasPendingRelease = true;
        mPreviousBufferSlot = mCurrentBufferSlot;
        mPreviousBuffer = mCurrentBuffer;
    }
    
    // 3. 将新获取的 GraphicBuffer 置为当前帧
    mCurrentBufferSlot = item.mSlot;
    mCurrentBuffer = mSlots[mCurrentBufferSlot].mGraphicBuffer;
    mCurrentFence = item.mFence;

    outFence = item.mFence;
    
    // 4. 将当前帧推入 HAL 的缓冲区中
    mHwcBufferCache.getHwcBuffer(mCurrentBufferSlot, mCurrentBuffer,
            &outSlot, &outBuffer);
    ......
    return NO_ERROR;
}
```
好的, 这里可以看到 FramebufferSurface::advanceFrame 中的主要操作, 便是将合成后的数据, 推入 HAL 层让屏幕展示
- **若渲染队列中无数据, 则继续使用当前帧渲染, 将其推入 HAL 层缓冲**
- **若渲染队列中若有数据, 则将当前帧置为上一帧, 新数据置为当前帧, 再将当前帧推入 HAL 层缓冲**

到这里, 显示器就可以输出我们合成后的数据帧了

## 总结
![SurfaceFlinger 渲染流程](https://i.loli.net/2019/10/23/e1kBUtsMXKR7JbW.png)

通过本片的分析, 我们得知 SurfaceFlinger 的渲染操作主要有如下几步
- INVALIDATE 消息类型
  - Layer 从图形队列中获取 GraphicBuffer
- REFESH 消息类型
  - 将 Layer 按照 Z 轴排序
  - 通过 RE::Surface 进行图形合成 
  - 通过 RE::Surface 将渲染后的数据推入队列
  - FramebufferSurface 从队列中取数据, 推入到 HAL 层的缓冲, 由硬件驱动完成图像的输出

好的, 到这里 SurfaceFlinger 图像渲染的分析就结束了, 其代码还是非常复杂的, 这里只是梳理了其中的流程, 还有很多细节需要我们去探究