---
title: Android 系统架构 —— AMS 的启动
permalink: android-source/ams-start
key: android-source-ams-start
tags: AndroidFramework
---

## 前言
从 SystemServer 启动的分析中我们得知, 在其 startBootstrapServices 方法中, 启动引导服务的过程中, 会优先启动 AMS

<!--more-->

```
public class ActivityManagerService extends IActivityManager.Stub
        implements Watchdog.Monitor, BatteryStatsImpl.BatteryCallback {
        
    private void startBootstrapServices() {
        ......
        // 3.1 启动 AMS
        mActivityManagerService = mSystemServiceManager.startService(
                ActivityManagerService.Lifecycle.class).getService();
        mActivityManagerService.setSystemServiceManager(mSystemServiceManager);
        ......
        // 3.3 AMS 设置系统服务进程的程序
        mActivityManagerService.setSystemProcess();
        ......
    }
    
    private void startOtherServices() {
        // 安装系统 Provider
        mActivityManagerService.installSystemProviders();
        ......
        // 3.5 所有服务初始化完毕了, 调用 AMS 的 systemReady
        mActivityManagerService.systemReady(new Runnable() {
            public void run() {
                // 启动服务 "com.android.systemui" 和 "com.android.systemui.SystemUIService"
                startSystemUi(context);
                // 调用其他服务的 systemReady 方法
                ...
                // 调用其他服务的 systemRunning 方法
                ...
            }
         });
         ......
    }
    
}
```
AMS 维护各个进程的四大组件的状态, 是我们应用层开发必不可少的一个进程, 这里主要从下面三个方面来查看 AMS 的启动
- AMS 的启动
- 设置系统服务程序
- AMS 的 systemReady

## 一. AMS 的启动
从 startBootstrapServices 中关于 AMS 启动的代码中可知, 它的优先调用了 mSystemServiceManager.startService 获取一个 ActivityManagerService.Lifecycle 对象, 进而通过 getService 获取了 ActivityManagerService 对象, 其代码实现如下
```java
public class SystemServiceManager {
    
     public <T extends SystemService> T startService(Class<T> serviceClass) {
        try {
            final String name = serviceClass.getName();
            ......
            final T service;
            try {
                // 1. 构造 Service 对象
                // 1.1 获取含有 Context 的构造函数
                Constructor<T> constructor = serviceClass.getConstructor(Context.class);
                // 1.2 创建 Service 对象
                service = constructor.newInstance(mContext);
            } catch (InstantiationException ex) {
                ......
            }
            // 2. 启动 Service
            startService(service);
            return service;
        }
        ......
    }
    
    // Services that should receive lifecycle events.
    private final ArrayList<SystemService> mServices = new ArrayList<SystemService>();
    
    public void startService(@NonNull final SystemService service) {
        // 添加到缓存
        mServices.add(service);
        ......
        try {
            // 2.1 调用了  ActivityManagerService.Lifecycle 的 start 方法
            service.onStart();
        } catch (RuntimeException ex) {
            ......
        }
        ......
    }
    
}
```
这里主要有两个步骤
- 通过反射构建  ActivityManagerService.Lifecycle 对象
- 回调 ActivityManagerService.Lifecycle 的 onStart 函数

接下来我们就从这两个方面来分析 AMS 的创建和启动流程

