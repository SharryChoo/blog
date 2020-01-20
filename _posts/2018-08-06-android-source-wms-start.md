---
title: Android 系统架构 —— WMS 的启动
permalink: android-source/wms-start
key: android-source-wms-start
tags: AndroidFramework
sidebar:
  nav: android-source
---
## 前言
之前的文章介绍了 IMS 的启动过程, 也了解了 IMS 的事件分发与窗体是息息相关的, 这里我们就来学习一下 WMS 的启动流程

<!--more-->

```java
public final class SystemServer {

    private void startOtherServices() {
        ......
        WindowManagerService wm = null;
        ......
        InputManagerService inputManager = null;
        try {
            ......
            // 1. 创建 WMS
            wm = WindowManagerService.main(context, inputManager,
                    mFactoryTestMode != FactoryTest.FACTORY_TEST_LOW_LEVEL,
                    !mFirstBoot, mOnlyCore, new PhoneWindowManager());
            // 发布到 ServiceManager 中
            ServiceManager.addService(Context.WINDOW_SERVICE, wm, /* allowIsolated= */ false,
                    DUMP_FLAG_PRIORITY_CRITICAL | DUMP_FLAG_PROTO);
            ......
            // 2. 准备初始化
            wm.onInitReady();
            
            // 3. 调用 displayReady 准备显示
            wm.displayReady();
            ......
            // 4. 调用 systemReady
            wm.systemReady();
        }
        ......
    }
    
}
```
WindowManagerService 的启动流程如同上述所示, 我们这里主要从以下几个方面来分析
- WMS 的创建
- onInitReady
- displayReady
- systemReady

## 一. WMS 的创建
```java
public class WindowManagerService extends IWindowManager.Stub
        implements Watchdog.Monitor, WindowManagerPolicy.WindowManagerFuncs {
    
    private static WindowManagerService sInstance;
    
    public static WindowManagerService main(final Context context, final InputManagerService im,
            final boolean haveInputMethods, final boolean showBootMsgs, final boolean onlyCore,
            WindowManagerPolicy policy) {
        // 在 DisplayThread 中创建 WMS
        DisplayThread.getHandler().runWithScissors(() ->
                // 调用了 WMS 的构造函数, 为静态成员变量 sInstance 赋值
                sInstance = new WindowManagerService(context, im, haveInputMethods, showBootMsgs,
                        onlyCore, policy), 0);
        return sInstance;
    }
            
}
```
这里的构建过程非常有意思, 它调用了 DisplayThread 线程的 getHandler 的 runWithScissors 方法, 我们知道 DisplayThread 在 IMS 构建时便会创建

这个 Handler 绑定的是 DisplayThread 的 Looper, 我们看看 Handler.runWithScissors 操作

### Handler.runWithScissors
```java
public class Handler {

    public final boolean runWithScissors(final Runnable r, long timeout) {
        ......
        // 若当前线程的 Looper 与 Handler 的绑定的 Looper 相同, 则直接执行
        if (Looper.myLooper() == mLooper) {
            r.run();
            return true;
        }
        // 包装成 BlockingRunnable
        BlockingRunnable br = new BlockingRunnable(r);
        // 调用 postAndWait
        return br.postAndWait(this, timeout);
    }
    
    private static final class BlockingRunnable implements Runnable {
    
        private final Runnable mTask;
        private boolean mDone;

        public BlockingRunnable(Runnable task) {
            mTask = task;
        }

        @Override
        public void run() {
            try {
                mTask.run();
            } finally {
                // 3. 唤醒 postAndWait 中的阻塞
                synchronized (this) {
                    mDone = true;
                    notifyAll();
                }
            }
        }
        
        public boolean postAndWait(Handler handler, long timeout) {
            // 1. 投递到 Handler 绑定 Looper 的消息队列中执行
            if (!handler.post(this)) {
                return false;
            }
            synchronized (this) {
                // 由传参可只 timeout 为 0
                if (timeout > 0) {
                   ......
                } else {
                    // 2. 使用并发管程模型, 等待 mTask 完成
                    while (!mDone) {
                        try {
                            wait();
                        } catch (InterruptedException ex) {
                        }
                    }
                }
            }
            return true;
        }
    }
    
}
```
好的, Handler 的 runWithScissors 做了如下的操作
- 当前处于 Handler 的 Looper 绑定的线程, 则直接回调 run 方法, 如同同步代码
- 当前非 Handler 的 Looper 绑定的线程, 则通过 postAndWait 投递到 DisplayThread 中进行初始化
  - 使用并发管程模型的阻塞唤醒机制, 保证代码的串行

