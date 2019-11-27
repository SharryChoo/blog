---
title: Android 系统架构 ——  Activity 的启动 之 目标的启动
permalink: android-source/activity-launch3
key: android-source-activity-launch3
tags: AndroidFramework
---

## 前言
通过 Step 2 我们知道, Activity 的启动第二阶段会
- 先创建要启动 Activity 的进程
- 进程创建成功后会调用 AMS.attachApplication 通知 AMS 这个进程初始化完成了

这篇文章我们从 AMS 的 attachApplication 方法看起, 走完 Activity 创建的整个流程

<!--more-->

## 一. AMS 绑定应用进程
```
public class ActivityManagerService {

    public final void attachApplication(IApplicationThread thread, long startSeq) {
        synchronized (this) {
            // 获取请求调用进程的 PID 
            int callingPid = Binder.getCallingPid();
            // 获取请求调用进程的 UID 
            final int callingUid = Binder.getCallingUid();
            // 调用了 attachApplicationLocked
            attachApplicationLocked(thread, callingPid, callingUid, startSeq);
        }
    }
    
    private final boolean attachApplicationLocked(IApplicationThread thread,
            int pid, int callingUid, long startSeq) {
        // 获取进程描述
        ProcessRecord app;
        if (pid != MY_PID && pid >= 0) {
            synchronized (mPidsSelfLocked) {
                app = mPidsSelfLocked.get(pid);
            }
        } else {
            ......
        }
        final String processName = app.processName;
        try {
            // 注册死亡监听器
            AppDeathRecipient adr = new AppDeathRecipient(
                    app, pid, thread);
            thread.asBinder().linkToDeath(adr, 0);
            app.deathRecipient = adr;
        } catch (RemoteException e) {
            ......
        }
        // 1. 移除了消息队列中的 PROC_START_TIMEOUT_MSG
        mHandler.removeMessages(PROC_START_TIMEOUT_MSG, app);
        ......
        if (normalMode) {
            try {
                // 调用 ActivityStackSupervisor 的 attachApplicationLocked 方法
                if (mStackSupervisor.attachApplicationLocked(app)) {
                    didSomething = true;
                }
            } catch (Exception e) {
                Slog.wtf(TAG, "Exception thrown launching activities in " + app, e);
                badApp = true;
            }
        }
        ......
        return true;
    }
}

public class  ActivityStackSupervisor {
    
    boolean attachApplicationLocked(ProcessRecord app) throws RemoteException {
        final String processName = app.processName;
        boolean didSomething = false;
        for (int displayNdx = mActivityDisplays.size() - 1; displayNdx >= 0; --displayNdx) {
            final ActivityDisplay display = mActivityDisplays.valueAt(displayNdx);
            for (int stackNdx = display.getChildCount() - 1; stackNdx >= 0; --stackNdx) {
                final ActivityStack stack = display.getChildAt(stackNdx);    // 获取任务栈
                final ActivityRecord top = stack.topRunningActivityLocked(); // 获取栈顶 ACT
                final int size = mTmpActivityList.size();
                for (int i = 0; i < size; i++) {
                    final ActivityRecord activity = mTmpActivityList.get(i);
                    if (activity.app == null && app.uid == activity.info.applicationInfo.uid
                            && processName.equals(activity.processName)) {
                        try {
                            // 2 这里重新调用了 realStartActivityLocked 执行进进程栈顶 Activity 的启动
                            if (realStartActivityLocked(activity, app,
                                    top == activity /* andResume */, true /* checkConfig */)) {
                                didSomething = true;
                            }
                        } catch (RemoteException e) {
                           ......
                        }
                    }
                }
            }
        }
        return didSomething;
    }

}
```
可以看到 AMS 在处理新的进程的启动后调用的 attachApplication 的过程中, 主要做了以下几点事情
- 移除了消息队列中之前发出的 PROC_START_TIMEOUT_MSG
- **重新调用 ActivityStackSupervisor.realStartActivityLocked 真正的执行 TargetActivity 的启动**

下面我们看看 ActivityStackSupervisor.realStartActivityLocked 是如何启动目标 Activity 的

