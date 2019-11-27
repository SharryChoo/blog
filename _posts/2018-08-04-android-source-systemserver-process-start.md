---
layout: article
title: Android 系统架构 —— SystemServer 进程的启动
permalink: android-source/systemserver-process-start
key: android-source-systemserver-process-start
tags: AndroidFramework
aside:
  toc: true
---

## 前言
在分析 Zygote 启动的时候, 我们注意到它调用了 ZygoteInit.forkSystemServer 函数来创建系统服务进程

这里我们追踪一下系统服务进程的创建, 在此之前我们先回顾一下 SystemServer 进程的发起

<!--more-->

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
- 构建孵化参数
  - 启动后的 main 方法在 "com.android.server.SystemServer" 中
- 会通过 Zygote.forkSystemServer 方法孵化进程
- 然后调用 handleSystemServerProcess 初始化子进程

下面我们就从进程的创建和初始化两个方面来看看 SystemServer 的启动

## 一. SystemServer 进程的创建
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

至此 SystemServer 进程便创建了, 接下来看看 handleSystemServerProcess 是如何初始化 SystemServer 进程的

## 二. SystemServer 进程的初始化
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
        ......
        // 调用了 ZygoteInit.nativeZygoteInit 执行 native 层的初始化操作
        ZygoteInit.nativeZygoteInit();
        // 转发给 RuntimeInit.applicationInit 继续执行
        return RuntimeInit.applicationInit(targetSdkVersion, argv, classLoader);
    }
    
}
``` 
好的可以看到 ZygoteInit.handleSystemServerProcess 最终会调用到 zygoteInit 方法中, 它做了如下两件事件
- **ZygoteInit.nativeZygoteInit**: 执行 native 初始化
- **RuntimeInit.applicationInit**: 执行 java 层初始化

我们先了解一下 ZygoteInit.nativeZygoteInit 这个函数的实现

### 一) Native 的初始化
```
// frameworks/base/core/jni/AndroidRuntime.cpp
static AndroidRuntime* gCurRuntime = NULL;

static void com_android_internal_os_ZygoteInit_nativeZygoteInit(JNIEnv* env, jobject clazz)
{
    gCurRuntime->onZygoteInit();
}

// frameworks/base/core/jni/include/android_runtime/AndroidRuntime.h
/**
 * This gets called after the JavaVM has initialized after a Zygote
 * fork. Override it to initialize threads, etc. Upon return, the
 * correct static main will be invoked.
 */
virtual void onZygoteInit() { }   
```
可以看到 nativeZygoteInit 函数调用了 AndroidRuntime 的 onZygoteInit 函数

在 AndroidRuntime 的头文件中, 我们可以看到了它是一个虚函数, 也就是说它是需要交由子类实现的

我们在 Zygote 启动的过程中知道 AndroidRuntime 的实现类为 AppRuntime, 因此我们在 AppRuntime 中追踪该函数的实现
```
/frameworks/base/cmds/app_process/app_main.cpp
class AppRuntime : public AndroidRuntime
{
    
    virtual void onZygoteInit()
    {
        sp<ProcessState> proc = ProcessState::self();
        // 启动了 Binder 驱动的线程池
        proc->startThreadPool();
    }
    
}

// frameworks/native/libs/binder/ProcessState.cpp
void ProcessState::startThreadPool()
{
    AutoMutex _l(mLock);
    if (!mThreadPoolStarted) {
        mThreadPoolStarted = true;
        // 孵化一个线程
        spawnPooledThread(true);
    }
}

void ProcessState::spawnPooledThread(bool isMain)
{
    if (mThreadPoolStarted) {
        String8 name = makeBinderThreadName();
        // 为 Binder 线程池创建一个线程, 并且标记为 Binder 线程池的主线程
        sp<Thread> t = new PoolThread(isMain);
        t->run(name.string());
    }
}
```
关于 Binder 驱动线程池的启动, 我们在 Binder 驱动的章节再去分析, 这里我们只需要知道, **Binder 驱动线程池的主线程启动之后就可以在 looper 中通过 Binder 驱动获取并处理其他进程的跨进程请求了**

下面我们看看 Java 层的初始化流程

### 二) Java 层初始化
最终这个请求会转发到了 RuntimeInit.applicationInit 中去, 我们继续往下分析
```
public class RuntimeInit {

