---
layout: article
title: "Android 系统架构 —— ViewRootImpl 的 Surface 创建"
key: "Android 系统架构 —— ViewRootImpl 的 Surface 创建" 
tags: AndroidFramework
aside:
  toc: true
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
  - **调用 WindowStateAnimator.createSurfaceLocked 获取画布控制器 surfaceController**
  - **通过控制器的 WindowSurfaceController.getSurface 将数据写入到 outSurface 中** 

接下来我们一步一步分析, 先看看如何创建画布控制器

### 一) 创建 SurfaceControl
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

#### 1. 构建 java 的 SurfaceControl
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


#### 2. 构建 native 的 SurfaceControl
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

// frameworks/native/libs/gui/ISurfaceComposerClient.cpp
class BpSurfaceComposerClient : public SafeBpInterface<ISurfaceComposerClient> {
public:
    // 1.1 调用 ISurfaceComposerClient 的 Binder 代理对象的 createSurface 方法
    // 请求 SurfaceFlinger 为其分配一个 IGraphicBufferProducer 图像缓存区
    status_t createSurface(......, sp<IGraphicBufferProducer>* gbp) override {
        return callRemote<decltype(&ISurfaceComposerClient::createSurface)>(......, gbp);
    }
}

// frameworks/native/libs/gui/SurfaceControl.cpp
// 2.1 将 gbp 这个 Binder 代理对象保存到了成员变量 mGraphicBufferProducer 中
SurfaceControl::SurfaceControl(
        const sp<SurfaceComposerClient>& client,
        const sp<IBinder>& handle,
        const sp<IGraphicBufferProducer>& gbp,
        bool owned)
    : mClient(client), mHandle(handle), mGraphicBufferProducer(gbp), mOwned(owned)
{}
```
好的, 可见构造 SurfaceControl 对象主要有两步操作
- 调用 BpSurfaceComposerClient.createSurface, 通过 SurfaceFlinger 进程获取一个 IGraphicBufferProducer 代理对象
- 创建一个 SurfaceControl 对象, 并且将 IGraphicBufferProducer 保存在 mGraphicBufferProducer 中

**这个 IGraphicBufferProducer 代理对象非常的重要, 它是 SurfaceFlinger 为这个画布控制器分配的一个数据缓冲描述, 用于缓冲要渲染的数据**

如此一来 Native 层的 SurfaceControl 就构建好了

#### 3. SurfaceControl 结构分析
![SurfaceControl 结构分析](https://i.loli.net/2019/10/23/8kXraMuEwUOnDNH.png)

创建好了 SurfaceControl, 接下来我们看看 WindowSurfaceController.getSurface 如何将构建好的 SurfaceControl 数据输出到 outSurface 中

### 二) 构建画布 Surface
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

#### 创建 Surface Native 对象
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

至此 Java 层的 outSurface 中就可以获取到创建的句柄值了, **接下来要做的就是将 WMS 中构建好的 outSurface, 传送回应用进程了**

## 二. 创建应用进程的 Surface
好的, 可以看到通过上面的操作, WMS 中就给这个 outSurface 对象已经绑定了 Native 层的 Surface 了, 后面要做的事情, 就是将这个 Surface 返回给应用进程了, 这个过程非常的有趣, 笔者苦思冥想良久, 也没有想出它是如何传递回应用进程的

```
// frameworks/base/core/java/android/view/IWindowSession.aidl
interface IWindowSession {
    int relayout(IWindow window, int seq, in WindowManager.LayoutParams attrs,
            int requestedWidth, int requestedHeight, int viewVisibility,
            int flags, long frameNumber, out Rect outFrame, out Rect outOverscanInsets,
            out Rect outContentInsets, out Rect outVisibleInsets, out Rect outStableInsets,
            out Rect outOutsets, out Rect outBackdropFrame,
            out DisplayCutout.ParcelableWrapper displayCutout,
            out MergedConfiguration outMergedConfiguration, 
            out Surface outSurface// 这个 outSurface 参数前被标明了为 out 类型
    );
}
```
我们知道 ViewRootImpl 是通过 IWindowSession 和 WMS 通信的, 请求重置 Surface 画布大小调用的是 IWindowSession.relayout 方法, 

从其 AIDL 文件中可以看到其中的 outSurface 参数为 out 类型, 将其 .aidl 还原成 Java 代码后, 如下所示
```
// IWindowSession 的 Binder 代理对象
public int relayout(android.view.IWindow window, int seq,
		android.view.WindowManager.LayoutParams attrs,
		int requestedWidth, int requestedHeight,
		...
		android.view.Surface outSurface)
		throws android.os.RemoteException {
	android.os.Parcel _data = android.os.Parcel.obtain();
	android.os.Parcel _reply = android.os.Parcel.obtain();
	int _result;
	try {
		_data.writeInterfaceToken(DESCRIPTOR);
		_data.writeStrongBinder((((window != null)) ? (window.asBinder()) : (null)));
		...
		_data.writeInt(viewVisibility);
		_data.writeInt(flags);
		// 1. 调用  IWindowSession 的 Binder 代理对象, 将 outSurface 等数据发送给 WMS 进程
		mRemote.transact(Stub.TRANSACTION_relayout, _data, _reply,0);
		_reply.readException();
		_result = _reply.readInt();
		…
        // 4. 从 WMS 进程中返回的 reply 中读取 Surface 对象数据, 重新给应用进程的 outSurface 赋值
		if ((0 != _reply.readInt())) {
			outSurface.readFromParcel(_reply);
		}
	} finally {
		_reply.recycle();
		_data.recycle();
	}
	return _result;
}

