---
layout: article
title: "Android 系统架构 —— 系统启动篇 之 SystemServer 进程的启动"
key: "Android 系统架构 —— 系统启动篇 之 SystemServer 进程的启动" 
tags: AndroidFramework
aside:
  toc: true
---

## 前言
在分析 Zygote 启动的时候, 其 调用了 ZygoteInit.forkSystemServer 函数来创建系统服务进程

这里我们追踪一下系统服务进程的创建

<!--more-->

## 一. 创建系统服务进程
先回顾一下 SystemServer 进程的发起
```
// frameworks/base/core/java/com/android/internal/os/ZygoteInit.java
public class ZygoteInit {

    private static Runnable forkSystemServer(String abiList, String socketName,
            ZygoteServer zygoteServer) {
        ......
        
        // 1. 构建系统服务进程的参数
        String args[] = {
            "--setuid=1000",
            "--setgid=1000",
            "--setgroups=1001,1002,1003,1004,1005,1006,1007,1008,1009,1010,1018,1021,1023,1024,1032,1065,3001,3002,3003,3006,3007,3009,3010",
            "--capabilities=" + capabilities + "," + capabilities,
            "--nice-name=system_server",
            "--runtime-args",
            "--target-sdk-version=" + VMRuntime.SDK_VERSION_CUR_DEVELOPMENT,
            "com.android.server.SystemServer",
        };
        ZygoteConnection.Arguments parsedArgs = null;
        int pid;
        try {
            // 2. 将 args 参数封装成 Arguments对象
            parsedArgs = new ZygoteConnection.Arguments(args);
            ......
           // 3. 调用 Zygote.forkSystemServer() 孵化系统服务进程
            pid = Zygote.forkSystemServer(
                    parsedArgs.uid, parsedArgs.gid,
                    parsedArgs.gids,
                    parsedArgs.runtimeFlags,
                    null,
                    parsedArgs.permittedCapabilities,
                    parsedArgs.effectiveCapabilities);
        } catch (IllegalArgumentException ex) {
            ......
        }
        // pid == 0 表示在新 fork 的子进程中调用(即系统服务进程)
        if (pid == 0) {
            ......
            // 处理系统服务进程的启动操作
            return handleSystemServerProcess(parsedArgs);
        }
        // 表示在 Zygote 进程调用, 返回 null
        return null;
    }

}
```
通过 Zygote 进程启动可知, SystemServer 进程在其启动过程中
- 首先会通过 Zygote.forkSystemServer 方法孵化出来
- 然后调用 handleSystemServerProcess 处理 SystemServer 的启动

### 孵化 SystemServer 进程
接下来就分析一下 Zygote.forkSystemServer() 这个方法
```
public final class Zygote {
    public static int forkSystemServer(int uid, int gid, int[] gids, int runtimeFlags,
            int[][] rlimits, long permittedCapabilities, long effectiveCapabilities) {
        ......
        int pid = nativeForkSystemServer(
                uid, gid, gids, runtimeFlags, rlimits, permittedCapabilities, effectiveCapabilities);
        ......
        return pid;
    }
    
    native private static int nativeForkSystemServer(int uid, int gid, int[] gids, int runtimeFlags,
            int[][] rlimits, long permittedCapabilities, long effectiveCapabilities);
            
}
```
可见 Zygote 中 forkSystemServer 将启动系统服务进程的工作转发到了 native 层去做, 我们继续追踪
```
// frameworks/base/core/jni/com_android_internal_os_Zygote.cpp
namespace android {
    
    static jint com_android_internal_os_Zygote_nativeForkSystemServer(
            JNIEnv *env, jclass, uid_t uid, gid_t gid, jintArray gids,
            jint runtime_flags, jobjectArray rlimits, jlong permittedCapabilities,
            jlong effectiveCapabilities) {
        // 调用了 ForkAndSpecializeCommon, 来孵化这个系统进程
        pid_t pid = ForkAndSpecializeCommon(env, uid, gid, gids,
                                            runtime_flags, rlimits,
                                            permittedCapabilities, effectiveCapabilities,
                                            MOUNT_EXTERNAL_DEFAULT, NULL, NULL, true, NULL,
                                            NULL, false, NULL, NULL);
        ......
        return pid;
    }
    
}

namespace {
    static pid_t ForkAndSpecializeCommon(......) {
        ......
        // fork 了一个 SystemService 进程
        pid_t pid = fork();
        ......
        return pid;
    }
}
```
可以看到, 最终会调用一个 fork 方法孵化一个进程

