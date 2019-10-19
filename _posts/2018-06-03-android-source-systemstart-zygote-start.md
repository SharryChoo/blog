---
layout: article
title: "Android 系统架构 —— 系统启动篇 之 Zygote 进程的启动"
key: "Android 系统架构 —— 系统启动篇 之 Zygote 进程的启动" 
tags: AndroidFramework
aside:
  toc: true
---

## 前言
通过 Init 进程启动的分析, 我们了解到它会读取 init.rc 脚本文件, zygote 进程启动相关的配置便是在此定义的, 这里我们系统的了解一下

## Zygote 进程的作用
顾明思议, Zygote 是孵化器的意思, 在 Android 系统中, 所有的应用程序进程以及用来运行系统关键服务的 System 进程, 都是由 Zygote 创建的, **它就是孵化器进程**

<!--more-->

## Zygote 启动脚本
这里选取 init.rc 分析
```
// system/core/rootdir/init.rc
......
import /init.${ro.zygote}.rc

// system/core/rootdir/init.zygote32.rc
service zygote /system/bin/app_process -Xzygote /system/bin --zygote --start-system-server
    ......
    socket zygote stream 660 root system
```
简单的分析一下这个脚本文件
- **service zygote**: 表示 Zygote 进程是以服务的形式启动的
- **/system/bing/app_process**: 表示 Zygote 进程的应用程序文件
- 后面四个选项表示 Zygote 进程的启动参数
  -  **--start-system-server**: 表示在 Zygote 进程启动过程中, 需要启动 System 进程
- **socket zygote stream 660 root system**: 表示 Zygote 进程启动过程中需要创建一个名为 zygote 的 Socket
  - 这个 Socket 的用于执行进程间的通信
  - 访问权限为 660 root system, 即所有用户都可以对它进行读写

好的, 我们关注到,  Zygote 的入口在 **/system/bing/app_process** 目录中, 接下来分析它的启动流程

## 启动流程
**/system/bing/app_process** 目录下的 Zygote 进程入口 main 函数在 app_main.cpp 中
### 一) app_main.cpp 的 main 函数
```
// frameworks/base/cmds/app_process/app_main.cpp
int main(int argc, char* const argv[])
{
    AppRuntime runtime(argv[0], computeArgBlockSize(argc, argv));
    if (zygote) {
        // 若是 zygote 则调用了 AppRuntime 的 start 方法, AppRuntime 继承 AndroidRuntime
        runtime.start("com.android.internal.os.ZygoteInit", args, zygote);
    } else if (className) {
        ......
    } else {
        ......
    }
}
```
可见 app_main.cpp 中的 main 方法, 将 Zygote 的启动分发给了 AndroidRuntime.start 中, 我们接着往下分析
```
// frameworks/base/core/jni/AndroidRuntime.cpp
void AndroidRuntime::start(const char* className, const Vector<String8>& options, bool zygote)
{
    ......
    // 1. 创建一个 ART 虚拟机
    JniInvocation jni_invocation;
    jni_invocation.Init(NULL);
    JNIEnv* env;
    if (startVm(&mJavaVM, &env, zygote) != 0) {
        return;
    }
    onVmCreated(env);

    // 2. 构建用于调用 java main() 方法的字符串数组
    jclass stringClass;    // 描述 String.class
    jobjectArray strArray; // 描述 String[] 数组
    jstring classNameStr;  // 描述字符串对象
    // 创建 java 字符串数组
    stringClass = env->FindClass("java/lang/String");
    strArray = env->NewObjectArray(options.size() + 1, stringClass, NULL);
    // 从 app_main.cpp 中的 main 函数可知, className 为 "com.android.internal.os.ZygoteInit"
    classNameStr = env->NewStringUTF(className);
    // 2.2 将 ZygoteInit 的全限定类名导入
    env->SetObjectArrayElement(strArray, 0, classNameStr);
    // 2.3 将 options 中的参数导入
    for (size_t i = 0; i < options.size(); ++i) {
        jstring optionsStr = env->NewStringUTF(options.itemAt(i).string());
        assert(optionsStr != NULL);
        env->SetObjectArrayElement(strArray, i + 1, optionsStr);
    }

    // 3. 调用 ZygoteInti 中的 main 方法
    char* slashClassName = toSlashClassName(className != NULL ? className : "");
    jclass startClass = env->FindClass(slashClassName);
    if (startClass == NULL) {
        ......
    } else {
        jmethodID startMeth = env->GetStaticMethodID(startClass, "main",
            "([Ljava/lang/String;)V");
        if (startMeth == NULL) {
            ......
        } else {
            // 调用了 ZygoteInti.main() 方法
            env->CallStaticVoidMethod(startClass, startMeth, strArray);

    }
    ......
}
```
好的, 可以看到 Zygote 进程的启动调用了 AppRuntime.start 方法这个, 这个方法做了如下几件事情
- 创建 ART 虚拟机
- 构建用于调用 java main() 方法的字符串数组
  - 将 ZygoteInit 的全限定类名导入
  - 将 options 中的参数导入
- 调用 ZygoteInit 中的 main 方法

好的, 至此 Zygote 进程的启动工作就转移到了 Java 中了, 我们看看 ZygoteInit 中做了什么

