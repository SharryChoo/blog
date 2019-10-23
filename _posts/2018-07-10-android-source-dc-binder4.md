---
layout: article
title: "Android 系统架构 —— 数据通信篇 之 Binder 驱动"
key: "Android 系统架构 —— 数据通信篇 之 Binder 驱动" 
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

Linux 万物皆文件, 驱动程序也是通过文件接口对上层提供服务的, Binder 驱动文件别于基于 ext4 文件系统的文件操作, 它有自己的文件操作实现, 其定义如下
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
    // 1. 初始化 binder_proc 结构体对象
	proc = kzalloc(sizeof(*proc), GFP_KERNEL);
	......
	proc->default_priority = task_nice(current);   // 进程的优先级
	binder_dev = container_of(filp->private_data, struct binder_device,  miscdev);
	proc->context = &binder_dev->context;
	
	binder_lock(proc->context, __func__);
	binder_stats_created(BINDER_STAT_PROC);
	// 2. 将这个 binder_proc 结构体添加内核驱动的缓存队列 binder_procs 中
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
- 为当前进程创建 binder_proc 对象 proc
- 初始化 binder_proc 对象 proc
- 将 proc 加入全局维护的 hash 队列 binder_procs 中, 因此遍历这个队列, 就可以知道当前有多少个进程在使用 binder 进程间的通信了
- 将这个 proc 写入打开的 binder 设备文件的结构体中
   - filp->private_data = proc;
- 在 /proc/binder/proc 中创建一个以进程 ID 为名的只读文件

Linux 内核打开了 Binder 设备文件 /dev/binder 便完成了, 接下来看看 mmap 的实现

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
	// 2. 从 binder 设备文件中获取进程描述, 它在上面的 open 操作时添加
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
可以看到 binder_mmap 分配内核缓冲区的操作主要有如下几步
- 分配当前**进程的用户虚拟地址空间**
  - 实参 vm_area_struct *vma 指针, 它用来描述当前进程的用户虚拟地址空间
    - 它由 Linux 操作系统传入
  - **最大为 4M, 只可读不可写, 不可复制**
- 分配当前**进程的 Linux 内核虚拟地址空间**
  - 声明的变量 vm_struct *area, 它用来描述当前进程的Linux 内核虚拟地址空间
- 给用户虚拟地址空间和 Linux 内核虚拟地址空间分配物理内存
  - 通过 **binder_update_page_range 给两个虚拟地址空间分配物理内存**
- 将这个内核缓冲区添加到 proc 空闲内核缓冲区的红黑树 free_buffer 中

可以看到, 整个操作类似于 ext4 文件系统的 mmap 匿名映射, 通过这个缓冲区用户空间和内核空间就可以实现数据互通了, 同一个进程内无需进行数据拷贝操作

### 一) 映射物理页面
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
至此 Binder 驱动为这个进程分配的一个 binder 缓冲区就完成了, 他们的映射关系如下

![用户空间与内核空间物理页的映射](4D69DF9649904D34BBDABD68A838E1F4)

使用这种方式, 我们将数据从用户空间发至内核空间时, 就不用在用户态和内核态之间相互拷贝了, **只需要拷贝到虚拟地址指向的物理内存, 即可快捷的进行数据的读取**

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
可看到缓存内核缓冲区的操作, 即将这个 binder_buffer 添加到 binder_proc 的红黑树中, 以便后续取出使用

关于 mmap 操作, 我们就了解到这里, 为了后续更好的分析 Binder 驱动, 下面我们再了解一下 Binder 驱动中比较重要的结构体