至此 SystemServer 进程便创建了, 接下来看看系统服务进程的启动

## 二. 启动系统服务进程
```
// frameworks/base/core/java/com/android/internal/os/ZygoteInit.java
public class ZygoteInit {
    
    private static Runnable handleSystemServerProcess(ZygoteConnection.Arguments parsedArgs) {
       
        if (parsedArgs.invokeWith != null) {
            ......
        } else {
            ClassLoader cl = null;
            if (systemServerClasspath != null) {
                // 给这个设置类加载器
                cl = createPathClassLoader(systemServerClasspath, parsedArgs.targetSdkVersion);
                Thread.currentThread().setContextClassLoader(cl);
            }
            // 将剩下参数传递给 zygoteInit 方法
            return ZygoteInit.zygoteInit(parsedArgs.targetSdkVersion, parsedArgs.remainingArgs, cl);
        }
    }
    
    public static final Runnable zygoteInit(int targetSdkVersion, String[] argv, ClassLoader classLoader) {
        
        // 调用了 commonInit() 设置 System 进程的时区和键盘布局等信息
        RuntimeInit.commonInit();
        // 初始化了 System 进程中的 Binder 线程池
        ZygoteInit.nativeZygoteInit();
        // 回调 main 方法
        return RuntimeInit.applicationInit(targetSdkVersion, argv, classLoader);
    }
    
}
``` 
好的可以看到 ZygoteInit.handleSystemServerProcess 最终将 SystemServer 的启动, 转发到了 RuntimeInit.applicationInit 中去, 我们继续往下分析

```
public class RuntimeInit {

   protected static Runnable applicationInit(int targetSdkVersion, String[] argv,
            ClassLoader classLoader) {
        ......
        // 将参数封装到 Arguments 对象中
        final Arguments args = new Arguments(argv);
        // 找寻 SystemService 进程的 main 函数入口, 并且回调它
        return findStaticMain(args.startClass, args.startArgs, classLoader);
    }
    
}
```
从 ZygoteInit.forkSystemServer 中 args 的构建可知, 这个 main 函数的入口其实在 "com.android.server.SystemServer" 中, 我们继续追踪下去
```
public final class SystemServer {

    private Context mSystemContext;
    private SystemServiceManager mSystemServiceManager;

    /**
     * The main entry point from zygote.
     */
    public static void main(String[] args) {
        new SystemServer().run();
    }
    
    private void run() {
        try {
            // 准备主线程的消息循环
            Looper.prepareMainLooper();
            ......
            // 1. 创建系统服务进程的上下文
            createSystemContext();
            // 2. 创建了一个 SystemServiceManager 的实例对象
            mSystemServiceManager = new SystemServiceManager(mSystemContext);
            mSystemServiceManager.setStartInfo(mRuntimeRestart,
                    mRuntimeStartElapsedTime, mRuntimeStartUptime);
            LocalServices.addService(SystemServiceManager.class, mSystemServiceManager);
            ......
        } finally {
            ......
        }
        
        // 3. 启动系统服务
        try {
            // 启动引导服务
            startBootstrapServices();
            // 启动系统核心服务
            startCoreServices();
            // 启动系统其他服务
            startOtherServices();
            ......
        } catch (Throwable ex) {
            ......
        } finally {
            ......
        }
        ......
        // 开启消息循环
        Looper.loop();
    }
    
    private void createSystemContext() {
        // 1.1 调用 ActivityThread.systemMain 获取一个 ActivityThread 实例
        ActivityThread activityThread = ActivityThread.systemMain();
        // 1.2 通过这个实例获取 Context 对象
        mSystemContext = activityThread.getSystemContext();
        mSystemContext.setTheme(DEFAULT_SYSTEM_THEME);
        ......
    }
    
}
```
可见 SystemServer 的 main 函数, 主要做了入下操作 
- 创建了 SystemServer 的上下文
  - 调用 ActivityThread.systemMain 获取一个 ActivityThread 实例
  -  通过 ActivityThread 实例获取 Context 对象
