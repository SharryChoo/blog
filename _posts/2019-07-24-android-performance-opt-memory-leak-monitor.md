---
title: Android 性能优化 —— LeakCanary 内存泄漏的监控
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
```java
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
```java
public abstract class BaseFragment extends Fragment {
   
    @Override
    public void onDetach() {
        // ......
        LeakCanaryUtil.watch(this);
    }

}
```

## 二. 内存泄漏的检测
```java
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
```java
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
```java
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
```java
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
```java
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

```java
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
```java
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
      // Dump Hrof 文件
      File heapDumpFile = heapDumper.dumpHeap();
      ......
      // 分析 Hrof 的文件
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

如果 GC 之后, 这个对象的弱引用依旧存在, 那么就说明可能发生内存泄漏了, 会执行如下的操作
- 通过 heapDumper.dumpHeap() 获取此刻的内存镜像 HPROF 文件
  - 最终会调用获取 Debug.dumpHprofData(heapDumpFile.getAbsolutePath());
- 通过 heapdumpListener.analyze 来分析内存泄漏的引用链

下面我们便看看它是如何分析数据的并从堆中找到相关泄漏信息的

## 三. 泄漏引用链的分析与查找
```java
public final class ServiceHeapDumpListener implements HeapDump.Listener {

  ......

  @Override public void analyze(HeapDump heapDump) {
    checkNotNull(heapDump, "heapDump");
    HeapAnalyzerService.runAnalysis(context, heapDump, listenerServiceClass);
  }
  
}

public final class HeapAnalyzerService extends IntentService {

  private static final String LISTENER_CLASS_EXTRA = "listener_class_extra";
  private static final String HEAPDUMP_EXTRA = "heapdump_extra";

  public static void runAnalysis(Context context, HeapDump heapDump,
      Class<? extends AbstractAnalysisResultService> listenerServiceClass) {
    // 通知 HeapAnalyzerService 远程服务处理这个 Intent
    Intent intent = new Intent(context, HeapAnalyzerService.class);
    intent.putExtra(LISTENER_CLASS_EXTRA, listenerServiceClass.getName());
    intent.putExtra(HEAPDUMP_EXTRA, heapDump);
    context.startService(intent);
  }
```
可以看到 heapdumpListener.analyze 最终会通知远程的 HeapAnalyzerService 服务, 来解析这个 HPROF 文件
```java
public final class HeapAnalyzerService extends IntentService {

  @Override protected void onHandleIntent(Intent intent) {
    if (intent == null) {
      CanaryLog.d("HeapAnalyzerService received a null intent, ignoring.");
      return;
    }
    String listenerClassName = intent.getStringExtra(LISTENER_CLASS_EXTRA);
    HeapDump heapDump = (HeapDump) intent.getSerializableExtra(HEAPDUMP_EXTRA);
   
    // 1. 创建了 HeapAnalyzer 对象, 用于分析 HPROF 文件
    HeapAnalyzer heapAnalyzer = new HeapAnalyzer(heapDump.excludedRefs);
    // 2. 执行泄漏引用链的分析
    AnalysisResult result = heapAnalyzer.checkForLeak(heapDump.heapDumpFile, heapDump.referenceKey);
    ......
  }
  
}
```
HeapAnalyzer 就是我们的内存泄漏引用链分析的核心所在了, 下面我们看看它的 heapAnalyzer.checkForLeak 方法实现
```java
public final class HeapAnalyzer {
    
  /**
   * Searches the heap dump for a {@link KeyedWeakReference} instance with the corresponding key,
   * and then computes the shortest strong reference path from that instance to the GC roots.
   */
  public AnalysisResult checkForLeak(File heapDumpFile, String referenceKey) {
    long analysisStartNanoTime = System.nanoTime();
    ......
    try {
      // square haha 提供技术实现
      // 1. 将文件映射到内存, 内部实现很有意思, 将文件流读入 ByteBuffer 的数组集合
      HprofBuffer buffer = new MemoryMappedFileBuffer(heapDumpFile);
      // 2. 构建 HPROF 数据到 Snapshot 对象中
      HprofParser parser = new HprofParser(buffer);
      Snapshot snapshot = parser.parse();
      // 3. 从 Snapshot 获取 Java 的 GC Roots
      deduplicateGcRoots(snapshot);
      // 校验在我们 dump Hrof 期间, 这个对象是否被回收了
      Instance leakingRef = findLeakingReference(referenceKey, snapshot);
      // False alarm, weak reference was cleared in between key check and heap dump.
      if (leakingRef == null) {
        return noLeak(since(analysisStartNanoTime));
      }
      // 4. 从 GC Roots 中找寻到泄漏对象的引用链
      return findLeakTrace(analysisStartNanoTime, snapshot, leakingRef);
    } catch (Throwable e) {
      return failure(e, since(analysisStartNanoTime));
    }
  }
    
}
```
HeapAnalyzer 中 checkForLeak 的实现是非常有意思的, 主要流程如下
- 解析 Hprof 文件到 Snapshot 对象中
- 在 Snapshot 中找寻 GC Roots
- 从 GC Roots 中找寻到泄漏对象的引用链

