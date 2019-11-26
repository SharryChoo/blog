---
layout: article
title: "Android 系统架构 —— Activity 的启动 之 应用进程的启动"
permalink: android-source/activity-launch2
key: android-source-activity-launch2
tags: AndroidFramework
---

## 前言
通过 Step 1 我们知道, Activity 的启动会
- 通知 Client 进程将请求的 SourceActivity 置为 Paused 状态
- 通知 AMS 所在进程, SourceActivity 已经 Pause 成功了
- 调用 ActivityStackSupervisor.resumeFocusedStackTopActivityLocked 继续执行目标 Activity 的启动
  - 这个方法我们上面分析过, **它内部会回调 ActivityStack.resumeTopActivityInnerLocked 来激活当前栈顶的 Activity, 即我们的 TargetActivity**
 
接下来就看看 resumeTopActivityInnerLocked 的实现

<!--more-->

## 一. 恢复焦点栈顶 Activity 的启动
```
public class ActivityStack {
    
    private boolean resumeTopActivityInnerLocked(ActivityRecord prev, ActivityOptions options) {
        ......
        boolean pausing = mStackSupervisor.pauseBackStacks(userLeaving, next, false);
        if (mResumedActivity != null) {
            ......
            // 显然不会走这里了
            pausing |= startPausingLocked(userLeaving, false, next, false);
        }
        ......// 执行一些 prev 的动画

        // 很显然, next 还没有创建, 
        if (next.app != null && next.app.thread != null) {
            // next 的 app 已经创建, 他的 ApplicationThread 已经创建了会走这里
            ......
        } else {
            ......
            // 很显然我们的 TargetActivity 所在的进程还没有创建
            mStackSupervisor.startSpecificActivityLocked(next, true, true);
        }
        return true;
    }
}

public class ActivityStackSupervisor {
    
    void startSpecificActivityLocked(ActivityRecord r,
            boolean andResume, boolean checkConfig) {
        // 尝试获取要启动的 Activity 的进程描述
        ProcessRecord app = mService.getProcessRecordLocked(r.processName,
                r.info.applicationInfo.uid, true);
        // 此时我们的进程没有创建, 所以这里进不来
        if (app != null && app.thread != null) {
            try {
                ......
                // 2. 执行 Activity 的启动
                realStartActivityLocked(r, app, andResume, checkConfig);
                return;
            } catch (RemoteException e) {
                ......
            }
        }
        // 1. 调用了 AMS 的 startProcessLocked 方法
        mService.startProcessLocked(r.processName, r.info.applicationInfo, true, 0,
                "activity", r.intent.getComponent(), false, false, true);
    }
    
}
```
好的, 到这里我们可以看到
- 当要启动的 Activity 的进程已经启动的时候, 会直接调用 ActivityStackSupervisor.realStartActivityLocked 来执行目标 Activity 的启动
- 当要启动的 Activity 所对应的进程未启动时, 会先调用 AMS 的 startProcessLocked 方法执行进程的启动

