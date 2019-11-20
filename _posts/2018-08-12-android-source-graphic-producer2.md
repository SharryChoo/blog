---
layout: article
title: "Android 系统架构 —— Window 与 View 的关联"
key: "Android 系统架构 —— Window 与 View 的关联" 
tags: AndroidFramework
aside:
  toc: true
---

## 前言
通过前面文章的分析, 我们得知 Window 是通过内部维护的 DecorView 与 View 建立联系的, 本次我们便看看 View 是如何确定自己的位置的

<!--more-->

分析过 Activity 的启动流程, 我们知道 Activity 视图的测量摆放和渲染是在 onResume 之后执行的, 它的发起方如下
```
public final class ActivityThread extends ClientTransactionHandler {

    @Override
    public void handleResumeActivity(IBinder token, boolean finalStateRequest, boolean isForward,
            String reason) {
        // 执行 Activity 的 Start 和 Resume 操作
        final ActivityClientRecord r = performResumeActivity(token, finalStateRequest, reason);
        final Activity a = r.activity;
        // mStartedActivity 在这个 Activity 准备跳转到其他页面时, 会被赋值为 true
        // 若并非需要跳转到其他页面, 则准备进行这个 Activity 的视图的测量、布局和绘制
        boolean willBeVisible = !a.mStartedActivity;
        if (r.window == null && !a.mFinished && willBeVisible) {
            // 1. 获取 Activity 绑定的 Window 
            r.window = r.activity.getWindow();
            // 2. 获取 Window 中的 DecorView
            View decor = r.window.getDecorView();
            decor.setVisibility(View.INVISIBLE);
            // 3. 获取 Activity 的 WindowManager
            ViewManager wm = a.getWindowManager();
            // 4. 获取 Window 的布局参数
            WindowManager.LayoutParams l = r.window.getAttributes();
            ......
            if (a.mVisibleFromClient) {
                if (!a.mWindowAdded) {
                    a.mWindowAdded = true;
                    // 5. 将这 DecorView 和该 Window 的布局参数传入 WidnowManager.addView 方法
                    wm.addView(decor, l);
                } else {
                    ......
                }
            }
            .......
        } else if (!willBeVisible) {
            ......
        }
        ......
    }
}
```
可以看到 ActivityThread.handleResumeActivity 在回调了 onStart 和 onResume 之后, 便会进行调用 WindowManager.addView

**这里有个疑问, 我们知道在 Window.setContentView 时, 就已经将 DecorView 保存到 Window 中的成员变量 mDecor 中, 这里调用 WindowManager.addView 方法的意义是什么呢? 难道再添加一次? 叫 WindowManager.setupView 是否会更合适呢?**

好的, 带着心中的疑惑, 接下来我们看看 WindowManager.addView 中做了哪些操作

## 一. WindowManager.addView
```
public class WindowManagerImpl implements WindowManager {
    
    // 桥接对象, 用于转移 WindowManager 中的具体操作
    private final WindowManagerGlobal mGlobal = WindowManagerGlobal.getInstance();
    private final Context mContext;
    private final Window mParentWindow;
    
    @Override
    public void addView(@NonNull View view, @NonNull ViewGroup.LayoutParams params) {
        ......
        mGlobal.addView(view, params, mContext.getDisplay(), mParentWindow);
    }
}

public final class WindowManagerGlobal {
    
    private static WindowManagerGlobal sDefaultWindowManager;
    
    // 获取 WindowManagerGlobal 单例
    // 这种方式显然不够优雅
    public static WindowManagerGlobal getInstance() {
        synchronized (WindowManagerGlobal.class) {
            if (sDefaultWindowManager == null) {
                sDefaultWindowManager = new WindowManagerGlobal();
            }
            return sDefaultWindowManager;
        }
    }
}
```
好的, 可以看到 WindowManagerImpl.addView 的操作, 直接转发给了 WindowManagerGlobal

我们知道 WindowManager 是为了管理与之绑定的 Window 的, 那么 **WindowManagerGlobal 顾名思义它是用于管理当前进程中所有的 Window 的**, 从它使用了单例设计模式便可初步印证我们的想法

