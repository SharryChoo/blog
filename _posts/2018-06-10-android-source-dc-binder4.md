---
layout: article
title: "Android 系统架构 —— Binder 驱动"
key: "Android 系统架构 —— Binder 驱动" 
tags: AndroidFramework
aside:
  toc: true
---

## 前言
前面学习了 Binder 对象的实例化流程, 了解了 Android 运行时库中对 Binder 的封装, 为了更好的了解 Binder 驱动的工作机制, 这里我们再学习一下 Linux 内核中 Binder 驱动相关知识
```
// 目录结构
- ~Android/kernel/goldfish
  - drivers/staging/android
    - binder.h
    - binder.c
```

<!--more-->

Linux 万物皆文件, 驱动程序也是通过文件接口对上层提供服务的, Binder 驱动文件有别于基于 ext4 文件系统的文件操作, 它有自己的文件操作实现, 其定义如下
```
// /drivers/staging/android/binder.c
static const struct file_operations binder_fops = {
	.owner = THIS_MODULE,
	.poll = binder_poll,
	.unlocked_ioctl = binder_ioctl,
	.compat_ioctl = binder_ioctl,
	.mmap = binder_mmap,
	.open = binder_open,
	.flush = binder_flush,
	.release = binder_release,
};
```
这里我们解析几个很重要的初始化操作
- 驱动的初始化
- open 系统调用
- mmap 系统调用
- ioctl 系统调用

## 一. 驱动的初始化
Linux 设备驱动程序是一个内核模块, 以 ko 的文件形式存在, 通过 insmod 系统调用加载到内核中, Binder 驱动程序的初始化是在 init 进程中进行的, 由 binder_init 函数实现, 接下来看看这个函数做了哪些操作
```
// kernel/goldfish/drivers/staging/android/binder.c
static int __init binder_init(void)
{
	int ret = 0;
	char *device_name, *device_names;
	struct binder_device *device;
	struct hlist_node *tmp;
	......
	// 创建设备名称
	device_names = kzalloc(strlen(binder_devices_param) + 1, GFP_KERNEL);
	strcpy(device_names, binder_devices_param);
	while ((device_name = strsep(&device_names, ","))) {
	    // 1. 调用了 init_binder_device 初始化设备
		ret = init_binder_device(device_name);
		......
	}
	// 声明了 Binder 驱动可使用的目录: /proc/binder
	binder_debugfs_dir_entry_root = debugfs_create_dir("binder", NULL);
	// 2. 为当前进程创建了: /proc/binder/proc 目录
	// 每个使用了 Binder 进程通信机制的进程在该目录下都对应一个文件
	if (binder_debugfs_dir_entry_root) {
	    binder_debugfs_dir_entry_proc = debugfs_create_dir("proc",binder_debugfs_dir_entry_root);
	}
	// 3. 在 /proc/binder 目录下创建 state, stats, transactions, transaction_log, failed_transaction_log 这几个文件
	// 用于读取 Binder 驱动运行状况
	if (binder_debugfs_dir_entry_root) {
		debugfs_create_file("state",
				    S_IRUGO,
				    binder_debugfs_dir_entry_root,
				    NULL,
				    &binder_state_fops);
		debugfs_create_file("stats",
				    S_IRUGO,
				    binder_debugfs_dir_entry_root,
				    NULL,
				    &binder_stats_fops);
		debugfs_create_file("transactions",
				    S_IRUGO,
				    binder_debugfs_dir_entry_root,
				    NULL,
				    &binder_transactions_fops);
		debugfs_create_file("transaction_log",
				    S_IRUGO,
				    binder_debugfs_dir_entry_root,
				    NULL,
				    &binder_transaction_log_fops);
		debugfs_create_file("failed_transaction_log",
				    S_IRUGO,
				    binder_debugfs_dir_entry_root,
				    NULL,
				    &binder_failed_transaction_log_fops);
	}
	return ret;
}

static int __init init_binder_device(const char *name)
{
	int ret;
	struct binder_device *binder_device;
	struct binder_context *context;

	binder_device = kzalloc(sizeof(*binder_device), GFP_KERNEL);
	if (!binder_device)
		return -ENOMEM;

	binder_device->miscdev.fops = &binder_fops;
	binder_device->miscdev.minor = MISC_DYNAMIC_MINOR;
	binder_device->miscdev.name = name;
    ......
    // 调用了 misc_register 来注册一个 Binder 设备
	ret = misc_register(&binder_device->miscdev);
    ......
	return ret;
}
```
可以看到驱动设备的初始化主要做了以下事情
- 调用了 init_binder_device 初始化 Binder 设备
   - 最终调用了 misc_register 来注册了一个 Binder 设备