### 二) ZygoteInit 的 main 方法
```
// frameworks/base/core/java/com/android/internal/os/ZygoteInit.java
public class ZygoteInit {
    
    public static void main(String argv[]) {
        ZygoteServer zygoteServer = new ZygoteServer();
        // 将 Zygote 标记为启动了
        ZygoteHooks.startZygoteNoThreadCreation();
        final Runnable caller;
        try { 
            // 用于判断在 Zygote 进程启动之后, 是否启动 SystemService 进程
            boolean startSystemServer = false;
            String socketName = "zygote";
            String abiList = null;
            boolean enableLazyPreload = false;
            for (int i = 1; i < argv.length; i++) {
                // 若 argv 中包含 "start-system-server" 表示紧接着启动 SystemService 进程
                if ("start-system-server".equals(argv[i])) {
                    startSystemServer = true;
                } else if ("--enable-lazy-preload".equals(argv[i])) {
                    enableLazyPreload = true;
                } else if (argv[i].startsWith(ABI_LIST_ARG)) {
                    abiList = argv[i].substring(ABI_LIST_ARG.length());
                } else if (argv[i].startsWith(SOCKET_NAME_ARG)) {
                    socketName = argv[i].substring(SOCKET_NAME_ARG.length());
                } else {
                    throw new RuntimeException("Unknown command line argument: " + argv[i]);
                }
            }
            // 1. 注册 Zygote 进程的 Socket, 用于监听是否有 fork 请求
            zygoteServer.registerServerSocketFromEnv(socketName);
            // 2. 创建 SystemService 系统服务进程
            if (startSystemServer) {
                Runnable r = forkSystemServer(abiList, socketName, zygoteServer);
                // r == null 说明当前是 Zygote 进程执行该 main 方法
                // r != null 说明当前是 SystemService 进程执行该 main 方法
                if (r != null) {
                    r.run();
                    return++;++
                }
            }
            // 3. 开启 Zygote 进程的死循环, 其他进程发送的进程创建请求
            caller = zygoteServer.runSelectLoop(abiList);
        } catch (Throwable ex) {
            ......
        } finally {
            // 关闭
            zygoteServer.closeServerSocket();
        }
        ......
    }
    
}
```
可见 ZygoteInit 的 main 方法主要做了如下几件事情
- 调用 ZygoteServer.registerServerSocketFromEnv() 方法, 注册 Zygote 进程的 Socket, 用于监听是否有 fork 请求
  - 该 Socket 负责与其他进程通信, 如: 接收 AMS 的进程创建请求 
-  调用 ZygoteInit.forkSystemServer() 创建 SystemService 系统服务进程
- 调用 ZygoteServer.runSelectLoop() 开启 Zygote 进程的死循环, 其他进程发送的进程创建请求

接下来逐一分析

#### 1. 创建 Zygote 进程的 Socket
```
// frameworks/base/core/java/com/android/internal/os/ZygoteServer.java
class ZygoteServer {

    private static final String ANDROID_SOCKET_PREFIX = "ANDROID_SOCKET_";
    
    /**
     * Listening socket that accepts new server connections.
     */
    private LocalServerSocket mServerSocket;
   
    void registerServerSocketFromEnv(String socketName) {
        if (mServerSocket == null) {
            int fileDesc;
            final String fullSocketName = ANDROID_SOCKET_PREFIX + socketName;
            try {
                // 获取环境变量的值 ANDROID_SOCKET_zygote
                String env = System.getenv(fullSocketName);
                // 转化为一个文件描述符
                fileDesc = Integer.parseInt(env);
            } catch (RuntimeException ex) {
                throw new RuntimeException(fullSocketName + " unset or invalid", ex);
            }

            try {
                // 创建了一个文件描述对象
                FileDescriptor fd = new FileDescriptor();
                fd.setInt$(fileDesc);
                // 以 fd 为参, 创建了一个 LocalServerSocket 实例对象
                mServerSocket = new LocalServerSocket(fd);
            } catch (IOException ex) {
                ......
            }
        }
    }
    
}
```
好的可以看到创建 Zygote 进程的 Socket 最终 new 了一个 LocalServerSocket, 并将它保存在 ZygoteServer 的成员变量 mServerSocket 中, 用于监听该端口的 Socket 请求<br>

#### 2. 启动 SystemService 进程
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

这不是本次分析的重点, 我们接着往下看

#### 3. Zygote 进程的循环等待其他进程的连接请求
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
好的, 可见 ZygoteServer 中, 会开启一个死循环
- 通过 fds 持续的判断 Socket 中是否有数据可读
- **i == 0**: 表示其他进程通过 Socket 与当前 Zygote 进程建立了连接 
  - 创建了一个连接对象加入 peers 缓存
  - 从连接对象中获取文件描述符加入 fds 缓存
- **i > 0**: 执行子进程孵化
   - 调用 ZygoteConnection.processOneCommand 孵化进程   

### 回顾
![Zygote 启动流程图](https://i.loli.net/2019/10/19/vEPkoZRIGQhr19j.png)

## 总结
至此 Zygote 的进程启动分析就结束了, 从 ZygoteInit.main 中可以很清晰的看到 Zygote 进程的职责, 这里总结一下
- Zygote 是在 init 进程启动过程中启动的
- 调用 app_main.cpp 的 main 函数
- 调用 ZygoteInit 的 main 方法

**Zygote 进程的职责如下**
- 创建 Android 虚拟机 Dalvik/ART
- 孵化 SystemServer 进程
- 通过监听 Socket 端口, 处理进程的 fork 请求