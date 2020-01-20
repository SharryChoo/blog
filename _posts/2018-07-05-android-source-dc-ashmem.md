---
title: Android 系统架构 —— Ashmem 共享内存驱动
permalink: android-source/dc-ashmem
key: android-source-dc-ashmem
tags: AndroidFramework
---

## 前言
Ashmen(Anonymous Shared Memory) 匿名共享内存是 Android 的 Linux 内核实现的一个驱动, 它以驱动程序的形式实现在内核空间, 用于在进程间进行数据共享

我们知道 Binder 的内核缓冲区空间最大为 4M, 因此它无法一次性传递大数据, 在 Android UI 渲染时, 我们需要将我们需要渲染的 Surface 数据发送到 SurfaceFlinger 进程进行最后的绘制, Binder 驱动传输显然是无法满足这么大的数据量, 因此这里使用到了 Ashmem 共享内存

接下来我们从下面几个部分分别来解读 Ashmem 共享内存机制
- 驱动的初始化
- open 系统调用
- mmap 系统调用

## 一. 驱动程序的初始化
Ashmem 驱动程序的初始化工作在 Linux 内核中 **ashmem_init** 函数中执行, 下面看看它的实现
```
// kernel/goldfish/mm/ashmem.c
static struct kmem_cache *ashmem_area_cachep __read_mostly;

/** Ashmem 驱动初始化
 *
 */
static int __init ashmem_init(void)
{
	int ret;
    
    // 创建了 kmem_cache, 用于分配 ashmem_area 结构体的内存
	ashmem_area_cachep = kmem_cache_create("ashmem_area_cache",
					  sizeof(struct ashmem_area),
					  0, 0, NULL);
    .......

	return 0;
}
```
其初始化操作调用了 kmem_cache_create 函数创建了一个 kmem_cache 结构体对象

**kmem_cache 描述物理内存缓冲区, 使用的是 Linux 中分配小物理内存时的 slab allocator 技术**, 与通过伙伴系统分配大内存不同, 使用 kmem_cache 进行物理内存分配的策略会更加的高效, 内核中描述进程的 task_struct 也是通过 kmem_cache 来快速分配物理页面的, 这里就不再赘述了

这里在创建 kmem_cache 时, 指定了每一个区域的大小指定为 ashmem_area 的大小, 也就是说, 当我们在使用 Ashmem 共享内存创建 ashmem_area 结构体时, 就可以高效的使用 kmem_cache 来完成了

ashmem_area 的定义如下
```
// kernel/goldfish/mm/ashmem.c

#define ASHMEM_NAME_PREFIX "dev/ashmem/"
#define ASHMEM_NAME_PREFIX_LEN (sizeof(ASHMEM_NAME_PREFIX) - 1)
#define ASHMEM_FULL_NAME_LEN (ASHMEM_NAME_LEN + ASHMEM_NAME_PREFIX_LEN)

/**
 * ashmem_area - 描述一块共享内存区域的结构体
 * Lifecycle: From our parent file's open() until its release()
 * Locking: 受到 `ashmem_mutex' 互斥锁的保护
 * Big Note: Mappings do NOT pin this structure; it dies on close()
 */
struct ashmem_area {
    // 描述共享内存的名字, 名字会显示 /proc/<pid>/maps 文件中
    // pid 表示打开这个共享内存文件的进程ID
	char name[ASHMEM_FULL_NAME_LEN];
	
	// 描述一个链表头, 它把这块共享内存中所有被解锁的内存块连接在一起
	struct list_head unpinned_list;
	
	// 描述这个共享内存在临时文件系统 tmpfs 中对应的文件
	// 在内核决定要把这块共享内存对应的物理页面回收时，就会把它的内容交换到这个临时文件中去
	struct file *file;
	
	// 描述共享内存块的大小
	size_t size;		
	
	// 描述这块共享内存的访问保护位
	unsigned long prot_mask;	
};
```
可以看到 ashmem_area 用来描述一块共享内存区域, 了解了这个之后, 下面我们看看共享内存的文件的打开操作

## 二. open 系统调用
```
// system/core/libcutils/ashmem-dev.cpp
static int __ashmem_open()
{
    int fd;

    pthread_mutex_lock(&__ashmem_lock);
    fd = __ashmem_open_locked();
    pthread_mutex_unlock(&__ashmem_lock);

    return fd;
}

