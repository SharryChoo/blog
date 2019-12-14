---
title: Android 内存监控 —— LeakCanary 检测内存泄漏
permalink: android-performance-opt/memory-leak-monitor
key: android-performance-opt-memory-leak-monitor
tags: PerformanceOptimization
tags: PerformanceOptimization
---

## 前言
[LeakCanary](https://github.com/square/leakcanary) 是 Square 用于 Android 端用于自动检测内存泄漏的开源库

使用这个工具可以方便的监控 Activity 和 Fragment 的内存泄漏情况, 并且提供了可视化界面, 可以在开发过程中很好的暴露和排查问题

![Sample](https://i.loli.net/2019/07/31/5d40faf65a2a183363.jpg)

这里主要分析 LeakCanary 的 **使用流程** 和 **工作原理**

<!--more-->

## 一. 使用流程
### 一) 添加依赖
在 build.gradle 中
```
dependencies {
    ......
    implementation 'com.squareup.leakcanary:leakcanary-android:1.5.4'
    ......
}
```
### 二) 编写工具类
编写 LeakCanary 工具类
```
public class LeakCanaryUtil {

    private static RefWatcher sRefWatcher;

    /**
     * Initialize, please invoke in {@link Application#onCreate()}
     */
    public static void init(@NonNull Application application) {
        if (BuildConfig.IS_RELEASE) {
            return;
        }
        if (LeakCanary.isInAnalyzerProcess(application)) {
            return;
        }
        sRefWatcher = LeakCanary.install(application);
    }

    /**
     * Watches the provided references and checks if it can be GCed. This method is non blocking,
     * the check is done on the {@link com.squareup.leakcanary.WatchExecutor} this {@link RefWatcher} has been constructed
     * with.
     */
    public static void watch(@Nullable Object obj) {
        if (sRefWatcher == null || obj == null) {
            return;
        }
        sRefWatcher.watch(obj);
    }

}
```
RefWatcher 便是进行内存泄漏分析的接口类, 当我们调用了 LeakCanary.install 之后便会自动进行 Activity 的监控

### 三) 使用
#### 1. 初始化
在 Application 中
```
class BaseApplication : android.app.Application() {

    private RefWatcher mRefWatcher;

    override fun onCreate() {
        super.onCreate()
        ......
        LeakCanaryUtil.init(this);
    }

}
```

#### 2. 监控其他对象
若想监控其他对象, 使用只需要调用 RefWatcher.watch 便可, Fragment 的监控如下
```
public abstract class BaseFragment extends Fragment {
   
    @Override
    public void onDetach() {
        // ......
        LeakCanaryUtil.watch(this);
    }

}
```

## 二. 工作原理
```
public final class LeakCanary {
    
  public static RefWatcher install(Application application) {
    // 可以看到这是一个构建者的链式调用, 最终通过 buildAndInstall 来完成对 RefWatcher 的构建和安装
    return refWatcher(application)
               .listenerServiceClass(DisplayLeakService.class)
               .excludedRefs(AndroidExcludedRefs.createAppDefaults().build())
               .buildAndInstall();
  }
    
  public static AndroidRefWatcherBuilder refWatcher(Context context) {
    return new AndroidRefWatcherBuilder(context);
  }
  
}
```
可以看到 LeakCanary 的 install 方法的目的即, 构建一个 RefWatcher 对象, 并且初始化它

这个 RefWatcher 对象通过 AndroidRefWatcherBuilder.buildAndInstall 创建
```
public final class AndroidRefWatcherBuilder extends RefWatcherBuilder<AndroidRefWatcherBuilder> {
    
  public RefWatcher buildAndInstall() {
    // 1. 通过 build 构建实例对象
    RefWatcher refWatcher = build();
    if (refWatcher != DISABLED) {
      ......
      // 2. 注册 Activity 的内存泄漏监听
      ActivityRefWatcher.install((Application) context, refWatcher);
    }
    return refWatcher;
  }  
    
}
```
可以看到 AndroidRefWatcherBuilder.buildAndInstall 中, 主要有两个任务
- 通过 build 方法, 创建 RefWatcher 对象
- 调用 ActivityRefWatcher.install 来监听 Activity 的内存泄漏

接下来从这两个方面来分析 LeakCanary 的工作流程

### 一) RefWatcher 的构建
```
public class RefWatcherBuilder<T extends RefWatcherBuilder<T>> {
    
  /** Creates a {@link RefWatcher}. */
  public final RefWatcher build() {
    if (isDisabled()) {
      return RefWatcher.DISABLED;
    }
    ......
    // 构建 RefWatcher 对象
    return new RefWatcher(watchExecutor, debuggerControl, gcTrigger, heapDumper, heapDumpListener,
        excludedRefs);
  }  
    
}

public final class RefWatcher {

  public static final RefWatcher DISABLED = new RefWatcherBuilder<>().build();

  private final WatchExecutor watchExecutor;
  private final DebuggerControl debuggerControl;
  private final GcTrigger gcTrigger;
  private final HeapDumper heapDumper;
  private final Set<String> retainedKeys;
  private final ReferenceQueue<Object> queue;
  private final HeapDump.Listener heapdumpListener;
  private final ExcludedRefs excludedRefs;

  RefWatcher(WatchExecutor watchExecutor, DebuggerControl debuggerControl, GcTrigger gcTrigger,
      HeapDumper heapDumper, HeapDump.Listener heapdumpListener, ExcludedRefs excludedRefs) {
    // 用于执行监听引用
    this.watchExecutor = checkNotNull(watchExecutor, "watchExecutor");
    // 判断是否在调试中
    this.debuggerControl = checkNotNull(debuggerControl, "debuggerControl");
    // 用于执行 GC
    this.gcTrigger = checkNotNull(gcTrigger, "gcTrigger");
    // 用于将 Heap dump 到文件中
    this.heapDumper = checkNotNull(heapDumper, "heapDumper");
    // 用于接收并分析 heap 信息
    this.heapdumpListener = checkNotNull(heapdumpListener, "heapdumpListener");
    // 排除系统引起的内存泄露
    this.excludedRefs = checkNotNull(excludedRefs, "excludedRefs");
    // 创建集合, 用于保存待分析对象弱引用对应的 key
    retainedKeys = new CopyOnWriteArraySet<>();
    // 创建引用队列, 用于存储待 GC 的元素
    queue = new ReferenceQueue<>();
  }
    
}
```
从 RefWatcher 的构造中, 还是可以看到其功能的职责分配的是非常明确的

RefWatcher 是一个门面类, 其具体的实现分离到各个成员变量中, 接下来看看如何监听 Activity 的内存泄漏

### 二) Activity 内存泄漏的监听
Activity 内存泄漏的监听, 是通过 ActivityRefWatcher.install 发起的
```
public final class ActivityRefWatcher {

  public static void install(Application application, RefWatcher refWatcher) {
    // 1. 创建 ActivityRefWatcher 对象
    // 2. 调用 ActivityRefWatcher.watchActivities
    new ActivityRefWatcher(application, refWatcher).watchActivities();
  }
  
  private final Application application;
  private final RefWatcher refWatcher;
  
  public ActivityRefWatcher(Application application, RefWatcher refWatcher) {
    this.application = checkNotNull(application, "application");
    this.refWatcher = checkNotNull(refWatcher, "refWatcher");
  }

}
```
可以看到 ActivityRefWatcher.install 中, 首先创建了 ActivityRefWatcher 实例, 然后调用了它的 watchActivities 方法
- 其构造函数将传入参数保存到成员变量

接下来看看 watchActivities 的实现
```
public final class ActivityRefWatcher {
    
    private final Application.ActivityLifecycleCallbacks lifecycleCallbacks =
      new Application.ActivityLifecycleCallbacks() {
        @Override public void onActivityCreated(Activity activity, Bundle savedInstanceState) {
        }

        @Override public void onActivityStarted(Activity activity) {
        }

        @Override public void onActivityResumed(Activity activity) {
        }

        @Override public void onActivityPaused(Activity activity) {
        }

        @Override public void onActivityStopped(Activity activity) {
        }

        @Override public void onActivitySaveInstanceState(Activity activity, Bundle outState) {
        }

        @Override public void onActivityDestroyed(Activity activity) {
          // 调用 onActivityDestroyed 来处理 Destroy 之后的 Activity
          ActivityRefWatcher.this.onActivityDestroyed(activity);
        }
      };
      
  public void watchActivities() {
    // 确保不会注册两个 lifecycle
    stopWatchingActivities();
    application.registerActivityLifecycleCallbacks(lifecycleCallbacks);
  }
  
  void onActivityDestroyed(Activity activity) {
    refWatcher.watch(activity);
  }
  
}
```
好的, 可以看到当 Activity 被销毁的回调后, 便会调用 onActivityDestroyed 方法, 进而通过 refWatcher.watch 来处理后续操作

```
public final class RefWatcher {
    
  public void watch(Object watchedReference) {
    watch(watchedReference, "");
  }

  public void watch(Object watchedReference, String referenceName) {
    ......
    // 1. 构建 要监听对象(Activity) 的 key
    String key = UUID.randomUUID().toString();
    // 2. 添加到 key 缓存
    retainedKeys.add(key);
    // 3. 构建要监听对象(Activity)的弱引用
    final KeyedWeakReference reference =
        new KeyedWeakReference(watchedReference, key, referenceName, queue);
    // 4. 确认一下被弱引用的对象是否被回收了
    ensureGoneAsync(watchStartNanoTime, reference);
  }

  private void ensureGoneAsync(final long watchStartNanoTime, final KeyedWeakReference reference) {
    watchExecutor.execute(new Retryable() {
      @Override public Retryable.Result run() {
        // 通过线程池去执行确认操作
        return ensureGone(reference, watchStartNanoTime);
      }
    });
  }
}
```
好的, 可以看到 RefWatcher.watch 方法的任务非常清晰, 其中可以看到它为这个对象构建了一个 key, 并且为它构建了一个弱引用

接下来看看它是通过什么途径确定这个被弱引用的对象是否被回收了
```
public final class RefWatcher {

  @SuppressWarnings("ReferenceEquality") // Explicitly checking for named null.
  Retryable.Result ensureGone(final KeyedWeakReference reference, final long watchStartNanoTime) {
    ......
    removeWeaklyReachableReferences();
    ......
    if (gone(reference)) {
      return DONE;
    }
    // 执行 GC
    gcTrigger.runGc();
    // 尝试移除弱可及的引用, 即即将被 GC 的对象
    removeWeaklyReachableReferences();
    // 判断弱引用是否已经被移除了
    if (!gone(reference)) {
      ......
      // 没有被移除, 则说明发生了内存泄漏
      File heapDumpFile = heapDumper.dumpHeap();
      ......
      // 分析 Dump 的文件
      heapdumpListener.analyze(
          new HeapDump(heapDumpFile, reference.key, reference.name, excludedRefs, watchDurationMs,
              gcDurationMs, heapDumpDurationMs));
    }
    return DONE;
  }

  private boolean gone(KeyedWeakReference reference) {
    return !retainedKeys.contains(reference.key);
  }

  private void removeWeaklyReachableReferences() {
    KeyedWeakReference ref;
    // 从引用队列中获取即将被 GC 的对象
    while ((ref = (KeyedWeakReference) queue.poll()) != null) {
      // 既然即将被 GC 了, 那么说明就不会内存泄漏
      // 尝试从 retainedkey 中移除
      retainedKeys.remove(ref.key);
    }
  }
    
}
```
好的, 可以看到, 这里巧妙的利用了 ReferenceQueue 这个引用队列
- **若一个对象的弱引用, 设置了这个引用队列, 那么这个被弱引用的对象被 GC 时, 会将其弱引用添加到引用队列中, 用于通知外界这个对象即将被回收了**

如果 GC 之后, 这个对象的弱引用依旧存在, 那么就说明可能发生内存泄漏了, 然后辩护通过 heapdumpListener.analyze 来分析内存泄漏的原因 

## 总结
通过上面的分析可知 LeakCanary 的工作原理还是比较简单的
- **它利用了 WeakReference 的特性, 即被弱引用的对象, 在 GC 发生时, 便会被回收**
- 通过为 WeakReference 添加 ReferenceQueue, 记录被 GC 的弱引用对象
- 若我们分析的对象没有被 GC, 那么说明发生了内存泄漏

便是通过这种方式巧妙的实现了 Activity 的内存泄漏分析

## 参考文献
- https://square.github.io/leakcanary/
- https://cloud.tencent.com/developer/article/1169327