**当 runWithScissors 执行完成之后, sInstance 就已经被 WMS 赋值了, 使用 Handler 大费周折的操作的意义是什么呢?** 
- 这保证了 WMS 在 DisplayThread 线程中创建

下面我们看看 WMS 的构造流程

### WMS 的构造
```
public class WindowManagerService extends IWindowManager.Stub
        implements Watchdog.Monitor, WindowManagerPolicy.WindowManagerFuncs {

    // 窗体动画
    final WindowAnimator mAnimator;
    // 所有窗体的根容器    
    RootWindowContainer mRoot;
    // 窗体管理策略
    final WindowManagerPolicy mPolicy;
    
    private WindowManagerService(Context context, InputManagerService inputManager,
            boolean haveInputMethods, boolean showBootMsgs, boolean onlyCore,
            WindowManagerPolicy policy) {
        ......
        mContext = context;
        ......
        // 保存 IMS
        mInputManager = inputManager;
        ....
        // 保存 PhoneWindowManager
        mPolicy = policy;
        // 窗体动画控制器
        mAnimator = new WindowAnimator(this);
        // 初始化根容器
        mRoot = new RootWindowContainer(this);
        ......
        // DMS
        mDisplayManager = (DisplayManager)context.getSystemService(Context.DISPLAY_SERVICE);
        // PMS
        mPowerManager = (PowerManager)context.getSystemService(Context.POWER_SERVICE);
        ......
        // AMS
        mActivityManager = ActivityManager.getService();
        ......
    }   
}
```
在 WMS 的构建中, 创建了一些核心对象
- mInputManager: IMS
- mPolicy: 窗体管理策略, 实现类为 PhoneWindowManager
- WindowAnimator mAnimator: 窗体动画
- RootWindowContainer mRoot: Window 的根容器
- mDisplayManager: 显示管理者
- mActivityManager: AMS

WMS 的构造中关联的数据对象非常多, 这里笔者只列出常见的一些, 关于他们的作用, 到后面应用进程与 WMS 交互时再具体分析

下面我们看看 onInitReady 中做了什么操作

## 二. WMS.onInitReady
```
public class WindowManagerService extends IWindowManager.Stub
        implements Watchdog.Monitor, WindowManagerPolicy.WindowManagerFuncs {
   
    public void onInitReady() {
        // 初始化管理策略
        initPolicy();
        ......
    }
    
    private void initPolicy() {
        UiThread.getHandler().runWithScissors(new Runnable() {
            @Override
            public void run() {
                WindowManagerPolicyThread.set(Thread.currentThread(), Looper.myLooper());
                // 调用 PhoneWindowManager 的初始化方法
                mPolicy.init(mContext, WindowManagerService.this, WindowManagerService.this);
            }
        }, 0);
    }
}
```
onInitReady 方法中执行了初始化管理策略的操作, **mPolicy 的初始化操作在 UiThread 中进行**, 下面看看 PhoneWindowManager 的 init 中做了什么处理

```
public class PhoneWindowManager implements WindowManagerPolicy {
    
    @Override
    public void init(Context context, IWindowManager windowManager,
            WindowManagerFuncs windowManagerFuncs) {
        mContext = context;
        mWindowManager = windowManager;
        mWindowManagerFuncs = windowManagerFuncs;
        // 这个 Handler 运行在 UiThread 上
        mHandler = new PolicyHandler();
        ......
        mWakeGestureListener = new MyWakeGestureListener(mContext, mHandler);
        mOrientationListener = new MyOrientationListener(mContext, mHandler);
        mPowerManager = (PowerManager)context.getSystemService(Context.POWER_SERVICE);
        mBroadcastWakeLock = mPowerManager.newWakeLock(PowerManager.PARTIAL_WAKE_LOCK,
                "PhoneWindowManager.mBroadcastWakeLock");
        mPowerKeyWakeLock = mPowerManager.newWakeLock(PowerManager.PARTIAL_WAKE_LOCK,
                "PhoneWindowManager.mPowerKeyWakeLock");
        ......
        mAccessibilityManager = (AccessibilityManager) context.getSystemService(
                Context.ACCESSIBILITY_SERVICE);
        mVibrator = (Vibrator)context.getSystemService(Context.VIBRATOR_SERVICE);
        ......
        mWindowManagerInternal.registerAppTransitionListener(......);
    }
}
```
PhoneWindowManager.init 中需要注意的是, 它创建一个 PolicyHandler, 因为 init 在 UiThread 的下执行, 所以这个 PolicyHandler 绑定的也是 UIThread 的 Looper

