---
layout: article
title: "Android 系统架构 —— 数据通信篇 之 ServiceManager 进程的启动"
key: "Android 系统架构 —— 数据通信篇 之 ServiceManager 进程的启动" 
tags: AndroidFramework
aside:
  toc: true
---

## 前言
我们之前在学习系统进程启动的过程中, 知道 ServiceManager 服务管理进程是通过 init 进程 fork 出来的, 但是并没有立即分析

这是因为 ServiceManager 与 Binder 驱动是强关联的, 因此在了解了 Binder 驱动之后, 再回过头来学习 ServiceManager 会更加清晰, 同时也是检验学习成果的一种

## ServiceManager 的作用
- Service Manager 是 Binder 进程间通信的核心组件之一
- 它扮演着 Binder 进程间通信机制的上下文管理者的角色
- 负责管理系统中的 Service 组件, 并且向 Client 端提供获取 Service 代理对象的服务

<!--more-->

## 一. 启动流程
该程序的入口函数 main 实现在 service_manager.c 中
```
// frameworks/base/cmds/servicemanager/service_manager.c
int main(int argc, char **argv) {
    struct binder_state *bs;
    void *svcmgr = BINDER_SERVICE_MANAGER;
    // 打开设备文件
    bs = binder_open(128*1024);
    // 将自己注册为 Binder 驱动的上下文管理者
    if (binder_become_context_manager(bs)) {
        return -1;
    }
    svcmgr_handle = svcmgr;
    // 循环等待和处理 Client 进程的通信请求
    binder_loop(bs, svcmgr_handler);
    return 0;
}
```
可见 service_manager 的主函数中主要做了三件事情
1. 调用 binder_open 打开 binder 设备文件 /dev/binder, 并且将其映射到本进程的地址空间, 返回一个 binder_state 结构体
2. 调用 binder_become_context_manager 将自己注册成为一个 Binder 进程间通信的上下文管理者
3. 调用函数 binder_loop 来循环等待和处理 Client 进程的通信请求

## 二. 打开 Binder 驱动设备文件
```
// frameworks/base/cmds/servicemanager/binder.c
struct binder_state *binder_open(size_t mapsize)
{
    struct binder_state *bs;
    struct binder_version vers;
    // 在堆内存中创建了 binder_state 的实例
    bs = malloc(sizeof(*bs));
    if (!bs) {
        errno = ENOMEM;
        return NULL;
    }
    // 1. 调用 open 函数打开 Binder 设备文件
    bs->fd = open("/dev/binder", O_RDWR | O_CLOEXEC);
    if (bs->fd < 0) {
        goto fail_open;
    }

    if ((ioctl(bs->fd, BINDER_VERSION, &vers) == -1) ||
        (vers.protocol_version != BINDER_CURRENT_PROTOCOL_VERSION)) {
        goto fail_open;
    }
    // 将给进程分配的内核缓冲区大小记录到 binder_state 结构体对象中
    bs->mapsize = mapsize;
    // 2. 调用函数 mmap, 获取内核缓冲区用户虚拟地址空间首地址
    bs->mapped = mmap(NULL, mapsize, PROT_READ, MAP_PRIVATE, bs->fd, 0);
    if (bs->mapped == MAP_FAILED) {
        fprintf(stderr,"binder: cannot map device (%s)\n",
                strerror(errno));
        goto fail_map;
    }
    // 返回这个 binder_state 这个结构体对象
    return bs;

fail_map:
    close(bs->fd);
fail_open:
    free(bs);
    return NULL;
}
```
可见 ServiceManager 的打开设备文件的操作非常简单
- 调用 open 函数打开 Binder 设备文件
- 调用函数 mmap, 获取内核缓冲区用户虚拟地址空间首地址
- 将给进程分配的内核缓冲区大小记录到 binder_state 结构体对象中

关于 Binder 驱动设备文件的 open 和 mmap 系统调用的实现, 我们在上一篇文章中已经仔细分析过了, 这里我们就不赘述了

下面我们看看 ServiceManager 是如何将自己注册成上下文管理者的

## 三. 注册为 Binder 的上下文管理者
```
// frameworks/base/cmds/servicemanager/binder.c
int binder_become_context_manager(struct binder_state *bs)
{
    return ioctl(bs->fd, BINDER_SET_CONTEXT_MGR, 0);
}
```
可以看到注册上下文管理者的函数中, 调用了 ioctl 系统调用
- BINDER_SET_CONTEXT_MGR 为 IO 控制命令
  - 这个标记位代表将当前进程注册为 binder context manager 即 Binder 的上下文管理者

接下来了解一下, Binder 内核驱动中对这个 IO 控制命令做了哪些处理
```
// Binder 通信上下文管理者的在 Binder 内核驱动中的 Binder 实体对象
static struct binder_node *binder_context_mgr_node;
// 描述了注册了 Binder 通信上下文管理者的有效用户 ID
static struct binder_context_mgr_uid = -1;

static long binder_ioctl(struct file *filp, unsigned int cmd, unsigned long arg) {
    // 获取当前进程的 binder 线程, 没有则创建一个
    thread = binder_get_thread(proc);
    
    switch(cmd) {
        ......
        case BINDER_SET_CONTEXT_MGR:
            // 说明 Binder 上下文管理者已经注册过了
            if (binder_context_mgr_node != NULL) {
                goto error;
            }
            // 说明 Binder 上下文管理者已经注册过了
            if (binder_context_mgr_uid != -1) {
                goto error;
            } else {
                // 经过一系列验证之后, 给当前进程创建其对应的 binder 实体对象保存在全局的 binder_context_mgr_node 变量中
                binder_context_mgr_node = binder_new_node(proc, NULL, NULL);
            }
            ......
            break;
    } 
}
```
Binder 内核驱动中针对 BINDER_SET_CONTEXT_MGR 这个控制码, 主要做了以下操作
- 将为这个请求成为 Binder 上下文管理者的进程创建其对应的 binder 实体对象
- 保存在内核驱动的静态变量 binder_context_mgr_node 中

