---
layout: article
title: "Android 系统架构 —— 系统启动篇之 Init 进程的启动"
key: "Android 系统架构 —— 系统启动篇之 Init 进程的启动" 
tags: AndroidFramework
aside:
  toc: true
---

## 前言
我们知道 Android 是基于 Linux 内核的, 因此 Android 系统的启动与 Linux 内核类似

当 bootloader 中的 kernal.img 选择了操作系统之后, 便会进入 Linux 内核的初始化流程了, 内核启动的初始化由 start_kernel() 开始, 在 init/main.c 文件中, 这是 Linux 的基础知识, 这里就不赘述了

Linux 的启动的参与的进程如下
- **0 号进程**
  - 所有进程的祖先, 这是 Linux 唯一一个没有通过 fork 或者 kernel_thread 产生的进程
- **1 号进程**
  - 为用户进程, 是所有用户态进程的祖先
- **2 号进程**
  - 为内核进程, 负责内核态所有进程的管理和调度, 是所有 Linux 内核进程的祖先

1 号进程也称之为 Init 进程, 是我们需要重点关心的对象, 接下来便着重分析一下 init 进程的启动流程

<!--more-->

## Init 进程启动
Init 进程启动的文件在 /system/core/init/init.cpp 目录下, 我们看看它的 main 方法
```
// /system/core/init/init.cpp
int main(int argc, char** argv) {
    ......
    // 1. 挂载文件系统, 创建驱动目录, 创建驱动设备文件
    if (is_first_stage) {
        boot_clock::time_point start_time = boot_clock::now();
        
        umask(0);

        clearenv();
        setenv("PATH", _PATH_DEFPATH, 1);
        // 挂载 tmpfs 文件系统
        mount("tmpfs", "/dev", "tmpfs", MS_NOSUID, "mode=0755");
        // 创建驱动设备目录
        mkdir("/dev/pts", 0755);
        mkdir("/dev/socket", 0755);
        // 挂载 devpts文件系统
        mount("devpts", "/dev/pts", "devpts", 0, NULL);
        // 挂载 proc 文件系统
        mount("proc", "/proc", "proc", 0, "hidepid=2,gid=" MAKE_STR(AID_READPROC));
        mount("sysfs", "/sys", "sysfs", 0, NULL);
        mount("selinuxfs", "/sys/fs/selinux", "selinuxfs", 0, NULL);
        // 创建驱动设备文件
        mknod("/dev/kmsg", S_IFCHR | 0600, makedev(1, 11));
        ......
        
        // 初始化内核log
        InitKernelLogging(argv);
        ......
    }

    ......

    // 初始化信号的处理
    sigchld_handler_init();
    
    ......
    
    // 2. 解析启动脚本
    ActionManager& am = ActionManager::GetInstance();
    ServiceList& sm = ServiceList::GetInstance();
    LoadBootScripts(am, sm);
    ......

    while (true) {
        ......
        if (!(waiting_for_prop || Service::is_exec_service_running())) {
            // 3. 执行脚本指令
            am.ExecuteOneCommand();
        }
        // 循环等待事件的发生
        epoll_event ev;
        int nr = TEMP_FAILURE_RETRY(epoll_wait(epoll_fd, &ev, 1, epoll_timeout_ms));
        if (nr == -1) {
            PLOG(ERROR) << "epoll_wait failed";
        } else if (nr == 1) {
            ((void (*)()) ev.data.ptr)();
        }
    }
    return 0;
}
```
从 Init 进程的 main 函数中可以了解到, 它主要职责如下
- 挂载文件系统, 创建驱动目录和驱动设备文件
- 解析启动脚本
- 执行脚本指令

关于文件系统挂载和驱动设备的文件的创建, 都是 Linux 内核的知识, 这里不做探讨, 主要看看解析启动脚本和脚本执行的相关代码