## 二. 启动目标 Activity
```
    final boolean realStartActivityLocked(ActivityRecord r, ProcessRecord app,
            boolean andResume, boolean checkConfig) throws RemoteException {
        try {
            ......
            try {
                ......
                // 1. 获取一个让客户端处理事务的对象 ClientTransaction
                final ClientTransaction clientTransaction = ClientTransaction.obtain(app.thread,
                        r.appToken);
                // 2. 将一个启动 Activity 的事务请求添加到的 Callback 中去
                clientTransaction.addCallback(LaunchActivityItem.obtain(new Intent(r.intent),
                        System.identityHashCode(r), r.info,
                        // TODO: Have this take the merged configuration instead of separate global
                        // and override configs.
                        mergedConfiguration.getGlobalConfiguration(),
                        mergedConfiguration.getOverrideConfiguration(), r.compat,
                        r.launchedFromPackage, task.voiceInteractor, app.repProcState, r.icicle,
                        r.persistentState, results, newIntents, mService.isNextTransitionForward(),
                        profilerInfo));
                // 判断启动之后需要执行的下一个生命周期
                final ActivityLifecycleItem lifecycleItem;
                if (andResume) { // 由传值可知, andResume 为 true
                    // 3. 这里构建了一个描述 Resume 生命周期的事务条目
                    lifecycleItem = ResumeActivityItem.obtain(mService.isNextTransitionForward());
                } else {
                    ......
                }
                // 添加上面创建的声明周期请求
                clientTransaction.setLifecycleStateRequest(lifecycleItem);
                // 4. 执行构建的事务
                mService.getLifecycleManager().scheduleTransaction(clientTransaction);
                ......
                }

            } catch (RemoteException e) {
                .......
            }
        } finally {
           ......
        }
        ......
        return true;
    }
```
可以看到 realStartActivityLocked 这个方法中, 主要做了如下的操作
- 构建一个客户端执行事务的描述 ClientTransaction
- 给这个事务添加一个 LaunchActivityItem 的 Callback
- 给这个事务设置一个 ResumeActivityItem 的 Request

我们之前分析过 Pause 的过程, 可知当客户端收到这个事务时, 会通过 TransactionExecutor.execute 来处理这个事务, 如下所示
```
public class TransactionExecutor {
    public void execute(ClientTransaction transaction) {
        // 1. 先执行 ClientTransaction 的 callback.
        executeCallbacks(transaction);
        // 2. 再执行生命周期的请求, 我们只关注上面一个
        executeLifecycleState(transaction);
        ......
    }
}
```
所以我们的 LaunchActivityItem 会优先执行, 执行结束之后, 会执行 ResumeActivityItem, 所以接下来我们先看看 LaunchActivityItem 的 execute