- 创建目录
  - /proc/binder/
    - proc(folder)
       - 每一个使用了 Binder 进程间通信机制的进程, 在该目录下都对应一个文件, 文件以进程 ID 命名
       - 通过他们就可以读取到各个进程的 Binder 线程池, Binder 实体对象, Binder 引用对象以及内核缓冲区等消息
    - 创建了 5个文件 state, stats, transactions, transaction_log, failed_transaction_log
       - 这 5 个文件可以读取到 Binder 驱动的运行状况 

好的, Binder 驱动初始化成功之后, 用户空间就可以正常的使用了, 下面我们看看 Binder 驱动设备文件的 open 系统调用

## 二. open 系统调用
Binder 设备文件的 open, 陷入内核之后会调用 binder_open 函数
```
// kernel/goldfish/drivers/staging/android/binder.c

static int binder_open(struct inode *nodp, struct file *filp)
{
	struct binder_proc *proc;
	struct binder_device *binder_dev;

	binder_debug(BINDER_DEBUG_OPEN_CLOSE, "binder_open: %d:%d\n",
		     current->group_leader->pid, current->pid);
    // 1. 创建 binder_proc 结构体对象
	proc = kzalloc(sizeof(*proc), GFP_KERNEL);
	......
	// 为 binder_proc 注入数据
	proc->default_priority = task_nice(current);   // 进程的优先级
	binder_dev = container_of(filp->private_data, struct binder_device,  miscdev);
	proc->context = &binder_dev->context;
	binder_lock(proc->context, __func__);
	binder_stats_created(BINDER_STAT_PROC);
	
	// 2. 将这个 binder_proc 结构体添加内核驱动的 binder_procs 中缓存
	hlist_add_head(&proc->proc_node, &proc->context->binder_procs);
	proc->pid = current->group_leader->pid;
	INIT_LIST_HEAD(&proc->delivered_death);
	// 将进行 binder 驱动的进程描述写入 file 的 private_data 中保存
	filp->private_data = proc;
	binder_unlock(proc->context, __func__);

    // 3. 在目标设备的目录下 /proc/binder/proc 创建以该进程 ID 为名的只读文件
	if (binder_debugfs_dir_entry_proc) {
		char strbuf[11];
		snprintf(strbuf, sizeof(strbuf), "%u", proc->pid);
		proc->debugfs_entry = debugfs_create_file(strbuf, S_IRUGO,
			binder_debugfs_dir_entry_proc,
			(void *)(unsigned long)proc->pid,
			&binder_proc_fops);
	}

	return 0;
}
```
binder_open 的主要流程如下
- 为当前进程创建 **binder_proc** 对象 proc
- 初始化 binder_proc 对象 proc
- **将 proc 加入全局维护的 hash 队列 binder_procs 中**
  - 因此遍历这个队列, 就可以知道当前有多少个进程在使用 binder 进程间的通信了
- 将这个 proc 写入打开的 binder 设备文件的结构体中
   - filp->private_data = proc;
- 在 /proc/binder/proc 中创建一个以进程 ID 为名的只读文件

从流程中我们可以了解到这些信息 binder_proc 与进程是一一对应的, Linux 内核只需要根据从 binder_procs 中找到当前进程对应的 binder_proc, 就可以进行 Binder 驱动的相关操作了

**因此 binder_proc 可以理解为进程 Binder 驱动的上下文对象**, 下面我们看看 binder_proc 的相关定义