// IWindowSession 的 Binder 实体对象
public boolean onTransact(int code, android.os.Parcel data,android.os.Parcel reply, int flags)throws android.os.RemoteException {
	switch (code) {
	case TRANSACTION_relayout: {
		data.enforceInterface(DESCRIPTOR);
		android.view.IWindow _arg0;
		_arg0 = android.view.IWindow.Stub.asInterface(data.readStrongBinder());
		…
		android.view.Surface _arg13;
		_arg13 = new android.view.Surface();
		// 2. IWindowSession 实体对象接收应用进程传递的参数, 调用 Session.relayout 方法执行操作
		int _result = this.relayout(_arg0, _arg1, _arg2, _arg3, _arg4,
				_arg5, _arg6, _arg7, _arg8, _arg9, _arg10, _arg11,_arg12, _arg13);
		reply.writeNoException();
		reply.writeInt(_result);
		….
		if ((_arg13 != null)) {
			reply.writeInt(1);
			// 3. 操作执行成功之后, writeToParcel 将 WMS 中构建的 outSurface 写入 reply 中
			_arg13.writeToParcel(reply,android.os.Parcelable.PARCELABLE_WRITE_RETURN_VALUE);
		} else {
			reply.writeInt(0);
		}
		return true;
	}
	}
	return super.onTransact(code, data, reply, flags);
}
```
非常精彩的部分来了, 这次的 Binder 通信主要有如下操作
- 调用  IWindowSession 的 Binder 代理对象, **将 outSurface 等数据封装成 Parcel, 发送给 WMS 进程**
- IWindowSession Binder 实体对象接收应用进程传递的参数, **调用 Session.relayout 方法执行操作**
  - 这个方法就是我们上面在 WMS 中构建 Surface 的所有操作
- 操作执行成功之后, writeToParcel **将 WMS 中构建的 outSurface 写入 reply 中, 即将返回给应用进程**
- 从 WMS 进程中返回的 reply 中读取 Surface 对象数据, **调用 outSurface.readFromParcel 重新给应用进程的 outSurface 赋值**

从上面的操作中, 有两个关键点
- 系统服务进程: Surface.writeToParcel(reply, android.os.Parcelable.PARCELABLE_WRITE_RETURN_VALUE);
- 应用进程: 	Surface.readFromParcel(_reply);

因此, 从这两个关键点入手, 我们继续往下探究

### 一) Surface.writeToParcel 
```
public class Surface implements Parcelable {
    
    @Override
    public void writeToParcel(Parcel dest, int flags) {
        ......
        synchronized (mLock) {
            ......
            // 调用了 nativeWriteToParcel 到了 Native 层执行操作
            nativeWriteToParcel(mNativeObject, dest);
        }
        ......
    }
    
}
```
这里看看 native 实现
```
// frameworks/base/core/jni/android_view_Surface.cpp
static void nativeWriteToParcel(JNIEnv* env, jclass clazz,
        jlong nativeObject, jobject parcelObj) {
    // 1. 获取 Java 层 Parcel 对应的 Native 实例
    Parcel* parcel = parcelForJavaObject(env, parcelObj);
    ......
    // 2. 获取 Java 层 Surface 对应的 Native 强引用
    sp<Surface> self(reinterpret_cast<Surface *>(nativeObject));
    // 3. 声明了一个 Native 层的 Surface 对象 surfaceShim
    android::view::Surface surfaceShim;
    // 4. 将 self 中的图像缓冲的生产者 Binder 实例拷贝给 surfaceShim 中
    if (self != nullptr) {
        surfaceShim.graphicBufferProducer = self->getIGraphicBufferProducer();
    }
    // 5. 调用了 Surface.writeToParcel 
    surfaceShim.writeToParcel(parcel, /*nameAlreadyWritten*/true);
}

// frameworks/native/libs/gui/view/Surface.cpp
status_t Surface::writeToParcel(Parcel* parcel, bool nameAlreadyWritten) const {
    if (parcel == nullptr) return BAD_VALUE;

    status_t res = OK;
    // 已经写过 name 了
    if (!nameAlreadyWritten) {
        res = parcel->writeString16(name);
        if (res != OK) return res;

        /* isSingleBuffered defaults to no */
        res = parcel->writeInt32(0);
        if (res != OK) return res;
    }
    // 调用 IGraphicBufferProducer 的静态方法 exportToParcel
    return IGraphicBufferProducer::exportToParcel(graphicBufferProducer, parcel);
}