```
public class ActivityManagerService {
    
    final ProcessRecord startProcessLocked(......) {
        ProcessRecord app;
        if (!isolated) {
            // 再次尝试获取进程描述
            app = getProcessRecordLocked(processName, info.uid, keepIfLarge);
        } else {
           ......
        }
        if (app == null) {
            // 1. 这里创建了一个进程描述 ProcessRecord 对象
            app = newProcessRecordLocked(info, processName, isolated, isolatedUid);
            app.crashHandler = crashHandler;
            app.isolatedEntryPoint = entryPoint;
            app.isolatedEntryPointArgs = entryPointArgs;
        } else {
            ......
        }
        // 回调另一个重载方法, 最终会进入下面
        final boolean success = startProcessLocked(app, hostingType, hostingNameStr, abiOverride);
        return success ? app : null;
    }
    
    private boolean startProcessLocked(......) {
        app.pendingStart = true;
        app.killedByAm = false;
        app.removed = false;
        app.killed = false;
        // 设置启动新的进程一系列的参数
        app.setStartParams(uid, hostingType, hostingNameStr, seInfo, startTime);
        if (mConstants.FLAG_PROCESS_START_ASYNC) {
            ......
        } else {
            try {
                // 2. Fork 一个新的进程
                final ProcessStartResult startResult = startProcess(hostingType, entryPoint, app,
                        uid, gids, runtimeFlags, mountExternal, seInfo, requiredAbi, instructionSet,
                        invokeWith, startTime);
                // 处理进程的启动后的事宜
                handleProcessStartedLocked(app, startResult.pid, startResult.usingWrapper,
                        startSeq, false);
            } catch (RuntimeException e) {
                ......
            }
            return app.pid > 0;
        }
    }
    
    private boolean handleProcessStartedLocked(ProcessRecord app, int pid, boolean usingWrapper,
            long expectedStartSeq, boolean procAttached) {
        
        synchronized (mPidsSelfLocked) {
            this.mPidsSelfLocked.put(pid, app);
            if (!procAttached) {
                Message msg = mHandler.obtainMessage(PROC_START_TIMEOUT_MSG);
                msg.obj = app;
                // 3. 可以看到, 这里发送了一个 Delayed Msg , 超过 PROC_START_TIMEOUT 进程没有给予 AMS 回应, 则会发送该消息
                mHandler.sendMessageDelayed(msg, usingWrapper
                        ? PROC_START_TIMEOUT_WITH_WRAPPER : PROC_START_TIMEOUT);
            }
        }
        return true;
    }
}
```
好的, 到这里总结一下, 我们回调了一系列的 startProcessLocked 重载方法, 它主要做了如下几件事情
- 创建了一个进程描述 ProcessRecord 对象
- 调用 startProcess 创建新进程
- 将 PROC_START_TIMEOUT_MSG 延时投递到消息队列
  - 若在 PROC_START_TIMEOUT 时间内没有接收到新进程初始化完毕的回调, 则会发送这个消息

好的, 接下来我们便重点看看 startProcess 创建进程的操作

## 二. 进程创建发起
```java
public class Process { 
    public static final ProcessStartResult start(final String processClass,
                                  final String niceName,
                                  int uid, int gid, int[] gids,
                                  int runtimeFlags, int mountExternal,
                                  int targetSdkVersion,
                                  String seInfo,
                                  String abi,
                                  String instructionSet,
                                  String appDataDir,
                                  String invokeWith,
                                  String[] zygoteArgs) {
        
        return zygoteProcess.start(processClass, niceName, uid, gid, gids,
                    runtimeFlags, mountExternal, targetSdkVersion, seInfo,
                    abi, instructionSet, appDataDir, invokeWith, zygoteArgs);
    }
}

public class ZygoteProcess { 
    public final Process.ProcessStartResult start(final String processClass,
                                                  final String niceName,
                                                  int uid, int gid, int[] gids,
                                                  int runtimeFlags, int mountExternal,
                                                  int targetSdkVersion,
                                                  String seInfo,
                                                  String abi,
                                                  String instructionSet,
                                                  String appDataDir,
                                                  String invokeWith,
                                                  String[] zygoteArgs) {
        try {
            return startViaZygote(processClass, niceName, uid, gid, gids,
                    runtimeFlags, mountExternal, targetSdkVersion, seInfo,
                    abi, instructionSet, appDataDir, invokeWith, false /* startChildZygote */,
                    zygoteArgs);
        } catch (ZygoteStartFailedEx ex) {
            ......
        }
    }
    
    private Process.ProcessStartResult startViaZygote(final String processClass,
                                                      final String niceName,
                                                      final int uid, final int gid,
                                                      final int[] gids,
                                                      int runtimeFlags, int mountExternal,
                                                      int targetSdkVersion,
                                                      String seInfo,
                                                      String abi,
                                                      String instructionSet,
                                                      String appDataDir,
                                                      String invokeWith,
                                                      boolean startChildZygote,
                                                      String[] extraArgs)
                                                      throws ZygoteStartFailedEx {
        // 1. 创建了一个用于存储参数的集合
        ArrayList<String> argsForZygote = new ArrayList<String>();
        // 2. 向集合中, 添加进程启动时的必要参数
        // --runtime-args, --setuid=, --setgid=,
        // and --setgroups= must go first
        argsForZygote.add("--runtime-args");
        argsForZygote.add("--setuid=" + uid);
        argsForZygote.add("--setgid=" + gid);
        argsForZygote.add("--runtime-flags=" + runtimeFlags);
        ......
        // 调用了 zygoteSendArgsAndGetResult 方法
        synchronized(mLock) {
            // 3. 先调用了 openZygoteSocketIfNeeded 与 Zygote 建立 Socket 连接
            // 4. 调用 zygoteSendArgsAndGetResult 进行进程的启动
            return zygoteSendArgsAndGetResult(openZygoteSocketIfNeeded(abi), argsForZygote);
        }
    }
    
}
```
可以看到, startProcess 经过了一系列的重载方法, 最终会调用 startViaZygote 来执行后续操作, 这个方法主要做了如下几件事情
- 创建一个集合, 向集合中添加进程启动时的必要参数
- 调用 openZygoteSocketIfNeeded 尝试与 Zygote 进程建立 Socket 连接
- 调用 zygoteSendArgsAndGetResult 执行进程的启动