- 创建了 SystemServiceManager 系统服务的管理者
- 启动系统服务
  - 引导服务
    - ActivityManagerService、PowerManagerService、LightsService、DisplayManagerService、PackageManagerService、UserManagerService、SensorService 
  - 核心服务
    - BatteryService、UsageStatsService、WebViewUpdateService 
  - 其他服务
    - AlarmManagerService、VibratorService...

一般在我们手机开启启动之后便会进入到 launcher 页, 那么系统服务进程是如何触发 Launcher 启动的呢? 

我们带着这个问题继续探究

### Launcher 的启动
```
public final class SystemServer {

    private Context mSystemContext;
    private SystemServiceManager mSystemServiceManager;

    /**
     * The main entry point from zygote.
     */
    public static void main(String[] args) {
        new SystemServer().run();
    }
    
    private void run() {
        ......
        // 3. 启动系统服务
        try {
            // 启动引导服务
            startBootstrapServices();
            // 启动系统核心服务
            startCoreServices();
            // 启动系统其他服务
            startOtherServices();
            ......
        } catch (Throwable ex) {
            ......
        } finally {
            ......
        }
        ......
        // 开启消息循环
        Looper.loop();
    }
    
    private void startBootstrapServices() {
        ......
        Installer installer = mSystemServiceManager.startService(Installer.class);
        ......
        // Activity manager runs the show.
        mActivityManagerService = mSystemServiceManager.startService(
                ActivityManagerService.Lifecycle.class).getService();
        mActivityManagerService.setSystemServiceManager(mSystemServiceManager);
        mActivityManagerService.setInstaller(installer);
        ......
    }
    
    private void startOtherServices() {
        ......// 初始化其他服务
        // 所有服务初始化完毕了, 调用 AMS 的 systemReady
        mActivityManagerService.systemReady(....);
        ......
    }
    
}
```
从这里可以看到, **系统服务进程在其他服务启动完毕之后, 会调用 ActivityManagerService 的 systemReady 方法**
- ActivityManagerService 是管理 Android 四大组件的服务, 相当于 Android 系统的路由中转站, 这里不做系统分析, 先作为了解

接下来我们去 ActivityManagerService.systemReady 方法中探索一下应用进程的启动

```
public class ActivityManagerService extends IActivityManager.Stub
        implements Watchdog.Monitor, BatteryStatsImpl.BatteryCallback {
            
    public void systemReady(final Runnable goingCallback, TimingsTraceLog traceLog) {
        ......
         synchronized (this) {
            .....
            // 启动开启必要的 app
            startPersistentApps(PackageManager.MATCH_DIRECT_BOOT_AWARE);
            ......
            // 启动 HomeActivity
            startHomeActivityLocked(currentUserId, "systemReady");
        }
        ......
    }
    
}
```
好的, 到这里我们就清晰了, 当系统服务中的其他服务启动完毕之后, 便会回调 AMS 的 systemReady, 这里执行了 Launcher 的启动

具体的启动流程比较复杂, 会到后续的文章中抽丝剥茧的探究

## 总结
系统服务进程是由 Zygote 进程孵化的第一个进程, 它的职责是实现 Android 必要的一些服务, 主要的服务类型如下
- **引导服务**
- **核心服务**
- **其他服务**
  - 服务启动完毕之后, 执行 Launcher 的启动 