接下来我们来看看 WindowManagerGlobal 是如何执行 addView 操作的
```
public final class WindowManagerGlobal {

    // 缓存当前进程中所有的正在展示 DecorView
    private final ArrayList<View> mViews = new ArrayList<View>();
    // 缓存当前进程中所有真正展示的 DecorView 的管理者
    private final ArrayList<ViewRootImpl> mRoots = new ArrayList<ViewRootImpl>();
    // 缓存当前进程中所有 DecorView 的布局参数
    private final ArrayList<WindowManager.LayoutParams> mParams =
            new ArrayList<WindowManager.LayoutParams>();
    
    public void addView(View view, ViewGroup.LayoutParams params,
            Display display, Window parentWindow) {
        ......
        final WindowManager.LayoutParams wparams = (WindowManager.LayoutParams) params;
        ......
        // 声明 ViewRootImpl
        ViewRootImpl root;
        View panelParentView = null;
        synchronized (mLock) {
            ......// 添加系统属性变更的观察者
            // 1. 查找当前 View 在 mViews 中的索引值
            // 这个 Window 是首次 addView 操作, 显然索引为 -1 
            int index = findViewLocked(view, false);
            if (index >= 0) {
                // 1.1 若当前 View 已经被添加到解绑结合中, 则执行 ViewRootImpl 的 doDie 操作
                if (mDyingViews.contains(view)) {
                    mRoots.get(index).doDie();
                } else {
                    // 1.2 这是一个非常常见的异常, 表示这个 DecorView 被重新添加了
                    throw new IllegalStateException("View " + view
                            + " has already been added to the window manager.");
                }
            }
            // 2. 判断当前 Window 是否为 SUB_WINDOW 类型
            if (wparams.type >= WindowManager.LayoutParams.FIRST_SUB_WINDOW &&
                    wparams.type <= WindowManager.LayoutParams.LAST_SUB_WINDOW) {
                // 若是子 Window, 找寻它父 Window 的 DecorView
                final int count = mViews.size();
                for (int i = 0; i < count; i++) {
                    // 通过 wparams 的 token 进行比较
                    // 可以看到找到之后并没有立即 break, 它会找到集合中最后一个匹配的 View
                    // 因为最晚添加的 View 才有可能是它的直接父容器
                    if (mRoots.get(i).mWindow.asBinder() == wparams.token) {
                        panelParentView = mViews.get(i);
                    }
                }
            }
            // 3. 构建了 ViewRootImpl 对象
            root = new ViewRootImpl(view.getContext(), display);
            // 给 DecorView 设置 Window 的布局参数
            view.setLayoutParams(wparams);
            // 4. 添加到缓存中
            mViews.add(view);
            mRoots.add(root);
            mParams.add(wparams);
            try {
                // 5. 给 ViewRootImpl 设置这个 DecorView
                root.setView(view, wparams, panelParentView);
            } catch (RuntimeException e) {
                ......
            }
        }
    }
}
```
可以看到 WindowManagerGlobal 的 addView 中做了非常重要的操作
- 查找传入的 DecorView 是否已经在 mViews 缓存中了
  - 若在缓存中, 并且不再待清空的集合中, 则抛出我们熟知的异常
- 判断当前 DecorView 的 LayoutParams 是否 SUB_WINDOW 类型
  - 若为子窗口类型, 则遍历集合查找其父窗口的 DecorView
- **构建一个 ViewRootImpl** 
  - 给 DecorView 设置 Window 的布局参数 
- 将相关对象添加到 WindowManagerGlobal 的缓存集合中
- **调用 ViewRootImpl.setView 配置这个 DecorView**

在 WindowManagerGlobal.addView 中, 的确没有看到添加操作 Window 的代码, 它创建了一个 ViewRootImpl, 并且将这个 DecorView 和与之相关的对象, 添加到了内部维护的集合中

**看到这里就可以很好地回答前言中的疑问了, WindowManager.addView 操作是将 Window 中的 DecorView 添加到 WindowManagerGlobal 的缓存中**, 因此 Google 工程师的命名还是非常合理的

解决了 WindowManager.addView 命名的疑惑, 又产生了另一个疑惑, **这个 ViewRootImpl 究竟是什么呢? 我们的 DecorView 不就是最顶层容器吗, 为什么将这个类命名为视图根呢?**

带着疑惑, 我们分析一下 ViewRootImpl.setView 方法