首先看看与 Zygote 建立连接的过程

### 一) 与 Zygote 进程建立连接
```
public class ZygoteProcess {

    /**
     * 描述 Zygote 进程的 Socket 地址
     * 这个地址在 Process 中使用常量定义
     * {@code #public static final String ZYGOTE_SOCKET = "zygote";}
     */
    private final LocalSocketAddress mSocket;
    
    private ZygoteState openZygoteSocketIfNeeded(String abi) throws ZygoteStartFailedEx {
    
        // 获取之前的 Zygote 状态
        if (primaryZygoteState == null || primaryZygoteState.isClosed()) {
            try {
                // 1. 若为 null, 则调用 ZygoteState.connect 获取一个 ZygoteState 的对象
                primaryZygoteState = ZygoteState.connect(mSocket);
            } catch (IOException ioe) {
                ......
            }
            .....
        }
        // 若这个 ZygoteState 与 abi 匹配, 则说明连接创建成功了
        if (primaryZygoteState.matches(abi)) {
            return primaryZygoteState;
        }

        // 与初始的 Zygote 不匹配, 尝试使用第二个 State
        if (secondaryZygoteState == null || secondaryZygoteState.isClosed()) {
            // ...... 过程与 primaryZygote 一致
        }
        if (secondaryZygoteState.matches(abi)) {
            return secondaryZygoteState;
        }
        ......
    }
    
    public static class ZygoteState {
         final LocalSocket socket;
        final DataInputStream inputStream;
        final BufferedWriter writer;
        final List<String> abiList;

        boolean mClosed;
        
        public static ZygoteState connect(LocalSocketAddress address) throws IOException {
            DataInputStream zygoteInputStream = null;
            BufferedWriter zygoteWriter = null;
            // 2. 为当前进程创建了一个 Socket 对象
            final LocalSocket zygoteSocket = new LocalSocket();
            try {
                // 让当前进程的 Socket 与 address 地址映射的 Socke 建立起连接, 即与 Zygote 进程的 socket 建立起连接
                zygoteSocket.connect(address);
                // 获取输入流
                zygoteInputStream = new DataInputStream(zygoteSocket.getInputStream());
                // 获取输出流
                zygoteWriter = new BufferedWriter(new OutputStreamWriter(
                        zygoteSocket.getOutputStream()), 256);
            } catch (IOException ex) {
               ......
            }
            ......
            return new ZygoteState(zygoteSocket, zygoteInputStream, zygoteWriter,
                    Arrays.asList(abiListString.split(",")));
        }
    }
    
}
```
AMS 所在的系统服务进程与 Zygote 进程建立起连接的方式如下
- 通过 ZygoteState.connect(mSocket) 将 Zygote 进程的 Socket 地址传入(端口号)
  - mSocket 指向的设备文件地址为 **"dev/socket/zygoet"**
- 在当前进程中创建一个 LocalSocket 对象 zygoteSocket
  - 调用 **zygoteSocket.connect(address)** 与 Zygote 进程的 Socket 建立起连接
- 获取当前进程与 Zygote 进程之间的输入流 **zygoteInputStream**
- 获取当前进程与 Zygote 进程之间的输出流 **zygoteWriter**

知晓了连接的建立, 接下来看看请求的发送