## 三. 应用进程处理 Activity 的启动
```
public class LaunchActivityItem extends ClientTransactionItem {
    @Override
    public void execute(ClientTransactionHandler client, IBinder token,
            PendingTransactionActions pendingActions) {
        // 1. 封装了一个 ActivityClientRecord 的对象, 用于在客户端描述一个 Activity 对象
        ActivityClientRecord r = new ActivityClientRecord(token, mIntent, mIdent, mInfo,
                mOverrideConfig, mCompatInfo, mReferrer, mVoiceInteractor, mState, mPersistentState,
                mPendingResults, mPendingNewIntents, mIsForward,
                mProfilerInfo, client);
        // 2. 通过 ClientTransactionHandler 对象执行我们 TargetActivity 的启动
        client.handleLaunchActivity(r, pendingActions, null /* customIntent */);
    }
}
```
我们知道 ActivityThread 即 ClientTransactionHandler 的实现子类, 我们看看它的 handleLaunchActivity 方法做了些什么
```
public final class ActivityThread extends ClientTransactionHandler {

    public Activity handleLaunchActivity(ActivityClientRecord r,
            PendingTransactionActions pendingActions, Intent customIntent) {
        ......
        // 执行 activity 的启动
        final Activity a = performLaunchActivity(r, customIntent);
        ......
        return a;
    }
    
    private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
        ActivityInfo aInfo = r.activityInfo;
        if (r.packageInfo == null) {
            // 1. packageInfo 是 LoadedApk 对象, 若不存在则获取一个 LoadedApk 对象, 描述当前应用的安装包资源信息
            r.packageInfo = getPackageInfo(aInfo.applicationInfo, r.compatInfo,
                    Context.CONTEXT_INCLUDE_CODE);
        }
        ......
        // 2. 为 Activity 创建一个上下文对象
        ContextImpl appContext = createBaseContextForActivity(r);
        Activity activity = null;
        try {
            // 3. 构建 Activity 的对象
            java.lang.ClassLoader cl = appContext.getClassLoader();
            activity = mInstrumentation.newActivity(
                    cl, component.getClassName(), r.intent);
            ......
        } catch (Exception e) {
            ......
        }
        try {
            // 4. 调用 LoadedApk.makeApplication 获取当前进程的 Application
            Application app = r.packageInfo.makeApplication(false, mInstrumentation);

            if (activity != null) {
                ......
                // 5. 让 Activity 的上下文绑定 Activity 对象
                appContext.setOuterContext(activity);
                // 6. 给 Activity 注入参数
                activity.attach(appContext, this, getInstrumentation(), r.token,
                        r.ident, app, r.intent, r.activityInfo, title, r.parent,
                        r.embeddedID, r.lastNonConfigurationInstances, config,
                        r.referrer, r.voiceInteractor, window, r.configCallback);
                // 7. 回调 onCreate 方法
                if (r.isPersistable()) {
                    mInstrumentation.callActivityOnCreate(activity, r.state, r.persistentState);
                } else {
                    mInstrumentation.callActivityOnCreate(activity, r.state);
                }
            }
            // 8. 添加到 ActivityClientRecord 缓存池中
            mActivities.put(r.token, r);
        } catch (SuperNotCalledException e) {
            ......
        } catch (Exception e) {
            ......
        }
        return activity;
    }
}
```
performLaunchActivity 是真正创建并且回调 Activity 的 onCreate 生命周期的地方 
- **获取 LoadedApk 资源信息**
- **为 Activity 创建一个上下文对象**
- 反射构建 Activity 的实例
- **获取一个 Application 对象**
  - 当前进程不存在则创建一个新的 Application 对象
- 将 appContext 与 Activity 绑定
- **调用 Activity.attach 给 Activity 注入数据**
- 回调 Activity 的 onCreate 方法

这里我们选取几个重点的逻辑进行分析

### 一) 获取 LoadedApk 
```
public final class ActivityThread extends ClientTransactionHandler {
    
    public final LoadedApk getPackageInfo(ApplicationInfo ai, CompatibilityInfo compatInfo,
            int flags) {
        // 由上面的传参可知, includeCode 为 true
        boolean includeCode = (flags&Context.CONTEXT_INCLUDE_CODE) != 0;
        ......
        return getPackageInfo(ai, compatInfo, null, securityViolation, includeCode,
                registerPackage);
    }
    
    @GuardedBy("mResourcesManager")
    final ArrayMap<String, WeakReference<LoadedApk>> mPackages = new ArrayMap<>();
    
    private LoadedApk getPackageInfo(ApplicationInfo aInfo, CompatibilityInfo compatInfo,
            ClassLoader baseLoader, boolean securityViolation, boolean includeCode,
            boolean registerPackage) {
            
        final boolean differentUser = (UserHandle.myUserId() != UserHandle.getUserId(aInfo.uid));
        
        // 这个 mResourcesManager 在 ActivityThread 构造函数中创建
        synchronized (mResourcesManager) {
            WeakReference<LoadedApk> ref;
            if (differentUser) {
                // Caching not supported across users
                ref = null;
            } 
            // 由上面的传参可知, 这里会走到这个分支
            else if (includeCode) {
                // 1. 尝试从缓存中获取包名对应的 apk 资源
                ref = mPackages.get(aInfo.packageName);
            } else {
                ref = mResourcePackages.get(aInfo.packageName);
            }

            LoadedApk packageInfo = ref != null ? ref.get() : null;
            if (packageInfo == null || (packageInfo.mResources != null
                    && !packageInfo.mResources.getAssets().isUpToDate())) {
                    
                ......
                // 若为 nul, 则创建一个新的 LoadedApk 返回
                packageInfo =
                    new LoadedApk(this, aInfo, compatInfo, baseLoader,
                            securityViolation, includeCode &&
                            (aInfo.flags&ApplicationInfo.FLAG_HAS_CODE) != 0, registerPackage);
                ......
                
                if (differentUser) {
                    // Caching not supported across users
                } else if (includeCode) {
                    // 添加到 mPackages 中缓存
                    mPackages.put(aInfo.packageName,
                            new WeakReference<LoadedApk>(packageInfo));
                } else {
                    mResourcePackages.put(aInfo.packageName,
                            new WeakReference<LoadedApk>(packageInfo));
                }
            }
            return packageInfo;
        }
    }
    
}
```
从 ActivityThread 的 getPackageInfo 中可知, **LoadedApk 在 ActivityThread 的 mPackages 中缓存, 优先根据应用包名获取, 若不存在, 则创建一个新的对象, 并且添加到缓存**