## 二. ViewRootImpl.setView
```
public final class ViewRootImpl implements ViewParent,
        View.AttachInfo.Callbacks, ThreadedRenderer.DrawCallbacks {

    View mView;                           // 描述 ViewRootImpl 中的 DecorView
    final Rect mWinFrame;                 // 描述 ViewRootImpl 窗体的大小   
    final W mWindow;                      // 描述与需要添加到 WMS 中的窗口的 Binder 实体对象
    final IWindowSession mWindowSession;  // 描述一个与 WMS 建立的连接
    final View.AttachInfo mAttachInfo;    // 描述当前 View 依附的信息
    
    public ViewRootImpl(Context context, Display display) {
        ......
        mContext = context;
        mWindowSession = WindowManagerGlobal.getWindowSession();
        mWinFrame = new Rect();
        mWindow = new W(this);
        ......
        mAttachInfo = new View.AttachInfo(mWindowSession, mWindow, display, this, mHandler, this,
                context);
     }

    public void setView(View view, WindowManager.LayoutParams attrs, View panelParentView) {
        synchronized (this) {
            if (mView == null) {
                mView = view;
                ......
                if (panelParentView != null) {
                    // 若存在父窗体, 则保存父窗体的 token
                    mAttachInfo.mPanelParentWindowToken
                            = panelParentView.getApplicationWindowToken();
                }
                // 执行 View 绘制的三大流程
                requestLayout();
                ......
                try {
                    ......
                    // 将窗体信息添加到 WMS 中
                    res = mWindowSession.addToDisplay(mWindow, mSeq, mWindowAttributes,
                            getHostVisibility(), mDisplay.getDisplayId(), mWinFrame,
                            mAttachInfo.mContentInsets, mAttachInfo.mStableInsets,
                            mAttachInfo.mOutsets, mAttachInfo.mDisplayCutout, mInputChannel);
                } catch (RemoteException e) {
                    ......
                } finally {
                    ......
                }
            }
        }
    }
    
}
```
现在回过头来看看 ViewRootImpl 为什么命名为根视图, 就显得格外清晰了
- 在 ViewRootImpl.setView 的过程中它内部保存了 DecorView 这个对象
  - (注意是 setView, 不是 addView, 因此我认为它与 DecorView 是平级的, ViewRoot 描述的就是 DecorView)
- 绑定了 DecorView 之后, 还会调用 IWindowSession.addToDisplay 将窗体添加到 WMS 中

**因此为什么叫 ViewRootImpl 这个问题便迎刃而解了, ViewRootImpl 即负责根视图 DecorView 绘制渲染等操作的实现类**

渲染、绘制、事件监听等操作是非常复杂的, 它需要与系统服务进程 WMS 进行交互, 也需要与负责渲染的 SurfaceFlinger 进行交互, 因此它将这部分操作从 View 中抽离到 ViewRootImpl 中, **让我们应用层的开发者无需关心底层的细节**, 也是门面设计模式的一个很好的体现

### 应用进程窗体依赖关系
分析到这里, 简单的回顾一下应用进程中 Window 的相关依赖关系

