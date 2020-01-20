---
title: Android 系统架构 —— ServiceManager 进程的启动
permalink: android-source/servicemanager-process-start
key: android-source-servicemanager-process-start
tags: AndroidFramework
sidebar:
  nav: android-source
---
## 前言
我们之前在学习 Binder 驱动的过程中就提及过 ServiceManager 是 Binder 驱动的上下文管理者

从 init 进程启动过程中我们知道 **ServiceManager 服务管理进程是通过 init 进程 fork 出来的**, 这里我们就分析一下 ServiceManager 的启动流程

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
    // 1. 打开设备文件
    bs = binder_open(128*1024);
    // 2. 将自己注册为 Binder 驱动的上下文管理者
    if (binder_become_context_manager(bs)) {
        return -1;
    }
    svcmgr_handle = svcmgr;
    // 3. 循环等待和处理 Client 进程的通信请求
    binder_loop(bs, svcmgr_handler);
    return 0;
}
```
可见 service_manager 的主函数中主要做了三件事情
- 调用 **binder_open** 打开 binder 设备文件 "/dev/binder", 返回一个 binder_state 结构体
- 调用 **binder_become_context_manager** 将自己注册成为一个 Binder 进程间通信的上下文管理者
- 调用函数 **binder_loop** 来循环等待和处理 Client 进程的通信请求

接下来就这三个函数, 一一分析, 先看看 binder_open 打开 binder 驱动设备文件做了什么

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
    // 1. 通过 open 系统调用, 打开 Binder 设备文件
    bs->fd = open("/dev/binder", O_RDWR | O_CLOEXEC);
    if (bs->fd < 0) {
        goto fail_open;
    }
    
    // 2. 通过 ioctl 系统调用, 获取 binder 版本号, 验证是否打开成功
    if ((ioctl(bs->fd, BINDER_VERSION, &vers) == -1) ||
        (vers.protocol_version != BINDER_CURRENT_PROTOCOL_VERSION)) {
        goto fail_open;
    }
    
    // 将给进程分配的内核缓冲区大小记录到 binder_state 结构体对象中
    bs->mapsize = mapsize;
    
    // 3. 通过 mmap 系统调用, 获取内核缓冲区用户虚拟地址空间首地址
    bs->mapped = mmap(NULL, mapsize, PROT_READ, MAP_PRIVATE, bs->fd, 0);
    ......
    
    // 返回这个 binder_state 这个结构体对象
    return bs;
    ......
}
```
可见 ServiceManager 的打开设备文件的操作非常简单
- 通过 open 系统调用打开 Binder 设备文件
- 通过 ioctl 系统调用, 获取 binder 版本号, 验证是否打开成功
- 调用 mmap 系统调用, 初始化当前进程的 binder 内核缓冲区
  - 由传参可知, 其大小为 128 * 1024, 即 128 kb, 也就是说 ServiceManager 进程最高支持传输 128 kb 的数据
- 将给进程分配的内核缓冲区大小记录到 binder_state 结构体对象中

关于 Binder 驱动设备文件的系统调用的实现, 我们在上一篇文章中已经仔细分析过了, 这里我们就不赘述了

下面我们看看 ServiceManager 是如何通过 binder_become_context_manager 将自己注册成上下文管理者的

## 三. 注册为 Binder 驱动的上下文管理者
```
// frameworks/base/cmds/servicemanager/binder.c
int binder_become_context_manager(struct binder_state *bs)
{
    return ioctl(bs->fd, BINDER_SET_CONTEXT_MGR, 0);
}
```
可以看到注册上下文管理者的函数中, 调用了 ioctl 系统调用, 其 IO 指令码为 **BINDER_SET_CONTEXT_MGR**