### 二) 向 Zygote 发送请求
```
public class ZygoteProcess {
    private static Process.ProcessStartResult zygoteSendArgsAndGetResult(
            ZygoteState zygoteState, ArrayList<String> args)
            throws ZygoteStartFailedEx {
        try {
            // 1. 获取我们上面在 ZygoteState 中拿到的输入输出流
            final BufferedWriter writer = zygoteState.writer;
            final DataInputStream inputStream = zygoteState.inputStream;
            // 2. 将 args 参数中的数据, 通过输出流, 写入到 Zygote 进程中的 Socket 的设备文件地址 "dev/socket/zygoet" 中
            writer.write(Integer.toString(args.size()));
            writer.newLine();
            for (int i = 0; i < sz; i++) {
                String arg = args.get(i);
                writer.write(arg);
                writer.newLine();
            }
            writer.flush();
            
            // 3. 写入完成之后, 等待新进程的启动结果
            Process.ProcessStartResult result = new Process.ProcessStartResult();

            // 3.1 从输入流中等候新进程的创建返回的 pid
            result.pid = inputStream.readInt();
            result.usingWrapper = inputStream.readBoolean();
            return result;
        } catch (IOException ex) {
            ......
        }
    }
}
```
可见, AMS 所在的系统服务进程, 是通过将我们前面封装好的 args 参数, 通过与 Socket 连接传输到 Zygote 进程中, 通知其创建新的进程, 至此系统服务进程就与 Zygote 进程建立起 Socket 连接了

接下来看看 Zygote 进程是如何处理的子进程创建的

## 三. Zygote 创建应用进程
在 Zygote 进程启动的过程中我们知道, 它会有一个死循环, 来读取其他进程发送的进程孵化请求, 这里我们继续分析
```
// frameworks/base/core/java/com/android/internal/os/ZygoteServer.java
class ZygoteServer {

    Runnable runSelectLoop(String abiList) {
        // Socket 的文件描述集合, 从上面 Zygote 的 Socket 创建可知, 在构造 Socket 实例时, 会传入其相应的文件描述
        ArrayList<FileDescriptor> fds = new ArrayList<FileDescriptor>();   
        // 与 Zygote 建立连接的集合
        ArrayList<ZygoteConnection> peers = new ArrayList<ZygoteConnection>();
        // 将当前 Zygote 进程的 Socket 文件描述添加进去
        fds.add(mServerSocket.getFileDescriptor());
        peers.add(null);
        // 开启一个死循环
        while (true) {
            // 1. 通过 fds 持续的判断 Socket 中是否有数据可读
            StructPollfd[] pollFds = new StructPollfd[fds.size()];
            for (int i = 0; i < pollFds.length; ++i) {
                // 创建一个 StructPollfd 对象, 给相关属性赋值
                pollFds[i] = new StructPollfd();
                pollFds[i].fd = fds.get(i);
                pollFds[i].events = (short) POLLIN;
            }
            for (int i = pollFds.length - 1; i >= 0; --i) {
                if ((pollFds[i].revents & POLLIN) == 0) {
                    continue;
                }
                // 2. i == 0 表示 其他进程通过 Socket 与当前 Zygote 进程建立了连接 
                if (i == 0) {
                    // 2.1 创建了一个连接对象加入 peers 缓存
                    ZygoteConnection newPeer = acceptCommandPeer(abiList);
                    peers.add(newPeer);
                    // 2.2 从连接对象中获取文件描述符加入 fds 缓存
                    fds.add(newPeer.getFileDesciptor());
                } else {
                    // 3. i > 0 执行子进程孵化
                    try {
                        // 获取连接对象
                        ZygoteConnection connection = peers.get(i);
                        // 调用 ZygoteConnection.processOneCommand 孵化进程
                        final Runnable command = connection.processOneCommand(this);
                        if (mIsForkChild) {
                            .......
                            return command;
                        } else {
                            ......
                            // 孵化结束, 移除这个请求
                            if (connection.isClosedByPeer()) {
                                connection.closeSocket();
                                peers.remove(i);
                                fds.remove(i);
                            }
                        }
                    } catch (Exception e) {
                        ......
                    } finally {
                        ......
                    }
                }
            }
        }
    }
    
}
```
可以看到它最终会调用 ZygoteConnection.processOneCommand 孵化进程
```
class ZygoteConnection {

    Runnable processOneCommand(ZygoteServer zygoteServer) {
    
        String args[];
        Arguments parsedArgs = null;

        try {
            // 1. 读取参数列表
            args = readArgumentList();
            ......
        } catch (IOException ex) {
            ......
        }
        
        // 2. 将 arges 解析到 Arguments 对象中
        parsedArgs = new Arguments(args);
        // ...... 一系列的验参操作


        // 3. 调用 Zygote.forkAndSpecialize 来创建一个进程
        pid = Zygote.forkAndSpecialize(parsedArgs.uid, parsedArgs.gid, parsedArgs.gids,
                parsedArgs.runtimeFlags, rlimits, parsedArgs.mountExternal, parsedArgs.seInfo,
                parsedArgs.niceName, fdsToClose, fdsToIgnore, parsedArgs.startChildZygote,
                parsedArgs.instructionSet, parsedArgs.appDataDir);
        ......
         try {
            // 若 pid 为 0, 则说明当前方法是在新创建的子进程中执行的
            if (pid == 0) {
                ......
                // 处理进程创建成功之后的适宜
                return handleChildProc(parsedArgs, descriptors, childPipeFd,
                        parsedArgs.startChildZygote);
            }
            ......
        } finally {
            ......
        }
    }
    
}
```
ZygoteConnection.processOneCommand 的操作如下
- 从 Zygote 的 Socket 输入流中读取参数列表
- 将参数解析到 Argument 中
- 调用 Zygote.forkAndSpecialize 来创建子进程
  - 它最终会调用 fork() 来完成子进程的创建
