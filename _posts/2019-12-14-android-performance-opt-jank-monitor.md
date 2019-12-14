---
title: Android 性能优化 —— 卡顿的监控与信息采集
permalink: android-performance-opt/jank-monitor
key: android-performance-opt-jank-monitor
tags: PerformanceOptimization
---

## 前言
对用户来说，内存占用高、耗费电量、耗费流量可能不容易被发现，但是用户对卡顿特别敏感，很容易直观感受到。另一方面，对于开发者来说，卡顿问题又非常难以排查定位，产生的原因错综复杂，跟 CPU 负载、内存、磁盘 I/O 都可能有关，跟用户当时的系统环境也有很大关系。

## 一. 卡顿的基础知识
因为**连续丢帧**导致的视觉效果和操作不连贯的现象称之为卡顿

### 屏幕成像原理
刷新率为 60 Hz 的屏幕, 每秒会发出 60 个 Vsync 信号, 每一个信号到来时, 会触发一次 SurfaceFlinger 的图层合成与渲染, 也就是说只要 16.67 ms 能够完成一次绘制, 就能够保证每秒 60 帧

无论是游戏跑分, 还是其他领域, 通常认为每秒 60 帧是流畅度的保证, 那是不是低于 60 帧就一定会造成卡顿呢?
- 其实不然, 视频播放的帧率为 24 fps, 我们在观看的过程中也没有感觉卡顿, 这是因为它是连续的帧
 
**所以 FPS 低并不意味着一定会卡顿, 产生卡顿的原因是连续丢帧**

### 思考
**我们知道今年的一加 7 Pro 和三星定制了 90 Hz 刷新率的 OLEN 屏, 60 Hz 的屏幕意味着 1 秒钟最高可以渲染 60 帧不同的数据, 从视频播放来说, 视频 1 秒 24 帧我们都已经感觉不到卡顿了, 90 Hz 刷新率, 就算能够渲染 90 帧的图像数据, 又有什么意义呢?**

