---
title: Android 系统架构 —— SurfaceFlinger 对 Hotplug 事件的处理
permalink: android-source/surfaceflinger-hotplug
key: android-source-surfaceflinger-hotplug
tags: AndroidFramework
sidebar:
  nav: android-source
---

## 前言
## 前言
在前面的分析中, 我们了解了 SurfaceFlinger 的启动流程, 得知 SurfaceFlinger 实现了硬件驱动的回调 ComposerCallback
- onHotplugReceived
- onRefreshReceived
- onVsyncReceived

当 IComposer 接收到硬件设备发送的事件时, 会直接回调到 SurfaceFlinger 中统一处理, 这里我们主要分析 Hotplug 热插拔事件

<!--more-->

## 一. SurfaceFlinger 分发事件
在硬件驱动检测到外接显示设备变更时发出, 会由 IComposer 回调到 SurfaceFlinger 中, 下面看看 SurfaceFlinger 对其是如何处理的
```
// frameworks/native/services/surfaceflinger/SurfaceFlinger.cpp
void SurfaceFlinger::onHotplugReceived(int32_t sequenceId, hwc2_display_t display,
                                       HWC2::Connection connection) {
    // 检测 sequenceId 的正确性
    if (sequenceId != getBE().mComposerSequenceId) {
        return;
    }
    // 非主线程时, 进行加锁
    ConditionalLock lock(mStateLock, std::this_thread::get_id() != mMainThreadId);
    // 将热插拔的事件, 添加到 mPendingHotplugEvents 中缓存
    mPendingHotplugEvents.emplace_back(HotplugEvent{display, connection});
    if (std::this_thread::get_id() == mMainThreadId) {
        // 在主线程处理热插拔信号
        processDisplayHotplugEventsLocked();
    }
    ......
}
```
这里可以看到 onHotplugReceived 中, 将热插拔事件添加到待处理集合 mPendingHotplugEvents 中, 然后再主线程中处理热插拔进行
```
// frameworks/native/services/surfaceflinger/SurfaceFlinger.cpp
void SurfaceFlinger::processDisplayHotplugEventsLocked() {
    for (const auto& event : mPendingHotplugEvents) {
        auto displayType = determineDisplayType(event.display, event.connection);
        if (displayType == DisplayDevice::DISPLAY_ID_INVALID) {
            ......
            continue;
        }
        if (getBE().mHwc->isUsingVrComposer() && displayType == DisplayDevice::DISPLAY_EXTERNAL) {
            ......
            continue;
        }
        // 1. 回调了 HwComposer 的 onHotplug
        getBE().mHwc->onHotplug(event.display, displayType, event.connection);
        ......
        // 2. 调用 processDisplayChangesLocked 处理变更
        processDisplayChangesLocked();
    }
    mPendingHotplugEvents.clear();
}
```
SurfaceFlinger.processDisplayHotplugEventsLocked 函数中
- 调用 HwComposer.onHotplug 处理显示设备变更
- 调用 SurfaceFlinger.processDisplayChangesLocked 处理显示设备变更

接下来我们一一分析