### 一) AMS 的创建
```java
public class ActivityManagerService extends IActivityManager.Stub {
    
    public static final class Lifecycle extends SystemService {
    
        private final ActivityManagerService mService;

        public Lifecycle(Context context) {
            super(context);
            // 创建了 AMS
            mService = new ActivityManagerService(context);
        }
        
        ......
    }
    
    public ActivityManagerService(Context systemContext) {
        LockGuard.installLock(this, LockGuard.INDEX_ACTIVITY);
        mInjector = new Injector();
        mContext = systemContext;
        ......
        // 保存当前线程
        mSystemThread = ActivityThread.currentActivityThread();
        mUiContext = mSystemThread.getSystemUiContext();

        // 创建一个名为 "ActivityManager" 的前台线程, 并获取 mHandler
        mHandlerThread = new ServiceThread(TAG,
                THREAD_PRIORITY_FOREGROUND, false /*allowIo*/);
        mHandlerThread.start();
        
        // 绑定了 mHandlerThread 的 Looper
        mHandler = new MainHandler(mHandlerThread.getLooper());
        
        // 通过 UiThread 类，创建名为 "android.ui" 的线程
        mUiHandler = mInjector.getUiHandler(this);
        ......
        // 前台广播接收器，在运行超过 10s 将放弃执行
        mFgBroadcastQueue = new BroadcastQueue(this, mHandler,
                "foreground", BROADCAST_FG_TIMEOUT, false);
        // 后台广播接收器, 在运行超过 60s 将放弃执行
        mBgBroadcastQueue = new BroadcastQueue(this, mHandler,
                "background", BROADCAST_BG_TIMEOUT, true);
        mBroadcastQueues[0] = mFgBroadcastQueue;
        mBroadcastQueues[1] = mBgBroadcastQueue;
        
        // 创建 ActiveServices, 管控启动的服务
        mServices = new ActiveServices(this);
        // 创建 ProviderMap, 缓存 ContextProviders
        mProviderMap = new ProviderMap(this);
        ......
        
        // 创建目录/data/system
        File dataDir = Environment.getDataDirectory();
        File systemDir = new File(dataDir, "system");
        systemDir.mkdirs();

        // 创建服务 BatteryStatsService
        mBatteryStatsService = new BatteryStatsService(systemContext, systemDir, mHandler);
        mBatteryStatsService.getActiveStatistics().readLocked();
        mBatteryStatsService.scheduleWriteToDisk();
        ......
        // 创建进程统计服务，信息保存在目录/data/system/procstats，
        mProcessStats = new ProcessStatsService(this, new File(systemDir, "procstats"));
        ......
        // 任务栈监控者
        mStackSupervisor = createStackSupervisor();
        ......
        // Activity 启动控制器
        mActivityStartController = new ActivityStartController(this);
        // 最近的任务
        mRecentTasks = createRecentTasks();
        mStackSupervisor.setRecentTasks(mRecentTasks);
        mLockTaskController = new LockTaskController(mContext, mStackSupervisor, mHandler);
        
        // 客户端生命周期的管控者
        mLifecycleManager = new ClientLifecycleManager();
        
        // 创建 CpuTracker 线程
        mProcessCpuThread = new Thread("CpuTracker") {
            @Override
            public void run() {
                .......
            }
        };

    }
}
```
AMS 构造时的信息比较多, 很多内容都比较陌生, 大体可分为三类
- 额外的服务
  - BatteryStatsService 电池状态服务
  - ProcessStatsService 进程统计服务 
- 工作线程
  - ActivityManager 前台线程 
  - UiThread 线程
  - CpuTracker 线程 
- 四大组件管控者
  - ActiveServices mServices: 管控启动的服务
  - ActivityStackSupervisor mStackSupervisor: 任务栈的监视器
  - ActivityStartController mActivityStartController: Activity 启动控制器
  - ClientLifecycleManager mLifecycleManager: 应用进程生命周期管理者

好的, 笔者并不是 Framework 开发者, 这里看起来还是比较吃力的, 这里我们浅尝辄止, 下面看看它的 start 方法

### 二) start 方法
```
public class ActivityManagerService extends IActivityManager.Stub {
    
    public static final class Lifecycle extends SystemService {
    
        private final ActivityManagerService mService;
        
        @Override
        public void onStart() {
            mService.start();
        }
        
    }
    
    private void start() {
        // 移除所有的进程组
        removeAllProcessGroups();
        // mProcessCpuThread 在 AMS 的构造函数中创建, 这里启动它
        mProcessCpuThread.start();
        // 发布电池统计服务
        mBatteryStatsService.publish();
        mAppOpsService.publish(mContext);
        ......
        // 创建 LocalService
        LocalServices.addService(ActivityManagerInternal.class, new LocalService());
        ......
        try {
            mProcessCpuInitLatch.await();
        } catch (InterruptedException e) {
            ......
        }
    }
    
}
```
其 start 方法中主要处理了 AMS 构造函数中创建的服务和线程的启动操作, 笔者平时与之接触的比较少, 这里就不展开分析了

下面看看 AMS.setSystemProcess 设置服务程序的过程

## 二. AMS 设置服务程序
```
public class ActivityManagerService extends IActivityManager.Stub { 
    
    public void setSystemProcess() {
        try {
            // 注册自己
            ServiceManager.addService(Context.ACTIVITY_SERVICE, this, /* allowIsolated= */ true,
                    DUMP_FLAG_PRIORITY_CRITICAL | DUMP_FLAG_PRIORITY_NORMAL | DUMP_FLAG_PROTO);
            // 注册进程统计服务
            ServiceManager.addService(ProcessStats.SERVICE_NAME, mProcessStats);
            // 注册内存信息服务
            ServiceManager.addService("meminfo", new MemBinder(this), /* allowIsolated= */ false,
                    DUMP_FLAG_PRIORITY_HIGH);
            // 注册图像信息服务
            ServiceManager.addService("gfxinfo", new GraphicsBinder(this));
            // 注册数据库服务
            ServiceManager.addService("dbinfo", new DbBinder(this));
            if (MONITOR_CPU_USAGE) {
                // 注册 CPU 信息服务
                ServiceManager.addService("cpuinfo", new CpuBinder(this),
                        /* allowIsolated= */ false, DUMP_FLAG_PRIORITY_CRITICAL);
            }
            // 注册权限控制服务
            ServiceManager.addService("permission", new PermissionController(this));
            // 注册进程信息服务
            ServiceManager.addService("processinfo", new ProcessInfoService(this));

            ApplicationInfo info = mContext.getPackageManager().getApplicationInfo(
                    "android", STOCK_PM_FLAGS | MATCH_SYSTEM_ONLY);
            mSystemThread.installSystemApplicationInfo(info, getClass().getClassLoader());
            synchronized (this) {
                // 注册当前进程的记录
                ProcessRecord app = newProcessRecordLocked(info, info.processName, false, 0);
                app.persistent = true;
                app.pid = MY_PID;
                app.maxAdj = ProcessList.SYSTEM_ADJ;
                app.makeActive(mSystemThread.getApplicationThread(), mProcessStats);
                synchronized (mPidsSelfLocked) {
                    mPidsSelfLocked.put(app.pid, app);
                }
                // 维护进程的 LRU
                updateLruProcessLocked(app, false, null);
                updateOomAdjLocked();
            }
        } catch (PackageManager.NameNotFoundException e) {
            ......
        }
        ......
    }
    
}
```
可以看到 ActivityManagerService 的 setSystemProcess 中会向 ServiceManager 中注册一些额外的服务, 主要如下
- AMS 服务
- 进程统计服务
- 内存信息服务
- 图像信息服务
- 数据库服务
- CPU 信息服务
- 权限控制服务
- 进程信息服务