![90 赫兹刷新率](https://i.loli.net/2019/12/14/ZJa1dYI9XCNiDGP.png)

<!--more-->

答案是否定的, 刷新率的提高, 意味着每一帧动画之间的时间间隔缩小了, 每一帧动画之间的差异会随之缩小, 因此每一帧之间的色彩差异也会缩小, 因此过渡的会更加顺滑

虽然每秒 60 帧 和 90 帧动画, 都能够带来流畅的体验, 但帧率更高的流畅度也就更高, 操作起来就更加丝滑

## 二. 产生原因
卡顿产生的原因有很多, 一般有如下几个方面的原因
- **CPU 负载过高**
  - CPU 的时间片被占用, 导致我们的任务无法及时处理
- **内存不足**
  - 可用内存不足, 会导致 Dalvik/ART 虚拟机频繁 GC, 造成整个进程的停滞卡顿
- **UI 线程执行耗时操作**
  - 我们知道整个 View 绘制的三大流程都是在主线程执行, 可以想象的, 我们在执行动画期间穿插了其他的耗时操作势必会导致 UI 线程绘制的卡顿

从这里可以看到产生卡顿的原因是多种多样的, 想要解决卡顿问题, 首先要学会发现问题, 下面看看如何监控应用卡顿

## 三. 卡顿监控
我们知道卡顿产生的本质是连续的丢帧, 那么如何监控卡顿呢? 既然是连续丢帧, 那么监控每秒的帧率是行不通的, 因为它无法反应是否出现连续丢帧了, 我们转换一下思路就可以想到, **可以将监控卡顿的问题转化为监控丢帧数**

```
丢帧数 = 一帧准备的时长 / 16.67 ms(60 Hz 的刷新率)
```

若一帧的准备时长超过了 16.67 ms 则意味着出现了丢帧, 因此我们可以设置一个阈值, 比如说 300 ms, 当一帧的准备时长超过了 300 ms, 则说明出现连续丢帧了, 因此**丢帧的监控可以转化为每帧准备时长的监控**

4.1 之后帧的准备操作是在 Choreographer 收到 SurfaceFlinger 发送的 VSYNC 之后在主线程消息队列中执行的([关于 Choreographer 工作机制可以查看这篇文章](https://sharrychoo.github.io/blog/android-source/graphic-choreographer)), **因此我们也可以通过监控主线程消息执行耗时来监控卡顿**

### 一) 工具监控
#### [1. systrace](https://source.android.com/devices/tech/debug/systrace?hl=zh-cn)
systrace 是 Android 4.1 新增的性能分析工具。通常可以使用 systrace 跟踪系统的 I/O 操作、CPU 负载、Surface 渲染、GC 等事件。

systrace 用于监测系统的运行耗时是非常方便的, 但是对于我们的应用程序, 则需要自己进行插桩处理了
```
Trace.beginSection("sectionName");
{
    ......// 要监控的代码块
}
Trace.endSection();
```
图形界面使用的是 **CallChart**

![CallChart](https://i.loli.net/2019/12/14/LqvzSGg5JhViyWR.png)

我们可以看到在这一段时间内, 各个线程的具体工作, 比如是否存在线程间的锁、主线程是否存在长时间的 I/O 操作、是否存在空闲等

#### [2. Simpleperf](https://source.android.com/devices/tech/debug/eval_perf?hl=zh-cn)
Simpleperf 是一款适用于 Android 平台上的应用和本机进程的原生分析工具。CPU 分析器可用于实时检查应用的 CPU 使用率和线程 Activity。

它利用 CPU 的性能监控单元（PMU）提供的硬件 perf 事件。使用 Simpleperf 可以看到所有的 Native 代码的耗时，有时候一些 Android 系统库的调用对分析问题有比较大的帮助，例如加载 dex、verify class 的耗时等。

**Simpleperf 同时封装了 systrace 的监控功能**，通过 Android 几个版本的优化，现在 Simpleperf 比较友好地支持 Java 代码的性能分析。具体来说分几个阶段：
- 第一个阶段：在 Android M 和以前，Simpleperf 不支持 Java 代码分析。
- 第二个阶段：在 Android O 和以前，需要手动指定编译 OAT 文件。
- 第三个阶段：在 Android P 和以后，无需做任何事情，Simpleperf 就可以支持 Java 代码分析。

图形界面使用的 [FlameChart](http://www.brendangregg.com/flamegraphs.html)

![FlameChart](https://i.loli.net/2019/12/14/rHIXZ8xdvcsFALu.png)

它跟 Call Chart 不同的是，Flame Chart 以一个全局的视野来看待一段时间的调用分布，它就像给应用程序拍 X 光片，**可以很自然地把时间和空间两个维度上的信息融合在一张图上**。

#### 3. AS 自带的 Profiler
![Profiler](https://i.loli.net/2019/12/14/jNoFw2K7cPnUOIz.png)

在 Android Studio 3.2 的 Profiler 中直接集成了几种性能分析工具，其中：
- Sample Java Methods
  - 抽样的方式观察函数调用过程
- Trace Java Methods
  - 采集一段时间的 Java 方法的执行
- Trace System Calls 的功能类似于 systrace。
  - 采集 Trace.beginSection {...} Trace.endSection 埋点的执行时间
  - 类似于 systrace, 常用与监控 Android 源码的执行耗时
- SampleNative  (API Level 26+) 
  - 采集 Native 执行堆栈,  功能类似于 Simpleperf

![定位耗时方法](https://i.loli.net/2019/12/14/Tq1kNn54oIpmLiD.png)

除此之外还有 Uber 开源的 [Nanoscope](https://github.com/uber/nanoscope), 它通过直接修改 Android 虚拟机源码, 在ArtMethod执行入口和执行结束位置增加埋点代码, 将所有的信息先写到内存, 等到 trace 结束后才统一生成结果文件

### 二) 代码监控
使用代码监控卡顿, 的可选方式如下

#### 1. Choreographer
```
Choreographer.getInstance().postFrameCallback(new Choreographer.FrameCallback() {
    @Override
    public void doFrame(long frameTimeNanos) {
        // 统计与上一帧的差值, 超过阈值, 则认为出现了丢帧
        
        // 添加下一次回调
        Choreographer.getInstance().postFrameCallback(xxx);
    }
});
```
Choreographer 是 Android 4.1 引入的配合 VSYNC 共同工作的应用层渲染编舞者, 通过 postFrameCallback 我们可以向其类型为 CALLBACK_ANIMATION 的 CallbackQueue 中插入一条待执行指令

根据两次渲染时间的间隔来判断是否出现了丢帧

#### 2. Looper
```
final Printer logging = me.mLogging;
if (logging != null) {
    logging.println(">>>>> Dispatching to " + msg.target + " " +
            msg.callback + ": " + msg.what);
}
......
msg.target.dispatchMessage(msg);
......
if (logging != null) {
    logging.println("<<<<< Finished to " + msg.target + " " + msg.callback);
}
```
我们知道, 我们的 App 进程被 fork 出来之后, 会在 main 方法中启动 Looper, 此后所有的任务均由 Looper 从对应的 MessageQueue 中取消息执行, 我们可以通过为 Looper 注入 Printer 来 hook 消息执行的开始和结束

当一个 Message 执行时间超过了 16.6 ms, 我们就可以近似的认为它出现了丢帧的情况, 使用这种方式我们同样可以进行 ANR 的监控

具体的实现策略可以参考 https://blog.csdn.net/lmj623565791/article/details/58626355

### 三) 回顾
卡顿监控, 即丢帧的监控, 可选方式如下
- **工具监控: 监控范围广, 覆盖面全, 甚至可以监控系统的执行耗时, 难以部署到线上**
    - 如果需要分析 Native 代码的耗时, 可以选择 Simpleperf
    - 如果想分析 Android 系统代码执行耗时, 可以选择 systrace
    - 如果想分析整个程序执行流程的耗时, 可以选择插桩版本的 systrace
    - 若对监控精细度要求不高, 可以使用 AS 自带的 Profiler 进行监控 
- **代码监控: 运行时监控的性能消耗低, 可以部署到线上, 仅限 App 内部的卡顿监控**
  - Choreographer: 监控两帧准备的间隔
  - Looper: 通过监听主线程消息执行时长

## 四. 信息的采集
监控到了卡顿仅做到了第一步, 想要确认卡顿的原因进而进行优化, 还需要在出现卡顿时进行信息采集; 产生卡顿的原因多种多样, 即使能够定位到卡顿时的堆栈信息, 但也有可能是因为当时 CPU 负载过高, 内存不足 GC 频繁时导致的, 因此需要多方面采集
- 基本的设备信息
- 卡顿时主线程的堆栈
- CPU 负载信息
- 内存使用信息
- 帧率统计

关于设备的基本信息这里就不赘述了, 主要看看其他信息的监控

### 一) 当前页面卡顿信息
统计每个 Activity 的平均帧率和掉帧率, 方便问题的定位和针对性优化, 采集之后, 我们可以计算这个页面的卡顿率

#### 1. 帧率
当一帧准备完成之后, 我们可以通过以下方式计算帧率
```
// 记录当前帧准备耗时
long cost = dispatchEnd - dispatchStart;
// 记录绘制时间消耗总和
sumFrameCost += cost;
// 统计总帧数
sumFrameCount++;
// 计算平均每帧耗时
long averageCost = sumFrameCost / sumFrameCount;
// FPS = 1000 ms / 平均每帧耗时
FPS = 1000f / averageCost;
```

#### 2. 丢帧率
```
// 记录当前帧准备耗时
long cost = dispatchEnd - dispatchStart;
// 丢帧数量 = 当前帧准备耗时 / 屏幕刷新周期(60 Hz 屏幕为 16 ms)
int dropFrames = (int) (cost / 16f);
// 统计丢帧的次数
if (dropFrames > 0) {
    sumDropCount++;
}
// 记录采集次数
sumSampleCount++;
// 丢帧率 = 丢帧次数 / 采集次数
JankRatio = 100f * (sumDropCount / sumSampleCount)
```

#### 3. 卡顿等级
为了衡量卡顿程度, 我们需要将每秒的掉帧数划分出几个区间进行定级, 微信团队的 Matrix 采用了如下的方式进行定级

Best | Normal | Middle | High | Frozen
:---:|:---:|:---:|:---:|:---:
[0:3) | [3:9) | [9:24) | [24:42) | [42:∞)

- 每秒丢帧 0~3: 极度流畅
- 每秒丢帧 3~9: 流畅
- 每秒丢帧 9~24: 略有卡顿
- 每秒丢帧 24~42: 卡顿
- 每秒丢帧 42+: GG

计算之后我们可以得到下面的数据
```
>>>>>>>>>>>>>>>>>>>>>>>>>>>>> FpsTracker <<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<
Scene: com.sharry.lib.album.PickerActivity
FPS: 60.00
Jank Ratio: 3.95%
Drop Level: DROPPED_BEST: 739, DROPPED_NORMAL: 2, DROPPED_MIDDLE: 1, DROPPED_HIGH: 0, DROPPED_FROZEN: 0
Drop Sum: DROPPED_BEST: 12, DROPPED_NORMAL: 12, DROPPED_MIDDLE: 13, DROPPED_HIGH: 0, DROPPED_FROZEN: 0
```

### 二) 主线程堆栈信息
若是因为我们代码编写不当导致了主线程的耗时, 能够直接定位产生卡顿时的代码就非常容易定位问题了, 我们可以采用 AMS 中 ANR 监听的方式来监听卡顿
```
private Runnable mDumpBlockInfoRunnable = new Runnable() {
    @Override
    public void run() {
        StringBuilder builder = new StringBuilder("UI thread occurred evil method, execute time over than " + mAnrDuration + "ms \n");
        builder.append(">>>>>>>>>>>>>>>>>>>>>>>>>>>>> StackTrace <<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<\n");
        // Print main thread stack trace.
        StackTraceElement[] stackTrace = Looper.getMainLooper().getThread().getStackTrace();
        for (StackTraceElement s : stackTrace) {
            builder.append(s.toString());
            builder.append("\n");
        }
        builder.append("\n");
        String blockInfo = builder.toString();
        if (mListener != null) {
            mListener.onAnrOccurred(mAnrDuration, blockInfo);
        }
    }
};

Looper.getMainLooper().setMessageLogging(new Printer() {
    @Override
    public void println(String x) {
        // 消息分发开始
        if (x.charAt(0) == '>') {
            mMonitorThread.getHandler().post(mDumpBlockInfoRunnable, mAnrDuration);
        } 
        // 消息分发结束
        else if (x.charAt(0) == '<') {
            mMonitorThread.getHandler().removeCallbacks(mDumpBlockInfoRunnable);
        }
    }
});
```
这里以采用 Looper 监听方法耗时来举例, 我们可以先创建一个监控线程 HandlerThread 对象 mMonitorThread
- 埋炸弹: 在主线程消息分发开始时向监控线程的消息队列中发送一条延时消息
  - 这个消息时长可以自定义
- 拆炸弹: 在主线程消息分发结束之后, 从监控线程的消息队列中将这个消息移除 

若是主线程出现了耗时操作, 那么我们监控线程的消息队列就会执行我们预埋的消息, 通过 mDumpBlockInfoRunnable 就可以将当前主线程执行的堆栈打印出来了, 效果如下
```
>>>>>>>>>>>>>>>>>>>>>>>>>>>>> StackTrace <<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<
java.lang.Thread.sleep(Native Method)
java.lang.Thread.sleep(Thread.java:373)
java.lang.Thread.sleep(Thread.java:314)
// 卡顿位置
com.sharry.app.slibrary.sample2.Sample2Activity.testANR(Sample2Activity.java:81)
com.sharry.app.slibrary.sample2.Sample2Activity.initViews(Sample2Activity.java:70)
com.sharry.lib.framework.base.BaseActivity.onCreate(BaseActivity.java:36)
com.sharry.module.base.framework.mvp.BaseMvpActivity.onCreate(BaseMvpActivity.java:33)
android.app.Activity.performCreate(Activity.java:7224)
android.app.Activity.performCreate(Activity.java:7213)
android.app.Instrumentation.callActivityOnCreate(Instrumentation.java:1272)
android.app.ActivityThread.performLaunchActivity(ActivityThread.java:2926)
android.app.ActivityThread.handleLaunchActivity(ActivityThread.java:3081)
android.app.servertransaction.LaunchActivityItem.execute(LaunchActivityItem.java:78)
android.app.servertransaction.TransactionExecutor.executeCallbacks(TransactionExecutor.java:108)
android.app.servertransaction.TransactionExecutor.execute(TransactionExecutor.java:68)
android.app.ActivityThread$H.handleMessage(ActivityThread.java:1831)
android.os.Handler.dispatchMessage(Handler.java:106)
......
```

#### 使用 AMS 优化
上面的方式虽然能够巧妙的定位卡顿代码位置, 但它的不足之处是: **难以精确定位**

![执行示意图](https://i.loli.net/2019/12/14/ujQD5fpiPOHqeJv.png)

上图是监控主线程 2s 的卡顿, 可以看到当预埋的炸弹爆破时, 最终命中的堆栈是 B 方法, 当时 A 方法才是引起耗时操作的罪魁祸首, 如何解决这样的问题呢?

[Matrix](https://github.com/Tencent/matrix/wiki/Matrix-Android-TraceCanary) 通过 AMS 字节码注入的方式解决了这个问题
- **编译时**: 为每个方法进行插桩操作, 方便后续在运行时统计函数执行时间
- **运行时**: 出现卡顿时, 遍历 dispatchMessage 执行期间所有函数的执行时间, 找寻最大的即为耗时的罪魁祸首

通过这样的方式, 我们就可以精准定位到造成卡顿的元凶了, **当然这种方式也不是没有副作用, 首先实现成本会比较高, 其次会产生而外的内存开销**, Matrix 使用 long[] 数组作为 buffer, 内存占用约 7.6M

我们可以根据自身的需求, 灵活的选择实现方式, 若是对精准定位要求不高, 直接 dump 堆栈信息也未尝不可, 不过 dump 堆栈信息是一项耗时操作, 从紹文老师的文章中了解到, 可以使用 Facebook 开源库 [Profilo](http://github.com/facebookincubator/profilo), 更加优雅快速的 dump 堆栈信息, 感兴趣的可以了解一下

### 三) CPU 负载信息
熟悉 Linux 的内核机制的朋友肯定知道, 关于我们进程的运行状态信息, 都会存储到 /prco/ 这个文件夹中, 它由内存文件系统实现, 磁盘中并不是真是存在的, 我们可以从中读取信息

```
/proc/stat                   // 系统 CPU 使用率
/proc/loadavg                // 系统平均负载
/proc/[pid]/stat             // 进程CPU使用情况
/proc/[pid]/task/[tid]/stat  // 进程下面各个线程的CPU使用情况
/proc/[pid]/sched            // 进程CPU调度相关
```

#### 1. 使用率计算
关于 CPU 信息参数获取可以参考  http://www.samirchen.com/linux-cpu-performance/, 这里以系统 CPU 使用率 和 进程 CPU 使用率举例
##### 计算系统 CPU 的使用率
采样两个足够短的时间间隔的 CPU 快照, 即读取 /proc/stat 文件, 获取两个时间点的下列数据
```
SysCpuTime1 = user1 + nice1 + system1 + idle1 + iowait1 + irq1 + softirq1 + stealstolen1 + guest1;
SysCpuTime2 = user2 + nice2 + system2 + idle2 + iowait2 + irq2 + softirq2 + stealstolen2 + guest2;

// 计算两个节点的总使用的相对时间
RelSysTotalTime = SysCpuTime1 – SysCpuTime2;

// 计算两个节点的相对空闲时间
RelSysIdleTime = idle2 – idle1;

// CPU 使用率 = (CPU 使用总时间 - 空闲实现) / CPU 使用总时间
SystemRatio = (RelSysTotalTime – RelSysIdleTime) / RelSysTotalTime;
```
计算方式也很简单, 使用上面的方式就可以计算出 CPU 负载信息了

##### 计算进程 CPU 使用率
采集两个足够短的时间间隔的 CPU 快照, 读取 /proc/self/stat 文件
```
ProcCupTime1 = utime1 + stime1 + cutime1 + cstime1;
ProcCupTime2 = utime2 + stime2 + cutime1 + cstime2;

// 计算两个节点的进程对 CPU 使用的相对时间
RelProcTotalTime = ProcCupTime1 - ProcCupTime2;

// 进程对 CPU 的使用率 = 进程相对使用时间 / CPU使用总时间
ProcRatio = RelProcTotalTime / RelSysTotalTime
```
可以看到进程 CPU 使用率的计算也比较简单, 线程计算方式与进程计算一致, 就不再赘述了

#### 2. 实现
了解了 CPU 使用率计算的方式, 我们就可以采集我们想要的信息了, 方便在卡顿时 dump 出来

关于如何读取想要的信息我们可以参考 [Android 的 ProcessCpuTracker ](http://androidxref.com/9.0.0_r3/xref/frameworks/base/core/java/com/android/internal/os/ProcessCpuTracker.java), 获取到的信息如下
```
>>>>>>>>>>>>>>>>>>>>>>>>>>>>> ProcessCpuTracker <<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<
usage: CPU usage 240ms (2019-12-06 14:53:01.819 to 2019-12-06 14:53:02.059):
// 系统 CPU 负载信息
System TOTAL: 0% System(R): 0% user + 0% kernel
Load Average: 0.0 / 0.0 / 0.0
CPU Core: 8

// 进程 CPU 信息
Process: com.sharry.app.slibrary
100% 18270/com.sharry.app.slibrary(R): 88% user + 11% kernel, faults: 603 minor
Threads: 
34% 18270/ry.app.slibrary(R): 27% user + 6.9% kernel, faults: 255 minor
25% 18302/UIThreadMonitor(R): 20% user + 4.6% kernel, faults: 28 minor
4.6% 18314/RenderThread(S): 2.3% user + 2.3% kernel, faults: 303 minor
2.3% 18283/HeapTaskDaemon(S): 2.3% user + 0% kernel, faults: 14 minor
2.3% 18277/Jit thread pool(S): 2.3% user + 0% kernel, faults: 216 minor
0% 18304/Binder:intercep(S): 0% user + 0% kernel
0% 18285/Binder:18270_2(S): 0% user + 0% kernel
0% 18278/Signal Catcher(S): 0% user + 0% kernel
0% 18279/ADB-JDWP Connec(S): 0% user + 0% kernel
0% 18280/ReferenceQueueD(S): 0% user + 0% kernel
0% 18281/FinalizerDaemon(S): 0% user + 0% kernel, faults: 1 minor
0% 18282/FinalizerWatchd(S): 0% user + 0% kernel
0% 18284/Binder:18270_1(S): 0% user + 0% kernel
0% 18293/Binder:18270_3(S): 0% user + 0% kernel
0% 18297/Binder:18270_4(S): 0% user + 0% kernel, faults: 6 minor
0% 18299/Profile Saver(S): 0% user + 0% kernel
0% 18301/queued-work-loo(S): 0% user + 0% kernel
0% 18319/ConnectivityThr(S): 0% user + 0% kernel, faults: 6 minor
0% 18321/process reaper(S): 0% user + 0% kernel, faults: 7 minor
```
可以看到日志中包含两个部分的信息, 一个是系统的 CPU 使用率和负载, 另一个是当前进程和线程的使用率

其中系统 CPU 的使用率和负载是没有数据的, 这是**因为 Android6.0 之后出于安全考虑, 我们 App 在运行时是无权查看 /proc/ 下的文件信息的了, 因此我们只能退而求其次, 采集 /proc/self/xxx 下当前进程的信息**

我们可以统计出一个线程在当前进程中的使用率, 一样是能够帮助我们定位到发生卡顿的线程的, 其中需要注意的字段如下
- utime: CPU 在用户态执行时长
- stime: CPU 在内核态执行时长
- minor faults: 不需要从硬盘拷数据而发生的缺页次数
- major faults: 需要从硬盘拷数据而发生的缺页次数 

### 四) 内存信息
用户在前台的时候，可以每 5 分钟采集一次 PSS、Java 堆、图片总内存。我建议通过采样只统计部分用户，需要注意的是要按照用户抽样，而不是按次抽样。简单来说一个用户如果命中采集，那么在一天内都要持续采集数据。

#### 1. 内存触顶率
触顶率可以反映 Java 内存的使用情况, 如果超过 85% 最大堆限制, GC 会变得更加频繁, 容易造成 OOM 和卡顿。
```
内存 UV 触顶率 = Java 堆占用超过最大堆限制的 85% 的 UV / 采集 UV
```
是否触顶的计算方式
```
Runtime runtime = Runtime.getRuntime();
long javaMax = runtime.maxMemory();
long javaTotal = runtime.totalMemory();
long javaUsed = javaTotal - runtime.freeMemory();
// Java 内存使用超过最大限制的 85%
float proportion = (float) javaUsed / javaMax;
```

#### 2. GC 监控
```
// 运行的 GC 次数
Debug.getRuntimeStat("art.gc.gc-count");
// GC 使用的总耗时，单位是毫秒
Debug.getRuntimeStat("art.gc.gc-time");
// 阻塞式 GC 的次数
Debug.getRuntimeStat("art.gc.blocking-gc-count");
// 阻塞式 GC 的总耗时
Debug.getRuntimeStat("art.gc.blocking-gc-time");
```

#### 3. Bitmap 使用情况
版本 | 变更
---|---
Android 3.0 之前 | Bitmap 中的 pixels 存放在 Native, 释放时机依赖于 recycle 的调用
Android 3.0 ~ Android 7.0 之间 | Bitmap 中的 pixels 放在 Java 堆, 释放时机依赖于 finalize 方法
Andorid 8.0 之后 | Bitmap 中的 pixels 存放在 Native, 使用 NativeAllocationRegistry 辅助回收内存

虽然 Android 8.0 之后放置到了 Native 内存, 但依旧是内存消耗大户, **我们可以通过收拢 Bitmap 创建接口, 无侵入性的获取拦截 Bitmap 创建数据**
- 统计 Bitmap 创建数据和内存消耗总量
- 出现超宽 Bitmap, 弹窗提醒开发者进行修正
- **出现卡顿或者崩溃时, 将图片占用的总内存、Top N 图片的内存都写到崩溃日志中，帮助我们排查问题。**

如下所示
```
Bitmap status: 
    Total bitmap count: 74, Large Bitmap count: 0, rate 0.00% 
    Total bitmap allocation memory: 65MB, Large bitmap allocation memory: 0MB 
Top 10 Bitmaps: 
    android.graphics.Bitmap@e4ab75e size is [1080, 1440], allocation memory is 5.93MB 
    android.graphics.Bitmap@d334899 size is [1080, 1440], allocation memory is 5.93MB 
    android.graphics.Bitmap@d334899 size is [1080, 1440], allocation memory is 5.93MB 
    android.graphics.Bitmap@9e8bde0 size is [1080, 1080], allocation memory is 4.45MB 
    android.graphics.Bitmap@2772fe3 size is [1080, 1080], allocation memory is 4.45MB 
    android.graphics.Bitmap@2772fe3 size is [1080, 1080], allocation memory is 4.45MB 
    android.graphics.Bitmap@37dec12 size is [354, 354], allocation memory is 2.80MB 
    android.graphics.Bitmap@9dc4f9d size is [354, 354], allocation memory is 2.80MB 
```

### 回顾
采集到了上面的信息, 就很方便帮助我们定位问题了, 我们可以将卡顿的数据发布到线上, 然后按照卡顿率计算 TopK 的页面, 每个页面中维护该页面卡顿信息的集合, 如下所示

![卡顿信息采集](https://i.loli.net/2019/12/14/zgl61orQbvjHRNG.png)

## 总结
本篇文章主要阐述了如下的内容
- 卡顿的本质
  - 连续的丢帧 
- 卡顿的监控
  - **工具监控: 监控范围广, 覆盖面全, 甚至可以监控系统的执行耗时, 难以部署到线上**
    - 如果需要分析 Native 代码的耗时, 可以选择 Simpleperf
    - 如果想分析 Android 系统代码执行耗时, 可以选择 systrace
    - 如果想分析整个程序执行流程的耗时, 可以选择插桩版本的 systrace
    - 若对监控精细度要求不高, 可以使用 AS 自带的 Profiler 进行监控 
  - **代码监控: 运行时监控的性能消耗低, 可以部署到线上, 仅限 App 内部的卡顿监控**
    - Choreographer
    - Looper
- 卡顿时信息的采集
  - 设备基本信息 
  - 线程堆栈信息
  - CPU 负载
  - 内存使用情况
  - 当前页面卡顿信息

卡顿的优化并不是一个简单的任务, 每一个产生卡顿的子项内部都大有学问
- 若是因为 UI 布局复杂导致卡顿, 我们需要通过优化 UI 布局来解决卡顿问题
- 若是因为我们的 UI 线程 IO 操作不当导致卡顿, 我们可以使用并发编程的思路来解决卡顿问题
- 若是因为内存占用过高导致卡顿, 我们需要分析内存使用情况, 进而优化内存从而解决卡顿问题
- ......

不过有的时候发现问题可能会比解决问题更加困难, 关于每一项的优化方案, 后面有时间再补充吧 

## 参考
- [CPU 占用率](http://www.samirchen.com/linux-cpu-performance/)
- [Matrix 卡顿监控](https://github.com/Tencent/matrix/wiki/Matrix-Android-TraceCanary)