## 二. HwComposer 处理显示设备变更
```
// frameworks/native/services/surfaceflinger/DisplayHardware/HWComposer.cpp
void HWComposer::onHotplug(hwc2_display_t displayId, int32_t displayType,
                           HWC2::Connection connection) {
    ......
    mHwcDevice->onHotplug(displayId, connection);
    ......
}

// frameworks/native/services/surfaceflinger/DisplayHardware/HWC2.cpp
void Device::onHotplug(hwc2_display_t displayId, Connection connection) {
    // 1. 处理新的显示设备连接上了
    if (connection == Connection::Connected) {
        // 若之前的设备集合中存在, 则先将其移除
        auto oldDisplay = getDisplayById(displayId);
        mDisplays.erase(displayId);
        // 根据 displayId 获取设备类型
        DisplayType displayType;
        auto intError = mComposer->getDisplayType(displayId,
                reinterpret_cast<Hwc2::IComposerClient::DisplayType *>(
                        &displayType));
        ......
        // 创建一个显示设备的描述 Display
        auto newDisplay = std::make_unique<Display>(
                *mComposer.get(), mCapabilities, displayId, displayType);
        // 将这个设备置为连接状态
        newDisplay->setConnected(true);
        // 添加到设备集合中
        mDisplays.emplace(displayId, std::move(newDisplay));
    } 
    // 2. 处理显示设备断开连接
    else if (connection == Connection::Disconnected) {
        // 将这个 Display 的连接状态置为 false
        auto display = getDisplayById(displayId);
        if (display) {
            display->setConnected(false);
        }
    }
}
```
从这里可以看到 HwComposer.onHotplug 会调用 mHwcDevice 来更新显示器设备的状态, 显示器由 Display 描述, 其定义如下
```
// frameworks/native/services/surfaceflinger/DisplayHardware/HWC2.h
// Convenience C++ class to access hwc2_device_t Display functions directly.
class Display
{
public:
    Display(android::Hwc2::Composer& composer, const std::unordered_set<Capability>& capabilities,
            hwc2_display_t id, DisplayType type);
    ......
    hwc2_display_t getId() const { return mId; }
    bool isConnected() const { return mIsConnected; }
    void setConnected(bool connected);  // For use by Device only
private:
    android::Hwc2::Composer& mComposer;
    const std::unordered_set<Capability>& mCapabilities;
    hwc2_display_t mId;
    bool mIsConnected;
    DisplayType mType;
}
```

## 三. SurfaceFlinger 处理热插拔信号
```
// frameworks/native/services/surfaceflinger/SurfaceFlinger.cpp
DefaultKeyedVector< wp<IBinder>, sp<DisplayDevice> > mDisplays;

void SurfaceFlinger::processDisplayChangesLocked() {
    ......
    const KeyedVector<wp<IBinder>, DisplayDeviceState>& curr(mCurrentState.displays);
    const KeyedVector<wp<IBinder>, DisplayDeviceState>& draw(mDrawingState.displays);
    if (!curr.isIdenticalTo(draw)) {
        mVisibleRegionsDirty = true;
        const size_t cc = curr.size();
        size_t dc = draw.size();
        // 1. 处理设备被移除
        for (size_t i = 0; i < dc;) {
            ......
        }
        // 2. 处理显示设备添加了
        for (size_t i = 0; i < cc; i++) {
            if (draw.indexOfKey(curr.keyAt(i)) < 0) {
                const DisplayDeviceState& state(curr[i]);
                sp<DisplaySurface> dispSurface;
                sp<IGraphicBufferProducer> producer;
                sp<IGraphicBufferProducer> bqProducer;
                sp<IGraphicBufferConsumer> bqConsumer;
                // 2.1 为这个显示设备创建一个 生产者-消费者 队列
                mCreateBufferQueue(&bqProducer, &bqConsumer, false);

                int32_t hwcId = -1;
                
                if (state.isVirtualDisplay()) {
                    ......// 处理 VR 设备
                } else {
                    ......//  
                    hwcId = state.type;
                    // 2.2 创建显示设备的消费者
                    dispSurface = new FramebufferSurface(*getBE().mHwc, hwcId, bqConsumer);
                    producer = bqProducer;
                }
                const wp<IBinder>& display(curr.keyAt(i));
                if (dispSurface != nullptr) {
                    mDisplays.add(display,
                                  // 2.3 创建了一个新的 DisplayDevice 对象加入缓存
                                  setupNewDisplayDeviceInternal(display, hwcId, state, dispSurface,
                                                                producer));
                    if (!state.isVirtualDisplay()) {
                        // 回调 mEventThread 的热插拔
                        mEventThread->onHotplugReceived(state.type, true);
                    }
                }
            }
        }
    }
    mDrawingState.displays = mCurrentState.displays;
}
```
可以看到 processDisplayChangesLocked 也是从接入和拔出两个操作进行处理, 这里我们主要关心设备添加的过程
- **创建显示屏幕的 生产者-消费者 队列**
- **创建队列的消费者 FramebufferSurface**
- **创建 DisplayDevice 对象加入缓存**