接下来看看 Activity 上下文的创建过程

### 二) 构建 Activity 上下文
```
public final class ActivityThread extends ClientTransactionHandler {
    
    private ContextImpl createBaseContextForActivity(ActivityClientRecord r) {
        // 获取显示屏幕的 ID 
        final int displayId;
        try {
            displayId = ActivityManager.getService().getActivityDisplayId(r.token);
        } catch (RemoteException e) {
            ......
        }
        // 创建 Activity 的 Context
        ContextImpl appContext = ContextImpl.createActivityContext(
                this, r.packageInfo, r.activityInfo, r.token, displayId, r.overrideConfig);
        ......// 处理多显示屏
        return appContext;
    }
    
}
```
这里调用了 ContextImpl.createActivityContext 创建一个 ContextImpl 对象, 我们继续追踪
```
class ContextImpl extends Context {
    
    static ContextImpl createActivityContext(ActivityThread mainThread,
                                             LoadedApk packageInfo, ActivityInfo activityInfo, IBinder activityToken, int displayId,
                                             Configuration overrideConfiguration) {
        String[] splitDirs = packageInfo.getSplitResDirs();
        
        // 使用 LoadedApk 中的 ClassLoader
        ClassLoader classLoader = packageInfo.getClassLoader();

        if (packageInfo.getApplicationInfo().requestsIsolatedSplitLoading()) {
            try {
                // 获取分割资源包的 ClassLoader
                classLoader = packageInfo.getSplitClassLoader(activityInfo.splitName);
                splitDirs = packageInfo.getSplitPaths(activityInfo.splitName);
            } catch (NameNotFoundException e) {
                ......
            } finally {
                ......
            }
        }
        // 1. 创建 Context 对象
        ContextImpl context = new ContextImpl(null, mainThread, packageInfo, activityInfo.splitName,
                activityToken, null, 0, classLoader);

        ......
        final ResourcesManager resourcesManager = ResourcesManager.getInstance();

        // 2. 创建 Resource 并且注入到 Activity 的 Context 中
        context.setResources(resourcesManager.createBaseActivityResources(activityToken,
                packageInfo.getResDir(),
                splitDirs,
                packageInfo.getOverlayDirs(),
                packageInfo.getApplicationInfo().sharedLibraryFiles,
                displayId,
                overrideConfiguration,
                compatInfo,
                classLoader));
        // 注入 Display 对象
        context.mDisplay = resourcesManager.getAdjustedDisplay(displayId,
                context.getResources());
        return context;
    }
    
}
```
创建 Activity 的 Context 的主要流程如下
- 创建 ContextImpl 对象
- 创建 Resource 并且注入到 Activity 的 ContextImpl 中

关于创建 Resource 的过程, 涉及到 Android 资源管理的技术栈, 我们这里就不展开讨论了

下面看看获取 LoadedApk.makeApplication 的过程

### 三) 获取 Application
```
public final class LoadedApk {

    public Application makeApplication(boolean forceDefaultAppClass,
            Instrumentation instrumentation) {
        // 处理已创建的情况
        if (mApplication != null) {
            return mApplication;
        }
        ......
        // 处理 Application 还未创建的情况
        Application app = null;
        String appClass = mApplicationInfo.className;
        if (forceDefaultAppClass || (appClass == null)) {
            appClass = "android.app.Application";
        }

        try {
            java.lang.ClassLoader cl = getClassLoader();
            ......
            // 1. 创建 Application 的 Context
            ContextImpl appContext = ContextImpl.createAppContext(mActivityThread, this);
            // 2. 创建 Application 对象
            app = mActivityThread.mInstrumentation.newApplication(
                    cl, appClass, appContext);
            appContext.setOuterContext(app);
        } catch (Exception e) {
            ......
        }
        // 保存到成员变量
        mActivityThread.mAllApplications.add(app);
        mApplication = app;
        
        if (instrumentation != null) {
            try {
                // 3. 回调 Application 的 onCreate
                instrumentation.callApplicationOnCreate(app);
            } catch (Exception e) {
                ......
            }
        }
        return app;
    }
        
}
```
创建 Application 的流程如下
- 创建 Application 的 Context 对象
- 反射创建 Application 的实例
- 回调 Application 的 onCreate 方法