- 子进程的创建会复制 Zygote 的地址空间, 因此会它也会接着 ZygoteConnection.processOneCommand 往下执行, 当 pid 为 0 时说明是子进程在调用
- 调用 handleChildProc 来处理应用进程的初始化操作

下面我们看看应用进程的启动流程

## 三. 应用进程的启动
接下来我们分析一下 handleChildProc 方法的操作
```
public class ZygoteConnection {

    private Runnable handleChildProc(Arguments parsedArgs, FileDescriptor[] descriptors,
            FileDescriptor pipeFd, boolean isZygote) {
        ......
        if (parsedArgs.invokeWith != null) {
            ......
        } else {
            if (!isZygote) {
                // 初始化子进程
                return ZygoteInit.zygoteInit(parsedArgs.targetSdkVersion, parsedArgs.remainingArgs,
                        null /* classLoader */);
            } else {
                ......
            }
        }
    }

}
```
可以看到这里直接调用了一个 ZygoteInit.zygoteInit 去执行子进程的初始化操作, 这个方式与 SystemServer 初始化时的流程操作是完全一致的

不同的是这个 ZygoteInit.zygoteInit 返回的 Runnable 中的 run 方法执行的是 ActivityThread 中的 main 函数

下面我们就看看应用进程的启动做了哪些操作

```
class ActivityThread {

    static volatile Handler sMainThreadHandler;  // set once in main()
    
    public static void main(String[] args) {
        // 1. 为这个线程准备 Looper
        Looper.prepareMainLooper();
        
        // 2. 创建了一个 ActivityThread 实体对象
        ActivityThread thread = new ActivityThread();
        
        // 3. 调用它的 attach 方法, 初始化绑定一些参数
        thread.attach(false, startSeq);
        
        // 4. 创建了 sMainThreadHandler, 即主线程的 Handler
        if (sMainThreadHandler == null) {
            sMainThreadHandler = thread.getHandler();
        }
        
        // 5. 开启 Looper 循环
        Looper.loop();
    }
    
    
}
```
main 函数中的操作比较简单的
- 首先是创建并且初始化 ActivityThread
- 然后开启一个主线程的 Looper, 用于处理其他线程的发送过来的消息
  - Looper 退出, 也就意味着应用程序退出了

关于 Looper 的相关操作, 属于线程间通信的内容, 我们到后面的章节再进行分析

下面我们关注一下 ActivityThread 的创建和初始化