接下来看看 AMS 的 systemReady 方法

## 三. AMS 的 systemReady
```
public class ActivityManagerService extends IActivityManager.Stub { 

    public void systemReady(final Runnable goingCallback, TimingsTraceLog traceLog) {
        ......
        // 1. 准备 AMS 
        // 将非 persistent 进程加入 procsToKill 中, 到后面进行杀进程操作
        ArrayList<ProcessRecord> procsToKill = null;
        synchronized(mPidsSelfLocked) {
            for (int i=mPidsSelfLocked.size()-1; i>=0; i--) {
                ProcessRecord proc = mPidsSelfLocked.valueAt(i);
                if (!isAllowedWhileBooting(proc.info)){
                    if (procsToKill == null) {
                        procsToKill = new ArrayList<ProcessRecord>();
                    }
                    procsToKill.add(proc);
                }
            }
        }
        // 执行杀进程操作
        synchronized(this) {
            if (procsToKill != null) {
                for (int i=procsToKill.size()-1; i>=0; i--) {
                    ProcessRecord proc = procsToKill.get(i);
                     ......
                    removeProcessLocked(proc, true, false, "system update done");
                }
            }
            ......
            mProcessesReady = true;
        }
        ......
        // 2. 执行 goingCallback 回调, 准备其他服务
        if (goingCallback != null) goingCallback.run();
        ......
        // 3. 启动 App
        synchronized (this) {
            // 启动持久化的 App
            startPersistentApps(PackageManager.MATCH_DIRECT_BOOT_AWARE);
            // Start up initial activity.
            mBooting = true;
            // 启动 Launcher
            startHomeActivityLocked(currentUserId, "systemReady");
            ......
            // 恢复栈顶 Activity 的启动
            mStackSupervisor.resumeFocusedStackTopActivityLocked();
        }
    }
    
}
```
ActivityManagerService 的 systemReady 代码量非常多, 它主要的操作如下
- 准备 AMS
  - 清除非持久化的进程
- 执行 goingCallback 回调, 准备其他服务
  - startSystemUi
  - 调用其他服务的 systemReady 方法
  - 调用其他服务的 systemRunning 方法
- 启动 App
  - 启动持久化的 App
  - 启动 Launcher

好的, 当 goingCallback 回调完毕后, 这意味着其他服务都已经准备好了, 下面就可以正常的进行 App 的启动了

## 总结
AMS 在 SystemServer 中的启动主要有三个方面
- **构造 和 启动**
  - 额外的服务
    - BatteryStatsService 电池状态服务
    - ProcessStatsService 进程统计服务 
  - 工作线程
    - ActivityManager 前台线程 
    - UiThread 线程
    - CpuTracker 线程 
  - 四大组件管控者
    - ActiveServices mServices: 管控启动的服务
    - ActivityStackSupervisor mStackSupervisor: 任务栈的监视器
    - ActivityStartController mActivityStartController: Activity 启动控制器
    - ClientLifecycleManager mLifecycleManager: 应用进程生命周期管理者
- **设置系统服务进程的程序**
  - AMS 服务
  - 进程统计服务
  - 内存信息服务
  - 图像信息服务
  - 数据库服务
  - CPU 信息服务
  - 权限控制服务
  - 进程信息服务 
- **回调服务准备完成**
  - 准备 AMS
    - 清除非持久化的进程
  - 执行 goingCallback 回调, 准备其他服务
    - startSystemUi
    - 调用其他服务的 systemReady 方法
    - 调用其他服务的 systemRunning 方法
  - 启动 App
    - 启动持久化的 App
    - 启动 Launcher
    - 恢复栈顶 Activity 的启动

## 参考
- http://gityuan.com/2016/02/21/activity-manager-service/