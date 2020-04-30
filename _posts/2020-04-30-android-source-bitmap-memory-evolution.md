---
title: Android 源码 —— Bitmap 位图内存的演进流程
permalink: android-source/bitmap-memory-evolution
tags: AndroidSource
---
## 前言
Android 中的 Bitmap 像素内存演进流程如下, 摘录自
[Google 文档 - 管理位图内存](https://developer.android.com/topic/performance/graphics/manage-memory)

| 版本                         | 内存分布                                                     |
| ---------------------------- | ------------------------------------------------------------ |
| ~ Android 2.3.3(API 10)      | 1. 位图的后备像素数据存储在 Native 堆中。Native 内存中的像素数据并不以可预测的方式释放，可能会导致应用短暂超出其内存限制并崩溃。<br> 2. Android 2.2(API 8) 无并发垃圾回收功能, 当发生垃圾回收时，应用的线程会停止。这会导致延迟，从而降低性能。<br>3. Android 2.3 添加了并发垃圾回收功能，这意味着系统不再引用位图后，很快就会回收内存 |
| 3.0（API 11）~ 7.1（API 25） | 像素数据会与关联的位图一起存储在 Dalvik/ART 堆上。           |
| 8.0（API 26）~               | 位图像素数据存储在 Native 堆中。配合 NativeAllocationRegistry 进行垃圾回收 |

这里有一个疑问, 为什么 Android 2.3.3 中 Native 的内存释放不可预测, 在 Java 对象的 finalize 被调用时直接释放的方案有何不妥? Android 8.0 的 NativeAllocationRegistry 的引入是为了解决什么问题?

这里就系统的学习一下 NativeAllocationRegistry 的技术, 然后看看他解决了 Android2.3.3 的哪些问题

## 一. Bitmap 的构造

```java
public final class Bitmap implements Parcelable {
    /**
     * Private constructor that must received an already allocated native bitmap
     * int (pointer).
     */
    // Bitmap 对象无法直接 new, 其创建流程, 我们已经分析过了
    Bitmap(long nativeBitmap, int width, int height, int density,
            boolean isMutable, boolean requestPremultiplied,
            byte[] ninePatchChunk, NinePatch.InsetStruct ninePatchInsets) {
        if (nativeBitmap == 0) {
            throw new RuntimeException("internal error: native bitmap is 0");
        }

        mWidth = width;
        mHeight = height;
        mIsMutable = isMutable;
        mRequestPremultiplied = requestPremultiplied;

        mNinePatchChunk = ninePatchChunk;
        mNinePatchInsets = ninePatchInsets;
        if (density >= 0) {
            mDensity = density;
        }

        // 1. 保存 Native 层对应的 SkiaBitmap 的句柄值
        mNativePtr = nativeBitmap;
        long nativeSize = NATIVE_ALLOCATION_SIZE + getAllocationByteCount();
        // 2. nativeGetNativeFinalizer 获取 Native 的 Finalizer.
        // 3. 创建 NativeAllocationRegistry 对象
        NativeAllocationRegistry registry = new NativeAllocationRegistry(
            Bitmap.class.getClassLoader(), nativeGetNativeFinalizer(), nativeSize);
        // 4. 注册这个 java 对象和 native 对象
        registry.registerNativeAllocation(this, nativeBitmap);
        ......
    }

    // 获取 Native 的 Finalizer
    private static native long nativeGetNativeFinalizer();

}
```

Bitmap 创建之后, 其构造函数内部的操作如下

- nativeGetNativeFinalizer 获取 native 层的 Finalizer
- 实例化 NativeAllocationRegistry
- 通过 NativeAllocationRegistry.registerNativeAllocation 监听这个 Java 和 Native 对象

### 一) 获取 native 层的 Finalizer

```C++
namespace android {

 namespace bitmap {
  static void Bitmap_destruct(BitmapWrapper* bitmap) {
      delete bitmap;
  }

  // 这里把 Bitmap_destruct 函数地址转为 long 类型的句柄值返回给 Java 层
  static jlong Bitmap_getNativeFinalizer(JNIEnv*, jobject) {
      return static_cast<jlong>(reinterpret_cast<uintptr_t>(&Bitmap_destruct));
  }
 }

}
```

native 层的 Finalizer 即 Bitmap_destruct 函数, 有了这个函数, 我们便可以快速的是否 native 的 BitmapWrapper 内存了


### 二) NativeAllocationRegistry 的创建

```java
public class NativeAllocationRegistry {

    private final ClassLoader classLoader;
    private final long freeFunction;
    private final long size;

    public NativeAllocationRegistry(ClassLoader classLoader, long freeFunction, long size) {
        if (size < 0) {
            throw new IllegalArgumentException("Invalid native allocation size: " + size);
        }
        this.classLoader = classLoader;
        this.freeFunction = freeFunction;
        this.size = size;
    }

}
```

NativeAllocationRegistry 的构造函数, 即将形参保存到了成员变量, 并没有做多余的初始化操作, 下面看看它的 registerNativeAllocation 方法

## 二. NativeAllocationRegistry 注册 Native 内存大小

```java
public class NativeAllocationRegistry {

   public Runnable registerNativeAllocation(Object referent, long nativePtr) {
          ......
          CleanerThunk thunk;
          CleanerRunner result;
          try {
              // 1. 创建 native 内存块清理器 CleanerThunk
              thunk = new CleanerThunk();
              // 2. 创建清理者, 持有 Java 引用和 thunk
              Cleaner cleaner = Cleaner.create(referent, thunk);
              // 3 .创建 Runner 交由外界主动触发清理操作
              result = new CleanerRunner(cleaner);
              // 4.1 通知 Runtime 分配了 size 大小的内存空间
              registerNativeAllocation(this.size);
          } catch (VirtualMachineError vme /* probably OutOfMemoryError */) {
              // 4.2 注册失败, 释放 native 的内存
              applyFreeFunction(freeFunction, nativePtr);
              throw vme;
          } // Other exceptions are impossible.
          // 5. 让 ClearThunk 绑定 Native 的对象的句柄
          thunk.setNativePtr(nativePtr);
          return result;
      }

      private class CleanerThunk implements Runnable {
          private long nativePtr;

          public CleanerThunk() {
              this.nativePtr = 0;
          }

          public void run() {
              if (nativePtr != 0) {
                  // 释放 Native 对象
                  applyFreeFunction(freeFunction, nativePtr);
                  // 通知 Runtime 释放了 size 大小的 Native 内存
                  registerNativeFree(size);
              }
          }

          public void setNativePtr(long nativePtr) {
              this.nativePtr = nativePtr;
          }
      }

      private static class CleanerRunner implements Runnable {
          private final Cleaner cleaner;

          public CleanerRunner(Cleaner cleaner) {
              this.cleaner = cleaner;
          }

          public void run() {
              // 执行 cleaner 的 clean 操作
              cleaner.clean();
          }
      }

      // 通知 Runtime 分配了 size 大小的 Native 内存
      private static void registerNativeAllocation(long size) {
          VMRuntime.getRuntime().registerNativeAllocation((int)Math.min(size, Integer.MAX_VALUE));
      }

      // 通知 Runtime 释放了 size 大小的 Native 内存
      private static void registerNativeFree(long size) {
          VMRuntime.getRuntime().registerNativeFree((int)Math.min(size, Integer.MAX_VALUE));
      }

      // 通过构造时传入的 freeFunction 释放内存
      public static native void applyFreeFunction(long freeFunction, long nativePtr);
}
```

可以看到 registerNativeAllocation 方法中主要操作如下

- 创建 Native 内存清理器 CleanerThunk
  - 持有 native 对象指针和 freeFunction
- 创建清理者 Cleaner 它负责监听 Java 对象引用状态, 被 GC 时自动触发 CleanerThunk 的内存释放
  - 有些类似于之前我们在对象的 finalize 中释放 Native 内存
- 创建成功, 调用 VMRuntime.registerNativeAllocation, 注册 Native 分配的内存大小
- 创建失败, 直接通过 applyFreeFunction 释放 Native 对象

这里我们主要关注一下 Cleaner 是如何自动处理 Native 对象释放的, 以及 Runtime.registerNativeAllocation 的操作

### 一) Cleaner 的创建

```java
  public class Cleaner
      extends PhantomReference<Object>
  {

      // 虚引用的引用队列无实际意义, 用于构造函数传入
      private static final ReferenceQueue<Object> dummyQueue = new ReferenceQueue<>();

      // 双向链表的头结点
      static private Cleaner first = null;

      // 前驱&后继指针
      private Cleaner
          next = null,
          prev = null;

      //
      private static synchronized Cleaner add(Cleaner cl) {
          if (first != null) {
              cl.next = first;
              first.prev = cl;
          }
          first = cl;
          return cl;
      }

      private static synchronized boolean remove(Cleaner cl) {

          // If already removed, do nothing
          if (cl.next == cl)
              return false;

          // Update list
          if (first == cl) {
              if (cl.next != null)
                  first = cl.next;
              else
                  first = cl.prev;
          }          if (cl.next != null)
              cl.next.prev = cl.prev;
          if (cl.prev != null)
              cl.prev.next = cl.next;

          // Indicate removal by pointing the cleaner to itself
          cl.next = cl;
          cl.prev = cl;
          return true;

      }

      // 清理回调
      private final Runnable thunk;

      // 构造
      private Cleaner(Object referent, Runnable thunk) {
          super(referent, dummyQueue);
          this.thunk = thunk;
      }

      /**
       * Creates a new cleaner.
       *
       * @param  ob the referent object to be cleaned
       * @param  thunk
       *         The cleanup code to be run when the cleaner is invoked.  The
       *         cleanup code is run directly from the reference-handler thread,
       *         so it should be as simple and straightforward as possible.
       *
       * @return  The new cleaner
       */
      public static Cleaner create(Object ob, Runnable thunk) {
          if (thunk == null)
              return null;
          return add(new Cleaner(ob, thunk));
      }

      /**
       * Runs this cleaner, if it has not been run before.
       */
      public void clean() {
          if (!remove(this))
              return;
          try {
              thunk.run();
          } catch (final Throwable x) {
              AccessController.doPrivileged(new PrivilegedAction<Void>() {
                      public Void run() {
                          if (System.err != null)
                              new Error("Cleaner terminated abnormally", x)
                                  .printStackTrace();
                          System.exit(1);
                          return null;
                      }});
          }
      }

  }

```

Cleaner 对象是一个双向链表的结点, 最终在内存中组成一个 Cleaner 双向链表, 通过 Cleaner.clean 方法, 会触发 Cleaner 的 thunk.run 执行真正的清理操作

Cleaner 的构造函数中有如下的注释

The  cleanup code is run directly from the reference-handler thread, so it should be as simple and straightforward as possible.

从注释中可以看出, 这个 Cleaner.clean 方法由 引用处理线程 直接回调的, 并且提示我们不要让 thunk.run 执行耗时操作

**看到这里我们大概就能够明白, 为什么 Bitmap 中找不到 finalize 方法, 也能释放 Native 内存了, 因为一个 Bitmap 对象会被包装成 Cleaner, 成为 Cleaner 链表中的一员, 它的清理操作由 引用处理线程 直接回调 Cleaner.clean 执行数据清理, 同样能够及时释放内存**

### 二) VMRuntime.registerNativeAllocation

```java
public final class VMRuntime {
      /**
       * Holds the VMRuntime singleton.
       */
      private static final VMRuntime THE_ONE = new VMRuntime();

      /**
       * Returns the object that represents the VM instance's Dalvik-specific
       * runtime environment.
       *
       * @return the runtime object
       */
      public static VMRuntime getRuntime() {
          return THE_ONE;
      }

      /**
       * Registers a native allocation so that the heap knows about it and performs GC as required.
       * If the number of native allocated bytes exceeds the native allocation watermark, the
       * function requests a concurrent GC. If the native bytes allocated exceeds a second higher
       * watermark, it is determined that the application is registering native allocations at an
       * unusually high rate and a GC is performed inside of the function to prevent memory usage
       * from excessively increasing.
       */
      public native void registerNativeAllocation(int bytes);
}
```

这里的 registerNativeAllocation 是一个 native 方法, 它的实现定义在 dalvik_system_VMRuntime.cc 中下面我们看看它的实现

```C++
// android-9.0.0_r3/art/runtime/native/dalvik_system_VMRuntime.cc
namespace art {
  ......
  static void VMRuntime_registerNativeAllocation(JNIEnv* env, jobject, jint bytes) {
    if (UNLIKELY(bytes < 0)) {
      ScopedObjectAccess soa(env);
      ThrowRuntimeException("allocation size negative %d", bytes);
      return;
    }
    // 将记录 Native 分配的内存的操作, 交由 ART 的 Heap 执行
    Runtime::Current()->GetHeap()->RegisterNativeAllocation(env, static_cast<size_t>(bytes));
  }
  ......
}

// android-9.0.0_r3/art/runtime/gc/heap.cc
namespace art {

namespace gc {
  ......
  void Heap::RegisterNativeAllocation(JNIEnv* env, size_t bytes) {
    // 交由 new_native_bytes_allocated_ 原子类累加 Native 对象分配的内存
    size_t old_value = new_native_bytes_allocated_.FetchAndAddRelaxed(bytes);
    // 判断 Native 分配的内存是否触发了 GC 阈值, 超过了则进行 GC 操作释放 Java 和 Native 内存
    if (old_value > NativeAllocationGcWatermark() * HeapGrowthMultiplier() &&
               !IsGCRequestPending()) {
      // Trigger another GC because there have been enough native bytes
      // allocated since the last GC.
      if (IsGcConcurrent()) {
        RequestConcurrentGC(ThreadForEnv(env), kGcCauseForNativeAlloc, /*force_full*/true);
      } else {
        CollectGarbageInternal(NonStickyGcType(), kGcCauseForNativeAlloc, false);
      }
    }
  }
  ......
}

}
```

Native 层的实现比较简单, 它将 Native 分配的内存进行累加, 当 Java 对象持有的 Native 内存超过了阈值, 则进行 GC 操作

这里看到阈值的是由 **NativeAllocationGcWatermark() * HeapGrowthMultiplier()** 组成, 下面探究一下 Native 内存 GC 阈值的值

### 注册 Native 内存触发虚拟机 GC 的阈值计算

```C++
// android-9.0.0_r3/art/runtime/gc/heap.h
class Heap {
 public:
  // 计算 Native 分配内存的乘数
  double foreground_heap_growth_multiplier_;

  // 计算 Native 内存阈值
  size_t max_free_;
  ALWAYS_INLINE size_t NativeAllocationGcWatermark() const {
    return max_free_;
  }
}

// android-9.0.0_r3/art/runtime/gc/heap.cc
double Heap::HeapGrowthMultiplier() const {
  // 不考虑后台暂停时长, 乘数则直接返回 1.0, 这样 Native 触发 GC 的阈值较少, GC 会更频繁
  if (!CareAboutPauseTimes()) {
    return 1.0;
  }
  // 默认使用 foreground_heap_growth_multiplier_
  return foreground_heap_growth_multiplier_;
}

```

这里并没有具体的赋值, Heap 的创建是在 Runtime::Init 中执行的, 我们看看这个两个数据注入的是多少

```C++
// android-9.0.0_r3/art/runtime/runtime.cc
static constexpr double kExtraDefaultHeapGrowthMultiplier = kUseReadBarrier ? 1.0 : 0.0;

bool Runtime::Init(RuntimeArgumentMap&& runtime_options_in) {
  ......
 float foreground_heap_growth_multiplier;
 if (is_low_memory_mode_ && !runtime_options.Exists(Opt::ForegroundHeapGrowthMultiplier)) {
   // 若为低内存设备, 乘数为 1.0
   foreground_heap_growth_multiplier = 1.0f;
 } else {
   // 从 runtime_options 取值
   foreground_heap_growth_multiplier =
       runtime_options.GetOrDefault(Opt::ForegroundHeapGrowthMultiplier) +
           kExtraDefaultHeapGrowthMultiplier;
 }
  heap_ = new gc::Heap(runtime_options.GetOrDefault(Opt::MemoryInitialSize),
                         runtime_options.GetOrDefault(Opt::HeapGrowthLimit),
                         runtime_options.GetOrDefault(Opt::HeapMinFree),
                         // 这里注入了 max_free_
                         runtime_options.GetOrDefault(Opt::HeapMaxFree),
                         runtime_options.GetOrDefault(Opt::HeapTargetUtilization),
                         // 阈值乘数
                         foreground_heap_growth_multiplier,
                         runtime_options.GetOrDefault(Opt::MemoryMaximumSize),
                         runtime_options.GetOrDefault(Opt::NonMovingSpaceCapacity),
                         runtime_options.GetOrDefault(Opt::Image),
                         runtime_options.GetOrDefault(Opt::ImageInstructionSet),
                         // Override the collector type to CC if the read barrier config.
                         kUseReadBarrier ? gc::kCollectorTypeCC : xgc_option.collector_type_,
                         kUseReadBarrier ? BackgroundGcOption(gc::kCollectorTypeCCBackground)
                                         : runtime_options.GetOrDefault(Opt::BackgroundGc),
                         runtime_options.GetOrDefault(Opt::LargeObjectSpace),
                         runtime_options.GetOrDefault(Opt::LargeObjectThreshold),
                         runtime_options.GetOrDefault(Opt::ParallelGCThreads),
                         runtime_options.GetOrDefault(Opt::ConcGCThreads),
                         runtime_options.Exists(Opt::LowMemoryMode),
                         runtime_options.GetOrDefault(Opt::LongPauseLogThreshold),
                         runtime_options.GetOrDefault(Opt::LongGCLogThreshold),
                         runtime_options.Exists(Opt::IgnoreMaxFootprint),
                         runtime_options.GetOrDefault(Opt::UseTLAB),
                         xgc_option.verify_pre_gc_heap_,
                         xgc_option.verify_pre_sweeping_heap_,
                         xgc_option.verify_post_gc_heap_,
                         xgc_option.verify_pre_gc_rosalloc_,
                         xgc_option.verify_pre_sweeping_rosalloc_,
                         xgc_option.verify_post_gc_rosalloc_,
                         xgc_option.gcstress_,
                         xgc_option.measure_,
                         runtime_options.GetOrDefault(Opt::EnableHSpaceCompactForOOM),
                         runtime_options.GetOrDefault(Opt::HSpaceCompactForOOMMinIntervalsMs));
  ......
}
```

可以看到这里的参数是从 runtime_options 中取到的, 他们的定义如下

```def
// android-9.0.0_r3/art/runtime/runtime_options.def
RUNTIME_OPTIONS_KEY (MemoryKiB, HeapMaxFree, gc::Heap::kDefaultMaxFree)
RUNTIME_OPTIONS_KEY (double, ForegroundHeapGrowthMultiplier gc::Heap::kDefaultHeapGrowthMultiplier)
```

可以看到这里最终又会回到 heap.h 中, 下面看看这个 Native 内存阈值到底是多少

```c++
// android-9.0.0_r3/art/runtime/gc/heap.h
class Heap {
 public:
  // If true, measure the total allocation time.
  static constexpr size_t kDefaultMaxFree = 2 * MB;
  static constexpr double kDefaultHeapGrowthMultiplier = 2.0;
  ......
}
```

故 Native 内存分配超过 4MB, 便会尝试触发虚拟机 GC

### 三) 回顾

通过源码分析可以发现, 原来向 ART 虚拟机注册 Native 分配大小, 是为了让 Native 分配的内存也成为虚拟机 GC 的权重之一, 当 Java 对应的 Native 对象内存分配总和超过 4MB, 便会尝试触发虚拟机的 GC

## 总结

为什么 2.3.3 ~ 7.0 要放到 Java 堆? 直接放到 Native 中, 然后在 Java 对象 finalize 调用的时候释放不行吗?

- Java 层的 Bitmap 对象是一个壳, 非常小, 因此有可能会出现 Native 堆快到了 3G, Java 堆才 10 MB, 10MB 是无法触发 Dalvik GC 的, 因此这个 java 对象的 finalize 并非那么容易调用, 因此可能会出现 Native 堆 OOM 的情况, 故需要我们手动 recycle
- 像素数据直接放置到 Java 堆, Java 堆就能直接统计到真正的内存数据, 能够根据内存使用情况准确触发 GC 回收数据
  -  隐患便是 Java 堆内存空间比较小, 容器造成 Java 堆的 OOM

为什么 8.0 又放置到了 Native 堆中?

- 使用 NativeAllocationRegistry 解决了这个问题, 触发 ART 堆 GC 的条件不仅仅是堆占用不足, 通过 VMRuntime.registerNativeAllocation 注册的 Native 内存累计超过了阈值(4MB)之后时也会触发 GC
-  而且 ART 的 GC 性能比 Dalvik 好的多, 不会轻易造成主线程卡顿

### 遗留问题
- 为什么要单独设计一个 Cleaner 链表, 让 reference-handler thread 单独处理内存的释放, NativeAllocationRegistry + finalize 的方案有何不妥?
- Native 内存 4MB 便会触发 GC, 阈值是不是太小了, 是不是自己什么地方没有看到? 会不会出现 GC 过于频繁的问题?
- ART 的 GC 机制和 Dailvk GC 原理有何异同, 是怎么实现的?

## 参考文献
- [管理位图内存](https://developer.android.com/topic/performance/graphics/manage-memory)