   protected static Runnable applicationInit(int targetSdkVersion, String[] argv,
            ClassLoader classLoader) {
        ......
        // 将参数封装到 Arguments 对象中
        final Arguments args = new Arguments(argv);
        // 通过 findStaticMain 获取一个 Runnable 对象
        return findStaticMain(args.startClass, args.startArgs, classLoader);
    }
    
}
```
下面我们看看 RuntimeInit.findStaticMain 是如何获取 Runnable 对象的

#### 获取 Runnable 对象
```
public class RuntimeInit {
    protected static Runnable findStaticMain(String className, String[] argv,
            ClassLoader classLoader) {
        Class<?> cl;
        // 找寻 main 函数启动类
        try { 
            cl = Class.forName(className, true, classLoader);
        } catch (ClassNotFoundException ex) {
            throw new RuntimeException(
                    "Missing class when invoking static main " + className,
                    ex);
        }
        // 找寻 main 函数
        Method m;
        try {
            m = cl.getMethod("main", new Class[] { String[].class });
        } catch (NoSuchMethodException ex) {
            ......
        }

        int modifiers = m.getModifiers();
        if (! (Modifier.isStatic(modifiers) && Modifier.isPublic(modifiers))) {
            throw new RuntimeException(
                    "Main method is not public and static on " + className);
        }
        // 创建一个 Runnable
        return new MethodAndArgsCaller(m, argv);
    }
    
    static class MethodAndArgsCaller implements Runnable {
    
        /** method to call */
        private final Method mMethod;

        /** argument array */
        private final String[] mArgs;

        public MethodAndArgsCaller(Method method, String[] args) {
            mMethod = method;
            mArgs = args;
        }

        public void run() {
            try {
                mMethod.invoke(null, new Object[] { mArgs });
            } catch (IllegalAccessException ex) {
                ......
            }
        }
    }
}
```
从 ZygoteInit.forkSystemServer 中 args 的构建可知, 这个 main 函数的入口其实在 "com.android.server.SystemServer" 中, 也就是说, 只要这个 Runnable 被调用了, 就正式的进入 SystemServer 的启动了

**那么我们可能就存在疑惑了创建了 Runnable 对象, 那这个 run 方法什么时候执行呢?**

因为应用进程复制了 Zygote 的地址空间, 所以它的函数函数调用栈与 Zygote 是一致的, 我们慢慢往上回溯, 最终到了 ZygoteInit.main 方法中
```
class ZygoteInit {

    public static void main(String argv[]) {
        ......
        final Runnable caller;
        try {
            // runSelectLoop 对于 Zygote 来说是一个死循环, 但对于子进程来说, 它会返回一个 Runnable
            caller = zygoteServer.runSelectLoop(abiList);
        } catch (Throwable ex) {
            ......
        } finally {
            zygoteServer.closeServerSocket();
        }
        // 调用 run 方法, 启动子进程的  run 方法
        if (caller != null) {
            caller.run();
        }
    }
    
}
```
好的, 到这里 SystemServer 进程的初始化操作就完成了, 下面看看它的启动流程

## 三. SystemServer 的启动
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

### Launcher 进程的启动
```
public final class SystemServer {

    ......
    
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
        .......
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
- ActivityManagerService 是管理 Android 四大组件的服务

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
**当系统服务中的其他服务启动完毕之后, 便会回调 AMS 的 systemReady, 执行了 Launcher 的启动**

Launcher 是一个 App, 也就是说会创建一个 Launcher 的应用进程, 至此便过渡到了应用进程的启动

关于应用进程的启动我们到后面的章节继续分析

## 总结
系统服务进程是由 Zygote 进程孵化的第一个进程, 它的职责是实现 Android 必要的一些服务, 创建之后会进行两个阶段的初始化操作
- **创建**
  - 通过 Zygote.fork 创建
- **初始化**
  - Native 层的初始化
    - 通过 AppRuntime.onZygoteInit 函数, 启动 Binder 驱动线程池的主线程, 监听 Binder 驱动的跨进程通信请求
  - Java 层的初始化
    - 创建 SystemServer 的 main 方法的 Runnable 对象
- **启动**
  - 引导服务
  - 核心
  - 其他服务
    - 其他服务启动完毕之后, 通过 AMS 的 systemReady 执行 Launcher 的启动