## 四. Binder 驱动相关结构体
### binder_work
**描述待处理的工作项**
 ```
 // kernel/goldfish/drivers/staging/android/binder.c
 struct binder_work {
     struct list_head entry;// 将该结构体嵌入到宿主的结构体中
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
### binder_node
**描述一个 Binder 的实体对象, 与 Service 一一对应**
```
 // kernel/goldfish/drivers/staging/android/binder.c
 struct binder_node {
     int debug_id;
     struct binder_work work;// 这个 binder 实体对象待处理的工作项
     union {
         struct rb_node rb_node;// 宿主进程通过红黑二叉树保存这个 binder 实体对象
         struct hlist_node dead_node;// 宿主进程死亡后, 将这个 binder 对象保存到 dead_node 中
     }
     struct binder_proc *proc; // 宿主进程
     struct hlist_head refs;// 保存引用了当前 binder 实体对象的 Client 组件的引用对象
     // 强引用/弱引用计数
     int internal_strong_refs;
     int local_strong_refs;
     int local_weak_refs;
     unsigned has_strong_ref : 1;
     unsigned pending_strong_ref : 1;
     unsigned has_weak_ref : 1;
     unsigned pending_weak_ref : 1;
     
     void __user *ptr;// 指向这个 binder 对象对应的 Service 的地址
     void __user *cookie;// 指向这个 binder 对象对应的 Service 的引用计数 weakref_impl 的地址
     
     unsigned has_async_transaction : 1;// 是否正在处理异步事务
     unsigned accept_fds : 1;// 是否可以接受包含文件描述的进程间通信数据
     int min_priority : 8;// 处理 Client 请求时, 处理线程的优先级
     struct list_head async_todo;// 若需要处理异步事务, 将 binder 保存在这个队列中
 }
```
### binder_death
- **描述一个 Binder 对象的死亡通知的实例**
```
 // kernel/goldfish/drivers/staging/android/binder.c
 struct binder_ref_death {
     struct binder_work work;
     void __user *cookie;// 保存接收死亡通知对象的地址
 }
```
### binder_ref
- **描述一个 Binder 对象的引用实例**
```
// kernel/goldfish/drivers/staging/android/binder.c
struct binder_ref {
    int debug_id;
    // 宿主进程使用两个红黑树保存所有的 binder 引用对象
    struct rb_node rb_node_desc;
    struct rb_node rb_node_node;
    struct hlist_node node_entry;// 这个 binder 引用对象, 在其所对应的 binder 实体对象的引用列表中的结点
    
    struct binder_proc *proc;// 指向这个 binder 引用对象的宿主进程
    struct binder_node *node;// 这个 binder 引用对象所引用的实体对象
    unit32_t desc;// 用于描述这个 binder 引用对象的句柄值
    // 描述一个 binder 引用对象的生命周期
    int strong;
    int weak;
    // 用于接收 Service 端发来的死亡通知
    struct binder_ref_death *death;
}
```
### binder_buffer
- **描述一个内核缓冲区, 它是用来在进程间传输数据的**
- 每一个使用 Binder 进程间通信机制的进程在 Binder 驱动程序中都有一个内核缓冲区列表, 用来保存 Binder 驱动程序为它分配的内核缓冲区
```
// kernel/goldfish/drivers/staging/android/binder.c
struct binder_buffer {
    struct list_head entry;// 内核缓冲区列表的结点
    struct rb_node rb_node;// 若内核缓冲区是空闲的, rb_node 就是空闲的内核缓冲区红黑树的节点, 否则是真正使用内核缓冲区树的结点
    
    unsigned free : 1;// 判断内核缓冲区是否空闲
    unsigned allow_user_free : 1;// 为 1 则请求 Binder 驱动程序释放这个缓冲区
    unsigned async_transaction : 1;// 是否为异步事务
    unsigned debug_id : 29;
    
    struct binder_transaction *transaction;// 内核缓冲区正在交给哪一个事务使用
    
    struct binder_node* target_node;// 内核缓冲区交给哪一个 binder 实体使用
    size_t data_size;      // 记录数据缓冲大小
    siez_t offset_size;    // 偏移数组的大小, 这个偏移数组在数据缓冲区之后, 用来记录数据缓冲中 binder 对象的位置
    uint8_t data[0];       // 描述真正的数据缓冲区
}
```

### binder_proc
- **描述一个正在使用 Binder 进程间通信机制的进程**
- 当一个进程调用函数 open 打开设备文件 /dev/binder 时, Binder 驱动就会为它创建一个 binder_proc 结构体
```
// kernel/goldfish/drivers/staging/android/binder.c
struct binder_proc {
    struct hlist_node proc_node;// 打开设备文件时, Binder 驱动会创建一个Binder_proc 保存在 hash 表中, proc_node 为其中的一个节点
    struct rb_root nodes;// 保存该进程中 Binder 的实体对象
    