```
class ActivityThread { 

    private final ResourcesManager mResourcesManager;
    
    ActivityThread() {
        // 创建资源管理对象
        mResourcesManager = ResourcesManager.getInstance();
    }

    final ApplicationThread mAppThread = new ApplicationThread();
    private static volatile ActivityThread sCurrentActivityThread;
    
    private void attach(boolean system, long startSeq) {
        sCurrentActivityThread = this;
        mSystemThread = system;
        // 1. 非系统应用
        if (!system) {
            ......
            RuntimeInit.setApplicationObject(mAppThread.asBinder());
            // 1. 通知 AMS, 应用进程创建完毕
            final IActivityManager mgr = ActivityManager.getService();
            try {
                mgr.attachApplication(mAppThread, startSeq);
            } catch (RemoteException ex) {
                ......
            }
            // 2. 创建 GC 监听器
            BinderInternal.addGcWatcher(new Runnable() {
                @Override public void run() {
                    if (!mSomeActivitiesChanged) {
                        return;
                    }
                    Runtime runtime = Runtime.getRuntime();
                    long dalvikMax = runtime.maxMemory();
                    long dalvikUsed = runtime.totalMemory() - runtime.freeMemory();
                    if (dalvikUsed > ((3*dalvikMax)/4)) {
                        if (DEBUG_MEMORY_TRIM) Slog.d(TAG, "Dalvik max=" + (dalvikMax/1024)
                                + " total=" + (runtime.totalMemory()/1024)
                                + " used=" + (dalvikUsed/1024));
                        mSomeActivitiesChanged = false;
                        try {
                            mgr.releaseSomeActivities(mAppThread);
                        } catch (RemoteException e) {
                            throw e.rethrowFromSystemServer();
                        }
                    }
                }
            });
        } 
        // 系统应用
        else {
            ......
            // 创建 Application
            try {
                mInstrumentation = new Instrumentation();
                mInstrumentation.basicInit(this);
                ContextImpl context = ContextImpl.createAppContext(
                        this, getSystemContext().mPackageInfo);
                mInitialApplication = context.mPackageInfo.makeApplication(true, null);
                mInitialApplication.onCreate();
            } catch (Exception e) {
                ......
            }
        }
        // 3. 为 ViewRootImpl 注入配置变更监听器
        ViewRootImpl.ConfigChangedCallback configChangedCallback
                = (Configuration globalConfig) -> {
            synchronized (mResourcesManager) {
                if (mResourcesManager.applyConfigurationToResourcesLocked(globalConfig,
                        null /* compat */)) {
                    updateLocaleListFromAppContext(mInitialApplication.getApplicationContext(),
                            mResourcesManager.getConfiguration().getLocales());
                    // This actually changed the resources! Tell everyone about it.
                    if (mPendingConfiguration == null
                            || mPendingConfiguration.isOtherSeqNewer(globalConfig)) {
                        mPendingConfiguration = globalConfig;
                        sendMessage(H.CONFIGURATION_CHANGED, globalConfig);
                    }
                }
            }
        };
        ViewRootImpl.addConfigCallback(configChangedCallback);
    }
    
}
```
ActivityThread 的创建非常的简单, 即创建了一个 ResourceManager 对象保存在成员变量中, 其 attach 方法中处理的事务如下
- 非系统应用
  - 通知 AMS 应用进程启动完毕, 可以继续执行后续操作了
  - 创建 GC 监听器
- 若为系统应用
  - 创建 Application 实例对象
- 为 ViewRootImpl 注入配置变更监听器

至此一次应用进程的创建就完成了

## 总结

![应用进程创建的流程图](https://i.loli.net/2019/11/20/83EGwkQq6YC5uMN.png)

通过 Zygote 创建应用进程还是非常清晰的, 其主要步骤如下
- 请求发起
  - 与 Zygote 进程建立 Socket 连接
  - 向 Zygote 进程发送创建子进程的请求
- 子进程的创建
  - Zygote 从 Socket 中获取进程创建请求数据
  - fork 子进程
- 应用进程的初始化和启动
  - Native 层的初始化
    - 通过 AppRuntime.onZygoteInit 函数, 启动 Binder 驱动线程池的主线程, 监听 Binder 驱动的跨进程通信请求
  - Java 层的初始化
    - 创建 SystemServer 的 main 方法的 Runnable 对象
  - 应用进程的启动
    - 创建并初始化 ActivityThread
    - **通过 attachApplication 通知 AMS 继续执行任务**
    - 创建主线程消息循环

好的应用进程到这里就创建和初始化完毕了, 下一篇文章我们会通过 attachApplication 回到了 AMS 中, 看看如何启动当前进程的 Activity