### 一) binder_proc 定义
```
// kernel/goldfish/drivers/staging/android/binder.c
struct binder_proc {
    // 打开设备文件时, Binder 驱动会创建一个Binder_proc 保存在 hash 表中 
    // proc_node 描述 binder_proc 为其中的一个节点
    struct hlist_node proc_node;
    
    // Binder 对象缓存
    struct rb_root nodes;             // 保存 binder_proc 进程内的 Binder 实体对象
    struct rb_root refs_by_desc;      // 保存其他进程 Binder 引用对象, 以句柄作 key 值来组织
    struct rb_root refs_by_node;      // 保存其他进程 Binder 引用对象, 以地址作 key 值来组织
    
    // 描述线程池
    struct rb_root threads;           // 以线程的 id 作为关键字来组织一个进程的 Binder 线程池
    struct max_threads;               // 进程本身可以注册的最大线程数
    int ready_threads;                // 空闲的线程数
    
    // 内核缓冲区首地址
    struct vm_area_struct *vma;       // 用户空间地址
    void *buffer;                     // 内核空间地址
    ptrdiff_t user_buffer_offset;     // 用户空间地址与内核空间地址的差值
    
    // 内核缓冲区
	struct list_head buffers;         // 内核缓冲区链表
	struct rb_root free_buffers;      // 空闲缓冲区红黑树
	struct rb_root allocated_buffers; // 被使用的缓冲区红黑树
    
    // binder 请求待处理工作项
    struct list_head todo;
}
```
binder_proc 中相关变量我们可以大致的看出 Binder 驱动的几个核心模块
- Binder 对象缓存
  - **binder_node 实体对象**
  - **binder_ref 引用对象**
- 用于处理任务的 **binder 线程**池
- binder 缓冲区的首地址
- 用于数据交互的 **binder 内核缓冲区**链表
- 待处理的任务队列

好的, binder_proc 中定义了当前进程下 Binder 驱动所有重要的结构体对象和缓存, 下面

### 二) binder 对象
#### 1. binder_node
描述一个 Binder 的实体对象, 在 Server 端使用
```
 // kernel/goldfish/drivers/staging/android/binder.c
 struct binder_node {
     int debug_id;
     // 这个 binder 实体对象待处理的工作项
     struct binder_work work;
     union {
         // 描述 binder_proc 中保存该实体对象的红黑树结点
         struct rb_node rb_node;
         // 宿主进程死亡后, 将这个 binder 对象保存到 dead_node 中
         struct hlist_node dead_node;
     }
     struct binder_proc *proc; // 当前进程的 binder 上下文
     // 保存引用了当前 binder 实体对象的 Client 组件的引用对象
     struct hlist_head refs;
     // 强引用/弱引用计数
     int internal_strong_refs;
     int local_strong_refs;
     int local_weak_refs;
     ......
     // 指向这个 binder 对象对应的 Service 的地址
     void __user *ptr;
     // 指向这个 binder 对象对应的 Service 的引用计数 weakref_impl 的地址
     void __user *cookie;
     // 是否正在处理异步事务
     unsigned has_async_transaction : 1;
     // 是否可以接受包含文件描述的进程间通信数据
     unsigned accept_fds : 1;
     // 处理 Client 请求时, 处理线程的优先级
     int min_priority : 8;
     // 若需要处理异步事务, 将 binder 保存在这个队列中
     struct list_head async_todo;
 }
```
binder_node 中的信息量还是很大的, 其中通过 **binder_work** 来描述待处理的工作项, 它的定义如下
```
// kernel/goldfish/drivers/staging/android/binder.c
struct binder_work {
    // 将该结构体嵌入到宿主的结构体中
    struct list_head entry;
    enum {
        BINDER_WORK_TRANSACTION = 1,
        BINDER_WORK_TRANSACTION_COMPLETE,
        BINDER_WORK_NODE,
        BINDER_WORK_DEAD_BINDER,
        BINDER_WORK_DEAD_BINDER_AND_CLEAR,
        BINDER_WORK_CLEAR_DEATH_NOTIFICATION,
    } type;
}
```
可以看到 binder_work 中有一个枚举, 定义了 Binder 处理的工作项的描述 在跨进程通信的实例中可以看到他们的妙用, 这里就先不展开讨论了