## 解析启动脚本
从 main 函数的分析可知, 启动脚本的解析主要由 LoadBootScripts 函数完成, 这里看看它的实现
```
// /system/core/init/init.cpp
static void LoadBootScripts(ActionManager& action_manager, ServiceList& service_list) {
    Parser parser = CreateParser(action_manager, service_list);

    std::string bootscript = GetProperty("ro.boot.init_rc", "");
    if (bootscript.empty()) {
        // 解析 /init.rc 脚本文件
        parser.ParseConfig("/init.rc");
        ...
    } else {
        parser.ParseConfig(bootscript);
    }
}
```
可以看到 LoadBootScripts 中主要解析了 init.rc 的脚本文件, 它的定义在 /system/core/rootdir/init.rc 中, 我们看看它的实现
```
import /init.environ.rc
import /init.usb.rc
import /init.${ro.hardware}.rc
import /vendor/etc/init/hw/init.${ro.hardware}.rc
import /init.usb.configfs.rc
import /init.${ro.zygote}.rc

on early-init
on init
on late-init

......
# 启动服务管理
on post-fs
    load_system_props
    # start essential services
    start logd
    start servicemanager
    start hwservicemanager
    start vndservicemanager

......
# 启动 zygote
on zygote-start && property:ro.crypto.state=unencrypted
    # A/B update verifier that marks a successful boot.
    exec_start update_verifier_nonencrypted
    start netd
    start zygote
    start zygote_secondary

......
# 启动 surfaceflinger
on property:vold.decrypt=trigger_restart_framework
    stop surfaceflinger
    start surfaceflinger
    # A/B update verifier that marks a successful boot.
    exec_start update_verifier
    class_start main
    class_start late_start

```
[rc 的语法参考](http://gityuan.com/2016/02/05/android-init/) 

可以看到 init.rc 中出现了一些非常重要关键字
- **servicemanager**
  - 服务管理进程, 用于向客户端提供发布的服务 
- **zygote**
  - Zygote 进程, 用于孵化 应用进程 和 系统服务 进程
- **surfaceflinger**
  - 用于执行 UI 渲染

这三个进程在整个 Android 系统中, 都起到了非常重要的作用, 其启动的流程将在后续一一展开

## 执行脚本指令
脚本的执行是通过 ActionManager.ExecuteOneCommand 触发的, 这里我们看看它的调用链
```
// /system/core/init/action_manager.cpp
void ActionManager::ExecuteOneCommand() {
    // Loop through the event queue until we have an action to execute
    while (current_executing_actions_.empty() && !event_queue_.empty()) {
        for (const auto& action : actions_) {
            if (std::visit([&action](const auto& event) { return action->CheckEvent(event); },
                           event_queue_.front())) {
                current_executing_actions_.emplace(action.get());
            }
        }
        event_queue_.pop();
    }
    if (current_executing_actions_.empty()) {
        return;
    }
    // 从脚本中获取可执行的动作
    auto action = current_executing_actions_.front();
    ...
    // 执行脚本
    action->ExecuteOneCommand(current_command_);
    // 若为一次性动作, 则执行结束之后, 从队列中移除
    ++current_command_;
    if (current_command_ == action->NumCommands()) {
        current_executing_actions_.pop();
        current_command_ = 0;
        if (action->oneshot()) {
            auto eraser = [&action](std::unique_ptr<Action>& a) { return a.get() == action; };
            actions_.erase(std::remove_if(actions_.begin(), actions_.end(), eraser));
        }
    }
}
```
可以看到脚本的执行动作由 Action.ExecuteOneCommand 发起, 我们接着追踪
```
// /system/core/init/action.cpp
void Action::ExecuteOneCommand(std::size_t command) const {
    // We need a copy here since some Command execution may result in
    // changing commands_ vector by importing .rc files through parser
    Command cmd = commands_[command];
    ExecuteCommand(cmd);
}

void Action::ExecuteCommand(const Command& command) const {
    ......
    // 执行脚本指令
    auto result = command.InvokeFunc(subcontext_);
    // 处理失败动作
    .....
}

Result<Success> Command::InvokeFunc(Subcontext* subcontext) const {
    ......
    return RunBuiltinFunction(func_, args_, kInitContext);
}

Result<Success> RunBuiltinFunction(const BuiltinFunction& function,
                                   const std::vector<std::string>& args,
                                   const std::string& context) {
    // 构建参数列表
    auto builtin_arguments = BuiltinArguments(context);
    builtin_arguments.args.resize(args.size());
    builtin_arguments.args[0] = args[0];
    for (std::size_t i = 1; i < args.size(); ++i) {
        if (!expand_props(args[i], &builtin_arguments.args[i])) {
            return Error() << "cannot expand '" << args[i] << "'";
        }
    }
    // 执行指令处理函数
    return function(builtin_arguments);
}
```
好的, 可看到最终这条指令会交由指令处理函数来执行, 执行处理函数定义在 [/system/core/init/builtins.cpp](http://androidxref.com/9.0.0_r3/xref/system/core/init/builtins.cpp) 中, 这里就不展开叙述了, 感兴趣可以点击查看

## 总结
从 Init 进程的启动主要的职责如下
- 挂载文件系统, 创建驱动目录和驱动设备文件
- 解析启动脚本
- 执行脚本指令
  - 启动服务管理进程
  - 启动图像渲染进程
  - 启动 Zygote 进程

![启动进程树](https://i.loli.net/2019/10/19/abnUSCpxGR7m6zZ.png)