关于 BINDER_SET_CONTEXT_MGR 我们在上一篇文章分析 ioctl 的时候就已经提到过, 这里我们就具体的分析一下
```
// Binder 通信上下文管理者的在 Binder 内核驱动中的 Binder 实体对象
static struct binder_node *binder_context_mgr_node;
// 描述了注册了 Binder 通信上下文管理者的有效用户 ID
static struct binder_context_mgr_uid = -1;

static long binder_ioctl(struct file *filp, unsigned int cmd, unsigned long arg) {
    
    int ret;
	struct binder_proc *proc = filp->private_data;
	struct binder_thread *thread;
	unsigned int size = _IOC_SIZE(cmd);
	void __user *ubuf = (void __user *)arg;
	
	......
	
	// 1. 获取当前线程的 binder_thread 描述
	thread = binder_get_thread(proc);
	
	// 2.  处理 IOCTL 指令
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
                // 3. 经过一系列验证之后, 给当前进程创建其对应的 binder 实体对象保存在全局的 binder_context_mgr_node 变量中
                binder_context_mgr_node = binder_new_node(proc, NULL, NULL);
            }
            ......
            break;
    } 
}
```
可以看到 binder_ioctl 中首先通过 **binder_get_thread** 获取了当前线程的 binder_thread 的描述

对于 BINDER_SET_CONTEXT_MGR 指令码主要是通过 **binder_new_node** 创建一个 binder_node 对象, 并且保存在静态变量 binder_context_mgr_node 中
- 并且通过一系列 if 判断, 保证了 binder_context_mgr_node 的唯一性, 足以看出 ServiceManager 的特殊

接下来我们分别看看 binder_get_thread 和 binder_new_node 的具体实现

### 一) binder_get_thread 获取 binder_thread 结构体
```
// kernel/goldfish/drivers/staging/android/binder.c
static struct binder_thread *binder_get_thread(struct binder_proc *proc)
{
	struct binder_thread *thread = NULL;
	struct rb_node *parent = NULL;
	struct rb_node **p = &proc->threads.rb_node;
	// 1. 以当前线程的 pid 为标示, 获取红黑树上缓存的结构体
	while (*p) {
		parent = *p;
		thread = rb_entry(parent, struct binder_thread, rb_node);
		// 以当前线程的 pid 为标示, 获取红黑树上缓存的结构体
		if (current->pid < thread->pid)
			p = &(*p)->rb_left;
		else if (current->pid > thread->pid)
			p = &(*p)->rb_right;
		else
			break;
	}
	// 2. 若无对应的 binder_thread 描述, 则创建一个新 binder_thread 结构体
	if (*p == NULL) {
		thread = kzalloc(sizeof(*thread), GFP_KERNEL);
	    ......
	    // 2.1 注入数据
		thread->proc = proc;
		thread->pid = current->pid;
		// 2.2 初始化当前线程的任务队列
		init_waitqueue_head(&thread->wait);
		INIT_LIST_HEAD(&thread->todo);
		// 2.3 插入到 binder_proc 中的缓存
		rb_link_node(&thread->rb_node, parent, p);
		rb_insert_color(&thread->rb_node, &proc->threads);
		// 2.4 状态为不需要 looper
		thread->looper |= BINDER_LOOPER_STATE_NEED_RETURN;
	}
	return thread;
}
```
binder_get_thread 操作以当前线程 pid 为 key 尝试 binder_proc 的红黑树中找寻对应的结构体描述 binder_thread 对象
- 若不存在则创建一个 binder_thread 并且插入 binder_proc 的红黑树中缓存
- 刚创建完毕时的 looper 状态为 BINDER_LOOPER_STATE_NEED_RETURN

关于获取 binder_thread 结构体就看到这里, 下面我们看看 binder_new_node 创建 binder_node 的流程