下面我们看看 binder_ref 的定义

#### 2. binder_ref
描述一个 Binder 对象的引用实例, 在 Client 端使用
```
// kernel/goldfish/drivers/staging/android/binder.c
struct binder_ref {
    int debug_id;
    // 当前 binder_ref 在 binder_proc 红黑树中的结点
    struct rb_node rb_node_desc;
    struct rb_node rb_node_node;
    
    // 当前 binder 引用对象, 对应的 binder_node 对象的结点
    struct hlist_node node_entry;
    
    // 指向这个 binder 引用对象的宿主进程
    struct binder_proc *proc;
    
    // 这个 binder 引用对象所引用的实体对象
    struct binder_node *node;
    
    // 用于描述这个 binder 引用对象的句柄值
    unit32_t desc;
    ......
}
```
可以看到 binder_ref 中保存了指向 binder_node 对象的指针, 因此通过 binder_ref 找到其对应的 binder_node 就可以进行跨进程数据的传输了

了解了 binder_node 和 binder_ref 的定义, 下面我们看看 binder 线程

### 三) binder 线程
```
// kernel/goldfish/drivers/staging/android/binder.c
struct binder_thread {
    struct binder_proc *proc;  // 宿主进程
    struct rb_node rb_node;    // 当前线程在宿主进程线程池中的结点
    int pid;                   // 线程的 id
    // 描述要处理的事务
    struct binder_transaction * transaction_stack;
    strcut list_head todo;     // 要处理的 client 请求
    wait_queue_head_t wait;    // 当前线程依赖于其他线程处理后才能继续, 则将它添加到 wait 中
    struct binder_stats stats; // 当前线程接收到进程间通信的次数
}
```
可以看到 binder_thread 中通过 binder_transaction 来描述一个待处理的事务, 它的定义如下

```
// kernel/goldfish/drivers/staging/android/binder.c
struct binder_transaction {
    struct binder_thread from;// 发起这个事务的线程
    struct binder_thread *to_thread;// 事务负责处理的线程
    struct binder_proc *to_proc;// 事务负责处理的进程
    
    // 当前事务所依赖的另一个事务
    struct binder_transaction *from_parent;
    struct binder_transaction *to_parent;
}
```
了解了描述 binder 驱动的事务之后, 下面我们看看 binder 用于和用户空间交互的缓冲区的定义

### 四) binder 缓冲区
每一个使用 Binder 进程间通信机制的进程在 Binder 驱动程序中都有一个内核缓冲区列表, 用来保存 Binder 驱动程序为它分配的内核缓冲区
```
// kernel/goldfish/drivers/staging/android/binder.c
struct binder_buffer {
    // 当前缓冲区在 binder_proc->buffers 中的结点
    struct list_head entry;
    
    // 若内核缓冲区是空闲的, rb_node 就是 binder_proc 空闲的内核缓冲区红黑树的结点
    // 若内核缓冲区正在使用的, rb_node 就是 binder_proc 使用的内核缓冲区红黑树的结点
    struct rb_node rb_node;
    
    ......
    
    // 保存使用缓冲区的事务
    struct binder_transaction *transaction;
    
    // 保存使用缓冲区的 binder 实体对象
    struct binder_node* target_node;
    
    // 记录数据缓冲大小
    size_t data_size;
    
    // 偏移数组的大小, 这个偏移数组在数据缓冲区之后, 用来记录数据缓冲中 binder 对象的位置
    siez_t offset_size;
    
    // 描述真正的数据缓冲区
    uint8_t data[0];       
}
```


### 回顾
Binder 驱动层的对象引用关系如下