## 三. WMS.displayReady
```
public class WindowManagerService extends IWindowManager.Stub
        implements Watchdog.Monitor, WindowManagerPolicy.WindowManagerFuncs {
        
    public void displayReady() {
        final int displayCount = mRoot.mChildren.size();
        // 1. 初始化 RootWindowContainer 的子窗体
        for (int i = 0; i < displayCount; ++i) {
            // 获取 DisplayContent 这也是一个 WindowContainer 的实现类, 描述一个窗体
            final DisplayContent display = mRoot.mChildren.get(i);
            // 1.1 初始化子窗体
            displayReady(display.getDisplayId());
        }
        synchronized(mWindowMap) {
            //2. 获取默认的窗体
            final DisplayContent displayContent = getDefaultDisplayContentLocked();
            // 2.1 设置 UI 的最大宽度
            if (mMaxUiWidth > 0) {
                displayContent.setMaxUiWidth(mMaxUiWidth);
            }
            // 2.2 表示窗体已经初始化完毕
            mDisplayReady = true;
        }
        ......
    }

    private void displayReady(int displayId) {
        synchronized(mWindowMap) {
            final DisplayContent displayContent = mRoot.getDisplayContent(displayId);
            if (displayContent != null) {
                ......
                // 1.2 初始化窗体的信息
                displayContent.initializeDisplayBaseInfo();
                ......
            }
        }
    }
    
}
```
displayReady 中主要任务如下
- 遍历并且初始化 RootWindowContainer 中的子窗体
  - 调用子窗体的 initializeDisplayBaseInfo 执行初始化操作
- 初始化默认窗体
  - 设置最大的 UI 宽度
  - 更新 flag 表示初始化完毕

## 四. WMS.systemReady
```
public class WindowManagerService extends IWindowManager.Stub
        implements Watchdog.Monitor, WindowManagerPolicy.WindowManagerFuncs {
        
    public void systemReady() {
        mPolicy.systemReady();
        ......
    }
    
}
```
WindowManagerService 的 systemReady 操作转发到了 PhoneWindowManager 中处理
```
public class PhoneWindowManager implements WindowManagerPolicy {

    @Override
    public void systemReady() {
        // In normal flow, systemReady is called before other system services are ready.
        // So it is better not to bind keyguard here.
        mKeyguardDelegate.onSystemReady();

        mVrManagerInternal = LocalServices.getService(VrManagerInternal.class);
        if (mVrManagerInternal != null) {
            mVrManagerInternal.addPersistentVrModeStateListener(mPersistentVrModeListener);
        }

        readCameraLensCoverState();
        updateUiMode();
        synchronized (mLock) {
            updateOrientationListenerLp();
            mSystemReady = true;
            mHandler.post(new Runnable() {
                @Override
                public void run() {
                    updateSettings();
                }
            });
            // If this happens, for whatever reason, systemReady came later than systemBooted.
            // And keyguard should be already bound from systemBooted
            if (mSystemBooted) {
                mKeyguardDelegate.onBootCompleted();
            }
        }

        mSystemGestures.systemReady();
        mImmersiveModeConfirmation.systemReady();

        mAutofillManagerInternal = LocalServices.getService(AutofillManagerInternal.class);
    }
    
}
```
到这里 WMS 的 systemReady 就执行完毕了

## 总结
WMS 的启动流程如下
- WMS 的创建
  - 在 DisplayThread 执行创建操作 
- WMS.onInitReady
  - 执行了 PhoneWindowManager 的初始化 
- WMS.displayReady
  - 初始化 RootWindowContainer 子窗体
- WMS.systemReady
  - 执行了 PhoneWindowManager 的初始化

比起前面的 AMS 和 IMS, WMS 的启动流程看起来感觉就有些过于简单了, 不过启动流程简单不代表背后的实现一样简单

关于 WMS 的更多的操作, 我们到应用窗体架构中再进行分析