![应用进程窗体依赖关系](https://i.loli.net/2019/10/23/yxLFKcpTAblj2Uu.png)

## 三. ViewRootImpl 与 WMS 的连接
在上面的代码中, 出现了两个陌生的对象
- WindowSession mWindowSession: **字面意思是窗体的会话, 如何理解呢?**
- W mWindow: 从命名上来看, 这是个窗体, **这个 W 与 之前看到的 Window 有什么关系呢?**

好的, 我们带着疑问先看看 IWindowSession 的构建流程

### 一) IWindowSession 的创建
在上面 ViewRootImpl 的构造方法中, 我们可以看到它调用了  WindowManagerGlobal.getWindowSession() 获取了一个窗体的会话
```
public final class WindowManagerGlobal {
    
    private static IWindowManager sWindowManagerService;
    private static IWindowSession sWindowSession;
    
    public static IWindowSession getWindowSession() {
        synchronized (WindowManagerGlobal.class) {
            if (sWindowSession == null) {
                try {
                    InputMethodManager imm = InputMethodManager.getInstance();
                    // 1. 获取 WMS 的客户端 Binder 代理对象
                    IWindowManager windowManager = getWindowManagerService();
                    // 2. 通过这个代理对象创建一个进程间单例的 IWindowSession Binder 代理对象
                    // 它描述了当前进程与系统服务进行的 WMS 进行会话的桥梁
                    sWindowSession = windowManager.openSession(
                            new IWindowSessionCallback.Stub() {
                                @Override
                                public void onAnimatorScaleChanged(float scale) {
                                    ValueAnimator.setDurationScale(scale);
                                }
                            },
                            imm.getClient(), imm.getInputContext());
                } catch (RemoteException e) {
                    throw e.rethrowFromSystemServer();
                }
            }
            return sWindowSession;
        }
    }
    
    public static IWindowManager getWindowManagerService() {
        synchronized (WindowManagerGlobal.class) {
            if (sWindowManagerService == null) {
                // 获取了系统服务进程的 WindowManagerService 的 Binder 代理对象, 保存在静态变量中
                sWindowManagerService = IWindowManager.Stub.asInterface(
                        ServiceManager.getService("window"));
                ......
            }
            return sWindowManagerService;
        }
    }
}
```
可以看到这个 IWindowSession 的构建是在 WindowManagerGlobal 中进行的, 它主要做了两部分操作
- 获取系统服务进程 WindowManagerService 的 Binder 代理对象, 我们称之为 WMS
- 调用了 WMS 代理对象的 openSession, 返回一个 Session 的 Binder 代理对象给我们客户端使用

可以看到这里出现了 WindowManagerService 这...是否与之前的 WindowManger 傻傻分不清了呢? 
- 其实很简单, WindowManager 是在我们的应用进程中对 Window 进行管理的服务, 并且是一一对应的关系, 可以理解为是更高层的抽象实现, 也就是门面; 
- 真正用于窗体管理的是在系统服务进程的 WMS, 它是不认是我们上面的 Window 对象的, 因为是跨进程通信, 它所能够识别的窗体是接下来我们要看到的 W 这个Binder 实体对象

接下来我们看看, WMS 是如何通过 openSession 方法建立和应用进程的通信的

#### 与 WMS 建立会话
```
public class WindowManagerService extends IWindowManager.Stub {
    
    public IWindowSession openSession(IWindowSessionCallback callback, IInputMethodClient client,
            IInputContext inputContext) {
        ......
        // 可以看到在系统服务进程, 创建了一个 Session 对象返回给客户端
        Session session = new Session(this, callback, client, inputContext);
        return session;
    }
    
}

class Session extends IWindowSession.Stub implements IBinder.DeathRecipient {
    
    final WindowManagerService mService;
    final IWindowSessionCallback mCallback;
    final IInputMethodClient mClient;
    
    public Session(WindowManagerService service, IWindowSessionCallback callback,
            IInputMethodClient client, IInputContext inputContext) {
        mService = service;
        mCallback = callback;
        mClient = client;
    }
    
}
```
可以看到在系统服务进程, WMS 创建了一个 Session 的 Binder 实体对象, 然后就返回给了客户端, 我们拿到这个 Session 对象之后就可以将我们的窗体发送给 WMS 进行统一管理了

### 二)  W 的创建
在 ViewRootImpl 的构造方法中我们可以看到, 成员变量 mWindow 是在其构造方法里 new 出来的, 这里我们看看它的构造器
```
public final class ViewRootImpl implements ViewParent,
        View.AttachInfo.Callbacks, ThreadedRenderer.DrawCallbacks {
            
    static class W extends IWindow.Stub {

        private final WeakReference<ViewRootImpl> mViewAncestor;
        private final IWindowSession mWindowSession;

        W(ViewRootImpl viewAncestor) {
            // 保存了 ViewRootImpl 的弱引用
            mViewAncestor = new WeakReference<ViewRootImpl>(viewAncestor);
            // 保存了当前进程与 WMS 通信的会话
            mWindowSession = viewAncestor.mWindowSession;
        }
    }            
   
}
```
可以看到它是一个 Binder 实体对象, 它的构造器中保存了 这个 ViewRootImpl 的弱引用和 Session 的代理对象

既然 **W 是 Binder 对象, 那么显然它用于和 WMS 进行交互, 描述当前 View 所在的窗体**

不知道大家有没有发现, 在 ViewRootImpl 中我们几乎看不到上面的 Window 和 WindowManager 了, 那是因为他们是上层高度抽象出来的壳, 将 DecorView 和相关参数导入 ViewRootImpl 之后, 它们的任务就完成了

接下来我们继续看看将这个 W 添加到 WMS 中的操作

### 三) 将窗体添加到 WMS 
从上面的分析可知, ViewRootImpl 中, 调用了 IWindowSession.addToDisplay 将窗体信息添加到了 WMS 中, 我们看看它是如何实现的
```
class Session extends IWindowSession.Stub implements IBinder.DeathRecipient {
    
    final WindowManagerService mService;

    @Override
    public int addToDisplay(IWindow window, int seq, WindowManager.LayoutParams attrs,
            int viewVisibility, int displayId, Rect outFrame, Rect outContentInsets,
            Rect outStableInsets, Rect outOutsets,
            DisplayCutout.ParcelableWrapper outDisplayCutout, InputChannel outInputChannel) {
        // 简单的进行了转发
        return mService.addWindow(this, window, seq, attrs, viewVisibility, displayId, outFrame,
                outContentInsets, outStableInsets, outOutsets, outDisplayCutout, outInputChannel);
    }
    
}

public class WindowManagerService extends IWindowManager.Stub {

    final WindowManagerPolicy mPolicy;// 在 WMS 的构造函数中赋值, 其实例为 PhoneWindowManager
    final WindowHashMap mWindowMap = new WindowHashMap();

    public int addWindow(Session session, IWindow client, int seq,
            LayoutParams attrs, int viewVisibility, int displayId, Rect outFrame,
            Rect outContentInsets, Rect outStableInsets, Rect outOutsets,
            DisplayCutout.ParcelableWrapper outDisplayCutout, InputChannel outInputChannel) {
            
        .......
        synchronized(mWindowMap) {
            // 1. 创建一个 DisplayContent, 它用来管理同一个 displayId 下的所有窗体
            final DisplayContent displayContent = getDisplayContentOrCreate(displayId);
            ......
            // 2. 创建一个窗体的描述
            final WindowState win = new WindowState(this, session, client, token, parentWindow,
                    appOp[0], seq, attrs, viewVisibility, session.mUid,
                    session.mCanAddInternalSystemWindow);
            ......
            // 3. 准备添加窗体
            res = mPolicy.prepareAddWindowLw(win, attrs);
            if (res != WindowManagerGlobal.ADD_OKAY) {
                return res;
            }
            // 4. 给这个窗体描述打开一个输入通道, 用于接收屏幕的点击事件(事件分发)
            final boolean openInputChannels = (outInputChannel != null
                    && (attrs.inputFeatures & INPUT_FEATURE_NO_INPUT_CHANNEL) == 0);
            if  (openInputChannels) {
                win.openInputChannel(outInputChannel);
            }
            if (res != WindowManagerGlobal.ADD_OKAY) {
                return res;
            }
            .......
            // 5. 会创建一个 SurfaceSession 与 SurfaceFlinger 交互
            win.attach();
            
            // 6. 以客户端的 W 的 Binder 代理对象为 key, 将 win 存入
            mWindowMap.put(client.asBinder(), win);
            .......
            
            // 7. 确认窗体中可用于布局区域的大小
            if (mPolicy.getLayoutHintLw(win.mAttrs, taskBounds, displayFrames, outFrame,
                    outContentInsets, outStableInsets, outOutsets, outDisplayCutout)) {
                res |= WindowManagerGlobal.ADD_FLAG_ALWAYS_CONSUME_NAV_BAR;
            }
        }
        ......
        return res;
    }
}
```
好的, 上面的省略的大量的代码, 保留了此次我们关注的点主要有以下几点
- 通过 displayId 创建/获取一个 DisplayContent 对象, 它用来管理同一个 displayId 下的所有窗体
- 创建一个窗体的描述对象 **WindowState**, 与应用进程的 IWindow 对应
- 调用 PhoneWindowManager.prepareAddWindowLw() 准备添加窗体
- 调用 WindowState.openInputChannel 给窗体打开一个输入通道, 用于接收屏幕的点击事件流
- **调用 WindowState.attach 方法, 为其内部创建一个 SurfaceSession**
  - 这个 SurfaceSession 非常重要, 它是 Surface 画布与 SurfaceFlinger 进行交互的桥梁 
- 将我们客户端的 W 为 key, WindowState 为 value , **将窗体添加到 mWindowMap 中缓存**
- 调用 PhoneWindowManager.getLayoutHintLw 确认窗体中可用于布局区域的大小
- 调用 WMS.sendNewConfiguration 通知应用进程窗体数据变更了

当构建的 WindowState 添加到了 WMS.mWindowMap 中缓存起来之后, 至此应用进程的窗体 IWindow 就与 WMS 联系在一起了, 他们交互的流程图如下所示

![IWindow 与 WMS 的联系](https://i.loli.net/2019/10/23/YhfxRkdtgsna3CL.png)

好的, 上面的方法中, 有一个非常重要的方法, 叫做 WindowState.attach, 它创建了这个窗体与 SurfaceFliger 交互的桥梁, 这里我们单独分析一下

## 四. WindowState.attach
```
class WindowState extends WindowContainer<WindowState> implements WindowManagerPolicy.WindowState {
    
    final Session mSession;
    
    void attach() {
        // 调用了 Session.windowAddedLocked 方法
        mSession.windowAddedLocked(mAttrs.packageName);
    }
    
}

class Session extends IWindowSession.Stub implements IBinder.DeathRecipient {
    
    SurfaceSession mSurfaceSession;
    private int mNumWindow = 0;

    void windowAddedLocked(String packageName) {
        mPackageName = packageName;
        if (mSurfaceSession == null) {
            // 这里创建了一个 SurfaceSession 对象
            mSurfaceSession = new SurfaceSession();
            // 添加到 WMS 的缓存中
            mService.mSessions.add(this);
            ......
        }
        // 表示这个 Session 对象所管理的进程中, 由多了一个 Window
        mNumWindow++;
    }
    
}
```
好的, 可以看到 WindowState.attach 方法, 会在这个窗体应用进程对应的 Session 中, 创建一个 mSurfaceSession 对象, 然后把它添加到 WMS 的缓存中, 这个 SurfaceSession 承担着与 Session 一样的作用
- Session 是 IWindowSession 的 Binder 实体对象, 它用与管理一个应用进程与 WMS 进程的交互
- SurfaceSession 也是如此, 它用来管理一个应用进程与 SurfaceFlinger 的交互
  - SurfaceFlinger 是专门用来渲染 UI 的进程 

接下来我们就看看这个 SurfaceSession 对象是如何构建的

### SurfaceSession 的构建
```
public final class SurfaceSession {
    // 描述 SurfaceComposerClient 的句柄
    private long mNativeClient;

    public SurfaceSession() {
        // 调用了 nativeCreate 获取了一个句柄值
        mNativeClient = nativeCreate();
    }
    
    private static native long nativeCreate();
}
```
好的, 可以看到 SurfaceSession 的构造方式非常简单, 它调用 nativeCreate 获取了一个句柄值, 接下来我们看看它在 Native 层的操作
```
// frameworks/base/core/jni/android_view_SurfaceSession.cpp
static jlong nativeCreate(JNIEnv* env, jclass clazz) {
    // 可以看到创建了一个 SurfaceComposerClient 的 C++ 对象
    SurfaceComposerClient* client = new SurfaceComposerClient();
    client->incStrong((void*)nativeCreate);// 增加一个强引用
    // 返回给 Java 层一个句柄值
    return reinterpret_cast<jlong>(client);
}
```
很简单, 创建了一个 SurfaceComposerClient C++ 对象, **它便承担着与 SurfaceFlinger 交互的重任**

## 总结
通过本次的分析, 我们了解了 ViewRootImpl 与根视图的关系、 ViewRootImpl 以及它是如何与 WMS 交互的, 以及各个对象的映射关系
- ViewRootImpl 通过 IWindowSession 与进行跨进程 WMS 交互
  - IWindowSession 的 Binder 实体对象为 Session 
- ViewRootImpl 中使用 ViewRootImpl.W 描述一个窗体
- WMS 中使用 WindowState 描述一个窗体
- Session 中会创建一个 SurfaceSession 用于和 SurfaceFlinger 交互
  -  SurfaceSession 对应的 Native 对象为 SurfaceComposerClient