主要的流程与 Activity 类似, 这里就不再赘述了

### 四) 为 Activity 注入数据
```
public class Activity extends ContextThemeWrapper {
    
    final void attach(Context context, ActivityThread aThread,
            Instrumentation instr, IBinder token, int ident,
            Application application, Intent intent, ActivityInfo info,
            CharSequence title, Activity parent, String id,
            NonConfigurationInstances lastNonConfigurationInstances,
            Configuration config, String referrer, IVoiceInteractor voiceInteractor,
            Window window, ActivityConfigCallback activityConfigCallback) {
        // 绑定 Context 实现类
        attachBaseContext(context);
        // 绑定依赖的 Activity
        mFragments.attachHost(null /*parent*/);
        // 创建 Activity 的 Window
        mWindow = new PhoneWindow(this, window, activityConfigCallback);
        mWindow.setWindowControllerCallback(this);
        mWindow.setCallback(this);
        mWindow.setOnWindowDismissedCallback(this);
        mWindow.getLayoutInflater().setPrivateFactory(this);
        if (info.softInputMode != WindowManager.LayoutParams.SOFT_INPUT_STATE_UNSPECIFIED) {
            mWindow.setSoftInputMode(info.softInputMode);
        }
        if (info.uiOptions != 0) {
            mWindow.setUiOptions(info.uiOptions);
        }
        // Activity 创建的 Thread 即为 UI Thread
        mUiThread = Thread.currentThread();
        mMainThread = aThread;
        ......
        // 为 Window 注入一个 WindowManger
        mWindow.setWindowManager(
                (WindowManager)context.getSystemService(Context.WINDOW_SERVICE),
                mToken, mComponent.flattenToString(),
                (info.flags & ActivityInfo.FLAG_HARDWARE_ACCELERATED) != 0);
        if (mParent != null) {
            mWindow.setContainer(mParent.getWindow());
        }
        mWindowManager = mWindow.getWindowManager();
        mCurrentConfig = config;
        ......
    }

}
```
Activity 的 attach 方法, 主要任务是绑定 Context 实现类, 为 Activity 创建一个 Window, 后续的视图均填充在这个 Window 下

至此 Activity 的启动就分析完了
                                                                  
## 总结
至此整个 Activity 创建的流程就结束了, 整个 Activity 的启动流程如下
- 准备阶段
  - 获取目标 Activity 的 ActivityRecord
    - 不存在则创建一个
  - 获取目标 Activity 所在的任务栈 ActivityStack
    - 不存在则创建一个
- 若发起方 Activity 未暂停, 则通知发起方的应用进程处理请求 Activity 的暂停
  - **暂停成功之后, 通知 AMS, 继续进行目标 Activity 的启动操作**
- 若目标 Activity 的进程未启动, 则先通过 Socket 请求 Zygote fork  应用进程
  - 应用进程初始化阶段会启动 Binder 的 loop 线程
  - **ActivityThread.main 中会回调 AMS.attachApplication 继续执行后续操作**
- AMS 发送请求到应用进程, 执行目标 Activity 的启动
  - 构建 App 的资源包
    - 优先从缓存中获取 LoadedApk
    - 不存在, 则构建一个对象, 添加到缓存 mPackages 中
  - **创建一个 Activity 的 Context**
    - 创建 ContextImpl
    - 注入 Resource 资源信息
  - 构建 Activity 实例
    - **创建进程唯一的 Application 对象**
    - Application 进程间单例
    - 创建 ContextImpl
    - 注入 Resource 资源信息
  - 让 appContext 绑定 Activity
  - **为 Activity 绑定参数**
    - 绑定 ContextImpl
    - 创建 Window 和 WindowManager
  - **回调 onCreate**