这里 Hprof 文件的解析操作是通过 squareup 的 haha 库提供的技术支持, 这对我们解析 HPROF 文件提供了很好的帮助, 我们可以移植它到其他地方使用

这里我们主要看看 GC Roots 的查找, 和引用队列的查找

### 一) GC Roots
GC Roots 对象是在将 Hprof 解析到 Snapshot 的时 Snapshot.addRoot 添加的

```java
public class Snapshot {

    public final void addRoot(@NonNull RootObj root) {
        mCurrentHeap.addRoot(root);
        root.setHeap(mCurrentHeap);
    }

}

public class Heap {

    //  Root objects such as interned strings, jni locals, etc
    @NonNull
    ArrayList<RootObj> mRoots = new ArrayList<RootObj>();

    public final void addRoot(@NonNull RootObj root) {
        root.mIndex = mRoots.size();
        mRoots.add(root);
    }
    
}
```
addRoot 的调用者有 6 处
```java
public class Snapshot {

    private int loadJniLocal(){...}

    private int loadJniMonitor() {...}

    private int loadThreadBlock() {...}
    
    private int loadBasicObj(RootType type) {...}
    
    private int loadNativeStack() {...}
    
    private int loadJavaFrame() {...}

}
```
它们分别代表的 GC Roots 类型如下
- JNILocalReference 持有的对象
- JNIGlobalReference 持有的对象
- 活动的 Thread 实例
- 类（被JVM加载的类是无法卸载的，因此无法被回收，导致被类持有（即通过静态成员持有）的对象也无法被回收）
- Native 的栈帧
- Java 线程栈的栈帧

了解了 GC Roots 的查找, 下面看看如何通过 GC Roots 找到与我们泄漏对象的引用链

### 二) 引用链查找
```java
public final class HeapAnalyzer {

  private AnalysisResult findLeakTrace(long analysisStartNanoTime, Snapshot snapshot,
      Instance leakingRef) {
    ShortestPathFinder pathFinder = new ShortestPathFinder(excludedRefs);
    // 找寻引用链
    ShortestPathFinder.Result result = pathFinder.findPath(snapshot, leakingRef);
    ......
  }
  
}
```
可以看到这里调用了 ShortestPathFinder 的 findPath 来执行引用链查找的任务
```java
final class ShortestPathFinder {
    
    
  Result findPath(Snapshot snapshot, Instance leakingRef) {
    clearState();
    canIgnoreStrings = !isString(leakingRef);
    // 将 snapshot 构建成邻接表的图结构
    enqueueGcRoots(snapshot);

    boolean excludingKnownLeaks = false;
    LeakNode leakingNode = null;
    // 使用图的广度优先遍历, 搜索距离 GC Roots 最近的引用链
    while (!toVisitQueue.isEmpty() || !toVisitIfNoPathQueue.isEmpty()) {
      LeakNode node;
      if (!toVisitQueue.isEmpty()) {
        node = toVisitQueue.poll();
      } else {
        node = toVisitIfNoPathQueue.poll();
        ......
        excludingKnownLeaks = true;
      }

      // 找到了目标的结点, 终止搜索
      if (node.instance == leakingRef) {
        leakingNode = node;
        break;
      }

      if (checkSeen(node)) {
        continue;
      }

      if (node.instance instanceof RootObj) {
        visitRootObj(node);
      } else if (node.instance instanceof ClassObj) {
        visitClassObj(node);
      } else if (node.instance instanceof ClassInstance) {
        visitClassInstance(node);
      } else if (node.instance instanceof ArrayInstance) {
        visitArrayInstance(node);
      } else {
        throw new IllegalStateException("Unexpected type for " + node.instance);
      }
    }
    return new Result(leakingNode, excludingKnownLeaks);
  }
    
}
```
这里就是**图的经典搜索算法广度优先搜索了**