#define ASHMEM_DEVICE "/dev/ashmem"

/* logistics of getting file descriptor for ashmem */
static int __ashmem_open_locked()
{
    ......
    // 调用了 open 函数获取文件描述符
    int fd = TEMP_FAILURE_RETRY(open(ASHMEM_DEVICE, O_RDWR | O_CLOEXEC));
    ......
    return fd;
}
```
当我们在用户空间调用了 open 函数打开设备文件 ASHMEM_DEVICE, 设备文件路径为 "/dev/ashmem"

这个 open 系统调用, 会触发 "/dev/ashmem" 所在的文件系统, 从而在 Linux 内核中触发 ashmem_open 的调用

```
// kernel/goldfish/mm/ashmem.c

#define ASHMEM_NAME_PREFIX "dev/ashmem/"
#define ASHMEM_NAME_PREFIX_LEN (sizeof(ASHMEM_NAME_PREFIX) - 1)

// 驱动初始化时, 创建的缓冲区
static struct kmem_cache *ashmem_area_cachep __read_mostly;

static int ashmem_open(struct inode *inode, struct file *file)
{
	struct ashmem_area *asma;
	......
    // 1. 通过 kmem_cache 创建一个 ashmem_area 结构体
	asma = kmem_cache_zalloc(ashmem_area_cachep, GFP_KERNEL);
	// 设置 ashmem_area 的其他字段
	// 初始化链表头
	INIT_LIST_HEAD(&asma->unpinned_list);
	// 给 name 属性设置固定前缀 "dev/ashmem/"
	memcpy(asma->name, ASHMEM_NAME_PREFIX, ASHMEM_NAME_PREFIX_LEN);
	......
	// 将结构体保存在设备文件的 private_data 中
	file->private_data = asma; 
	return 0;
}
```
可以看到 Linux 内核中的 ashmem_open 操作如下
- **通过 kmem_cache 创建一个 ashmem_area 结构体, 用于描述这个 ashmem 共享内存**
- 将这个 Ashmem 共享内存描述保存到设备文件的 private_data 的域中, 

至此共享内存文件的打开就完成了, 下面看看 mmap 系统调用

## 三. mmap 系统调用
Ashmem 驱动设备文件在 mmap 的系统调用, 对应 Linux 的 ashmem_mmap 函数
```
// kernel/goldfish/mm/ashmem.c
static int ashmem_mmap(struct file *file, struct vm_area_struct *vma)
{
    // 1. 从这个驱动文件的 private_data 中获取 ashmem_area 结构体
	struct ashmem_area *asma = file->private_data;
	int ret = 0;

	mutex_lock(&ashmem_mutex);
    ......
    
	if (!asma->file) {
		char *name = ASHMEM_NAME_DEF;
		struct file *vmfile;

		if (asma->name[ASHMEM_NAME_PREFIX_LEN] != '\0')
			name = asma->name;

		// 2. 通过 shmem_file_setup 创建一个共享内存文件
		vmfile = shmem_file_setup(name, asma->size, vma->vm_flags);
		// 3. 将这个共享内存文件保存在 asma 对象中
		asma->file = vmfile;
	}
	get_file(asma->file);
    // 4. 将这个共享内存文件 映射到 用户的虚拟地址空间上
	if (vma->vm_flags & VM_SHARED)
		shmem_set_file(vma, asma->file);
	else {
		if (vma->vm_file)
			fput(vma->vm_file);
		vma->vm_file = asma->file;
	}
	......
	mutex_unlock(&ashmem_mutex);
	return ret;
}
```
可以看到 ashmem_mmap 函数中主要做了这样几步操作
- **获取 ashmem_area 结构体对象**
- 通过 shmem_file_setup 创建共享内存文件 vmfile
- 将创共享内存文件 vmfile 保存在 ashmem_area 的 file 内
- 将这个 共享内存文件 映射到 用户态的虚拟地址空间上

好的, 可以看到这里的 ashmem_mmap 的操作完成后, 用户空间就可以通过往文件中读写, 从而将数据写到共享内存文件中了

## 四. 使用案例
### 一) Native 层使用
参考 MMKV 的 [MemoryFile](https://github.com/Tencent/MMKV/blob/master/Android/MMKV/mmkv/src/main/cpp/MmapedFile.cpp)
```
#pragma mark - ashmem
#include "native-bridge.h"
#include <dlfcn.h>