![Binder 驱动结构体依赖](https://i.loli.net/2019/10/23/pzIn2tfBis3yGTu.png)

了解了 Binder 驱动核心结构体之后, 下面我们看看 mmap 系统调用

## 三. mmap 系统调用
binder 设备文件的 mmap 系统调用, 由 binder_mmap 实现
```
// drivers/staging/android/binder.c

// vm_area_struct 由 Linux 内核传入, 它表示一个进程的用户虚拟地址空间
static int binder_mmap(struct file *filp, struct vm_area_struct *vma)
{
	int ret;
	// 1. 声明一个 vm_struct 结构体 area, 它描述的是一个进程的内核态虚拟内存
	struct vm_struct *area;
	
	// 2. 从 binder 设备文件中获取 binder_proc, 当进程调用了 open 函数时, 便会创建一个 binder_proc
	struct binder_proc *proc = filp->private_data;
	struct binder_buffer *buffer;
	
    // 3. 判断要映射的用户虚拟地址空间范围是否超过了 4M
	if ((vma->vm_end - vma->vm_start) > SZ_4M)
		vma->vm_end = vma->vm_start + SZ_4M;// 若超过 4M, 则截断为 4M
    // 3.1 检查要映射的进程用户地址空间是否可写
	if (vma->vm_flags & FORBIDDEN_MMAP_FLAGS) {
		ret = -EPERM;
		failure_string = "bad vm_flags";
		goto err_bad_arg;// 若可写, 则映射失败
	}
	// 3.2 给 binder 用户地址的 flags 中添加一条不可复制的约定
	vma->vm_flags = (vma->vm_flags | VM_DONTCOPY) & ~VM_MAYWRITE;
    // 3.3 验证进程的 buffer 中, 是否已经指向一块内存区域了
	if (proc->buffer) {
		ret = -EBUSY;
		failure_string = "already mapped";
		goto err_already_mapped;// 若已存在内存区域, 则映射失败
	}
	
    // 4. 分配一块内核态的虚拟内存, 保存在 binder_proc 中
	area = get_vm_area(vma->vm_end - vma->vm_start, VM_IOREMAP);// 空间大小为用户地址空间的大小, 最大为 4M
	proc->buffer = area->addr;
	// 计算 binder 用户地址与内核地址的偏移量
	proc->user_buffer_offset = vma->vm_start - (uintptr_t)proc->buffer;
    ......
    
    // 5. 创建物理页面结构体指针数组, PAGE_SIZE 一般定义为 4k
	proc->pages = kzalloc(sizeof(proc->pages[0]) * ((vma->vm_end - vma->vm_start) / PAGE_SIZE), GFP_KERNEL);
    ......
    
    // 6. 调用 binder_update_page_range 给进程的(用户/内核)虚拟地址空间分配物理页面
	if (binder_update_page_range(proc, 1, proc->buffer, proc->buffer + PAGE_SIZE, vma)) {
		.......
	}
	
	// 7. 物理页面分配成功之后, 将这个内核缓冲区添加到进程结构体 proc 的内核缓冲区列表总
	buffer = proc->buffer;
	INIT_LIST_HEAD(&proc->buffers);// 初始化链表头
	list_add(&buffer->entry, &proc->buffers);// 添加到 proc 的内核缓冲区列表中
	......
	
	// 8. 将这个内核缓冲区添加到 proc 空闲内核缓冲区的红黑树 free_buffer 中
	binder_insert_free_buffer(proc, buffer);
	// 将该进程最大可用于执行异步事务的内核缓冲区大小设置为总大小的一半
	// 防止异步事务消耗过多内核缓冲区, 影响同步事务的执行
	proc->free_async_space = proc->buffer_size / 2;
    ......
	return 0;
    ......
}
```
可以看到 binder_mmap 主要的任务是为当前进程的 binder_proc 初始化内核缓冲区, 相关步骤如下
- 验证是否满足创建条件
  - 需要注意的是 binder_mmap 只有第一次调用是有意义的, 若已经初始化过了, 则会直接返回失败
  - **一个进程的内核缓冲区最大为 4M**
- 创建内核虚拟地址空间
- **物理页面的创建与映射**
  - 将页面映射到用户虚拟地址空间和内核虚拟地址空间
- **缓存缓冲区**
  - 将初始化好的内核缓冲区 binder_buffer 保存到 binder_proc 中

下面我们看看物理页面的映射过程

### 一) 物理页面的创建与映射
从 binder_mmap 分析可知, 其调用了 binder_update_page_range 分配内核缓冲区
```
// drivers/staging/android/binder.c

/**
 *  // binder_mmap 调用代码
	if (binder_update_page_range(proc, 1, proc->buffer, proc->buffer + PAGE_SIZE, vma)) {
		.......
	}
 */
static int binder_update_page_range(
                    struct binder_proc *proc,   // 描述当前进程的 binder 驱动管理者
                    int allocate,
				    void *start, void *end,     // 内核地址的起始 和 结束地址
				    struct vm_area_struct *vma  // 与内核地址映射的用户地址
				    )
{
	void *page_addr;
	unsigned long user_page_addr;
	struct vm_struct tmp_area;
	struct page **page;
    ......
    // 若该参数为 0, 则说明为释放内存, 显然这里为 1
	if (allocate == 0)
		goto free_range;
    ......
    // 分配物理内存
	for (page_addr = start; page_addr < end; page_addr += PAGE_SIZE) {
		int ret;
		struct page **page_array_ptr;
		// 1. 从进程的物理页面结构体指针数组 pages 中, 获取一个与内核地址空间 page_addr~(page_addr + PAGE_SIZE)对应的物理页面指针
		page = &proc->pages[(page_addr - proc->buffer) / PAGE_SIZE];
		// 2. 给结构体指针分配物理内存
		*page = alloc_page(GFP_KERNEL | __GFP_ZERO);
		......
		// 3. 将物理内存地址映射给 Linux 内核地址空间
		tmp_area.addr = page_addr;
		tmp_area.size = PAGE_SIZE + PAGE_SIZE /* guard page? */;
		page_array_ptr = page;
		ret = map_vm_area(&tmp_area, PAGE_KERNEL, &page_array_ptr);
		// 4. 将物理内存地址映射给 用户地址空间
		user_page_addr =
			(uintptr_t)page_addr + proc->user_buffer_offset;
		ret = vm_insert_page(vma, user_page_addr, page[0]);
	}
	......
	return 0;

free_range:
    // allocate 为 0 的情况, 释放物理内存的操作
}
```
至此 Binder 驱动就为这块内核缓冲区, 分配了一个物理页面(一般为 4kb)了, 使用这种方式, 我们将数据从用户空间发至内核空间时, 就不用在用户态和内核态之间相互拷贝了, **只需要拷贝到虚拟地址指向的物理内存, 即可快捷的进行数据的读取**

接下来, 我们就看看它是如何添加到空闲缓冲区中的

### 二) 缓存空闲缓存区
```
// drivers/staging/android/binder.c

static int binder_mmap(struct file *filp, struct vm_area_struct *vma)
{
	......
	// 将这个内核缓冲区添加到 proc 空闲内核缓冲区的红黑树 free_buffer 中
	binder_insert_free_buffer(proc, buffer);
    ......
}

static void binder_insert_free_buffer(struct binder_proc *proc,
				      struct binder_buffer *new_buffer)
{
    // 1. 获取进程中维护的空闲缓冲区列表
	struct rb_node **p = &proc->free_buffers.rb_node;
	struct rb_node *parent = NULL;
	struct binder_buffer *buffer;
	size_t buffer_size;
	size_t new_buffer_size;
    ......
    // 2. 计算新加入的内核缓冲区的大小
	new_buffer_size = binder_buffer_size(proc, new_buffer);
    ......
    // 3. 从红黑树中找寻合适的插入结点
	while (*p) {
		parent = *p;
		buffer = rb_entry(parent, struct binder_buffer, rb_node);
		buffer_size = binder_buffer_size(proc, buffer);
		if (new_buffer_size < buffer_size)
			p = &parent->rb_left;
		else
			p = &parent->rb_right;
	}
	// 4. 链入红黑树中
	rb_link_node(&new_buffer->rb_node, parent, p);
	rb_insert_color(&new_buffer->rb_node, &proc->free_buffers);
}
```
可看到缓存内核缓冲区的操作, 即将这个 binder_buffer 添加到 binder_proc 的红黑树中, 以便后续取出使用, 此时的空闲缓冲区是刚分配的, 也是它最大的状态

### 回顾
这里我们可能会有如下的疑问
**最大的缓冲区为 4MB, 为什么只进行了一个物理页面的分配?**
- 这是因为 Binder 驱动采用了按需分配的策略, 当需要进行数据拷贝时, 会从空闲的缓冲区中找寻缓冲区, 进行物理页面的映射
- 采用按需分配的策略, 可以降低 Binder 驱动闲置时对物理内存的消耗, 也是内存优化的一种方式

**跨进程通讯频繁时 4MB 是否够用?**
- 4 MB 对于大数据跨进程通信是不够的, 这时就需要使用共享内存的方式进行跨进程通讯了, Binder 驱动是为了解决小规模跨进程通讯而生的

**跨进程共享内存支持的这么全全面, 为什么不代替 Binder 驱动?**
- 跨进共享内存虽然可以进行大数据传输, 不过我们需要手动的进行数据读写的同步, 即使是使用 Android 封装好的  Asheme 共享内存, 其使用成本也是比较高的
- 通过 Binder 驱动, 我们可以使用常规开发接口的方式进行跨进程代码的编写, 这也是 Google 工程师设计时一种利他的表现

## 三. ioctl 系统调用
binder 设备文件的 ioctl 系统调用, 由 binder_ioctl 实现
```
static long binder_ioctl(struct file *filp, unsigned int cmd, unsigned long arg)
{
	int ret;
	struct binder_proc *proc = filp->private_data;
	struct binder_thread *thread;
	unsigned int size = _IOC_SIZE(cmd);
	void __user *ubuf = (void __user *)arg;
	
	......
	
	ret = wait_event_interruptible(binder_user_error_wait, binder_stop_on_user_error < 2);
	if (ret)
		goto err_unlocked;

	binder_lock(__func__);
	
	// 2. 获取描述当前线程的 binder_thread 结构体
	thread = binder_get_thread(proc);
	......
	// IOCTL 请求码
	switch (cmd) {
	// 2.1 数据读写请求
	case BINDER_WRITE_READ:
		ret = binder_ioctl_write_read(filp, cmd, arg, thread);
		if (ret)
			goto err;
		break;
	......
	// 2.2 注册上下文的请求
	case BINDER_SET_CONTEXT_MGR:
		ret = binder_ioctl_set_ctx_mgr(filp);
		if (ret)
			goto err;
		ret = security_binder_set_context_mgr(proc->tsk);
		if (ret < 0)
			goto err;
		break;
	......
err:
	if (thread)
		thread->looper &= ~BINDER_LOOPER_STATE_NEED_RETURN;
	binder_unlock(__func__);
	wait_event_interruptible(binder_user_error_wait, binder_stop_on_user_error < 2);
	......
}
```
从这里可以看到 binder_ioctl 主要负责处理用户空间传递的 IOCTL 指令, 常用的指令如下
- BINDER_WRITE_READ: 数据的读写
- BINDER_SET_CONTEXT_MGR: 注册 Binder 上下文的管理者

关于这个两个 ioctl 指令的具体操作, 我们在实际场景下再进行分析, 这里先作为了解

## 总结
Binder 驱动程序设备文件的几个基本操作如下
- **init**: 创建 Binder 驱动相关的文件
- **open**: 打开 binder 设备文件, 为当前进程创建 **binder_proc**, 可以理解为 binder 驱动上下文
  - binder 对象
    - **binder_node**: 实体对象
    - **binder_ref**: 引用对象
  - **binder_thread**: binder 线程
  - **binder_buffer**: binder 缓冲区  
- **mmap**: 初始化当前进程第一个 binder 缓冲区 binder_buffer, 最大值为 4MB
  - 只有首次调用有效 
  - 为缓冲区分配物理页面, 将物理页映射到用户虚拟地址和内核虚拟地址
  - 将缓冲区添加到 binder_proc 中缓存
- **ioctl**: 找寻空闲的线程, 处理 ioctl 请求 
  - BINDER_WRITE_READ: 数据的读写
  - BINDER_SET_CONTEXT_MGR: 注册 Binder 上下文的管理者

好的, 这篇文章我们了解了 Binder 驱动文件对常见系统调用 api 的实现, 这有助于我们更好的了解 Binder 通信的流程, 为后面文章提供技术支持