接下来先看看他是如何创建生产者-消费者队列的

### 一) 创建显示屏幕的 生产者-消费者 队列
mCreateBufferQueue 是一个函数操作函数, 它在 SurfaceFlinger 构造时创建
```
SurfaceFlinger::SurfaceFlinger(SurfaceFlinger::SkipInitializationTag)
      : BnSurfaceComposer(),
        ......
        mCreateBufferQueue(&BufferQueue::createBufferQueue),
        ......
```
其实现如下
```
// frameworks/native/libs/gui/BufferQueue.cpp
void BufferQueue::createBufferQueue(
        // 传出参入
        sp<IGraphicBufferProducer>* outProducer,
        // 传出参数
        sp<IGraphicBufferConsumer>* outConsumer,
        bool consumerIsSurfaceFlinger) {
    // 1. 创建了 BufferQueueCore
    sp<BufferQueueCore> core(new BufferQueueCore());
    // 2. 创建生产者
    sp<IGraphicBufferProducer> producer(new BufferQueueProducer(core, consumerIsSurfaceFlinger));
    // 3. 创建消费者
    sp<IGraphicBufferConsumer> consumer(new BufferQueueConsumer(core));
    // 4. 为传出参数赋值
    *outProducer = producer;
    *outConsumer = consumer;
}
```
好的, 可以看到 BufferQueue::createBufferQueue 的实现非常清晰, 创建了三个非常重要的对象
- **BufferQueueCore**
- **BufferQueueProducer**
- **BufferQueueConsumer**

接下来看看他们的定义
#### 1. [BufferQueueCore](http://androidxref.com/9.0.0_r3/xref/frameworks/native/include/gui/BufferQueueCore.h)
```
namespace android {
class BufferQueueCore : public virtual RefBase {
    // 生产者
    friend class BufferQueueProducer;
    // 消费者
    friend class BufferQueueConsumer;
public:
    ......
    // 先入先出队列
    typedef Vector<BufferItem> Fifo;
    
    // BufferQueueCore manages a pool of gralloc memory slots to be used by
    // producers and consumers.
    BufferQueueCore();

private:
    // mQueue is a FIFO of queued buffers used in synchronous mode.
    Fifo mQueue;
    ......
    // mSlots is an array of buffer slots that must be mirrored on the producer
    // side. This allows buffer ownership to be transferred between the producer
    // and consumer without sending a GraphicBuffer over Binder. The entire
    // array is initialized to NULL at construction time, and buffers are
    // allocated for a slot when requestBuffer is called with that slot's index.
    BufferQueueDefs::SlotsType mSlots;
}
```
可以看到 BufferQueueCore 的定义如上述代码所示
- **其内部持有一个先入先出队列 mQueue, 用于存储生产者投递过来的图像缓冲 GraphicBuffer**
- **mSlots 可以理解为一个 Buffer 的享元复用池**, 在生产端使用, 它允许生产者通过 dequeueBuffer 从中获取一个空闲缓冲, 避免生产者自己开辟缓冲区造成的内存消耗