// frameworks/native/libs/gui/IGraphicBufferProducer.cpp
status_t IGraphicBufferProducer::exportToParcel(const sp<IGraphicBufferProducer>& producer,
                                                Parcel* parcel) {
    ......
    if (producer == nullptr) {
        status_t res = OK;
        res = parcel->writeUint32(IGraphicBufferProducer::USE_BUFFER_QUEUE);
        if (res != NO_ERROR) return res;
        return parcel->writeStrongBinder(nullptr);
    } else {
        // 调用了 IGraphicBufferProducer.exportToParcel
        return producer->exportToParcel(parcel);
    }
}

status_t IGraphicBufferProducer::exportToParcel(Parcel* parcel) {
    status_t res = OK;
    res = parcel->writeUint32(USE_BUFFER_QUEUE);
    ......
    // 将一个 IGraphicBufferProducer Binder 代理对象写入了 Parcel
    return parcel->writeStrongBinder(IInterface::asBinder(this));
}
```
好的, 可以看到服务进程的 **Surface.writeToParcel 操作将 SurfaceFlinger 为其分配的缓存区代理对象写入了 Parcel**

接下来我们看看应用进程的 Surface.readFromParcel 的操作

### 二) Surface.readFromParcel
```
public class Surface implements Parcelable {
    
    public void readFromParcel(Parcel source) {
        ......
        synchronized (mLock) {
            ......
            // 调用了 nativeReadFromParcel 获取一个 native 的 Surface 句柄, 保存在 mNativeObject 中
            setNativeObjectLocked(nativeReadFromParcel(mNativeObject, source));
        }
    }
    
    private static native long nativeReadFromParcel(long nativeObject, Parcel source);
 
    long mNativeObject;
 
    private void setNativeObjectLocked(long ptr) {
        if (mNativeObject != ptr) {
            ......
            mNativeObject = ptr;
            ......
        }
    }
    
}
```
可以看到 Surface.readFromParcel 中关键的操作在于**通过 nativeReadFromParcel 获取了一个 Surface(native) 的句柄**, 我们继续分析
```
// frameworks/base/core/jni/android_view_Surface.cpp
static jlong nativeReadFromParcel(JNIEnv* env, jclass clazz,
        jlong nativeObject, jobject parcelObj) {
    Parcel* parcel = parcelForJavaObject(env, parcelObj);
    ......
    // 1. 声明了一个 Native 层的 Surface 对象 surfaceShim
    android::view::Surface surfaceShim;
    // 2. 调用 Surface.readFromParcel 为其赋值
    surfaceShim.readFromParcel(parcel, /*nameAlreadyRead*/true);
    .....
    sp<Surface> sur;
    if (surfaceShim.graphicBufferProducer != nullptr) {
        // 3. 若 surfaceShim 中存在的缓冲区, 则说明这个 Surface 是有意义的
        sur = new Surface(surfaceShim.graphicBufferProducer, true);
        sur->incStrong(&sRefBaseOwner);
    }
    // 4. 将其句柄返回给 Java 层
    return jlong(sur.get());
}
```
其过程也非常简单, 就是从 Parcel 中获取服务进程写入的 IGraphicBufferProducer Binder 代理对象,  然后在应用进程中构建一个绑定了该缓冲区的 Surface(native) 对象, 如此一来这个应用进程的 Surface 就构建完毕了

## 总结
通过这次分析, 我们对 Surface 有了更深一层次的理解, 这里我们先回顾一下这个呢 relayoutWindow 的流程

![relayoutWindow 时序图](https://i.loli.net/2019/10/23/HAtmN5I9R6UprTb.png)

具体的流程如下
- 当客户端 View 测量的数据导致窗体大小变更时, 会调用 relayout 通知 WMS 中窗体的尺寸变更了
- WMS 中窗体是由 WindowState 描述的, 尺寸变更了自然要引起其对应的画布尺寸变更, WindowState 中有一个 WindowStateAnimator 类型的成员变量 mWinAnimator, 它用于管理这个窗体的画布
- WindowStateAnimator 内部有一个 WindowSurfaceControl 对象, 它是真正持有 SurfaceControl 的地方, 它会重新创建一个 SurfaceControl 保存在其成员变量 mSurfaceControl 中
- mSurfaceControl 创建好之后, 便会调用 outSurface 的 copyFrom 将 SurfaceControlNative 中持有的缓冲区拷贝到 outSurface 的 native 对象中
  - 一个拥有 IGraphicBufferProducer 的 Surface 才是有意义的, 否则他的数据无法得到 SurfaceFlinger 的渲染

**从上面的流程可以看到 WMS 中其实是不需要 Surface 对象的, 它只需要持有 SurfaceControl 就可以了, 为什么还要大费周折的将 SurfaceControl 的图像缓冲区拷贝给 outSurface 呢?**

通过给应用进程的 Surface 赋值我们知道, 这个 outSurface 是应用进程通过 Binder 传递过来的, 若不给它赋值, 那么 relayout 结束之后, 将一个没有绑定图像缓存区 的 Surface 写入 Parcel 是没有意义的

### 对象之间的依赖
![对象之间的依赖](https://i.loli.net/2019/10/23/gv7LjelH9IzQaYE.png)