    // 描述线城池
    struct rb_root threads;// 以线程的 id 作为关键字来组织一个进程的 Binder 线城池
    struct max_threads;// 进程本身可以注册的最大线程数
    int ready_threads;// 空闲的线程数
    
    // 内核缓冲区
	struct list_head buffers;         // 内核缓冲区链表
	struct rb_root free_buffers;      // 空闲缓冲区红黑树
	struct rb_root allocated_buffers; // 被使用的缓冲区红黑树
    
    // 相关地址
    struct vm_area_struct *vma;// 用户空间地址
    void *buffer; // 内核空间地址
    ptrdiff_t user_buffer_offset; // 用户空间地址与内核空间地址的差值
    

    struct list_head todo;// binder 请求待处理工作项
}
```
### binder_thread
- **描述 binder 线程池中的一个线程**
```
// kernel/goldfish/drivers/staging/android/binder.c
struct binder_thread {
    struct binder_proc *proc;// 宿主进程
    struct rb_node rb_node;// 当前线程在宿主进程线城池中的结点
    int pid;// 线程的 id
    struct binder_transaction * transaction_stack;// 保存要处理的事务
    strcut list_head todo;// 要处理的 client 请求
    wait_queue_head_t wait;// 当前线程依赖于其他线程处理后才能继续, 则将它添加到 wait 中
    struct binder_stats stats;// 当前线程接收到进程间通信的次数
}
```
### binder_transaction
- **描述进程间的通信过程, 又称为一个事务**
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
### binder_transaction_data
**描述进程间通信传递的数据**
```
// kernel/goldfish/drivers/staging/android/binder.c
struct binder_transaction_data {
    union {
        size_t handle;// binder 引用对象的句柄值
        void *ptr;// biner 实体对象的指针
    } target;// 描述目标 binder 实体对象/引用对象
    
    union {
        struct {
            const void *buffer;
            const void *offsets;
        } ptr;
        unit8_t buf[8];
    } data;// 指向通信数据的缓冲区
    size_t data_size;// 描述数据缓冲区大小
    size_t offset_size;// 描述数据缓存区一个偏移数组的大小
}
```
### flat_binder_object
- **描述 binder_transaction_data 中的数据缓冲区 binder 对象**
- 不仅可以描述从用户空间传递过来的 binder 实体对象, 也可以描述 binder 引用对象, 同时也可以描述句柄
```
// kernel/goldfish/drivers/staging/android/binder.c
struct flat_binder_object {

    unsigned long type;// 用于区分该结构体描述的对象类型(binder 实体对象/ binder 引用对象/文件)
    unsigned long flags;// 只有在 type 为 binder 实体对象时才有意义
    
    union {
        void *binder;// 当前 type 为 binder 实体对象时, 指向 service 组件内部的一个弱引用计数对象
        signed long handle;// 当 type 为 binder 引用对象时, 指向引用对象的句柄值
    }
    
    void *cookie;// 当 type 为 binder 实体对象时, 指向 service 组件的地址
}
```

### 回顾
Binder 驱动层的对象引用关系如下

![Binder 驱动结构体依赖](https://i.loli.net/2019/10/23/pzIn2tfBis3yGTu.png)

## 总结
Binder 驱动程序设备文件的几个基本操作如下
- init
  - 创建 Binder 驱动相关的文件
- open
  - 打开 binder 设备文件
  - 为当前进程创建 binder_proc
- mmap: 与 ext4 文件系统类似, 它负责为 Binder 驱动分配内核缓冲区
  - 创建缓冲区 binder_buffer, 最大值为 4MB
    - 因此 Binder 驱动无法传递大数据
  - 为缓冲区分配物理页面
  - 将缓冲区添加到 binder_proc 的链表中缓存

好的, 这篇文章我们了解了 Binder 驱动文件对常见系统调用 api 的实现, 这有助于我们更好的了解 Binder 通信的流程, 为后面文章提供技术支持