#### 2. [BufferQueueProducer](http://androidxref.com/9.0.0_r3/xref/frameworks/native/include/gui/BufferQueueProducer.h)
```
class BufferQueueProducer : public BnGraphicBufferProducer,
                            private IBinder::DeathRecipient {
public:
    friend class BufferQueue; // Needed to access binderDied

    BufferQueueProducer(const sp<BufferQueueCore>& core, bool consumerIsSurfaceFlinger = false);      
    
    // 从享元复用池中获取空闲缓冲
    virtual status_t dequeueBuffer(int* outSlot, sp<Fence>* outFence, uint32_t width,
                               uint32_t height, PixelFormat format, uint64_t usage,
                               uint64_t* outBufferAge,
                               FrameEventHistoryDelta* outTimestamps) override;  
    
    // 将数据投入队列
    virtual status_t queueBuffer(int slot,
            const QueueBufferInput& input, QueueBufferOutput* output);

private:
    sp<BufferQueueCore> mCore;
    // This references mCore->mSlots. Lock mCore->mMutex while accessing.
    BufferQueueDefs::SlotsType& mSlots;
}
```
可以看到 BufferQueueProducer 继承自 BnGraphicBufferProducer, 它是 IGraphicBufferProducer 的 Binder 本地对象, 相应的在应用进程中 Surface 持有的 IGraphicBufferProducer 即 Binder 代理对象, 所有的操作最终都会通过 BufferQueueProducer 来真正的落实
- dequeueBuffer: 从 mSlots 中获取空闲缓冲区
- queueBuffer: 将缓冲区投入队列

#### 3.[BufferQueueConsumer](http://androidxref.com/9.0.0_r3/xref/frameworks/native/include/gui/BufferQueueConsumer.h)
```
class BufferQueueConsumer : public BnGraphicBufferConsumer {
public:
    BufferQueueConsumer(const sp<BufferQueueCore>& core);
    
private:
    sp<BufferQueueCore> mCore;
    // This references mCore->mSlots. Lock mCore->mMutex while accessing.
    BufferQueueDefs::SlotsType& mSlots;
    
}
```
BufferQueueConsumer 继承自 BnGraphicBufferConsumer, 这是 IGraphicBufferConsumer 的 Binder 本地对象, 所有消费者相关的操作均在其中实现

好的, 这里主要是为了理清各个类的职责, 关于他们具体的实现到后面的章节进行重点分析

### 二) 创建队列的消费者 [FramebufferSurface](http://androidxref.com/9.0.0_r3/xref/frameworks/native/services/surfaceflinger/DisplayHardware/FramebufferSurface.h)
```
/frameworks/native/services/surfaceflinger/DisplayHardware/FramebufferSurface.cpp
FramebufferSurface::FramebufferSurface(HWComposer& hwc, int disp,
        const sp<IGraphicBufferConsumer>& consumer) :
    // 继承自 ConsumerBase
    ConsumerBase(consumer),
    mDisplayType(disp),
    mCurrentBufferSlot(-1),
    mCurrentBuffer(),
    mCurrentFence(Fence::NO_FENCE),
    mHwc(hwc),
    mHasPendingRelease(false),
    mPreviousBufferSlot(BufferQueue::INVALID_BUFFER_SLOT),
    mPreviousBuffer()
{   
    // 配置 Consumer 的相关选项
    mName = "FramebufferSurface";
    mConsumer->setConsumerName(mName);
    mConsumer->setConsumerUsageBits(GRALLOC_USAGE_HW_FB |
                                       GRALLOC_USAGE_HW_RENDER |
                                       GRALLOC_USAGE_HW_COMPOSER);
    // 1. 设置默认 Buffer 的大小
    const auto& activeConfig = mHwc.getActiveConfig(disp);
    mConsumer->setDefaultBufferSize(activeConfig->getWidth(),
            activeConfig->getHeight());
    // 2. 设置最大可请求的 Buffer 数量
    mConsumer->setMaxAcquiredBufferCount(
            SurfaceFlinger::maxFrameBufferAcquiredBuffers - 1);
}

// frameworks/native/services/surfaceflinger/SurfaceFlinger.cpp
maxFrameBufferAcquiredBuffers = getInt64< ISurfaceFlingerConfigs,
            &ISurfaceFlingerConfigs::maxFrameBufferAcquiredBuffers>(2);
```
**从 FramebufferSurface 的构造中可知, 它是 GraphicQueue 的消费者端**

接下来看看 DisplayDevice 的创建