### 二) binder_new_node 创建 binder_node
下面我们看看 binder_new_node 中做了哪些处理
```
// kernel/goldfish/drivers/staging/android/binder.c
static struct binder_node *binder_new_node(
                       struct binder_proc *proc,
					   binder_uintptr_t ptr,
					   binder_uintptr_t cookie)
{
    // 获取 binder_proc 中缓存的红黑树
	struct rb_node **p = &proc->nodes.rb_node;
	struct rb_node *parent = NULL;
	struct binder_node *node;

    // 1. 找寻合适的插入位置
	while (*p) {
		parent = *p;
		node = rb_entry(parent, struct binder_node, rb_node);
		if (ptr < node->ptr)
			p = &(*p)->rb_left;
		else if (ptr > node->ptr)
			p = &(*p)->rb_right;
		else
			return NULL;
	}
	
	// 2. 创建 binder_node 结构体
	node = kzalloc(sizeof(*node), GFP_KERNEL);
	......
	binder_stats_created(BINDER_STAT_NODE);
	// 3. 插入到红黑树中
	rb_link_node(&node->rb_node, parent, p);
	rb_insert_color(&node->rb_node, &proc->nodes);
	// 4. 初始化相关变量
	node->proc = proc;
	node->ptr = ptr;
	node->cookie = cookie;
	node->work.type = BINDER_WORK_NODE;
	INIT_LIST_HEAD(&node->work.entry);
	INIT_LIST_HEAD(&node->async_todo);
	return node;
}
```
binder_new_node 函数中的操作, 还是非常清晰, 即创建 binder_node 对象, 并且将其插入到 binder_proc 的红黑树中缓存

接下来我们看看 ServiceManager 的 binder_loop 是如何注册为 binder 驱动的 looper 线程的