## 四. 循环等待处理 Client 进程间的请求
```
// frameworks/base/cmds/servicemanager/binder.c
void binder_loop(struct binder_state *bs, binder_handler func)
{
    int res;
    struct binder_write_read bwr;
    uint32_t readbuf[32];

    bwr.write_size = 0;
    bwr.write_consumed = 0;
    bwr.write_buffer = 0;
    // BC_ENTER_LOOPER: 控制位的含义是, 将当前线程注册成为 Binder 线程
    // 以便 Binder 驱动程序可以将进程间的通信请求分发给它处理
    readbuf[0] = BC_ENTER_LOOPER;
    // 该函数通过 IO 控制命令将 readbuf 发送给 Binder 驱动程序, 通知其处理 readbuf 中的控制位
    binder_write(bs, readbuf, sizeof(uint32_t));
    // for 循环从 binder 驱动中获取需要处理的进程间通信请求
    for (;;) {
        bwr.read_size = sizeof(readbuf);
        bwr.read_consumed = 0;
        bwr.read_buffer = (uintptr_t) readbuf;
        // 通过 BINDER_WRITE_READ 控制位, 从 Binder 驱动中获取当前是否有新的进程间请求需要处理
        res = ioctl(bs->fd, BINDER_WRITE_READ, &bwr);
        if (res < 0) {
            ALOGE("binder_loop: ioctl failed (%s)\n", strerror(errno));
            break;
        }
        // 处理进程间的请求
        res = binder_parse(bs, 0, (uintptr_t) readbuf, bwr.read_consumed, func);
        if (res == 0) {
            ALOGE("binder_loop: unexpected reply?!\n");
            break;
        }
        if (res < 0) {
            ALOGE("binder_loop: io error %d %s\n", res, strerror(errno));
            break;
        }
    }
}
```
binder_loop 主要做了以下几件事情
- 通过 readbuf 记录 BC_ENTER_LOOPER 控制码, 将当前线程注册成为 Binder 线程, 以便 Binder 驱动程序可以将进程间的通信请求分发给它处理 
- 通过 binder_write 将 readbuf 控制码发送给 binder 驱动处理, 其内部同样是使用 ioctrl 与 binder 驱动通信
- for 循环不断的从 binder 驱动中获取新的进程间通信请求

### binder 驱动注册 looper 线程
接下来看看 binder_write 方法的实现
```
// frameworks/base/cmds/servicemanager/binder.c
int binder_write(struct binder_state *bs, void *data, size_t len)
{
    struct binder_write_read bwr;
    int res;

    bwr.write_size = len;
    bwr.write_consumed = 0;
    // 将数据存储在 write_buffer 中, 即 BC_ENTER_LOOPER 这个控制码
    bwr.write_buffer = (uintptr_t) data;
    bwr.read_size = 0;
    bwr.read_consumed = 0;
    bwr.read_buffer = 0;
    // 调用 ioctl 与 binder 驱动通信, 请求码为 BINDER_WRITE_READ
    res = ioctl(bs->fd, BINDER_WRITE_READ, &bwr);
    if (res < 0) {
        fprintf(stderr,"binder_write: ioctl failed (%s)\n",
                strerror(errno));
    }
    return res;
}
```
可见真正用于和 Binder 内核驱动交互的请求码为 BINDER_WRITE_READ, 接下来看看 binder 驱动做了哪些处理

```
static long binder_ioctl(struct file *filp, unsigned int cmd, unsigned long arg) {
    // 获取当前进程的 binder 线程, 没有则创建一个
    thread = binder_get_thread(proc);
    
    switch(cmd) {
        ......
        case BINDER_WRITE_READ:
            ......
            if (bwr.write_size > 0) {
                // 可见这里将 BC_ENTER_LOOPER 请求码转发给了 binder_thread_write 函数
                ret = binder_thread_write(proc, thread, (void __user *)bwr.write_buffer, bwr.write_size, &bwr.read_consumed);
            }
            ......
            break;
    } 
}

int binder_thread_write(......) {
    while(...) {
        switch(cmd) {
            case BC_ENTER_LOOPER:
                // 这里将这个线程注册成为了 looper 线程, 至此 Binder 进行间的通信请求便会交由这个线程处理
                thread->looper |= BINDER_LOOPER_STATE_ENTERED;
            break;
        }
    }
}
```
至此, ServiceManager 的主线程便可以接收到 Binder 驱动发送的通信请求了

## 总结
ServiceManager 为服务管理进程, 它是 Binder 驱动的上下文管理者, 其启动流程主要分为如下几步
- 打开 Binder 驱动
  - open/mmap
- 注册为当前进程为 Binder 上下文的管理者
- 调用 binder_loop 监听其他进程的通信请求

ServiceManager 启动的准备工作已经完成了, 后面的文章我们会通过一次 Binder 跨进程通信, 来将之前学习的知识点串联起来