### 三) 创建 [DisplayDevice](http://androidxref.com/9.0.0_r3/xref/frameworks/native/services/surfaceflinger/DisplayDevice.h)
```
sp<DisplayDevice> SurfaceFlinger::setupNewDisplayDeviceInternal(
        const wp<IBinder>& display, int hwcId, const DisplayDeviceState& state,
        // 队列消费者的 FramebufferSurface
        const sp<DisplaySurface>& dispSurface, 
        // 队列生产者
        const sp<IGraphicBufferProducer>& producer) {
    bool hasWideColorGamut = false;
    std::unordered_map<ColorMode, std::vector<RenderIntent>> hwcColorModes;

    ......

    HdrCapabilities hdrCapabilities;
    getHwComposer().getHdrCapabilities(hwcId, &hdrCapabilities);
    
    // 1. 创建显示设备的生产者 NativeWindowSurface
    auto nativeWindowSurface = mCreateNativeWindowSurface(producer);
    auto nativeWindow = nativeWindowSurface->getNativeWindow();

    // 2. 创建 GL 渲染工具
    std::unique_ptr<RE::Surface> renderSurface = getRenderEngine().createSurface();
    ......
    // 2.1 创建 EGLSurface (让 EGLSurface 绑定 Surface 中的缓冲区)
    renderSurface->setNativeWindow(nativeWindow.get());
    ......
    
    // 3. 创建显示设备的描述
    sp<DisplayDevice> hw =
            new DisplayDevice(this, state.type, hwcId, state.isSecure, display, nativeWindow,
                              dispSurface, std::move(renderSurface), displayWidth, displayHeight,
                              hasWideColorGamut, hdrCapabilities,
                              getHwComposer().getSupportedPerFrameMetadata(hwcId),
                              hwcColorModes, initialPowerMode);
    ......
    return hw;
}
```
setupNewDisplayDeviceInternal 的主要步骤如下
- 创建 NativeWindowSurface, 即屏幕显示数据的生产者
- 创建显示器的 GL 渲染器 RE::Surface
  - 其 EGLSurface 绑定了 Surface 中的缓冲区
- 将相关参数封装到 DisplayDevice 中

#### 1. NativeWindowSurface 的创建
```
// frameworks/native/services/surfaceflinger/SurfaceFlinger.cpp
class NativeWindowSurface final : public android::NativeWindowSurface {
public:
    static std::unique_ptr<android::NativeWindowSurface> create(
            const sp<IGraphicBufferProducer>& producer) {
        return std::make_unique<NativeWindowSurface>(producer);
    }

    // 初始化一个 Surface
    explicit NativeWindowSurface(const sp<IGraphicBufferProducer>& producer)
          : surface(new Surface(producer, false)) {}

    ~NativeWindowSurface() override = default;

private:
    // 获取 Surface
    sp<ANativeWindow> getNativeWindow() const override { return surface; }

    // 给 Surface 分配 Buffer
    void preallocateBuffers() override { surface->allocateBuffers(); }

    sp<Surface> surface;
};
```
NativeWindowSurface 实现在 SurfaceFlinger 中, 它内部持有了一个 Surface, 通过 getNativeWindow 便可以获取到这个 Surface

#### 2. [RE::Surface](http://androidxref.com/9.0.0_r3/xref/frameworks/native/services/surfaceflinger/RenderEngine/Surface.cpp)
```
// frameworks/native/services/surfaceflinger/RenderEngine/Surface.cpp
namespace android {

namespace RE {

namespace impl {

Surface::Surface(const RenderEngine& engine)
      :
      // 获取了 EGLDispalay
      mEGLDisplay(engine.getEGLDisplay()),
      // 获取了 EGLCongif
      mEGLConfig(engine.getEGLConfig()) {
    ......
}

void Surface::setNativeWindow(ANativeWindow* window) {
    // 销毁之前的 EGLSurface
    if (mEGLSurface != EGL_NO_SURFACE) {
        eglDestroySurface(mEGLDisplay, mEGLSurface);
        mEGLSurface = EGL_NO_SURFACE;
    }
    // 创建新的 EGLSurface
    mWindow = window;
    if (mWindow) {
        mEGLSurface = eglCreateWindowSurface(mEGLDisplay, mEGLConfig, mWindow, nullptr);
    }
}

}
```
好的, 可以看到 RE::Surface 中, 即 EGL 的工具类, 其 setNativeWindow 会上面将传入的 NativeWindowSurface 中的 Surfcae 封装成 EGLSurface