## 四. 注册 binder 驱动 looper 线程
```
// frameworks/base/cmds/servicemanager/binder.c
void binder_loop(struct binder_state *bs, binder_handler func)
{
    int res;
    // 描述对 binder 驱动读写的结构体
    struct binder_write_read bwr;
    uint32_t readbuf[32];

    bwr.write_size = 0;
    bwr.write_consumed = 0;
    bwr.write_buffer = 0;
    // 描述一个 binder_write_read 的控制码 BC_ENTER_LOOPER
    readbuf[0] = BC_ENTER_LOOPER;
    
    // 1. 调用 binder_write 该函数, 通知 binder 驱动 binder_write_read 的控制码 BC_ENTER_LOOPER
    binder_write(bs, readbuf, sizeof(uint32_t));
    
    // 2. 死循环从 binder 驱动中获取需要处理的进程间通信数据并处理
    for (;;) {
        bwr.read_size = sizeof(readbuf);
        bwr.read_consumed = 0;
        bwr.read_buffer = (uintptr_t) readbuf;
        
        // 2.1 获取 binder 驱动返回的数据
        res = ioctl(bs->fd, BINDER_WRITE_READ, &bwr);
        ......
        // 2.2 处理进程间的请求
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
binder_loop 主要做了如下两件事件
- 调用 binder_write, 通知 binder 驱动处理 binder_write_read 的读写控制码 BC_ENTER_LOOPER
- 通过一个死循环, 不断的通过 ioctl 系统调用, 获取 binder 驱动写入到 binder_write_read 的数据
  - 通过 ioctl 的指令码 BINDER_WRITE_READ 获取其他进程发送过来的数据
  - 通过 binder_parse 处理请求数据
  
好的, 这里我们看到了 binder_write_read 结构体, 它的定义如下
```
// bionic/libc/kernel/uapi/linux/android/binder.h
struct binder_write_read {
  binder_size_t write_size;
  binder_size_t write_consumed;
  binder_uintptr_t write_buffer;
  binder_size_t read_size;
  binder_size_t read_consumed;
  binder_uintptr_t read_buffer;
};
```
**binder_write_read 的作用是配合 ioctl 系统调用, 向 binder 驱动写入数据, 和从 binder 驱动读取数据**

接下来我们便看看 binder 驱动是如何处理 binder_write_read 的读写控制码 BC_ENTER_LOOPER 的

### 一) 处理 BC_ENTER_LOOPER 读写指令
```
// frameworks/base/cmds/servicemanager/binder.c
int binder_write(struct binder_state *bs, void *data, size_t len)
{
    struct binder_write_read bwr;
    int res;
    bwr.write_size = len;
    bwr.write_consumed = 0;
    // 1. 将读写指令码 BC_ENTER_LOOPER 保存在 write_buffer 中
    bwr.write_buffer = (uintptr_t) data;
    bwr.read_size = 0;
    bwr.read_consumed = 0;
    bwr.read_buffer = 0;
    // 2. 调用 ioctl 与 binder 驱动通信, IO 指令码为 BINDER_WRITE_READ
    res = ioctl(bs->fd, BINDER_WRITE_READ, &bwr);
    ......
    return res;
}
```
下面我们看看 ioctl 是如何处理 BC_ENTER_LOOPER 的
```
// kernel/goldfish/drivers/staging/android/binder.c
static long binder_ioctl(struct file *filp, unsigned int cmd, unsigned long arg) {

    ret = wait_event_interruptible(binder_user_error_wait, binder_stop_on_user_error < 2);

    // 获取当前线程的 binder_thread 描述
    thread = binder_get_thread(proc);
    
    // 处理 BINDER_WRITE_READ 指令码
    switch(cmd) {
        ......
        case BINDER_WRITE_READ:
            ......
            if (bwr.write_size > 0) {
                // 通过 binder_thread_write 处理 BC_ENTER_LOOPER
                ret = binder_thread_write(proc, thread, (void __user *)bwr.write_buffer, bwr.write_size, &bwr.read_consumed);
            }
            ......
            break;
    }
}
```
binder_ioctl 的 binder_get_thread 我们在上面已经分析过了, 再次获取便不会重新创建 binder_thread 对象了

这里我们主要看看 **binder_thread_write** 是如何处理 **BC_ENTER_LOOPER** 指令的
```
// kernel/goldfish/drivers/staging/android/binder.c
int binder_thread_write(......) {
    while(...) {
        switch(cmd) {
            case BC_ENTER_LOOPER:
                // 这里将这个线程注册成为了 looper 线程, 至此 Binder 进行间的通信请求便会交由这个线程分发处理
                thread->looper |= BINDER_LOOPER_STATE_ENTERED;
            break;
        }
    }
}
```
binder_thread_write 中对读写指令码 BC_ENTER_LOOPER 的操作也非常简单, 即将当前线程的 binder_thread 的描述中的 looper flag 置为 BINDER_LOOPER_STATE_ENTERED, 表示当前线程即将进入 looper 死循环了

**如此一来, 当其他进程向当前进程发起跨进程通信请求时, binder 驱动便会通过向 looper 的 thread 中写入工作项的方式来唤醒 looper 线程来处理这次跨进程通信了**

### 二) 死循环处理 Binder 驱动返回数据
```
void binder_loop(struct binder_state *bs, binder_handler func)
{
    int res;
    // 描述对 binder 驱动读写的结构体
    struct binder_write_read bwr;
    .....
    // 2. 死循环从 binder 驱动中获取需要处理的进程间通信数据并处理
    for (;;) {
        ......
        // 2.1 获取 binder 驱动返回的数据
        res = ioctl(bs->fd, BINDER_WRITE_READ, &bwr);
        ......
        // 2.2 处理进程间的请求
        res = binder_parse(bs, 0, (uintptr_t) readbuf, bwr.read_consumed, func);
        ......
    }
}
```
我们先看看 ioctl 系统调用如何通过 BINDER_WRITE_READ 获取其他进程的数据的

#### 1. IO 指令码 BINDER_WRITE_READ
```
// kernel/goldfish/drivers/staging/android/binder.c
static long binder_ioctl(struct file *filp, unsigned int cmd, unsigned long arg)
{
	int ret;
	struct binder_proc *proc = filp->private_data;
	struct binder_thread *thread;
	unsigned int size = _IOC_SIZE(cmd);
	void __user *ubuf = (void __user *)arg;
	
	......
	
    // 获取一个当前线程的 binder_thread 结构体描述
	thread = binder_get_thread(proc);
	......
	case BINDER_WRITE_READ:
	    // 调用 binder_ioctl_write_read 处理 BINDER_WRITE_READ 指令
		ret = binder_ioctl_write_read(filp, cmd, arg, thread);
		if (ret)
			goto err;
		break;
	......
}
```
对于 BINDER_WRITE_READ 指令, ioctl 系统调用分发给了 **binder_ioctl_write_read** 函数去处理, 下面我们看看它的实现
```
// kernel/goldfish/drivers/staging/android/binder.c
static int binder_ioctl_write_read(struct file *filp,
				unsigned int cmd, unsigned long arg,
				struct binder_thread *thread)
{
	int ret = 0;
	// 获取当前进程的 binder 驱动上下文
	struct binder_proc *proc = filp->private_data;
	unsigned int size = _IOC_SIZE(cmd);
	void __user *ubuf = (void __user *)arg;
	struct binder_write_read bwr;
	......
	// 1. 拷贝用户空间传递来的数据到 bwr 结构体中
	if (copy_from_user(&bwr, ubuf, sizeof(bwr))) {
		ret = -EFAULT;
		goto out;
	}
	// 2. 处理用户空间写入了数据的情况
	if (bwr.write_size > 0) {
	    // 调用 binder_thread_write 处理数据写入
		ret = binder_thread_write(proc, thread,
					  bwr.write_buffer,
					  bwr.write_size,
					  &bwr.write_consumed);
		......
	}
	// 3. 处理用户空间读取数据的情况
	if (bwr.read_size > 0) {
	    // 调用 binder_thread_read 函数, 读取数据
		ret = binder_thread_read(proc, thread, bwr.read_buffer,
					 bwr.read_size,
					 &bwr.read_consumed,
					 filp->f_flags & O_NONBLOCK);
		......
		// 若 proc 中仍然存在等待唤醒的线程, 则通知其唤醒
		if (!list_empty(&proc->todo))
			wake_up_interruptible(&proc->wait);
	    ......
	}
	// 4. 将处理好 bwr 结构体拷贝回用户空间
	if (copy_to_user(ubuf, &bwr, sizeof(bwr))) {
		ret = -EFAULT;
		goto out;
	}
out:
	return ret;
}
```
binder_ioctl_write_read 中主要操作入下
- 拷贝用户空间传递的数据到 binder_write_read 结构体中
- binder_thread_write: 向其他进程发送数据
- binder_thread_read: 读取其他进程发送的数据
- 将处理后的 binder_write_read 重新拷贝回用户空间

binder_ioctl_write_read 中的信息量非常丰富了, 内核中同样定义了 binder_write_read 结构体, 通过这个结构体就可以和用户空间进行相互拷贝了, 返回的数据也通过这个结构体拷贝回用户空间

**思考**<br>
这里我们可能会有疑问了, 网上不都说 Binder 驱动是一次拷贝吗? 为什么但是这里就已经出现两次拷贝了呢?
- 因为用户空间使用系统调用时, 会陷入内核态, 系统调用的中的参数必须通过拷贝传递

网上所述的一次拷贝指的是跨进程数据的拷贝, 在后面的章节在会着重分析

在 ServiceManager 的 binder_looper 中可知, bwr 的 read_size 为 32, 因此这里调用 binder_thread_read 进行读操作, 下面我们看看它是如何读取数据的
```
static int binder_thread_read(struct binder_proc *proc,
			      struct binder_thread *thread,
			      binder_uintptr_t binder_buffer, size_t size,
			      binder_size_t *consumed, int non_block)
{
	void __user *buffer = (void __user *)(uintptr_t)binder_buffer;
	void __user *ptr = buffer + *consumed;
	void __user *end = buffer + size;

	int ret = 0;
	int wait_for_proc_work;

	if (*consumed == 0) {
		if (put_user(BR_NOOP, (uint32_t __user *)ptr))
			return -EFAULT;
		ptr += sizeof(uint32_t);
	}

retry:
    // 1. 根据当前线程的 todo 队列是否为空, 判断是否需要执行等待操作
	wait_for_proc_work = thread->transaction_stack == NULL &&
				list_empty(&thread->todo);
    
    ......// 尝试从当前进程的 todo 队列中取任务执行
    
    // 2. 将当前线程状态标记为等待
    thread->looper |= BINDER_LOOPER_STATE_WAITING;
	......
	
	if (wait_for_proc_work) {
		if (!(thread->looper & (BINDER_LOOPER_STATE_REGISTERED |
					BINDER_LOOPER_STATE_ENTERED))) {
		    
			wait_event_interruptible(binder_user_error_wait,
						 binder_stop_on_user_error < 2);
		}
		binder_set_nice(proc->default_priority);
		if (non_block) {
			if (!binder_has_proc_work(proc, thread))
				ret = -EAGAIN;
		} else {
		    // 3. 通过 wait_event_interruptible_exclusive 函数进入休眠状态, 等待请求到来再唤醒了
		    ret = wait_event_freezable_exclusive(proc->wait, binder_has_proc_work(proc, thread));
		}
	}

	......// 4. 真正执行读取数据的操作
	
}
```
从这里可以看到 binder_thread_read 会根据当前进程是否有任务, 来判断是否需要进入睡眠等待, **若没有任务则睡眠在 wait_event_freezable_exclusive 函数上, 等待任务到来了再唤醒**

关于 binder_thread_read 获取其他进程发送的数据我们到跨进程通信实例中再分析, 接下来我们看看用户空间的 binder_parse 如何处理从 Binder 驱动读到的数据

#### 2. binder_parse 处理进程间通信请求
```
// frameworks/base/cmds/servicemanager/binder.c
void binder_loop(struct binder_state *bs, binder_handler func)
{
    int res;
    // 描述对 binder 驱动读写的结构体
    struct binder_write_read bwr;
    .....
    // 2. 死循环从 binder 驱动中获取需要处理的进程间通信数据并处理
    for (;;) {
        ......
        // 2.1 获取 binder 驱动返回的数据
        res = ioctl(bs->fd, BINDER_WRITE_READ, &bwr);
        ......
        // 2.2 处理进程间的请求
        res = binder_parse(bs, 0, (uintptr_t) readbuf, bwr.read_consumed, func);
        ......
    }
}