我们知道我们的泄漏对象可能和多个 GC Roots 有关联, LeakCanary 的做法是命中一条就结束搜索, 为什么不搜索出所有的引用链呢?
- 其实也不难理解, 因为修复了这条链, 若存在其他引用链会在下一次泄漏时找出来, 一次性探测出全部要遍历图的各个顶点, 导致算法耗时过长


## 三. 误判的改进
LeakCanary 虽然实现巧妙, 但并不是完美的, 在使用的过程中, 常会出现误判的情况, 这里就看看误判的原因和改进策略

### 一) 误判
- VM 并没有提供强制触发 GC 的 API，通过 System.gc() 或 Runtime.getRuntime().gc() 只能 **“建议”** 系统进行 GC，如果系统忽略了我们的 GC 请求，可回收的对象就不会被加入 ReferenceQueue
- 将可回收对象加入 ReferenceQueue 需要等待一段时间，LeakCanary 采用延时 100ms 的做法加以规避，但似乎并不绝对管用
- 监测逻辑是异步的，如果判断 Activity 是否可回收时某个 Activity 正好还被某个方法的局部变量持有，就会引起误判

### 二) 改进
- **增加一个一定能被回收的“哨兵”对象，用来确认系统确实进行了 GC**
- **直接通过 WeakReference.get() 来判断对象是否已被回收，避免因 GC 后加入引用队列存在延时导致误判**
- **若发现某个 Activity 无法被回收，再重复判断 3 次，以防在判断时该 Activity 被局部变量持有导致误判**

#### 1. 代码实现
```
public final class RefWatcher {
    
    /**
     * Sharry modified.
     */
    private final static int THRESHOLD = 3;
    private final ConcurrentLinkedQueue<KeyedWeakReference> mPendingDetectQueue = new ConcurrentLinkedQueue<>();

    @SuppressWarnings("ReferenceEquality")
        // Explicitly checking for named null.
    Retryable.Result ensureGone(final KeyedWeakReference newReference, final long watchStartNanoTime) {
        // 1. 将要检测的对象添加到待检测队列
        mPendingDetectQueue.offer(newReference);
        // 2. 尝试 GC
        if (debuggerControl.isDebuggerAttached()) {
            // The debugger can create false leaks.
            return Retryable.Result.RETRY;
        }
        // 创建哨兵对象
        WeakReference<Object> sentryObj = new WeakReference<>(new Object());
        // 执行 GC, 会执行延时 100 ms, 确认让被 GC 对象的弱引用添加到引用队列
        long gcStartNanoTime = System.nanoTime();
        long watchDurationMs = NANOSECONDS.toMillis(gcStartNanoTime - watchStartNanoTime);
        gcTrigger.runGc();
        // 判断是否真正发生 GC 了
        if (sentryObj.get() != null) {
            Log.i(TAG, "GC has been ignore. detect next time.");
            // GC has been ignore.
            return Retryable.Result.RETRY;
        } else {
            Log.i(TAG, "GC occurred.");
        }
        // 3. 遍历待检测的弱引用队列
        Iterator<KeyedWeakReference> iterator = mPendingDetectQueue.iterator();
        while (iterator.hasNext()) {
            KeyedWeakReference reference = iterator.next();
            // 已经被 GC 了
            if (gone(reference)) {
                // 已经被 GC 了, 从待检测项中移除
                iterator.remove();
                Log.i(TAG, reference.name + " is released.");
                continue;
            }
            // 采用分代回收思想, 被 GC 了 3 次还没有被回收, 则确定为泄漏
            if (reference.gcCount.incrementAndGet() < THRESHOLD) {
                Log.i(TAG, reference.name + " maybe leaked. gc count is " + reference.gcCount.get());
                continue;
            } else {
                Log.w(TAG, reference.name + " ensure leaked. do heap analyzing.");
            }
            // Dump 堆映像, 分析内存泄漏
            long startDumpHeap = System.nanoTime();
            long gcDurationMs = NANOSECONDS.toMillis(startDumpHeap - gcStartNanoTime);
            File heapDumpFile = heapDumper.dumpHeap();
            if (heapDumpFile == null) {
                // Could not dump the heap.
                continue;
            }
            long heapDumpDurationMs = NANOSECONDS.toMillis(System.nanoTime() - startDumpHeap);
            HeapDump heapDump = heapDumpBuilder.heapDumpFile(heapDumpFile).referenceKey(reference.key)
                    .referenceName(reference.name)
                    .watchDurationMs(watchDurationMs)
                    .gcDurationMs(gcDurationMs)
                    .heapDumpDurationMs(heapDumpDurationMs)
                    .build();
            heapdumpListener.analyze(heapDump);
            // 进行分析了, 从待检测项中移除
            iterator.remove();
        }
        return Retryable.Result.DONE;
    }

    private boolean gone(KeyedWeakReference reference) {
        return reference.get() == null;
    }
    
}
```