#define ASHMEM_NAME_LEN 256
#define __ASHMEMIOC 0x77
#define ASHMEM_SET_NAME _IOW(__ASHMEMIOC, 1, char[ASHMEM_NAME_LEN])
#define ASHMEM_GET_NAME _IOR(__ASHMEMIOC, 2, char[ASHMEM_NAME_LEN])
#define ASHMEM_SET_SIZE _IOW(__ASHMEMIOC, 3, size_t)
#define ASHMEM_GET_SIZE _IO(__ASHMEMIOC, 4)

void *loadLibrary() {
    auto name = "libandroid.so";
    static auto handle = dlopen(name, RTLD_LAZY | RTLD_LOCAL);
    if (handle == RTLD_DEFAULT) {
        MMKVError("unable to load library %s", name);
    }
    return handle;
}

typedef int (*AShmem_create_t)(const char *name, size_t size);

int ASharedMemory_create(const char *name, size_t size) {
    int fd = -1;
    // Android 8.0 以上使用 libandroid.so 的 ASharedMemory_create 创建
    if (g_android_api >= __ANDROID_API_O__) {
        static auto handle = loadLibrary();
        static AShmem_create_t funcPtr =
            (handle != nullptr)
                ? reinterpret_cast<AShmem_create_t>(dlsym(handle, "ASharedMemory_create"))
                : nullptr;
        if (funcPtr) {
            fd = funcPtr(name, size);
            if (fd < 0) {
                MMKVError("fail to ASharedMemory_create %s with size %zu, errno:%s", name, size,
                          strerror(errno));
            }
        } else {
            MMKVWarning("fail to locate ASharedMemory_create() from loading libandroid.so");
        }
    }
    // Android 8.0 以下, 直接操作 "dev/ashmem" 驱动文件
    if (fd < 0) {
        fd = open(ASHMEM_NAME_DEF, O_RDWR);
        if (fd < 0) {
            MMKVError("fail to open ashmem:%s, %s", name, strerror(errno));
        } else {
            // 设置共享内存区域的名称
            if (ioctl(fd, ASHMEM_SET_NAME, name) != 0) {
                MMKVError("fail to set ashmem name:%s, %s", name, strerror(errno));
            } 
            // 设置共享内存区域的大小
            else if (ioctl(fd, ASHMEM_SET_SIZE, size) != 0) {
                MMKVError("fail to set ashmem:%s, size %zu, %s", name, size, strerror(errno));
            }
        }
    }
    return fd;
}
```
因为文件是对多个进程是共享的, 只要两个进程都通过 mmap 映射这个文件, 那么两个进程就可以通过共享内存进行通信了

### 二) Java 层使用
```
// 创建 10 MB 的 Ashmem 共享内存
val memoryFile = MemoryFile("test shared memory", 10 * 1024 * 1024)

// 反射获取实现对象 mSharedMemory, 继承自 Parcelable
val sharedMemory = reflectObject(memoryFile, "mSharedMemory")

// 通过 Binder 驱动, 将 mSharedMemory 发送到另一个进程即可使用
```
Java 层的使用比较简单, Ashmem 的封装类为 SharedMemory, API 27 之前, 我们无法直接使用, 而是需要通过 MemoryFile 创建, 

## 总结
Ashmem 与 Binder 一样, 都是以驱动的形式存在于 Linux 内核中的, Linux 对上层提供服务的方式, 都是通过文件 api, 因此在 Linux 内核中 Ahsmem 文件系统的 api, 即可完成相应的操作
- init
  - 创建了用于分配小内存的 kmem_cache, 方便后续进行 ashmem_area 结构体的内存分配
- open
  - 打开驱动设备文件, 创建 ashmem_area 描述一块匿名共享内存，将其保存到共享内存驱动 fd 的 private 中 
- ioctl
  - ASHMEM_SET_NAME: 为这块匿名共享内存设置文件名
    - 只能在 mmap 之前调用
  - ASHMEM_SET_SIZE: 为这块匿名共享内存设置文件大小
    - 只能在 mmap 之前调用 
- mmap
  - 创建一个基于内存文件系统的共享内存文件 vmfile
  - 将这个 vmfile 文件映射到用户空间的虚拟地址上