int binder_parse(struct binder_state *bs, struct binder_io *bio,
                 uintptr_t ptr, size_t size, binder_handler func)
{
    int r = 1;
    uintptr_t end = ptr + (uintptr_t) size;
    while (ptr < end) {
        // 获取 binder 驱动的返回值指令码
        uint32_t cmd = *(uint32_t *) ptr;
        ptr += sizeof(uint32_t);
        ......
        switch(cmd) {
        ......
        case BR_TRANSACTION: {
            .......
            break;
        }
        case BR_REPLY: {
            ......
            break;
        }
        .......
    }
    return r;
}
```
这里通过解析 ptr 中 Binder 驱动返回的指令码, 进行相应的处理, 其中比较重要的有 **BR_TRANSACTION** 和 **BR_REPLY**

关于他们的具体实现, 我们到跨进程通信实例中再进行具体的分析

## 总结
ServiceManager 为服务管理进程, 它是 Binder 驱动的上下文管理者, 其启动流程主要分为如下几步
- 打开 Binder 驱动
  - open: 打开 Binder 设备文件
  - mmap: 初始化一个 128 kb 的缓冲区
- 注册为当前进程为 Binder 上下文的管理者
  - binder_node 对象通过静态变量 binder_context_mgr_node 保存
- 调用 binder_loop 维护一个死循环, 处理其他进程的通信请求
  - 当其他进程对当前进程发起 Binder 通信请求时, looper 线程用于接收通信数据分发处理
  - 死循环通过 ioctl 从 Binder 驱动中读取数据
    - 没有任务, 则阻塞在 wait_event_freezable_exclusive 函数上, 等待任务到来了再唤醒
  - 读到任务之后, 通过 binder_parse 处理任务数据

ServiceManager 进程的启动到这里就结束了, 通过 binder_loop 中的死循环, 就可以监听其他进程的 Binder 通信请求了
- 如此一来 Service Manager 就在 Android 系统的进程间通信机制 Binder 担负起守护进程的职责了

后面的文章我们会通过一次 Binder 跨进程通信, 来将之前学习的知识点串联起来