#### 2. 日志打印
```
I/RefWatcher: GC occurred.
I/RefWatcher: Main2Activity maybe leaked. gc count is 1
I/RefWatcher: GC occurred.
I/RefWatcher: Main2Activity maybe leaked. gc count is 2
I/RefWatcher: GC occurred.
W/RefWatcher: Main2Activity ensure leaked. do heap analyzing.
......
```
可以看到每一个泄漏项的确认, 都会经过三次 GC 确认, 通过这样的方式, 让项目中的 LeakCanary 误报的情况大大降低了

### 三) 其他
在这方面 Matrix 的 ResourceCanary 做的更多, 为了提升日志分析, 对 Hprof 文件的进行了裁剪

Hprof 文件的大小一般约为 Dump 时的内存占用大小，就微信而言 Dump 出来的 Hprof 大小通常为 150MB～200MB 之间，如果不做任何处理直接将此 Hprof 文件上传到服务端，一方面会消耗大量带宽资源，另一方面服务端将 Hprof 文件长期存档时也会占用服务器的存储空间。

通过分析 Hprof 文件格式可知，Hprof 文件中 buffer 区存放了所有对象的数据，包括字符串数据、所有的数组等，而我们的分析过程却只需要用到部分字符串数据和 Bitmap 的 buffer 数组，其余的 buffer 数据都可以直接剔除，这样处理之后的 Hprof 文件通常能比原始文件小 1/10 以上。

## 总结
- 内存泄漏的检测
  - **它利用了 WeakReference 的特性, 即被弱引用的对象, 在 GC 发生时, 便会被回收**
  - 为 WeakReference 添加 ReferenceQueue, 对象被 GC 之后弱引用对象会添加到引用队列中
  - 若我们分析的对象没有被 GC, 那么说明发生了内存泄漏
- 泄漏引用链的分析与查找
  - 通过 Debug.dumpHprofData(heapDumpFile.getAbsolutePath()); 获取 HPROF 内存镜像
  - 通知 HeapAnalyzerService 远程服务, 解析这个 HPROF 内存镜像
    - 解析 HPROF 文件数据到 Snapshot 对象中
    - 获取 Snapshot 中的 GC Roots
    - 使用图的广度优先搜索算法, 找寻 GC Roots 到泄漏对象的一条引用链
- 误判的改进
  - VM 并没有提供强制触发 GC 的 API，通过 System.gc() 或 Runtime.getRuntime().gc() 只能 **“建议”** 系统进行 GC，如果系统忽略了我们的 GC 请求，可回收的对象就不会被加入 ReferenceQueue
    - **增加一个一定能被回收的“哨兵”对象，用来确认系统确实进行了 GC**
  - 将可回收对象加入 ReferenceQueue 需要等待一段时间，LeakCanary 采用延时 100ms 的做法加以规避，但似乎并不绝对管用
    - **直接通过 WeakReference.get() 来判断对象是否已被回收，避免因 GC 后加入引用队列存在延时导致误判**
  - 监测逻辑是异步的，如果判断 Activity 是否可回收时某个 Activity 正好还被某个方法的局部变量持有，就会引起误判
    - **若发现某个 Activity 无法被回收，再重复判断 3 次，以防在判断时该 Activity 被局部变量持有导致误判**

LeakCanary 误判改进的 [Sample](https://github.com/SharryChoo/LeakCanarySample) 

## 参考文献
- https://square.github.io/leakcanary/
- https://github.com/Tencent/matrix/wiki/Matrix-Android-ResourceCanary