有了这个 RE::Surface 就可以使用 OpenGL 的渲染管线了, 渲染的数据会存储到 EGLSurface 中

#### 3. DisplayDevice 的定义
```
class DisplayDevice : public LightRefBase<DisplayDevice>
{
    // clang-format off
    DisplayDevice(
            const sp<SurfaceFlinger>& flinger,
            DisplayType type,
            int32_t hwcId,
            bool isSecure,
            const wp<IBinder>& displayToken,
            const sp<ANativeWindow>& nativeWindow,
            const sp<DisplaySurface>& displaySurface,
            std::unique_ptr<RE::Surface> renderSurface,
            int displayWidth,
            int displayHeight,
            bool hasWideColorGamut,
            const HdrCapabilities& hdrCapabilities,
            const int32_t supportedPerFrameMetadata,
            const std::unordered_map<ui::ColorMode, std::vector<ui::RenderIntent>>& hwcColorModes,
            int initialPowerMode);
    ......

private:
    /*
     *  Constants, set during initialization
     */
    sp<SurfaceFlinger> mFlinger;
    DisplayType mType;
    int32_t mHwcDisplayId;
    wp<IBinder> mDisplayToken;

    // ANativeWindow this display is rendering into
    sp<ANativeWindow> mNativeWindow;
    sp<DisplaySurface> mDisplaySurface;
    
    std::unique_ptr<RE::Surface> mSurface;
    ......
}
```
从这里可以看出 **DisplayDevice 其实可以理解为显示屏幕的上下文**, 它内部维护了这块显示设备与外界交互的数据

### 四) 回顾
![SurfaceFlinger 相关结构依赖图](https://i.loli.net/2019/12/13/2wOzjgBHVo38EJf.png)

SurfaceFlinger 处理新增显示屏的流程如下
- 创建屏幕的生产者-消费者队列
- 创建屏幕的消费者 FramebufferSurface
- 创建屏幕的对象 DisplayDevice
  - 创建屏幕生产者 NativeWindowSurface
  - 创建显示器的 GL 渲染器 RE::Surface
    - 其 EGLSurface 绑定了生产者中的缓冲区
  - 将相关数据封装到 DisplayDevice 中

## 总结
当硬件驱动出现热插拔信号时, 会回调 SurfaceFinger 的 onHotplugReceived 函数, 主要处理流程如下
- **HwComposer 处理显示设备变更**
  - HwComposer.onHotplug 会调用 mHwcDevice 来更新显示器设备的状态
- **SurfaceFlinger 处理新增显示屏的流程如下**
  - 创建屏幕的生产者-消费者队列 BufferQueueCore
  - 创建屏幕的消费者 FramebufferSurface
  - 创建屏幕的对象 DisplayDevice
    - 创建屏幕生产者 NativeWindowSurface
    - 创建显示器的 GL 渲染器 RE::Surface
      - 其 EGLSurface 绑定了生产者中的缓冲区
    - 将相关数据封装到 DisplayDevice 中

![屏幕渲染图示](https://i.loli.net/2019/12/13/mMuEUNZyKavVT8P.png)

到这里我们已经看到了一块显示屏幕有一个自己的生产者消费者队列
- **生产者为 NativeWindowSurface** 
  - RE::Surface 它是一个 EGL 的工具类
    - 其内部的 EGLSurface 绑定了生产者 NativeWindowSurface 中的缓冲区
    - 通过 EGL 的渲染之后的图像数据会存储到 EGLSurface 绑定的缓冲区中
    - 通过 eglSwapBuffer, 将生产好的数据推送到屏幕的 Buffer 队列
- **消费者为 FramebufferSurface**
  - 它负责从屏幕 Buffer 队列中取数据, 将其推送给硬件屏幕呈现出来
