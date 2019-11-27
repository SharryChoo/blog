---
layout: article
title: Android 系统架构 —— View 的测量
permalink: android-source/graphic-producer4
key: android-source-graphic-producer4
tags: AndroidFramework
---

## 前言
通过上面一篇的分析, 我们得知了 ViewRootImpl 的重要性, 这里我们接着往下分析, 看看 View 的布局操作
```
public final class ViewRootImpl implements ViewParent,
        View.AttachInfo.Callbacks, ThreadedRenderer.DrawCallbacks {

    public void setView(View view, WindowManager.LayoutParams attrs, View panelParentView) {
        synchronized (this) {
            if (mView == null) {
                mView = view;
                // 执行 View 绘制的三大流程
                requestLayout();
                ......
                // 将 W 添加到 WMS, 在上次已经分析过了
            }
        }
    }
    
}
```

<!--more-->

可以看到 ViewRootImpl 的 setView 方法中, 调用了我们非常熟悉的 requestLayout(), 这个方法在我们请求重新布局时经常会用到, 我们这次主要分析这个方法
```
public final class ViewRootImpl implements ViewParent,
        View.AttachInfo.Callbacks, ThreadedRenderer.DrawCallbacks {
    
    // 用于区分是否需要进行 View 的布局
    boolean mLayoutRequested;
    
    @Override
    public void requestLayout() {
        if (!mHandlingLayoutInLayoutRequest) {
            // 1. 检查请求 requestLayout 的线程
            checkThread();
            // 2. 表示接当前 View 树的布局被请求了
            mLayoutRequested = true;
            // 3. 调用了 scheduleTraversals
            scheduleTraversals();
        }
    }
    
    final Thread mThread;
   
    public ViewRootImpl(Context context, Display display) {
        ......
        mThread = Thread.currentThread();
        ......
    }
   
    // 检查当前的线程是否为 ViewRootImpl 创建的线程
    void checkThread() {
        // 1. 若非 ViewRootImpl 创建的线程会扔出异常
        // 因此 View 只能在主线程更新 UI 是伪命题, 它只能在其 ViewRootImpl 创建的线程更新 UI
        if (mThread != Thread.currentThread()) {
            throw new CalledFromWrongThreadException(
                    "Only the original thread that created a view hierarchy can touch its views.");
        }
    }
    
}
```
好的, 可以看到 ViewRootImpl 的 requestLayout 方法中做了如下几件事情
- 检查请求 requestLayout 的线程是否为 ViewRootImpl 创建的线程, 若非 ViewRootImpl 创建的线程会扔出异常
  - **因此 View 只能在主线程更新 UI 是伪命题, 它只能在其 ViewRootImpl 创建的线程更新 UI**
- 将 mLayoutRequested flag 标记为 true
  - 这个标记开启说名 View 是否需要重新布局
- 调用了 ViewRootImpl.scheduleTraversals

从 requestLayout 中, 我们明确了为什么 View 只能在主线程更新 UI 是个伪命题, 结下来继续分析 scheduleTraversals,  看看它为 View 的遍历做了哪些计划
```
public final class ViewRootImpl implements ViewParent,
        View.AttachInfo.Callbacks, ThreadedRenderer.DrawCallbacks {
    
    public boolean mTraversalScheduled;
    final TraversalRunnable mTraversalRunnable = new TraversalRunnable();
    
    void scheduleTraversals() {
        if (!mTraversalScheduled) {
            // 表示当前正在计划 View 的遍历
            mTraversalScheduled = true;
            // 插入同步屏障
            mTraversalBarrier = mHandler.getLooper().getQueue().postSyncBarrier();
            // post 到 Looper 的 MessageQueue 中
            mChoreographer.postCallback(
                    Choreographer.CALLBACK_TRAVERSAL, mTraversalRunnable, null);
            ......
        }
    }
    
    final class TraversalRunnable implements Runnable {
        @Override
        public void run() {
            doTraversal();
        }
    }
    
    void doTraversal() {
        if (mTraversalScheduled) {
            // 走到这里, 说明遍历的计划阶段已经完成, 即将进入执行遍历的阶段
            mTraversalScheduled = false;
            // 移除同步屏障
            mHandler.getLooper().getQueue().removeSyncBarrier(mTraversalBarrier);
            ......
            // 执行 View 的遍历
            performTraversals();
            ......
        }
    }
}
```
可以看到, 当计划成功之后, 它调用了 performTraversals, 进入了 View 遍历的执行阶段

## 一. ViewRootImpl.performTraversals
```
public final class ViewRootImpl implements ViewParent,
        View.AttachInfo.Callbacks, ThreadedRenderer.DrawCallbacks {
    
    @NonNull Display mDisplay;   // 描述屏幕的尺寸
    boolean mFirst;              // 描述是否是第一次进行 performTraversals 
    final Rect mWinFrame;        // frame given by window manager.

    public ViewRootImpl(Context context, Display display) {
        mDisplay = display;
        ......
        mWinFrame = new Rect();
        ......
        mFirst = true;
        ......
    }
    
    
    private void performTraversals() {
        // 获取 DecorView
        // 编程小技巧, 若一个成员变量在该方法中使用的比较多, 使用局部变量保存, 防止变更
        final View host = mView;
        
        WindowManager.LayoutParams lp = mWindowAttributes;   // 获取这个 DecorView 的布局参数
        boolean windowSizeMayChange = false;                 // 用于判断窗体大小是否有可能发生改变
        
        // 1. 用于设置窗体期望尺寸
        int desiredWindowWidth;
        int desiredWindowHeight;
        Rect frame = mWinFrame;
        // 1.1 若该 ViewRootImpl 第一次 Traversals
        if (mFirst) {
            ......
            mLayoutRequested = true;
            final Configuration config = mContext.getResources().getConfiguration();
            // 1.1.1 使用屏幕物理宽高作为窗体期望尺寸
            if (shouldUseDisplaySize(lp)) {
                Point size = new Point();
                mDisplay.getRealSize(size);
                desiredWindowWidth = size.x;
                desiredWindowHeight = size.y;
            } else {
                // 1.1.2 则使用 mWindFrame 中的宽高作为窗体期望尺寸
                desiredWindowWidth = mWinFrame.width();
                desiredWindowHeight = mWinFrame.height();
                ......
            }
            ......
        } else {
            // 1.2 若非第一次进行 Traversals, 那么窗体期望尺寸直接使用 frame(mWinFrame) 中记录的宽高
            desiredWindowWidth = frame.width();
            desiredWindowHeight = frame.height();
            ......
        }
        ......
        // Traversals 中的 layout 有两个部分, 分别是 measure 和 layout
        boolean layoutRequested = mLayoutRequested && (!mStopped || mReportNextDraw);
        if (layoutRequested) {
            final Resources res = mView.getContext().getResources();
            ......
             if (mFirst) {
                ......// 第一次的尺寸已经在上面设置过了, 这里不会进行窗体期望尺寸的设置操作
            } else {
                ......
                // 1.3 若 LayoutParams 为 WRAP_CONTENT, 那么会再次进行窗体期望尺寸的判断
                if (lp.width == ViewGroup.LayoutParams.WRAP_CONTENT
                        || lp.height == ViewGroup.LayoutParams.WRAP_CONTENT) {
                    windowSizeMayChange = true;
                    // 1.3.1 使用屏幕物理宽高作为窗体期望尺寸
                    if (shouldUseDisplaySize(lp)) {
                        Point size = new Point();
                        mDisplay.getRealSize(size);
                        desiredWindowWidth = size.x;
                        desiredWindowHeight = size.y;
                    } else { 
                        // 1.3.2 使用屏幕可用宽高作为窗体期望尺寸
                        Configuration config = res.getConfiguration();
                        desiredWindowWidth = dipToPx(config.screenWidthDp);
                        desiredWindowHeight = dipToPx(config.screenHeightDp);
                    }
                }
            }
            // 2. 调用 measureHierarchy 方法, 为了确定其布局位置, 先确定它的大小
            windowSizeMayChange |= measureHierarchy(host, lp, res,
                    desiredWindowWidth, desiredWindowHeight);
        }
        // 这个标记为用于判断是否开启了 fitSystemWindow
        if (mApplyInsetsRequested) {
            mApplyInsetsRequested = false;
            ......
            // 2.1 若设置了 fitSystemWindow , 则重新确定 View 大小
            if (mLayoutRequested) {
                windowSizeMayChange |= measureHierarchy(host, lp,
                        mView.getContext().getResources(),
                        desiredWindowWidth, desiredWindowHeight);
            }
        }
        ......
        
        // 3. 根据测量尺寸, 重置 ViewRootImpl 的 Surface 画布
        ......
        if (mFirst || windowShouldResize || insetsChanged ||
                viewVisibilityChanged || params != null || mForceNextWindowRelayout) {
            ......
            boolean hadSurface = mSurface.isValid();
            try {
                ......
                // 3.1 调用了 relayout window 重新设置窗体大小
                relayoutResult = relayoutWindow(params, viewVisibility, insetsPending);
                ......
            }
        }
        ......
        if (didLayout) {
            // 4. 进行视图的摆放
            performLayout(lp, mWidth, mHeight);
            ......
        }
        // 5. 进行视图的绘制
        if (!cancelDraw && !newSurface) {
            ......
            performDraw();
        }
    }
   
    // LayoutParams 若为如下三种, 则直接使用屏幕的大小       
    private static boolean shouldUseDisplaySize(final WindowManager.LayoutParams lp) {
        return lp.type == TYPE_STATUS_BAR_PANEL
                || lp.type == TYPE_INPUT_METHOD
                || lp.type == TYPE_VOLUME_OVERLAY;
    }   
}
```
好的, 可以看到 performTraversals 中便是 View 绘制的流程了, 中间简化了很多代码, 主要有以下几步
- **获取窗体期望尺寸**
- 将窗体期望尺寸传入**measureHierarchy** 方法, 进行 View 测量操作
- 当测量数据发生改变时, 调用 relayoutWindow 方法, 重新请求窗体大小
- 调用 performLayout(lp, mWidth, mHeight), 进行布局操作
- 调用 performDraw 进行视图的绘制

这里我们主要分析 View 的 measure 操作, 因此首先要关注的是如何获取窗体期望尺寸

## 二. 获取窗体期望尺寸
从上面的代码中, 我们可以看到若为第一次执行 performTraversals 时, 在 LayoutParams 不满足的情况下, 会使用 mWinFrame 中记录大小
```
public final class ViewRootImpl implements ViewParent,
        View.AttachInfo.Callbacks, ThreadedRenderer.DrawCallbacks {
    
    @NonNull Display mDisplay;   // 描述屏幕的尺寸
    boolean mFirst;              // 描述是否是第一次进行 performTraversals 
    final Rect mWinFrame;        // frame given by window manager.

    public ViewRootImpl(Context context, Display display) {
        mDisplay = display;
        ......
        mWinFrame = new Rect();
        ......
        mFirst = true;
        ......
    }
    
}
```
从构造函数中可以看到, 这个 mWinFrame 是一个空的 Rect 对象, 那么这里又是第一次进行 performTraversals 操作, **难道直接给 Window 期望的尺寸赋 0 值了吗?** 这很显然是不合理, 否则 match_parent 类型的 DecorView 则无法正常显示, 因此我们需要弄清楚它是如何赋值的

通过 ViewRootImpl 与 WMS 交互的分析, 我们知道在 ViewRootImpl.setView 中有如下两步操作
```
public final class ViewRootImpl implements ViewParent,
        View.AttachInfo.Callbacks, ThreadedRenderer.DrawCallbacks {
    
    public void setView(View view, WindowManager.LayoutParams attrs, View panelParentView) {
        synchronized (this) {
            if (mView == null) {
                // 调用 requestLayout 执行 View 绘制的三大流程
                requestLayout();
                ......
                try {
                    ......
                    // 将窗体信息添加到 WMS 中
                    res = mWindowSession.addToDisplay(......);
                } catch (RemoteException e) {
                    ......
                } 
            }
        }
    }
    
}
```
通过上面的分析 requestLayout 最终调用的 performTraversals 是 post 到主线程队列中执行的, **因此 mWindowSession.addToDisplay 这个与 WMS 建立联系的操作会先于 performTraversals 执行, 因此在我们执行首次 performTraversals 的时候, WinFrame 中已经保存了 WMS 计算好的窗体数据了**

## 三. 构建 Window 的 MeasureSpec
```
public final class ViewRootImpl implements ViewParent,
        View.AttachInfo.Callbacks, ThreadedRenderer.DrawCallbacks {
            
    private boolean measureHierarchy(final View host, final WindowManager.LayoutParams lp,
            final Resources res, final int desiredWindowWidth, final int desiredWindowHeight) {
        
        int childWidthMeasureSpec;                // 描述 Window 的宽度测量说明书
        int childHeightMeasureSpec;               // 描述 Window 的高度测量说明书
        boolean windowSizeMayChange = false;      // 描述 Window 尺寸是否变化
        boolean goodMeasure = false;              // 判断是否是好的测量
        
        // 1. 对 Window 布局参数为 WRAP_CONTENT 的情况进行特殊处理
        if (lp.width == ViewGroup.LayoutParams.WRAP_CONTENT) {
            // 则分配一个默认的宽度
            final DisplayMetrics packageMetrics = res.getDisplayMetrics();
            // 1.1 从系统文件 config_prefDialogWidth 中读取这个预制的宽度
            res.getValue(com.android.internal.R.dimen.config_prefDialogWidth, mTmpValue, true);
            int baseSize = 0;
            if (mTmpValue.type == TypedValue.TYPE_DIMENSION) {
                baseSize = (int)mTmpValue.getDimension(packageMetrics);
            }
            // 1.2 获取系统提供的宽度成功了, 但期望的窗体宽度比这个大, 则优先使用 baseSize
            // 若期望的比它小, 则直接使用期望的宽高即可
            if (baseSize != 0 && desiredWindowWidth > baseSize) {
                childWidthMeasureSpec = getRootMeasureSpec(baseSize, lp.width);              // 构建宽度测量说明书
                childHeightMeasureSpec = getRootMeasureSpec(desiredWindowHeight, lp.height); // 构建高度测量说明书
                performMeasure(childWidthMeasureSpec, childHeightMeasureSpec);               // 进行子 View 的测量操作
                // 1.4 若 DecorView 的状态值中发现大小合适, 说明是好的尺寸
                if ((host.getMeasuredWidthAndState()&View.MEASURED_STATE_TOO_SMALL) == 0) {
                    goodMeasure = true;
                } else {
                    // 走到这里, 说明上面的测量不太合适, 通过下面的算法, 再次构建 baseSize
                    baseSize = (baseSize+desiredWindowWidth)/2;
                    childWidthMeasureSpec = getRootMeasureSpec(baseSize, lp.width);// 构建测量说明书
                    performMeasure(childWidthMeasureSpec, childHeightMeasureSpec);  // 执行测量操作
                    // 再次判断是否是好的尺寸
                    if ((host.getMeasuredWidthAndState()&View.MEASURED_STATE_TOO_SMALL) == 0) {
                        goodMeasure = true;
                    }
                }
            }
        }
        // 2. 若非好的测量尺寸
        if (!goodMeasure) {
            // 2.1 通过 Window 期望的宽高和布局参数构建测量说明书
            childWidthMeasureSpec = getRootMeasureSpec(desiredWindowWidth, lp.width);
            childHeightMeasureSpec = getRootMeasureSpec(desiredWindowHeight, lp.height);
            // 2.2 执行测量操作
            performMeasure(childWidthMeasureSpec, childHeightMeasureSpec);
            if (mWidth != host.getMeasuredWidth() || mHeight != host.getMeasuredHeight()) {
                windowSizeMayChange = true;
            }
        }
        return windowSizeMayChange;
                     
   }
   
}
```
好的, 可以看到 measureHierarchy 中构建 Window 的测量说明书的过程非常有趣
- 若窗体宽度的 LayoutParam 为 WRAP_CONTENT
  - 从系统提供的资源文件中的  com.android.internal.R.dimen.config_prefDialogWidth 中读取一个最小的宽度, 记为 baseSize
  - 若 baseSize 有效并且比我们期望的窗体尺寸小
    - 使用 baseSize 为宽度**构建测量说明书**, 然后调用 **performMeasure** 进程测量
    - 若为好的测量, 则说明已经操作完成了
    - 若非好的测量
      - 则以 baseSize 和 期望的窗体宽高取平均值, 再次构建测量说明书, 再调用 **performMeasure** 进程测量
      - 若还不是好的测量, 进入后面的分支
- 若窗体宽度的 LayoutParam 非 wrap_content
  - 使用 窗体期望的宽高 来**构建测量说明书**
  - 调用 performMeasure 进行 View 的测量

好的, 分析了 measureHierarchy 的流程, 接下来看看它构建测量说明书的具体是实现
```
public final class ViewRootImpl implements ViewParent,
        View.AttachInfo.Callbacks, ThreadedRenderer.DrawCallbacks {
        
  private static int getRootMeasureSpec(int windowSize, int rootDimension) {
        int measureSpec;
        switch (rootDimension) {
        case ViewGroup.LayoutParams.MATCH_PARENT:
            // 若为 MATCH_PARENT, 说明书的类型为 MeasureSpec.EXACTLY
            measureSpec = MeasureSpec.makeMeasureSpec(windowSize, MeasureSpec.EXACTLY);
            break;
        case ViewGroup.LayoutParams.WRAP_CONTENT:
            // 若为 WRAP_CONTENT, 说明书的类型为 MeasureSpec.AT_MOST
            measureSpec = MeasureSpec.makeMeasureSpec(windowSize, MeasureSpec.AT_MOST);
            break;
        default:
            // 若为精确值, 说明书的类型也为 MeasureSpec.EXACTLY
            measureSpec = MeasureSpec.makeMeasureSpec(rootDimension, MeasureSpec.EXACTLY);
            break;
        }
        return measureSpec;
    }
    
}
```
- 若为 MATCH_PARENT
  - 测量说明书类型为 MeasureSpec.EXACTLY, 值为窗体的大小
- 若为 WRAP_CONTENT
  - 测量说明书类型为 MeasureSpec.AT_MOST, 值为窗体的大小
- 若设置了精确值
  - 测量说明书类型为 MeasureSpec.EXACTLY, 值为精确值大小

当窗体的测量说明书已经构建好了, 便调用 performMeasure 真正进行 View 树的测量操作了

## 四. View 树的测量分发
```
public final class ViewRootImpl implements ViewParent,
        View.AttachInfo.Callbacks, ThreadedRenderer.DrawCallbacks {
            
    private void performMeasure(int childWidthMeasureSpec, int childHeightMeasureSpec) {
        ......
        try {
            // 对 DecorView 进行测量, 它虽然是 ViewGroup, 但并没有重写 measure 方法, 因此我们直接去 View 中查看
            mView.measure(childWidthMeasureSpec, childHeightMeasureSpec);
        } finally {
            ......
        }
    }

}

public class View implements Drawable.Callback, KeyEvent.Callback,
        AccessibilityEventSource {

    int mOldWidthMeasureSpec = Integer.MIN_VALUE;
    int mOldHeightMeasureSpec = Integer.MIN_VALUE;
    private LongSparseLongArray mMeasureCache;
     
    public final void measure(int widthMeasureSpec, int heightMeasureSpec) {
        ......
        // 1. 构建测量缓存的 key
        long key = (long) widthMeasureSpec << 32 | (long) heightMeasureSpec & 0xffffffffL;
        if (mMeasureCache == null) {
            mMeasureCache = new LongSparseLongArray(2);
        }
        // 2. 根据 Flag 判断是否要强制重新布局
        final boolean forceLayout = (mPrivateFlags & PFLAG_FORCE_LAYOUT) == PFLAG_FORCE_LAYOUT;
        // 3. 判断是否需要重新测量
        final boolean specChanged = widthMeasureSpec != mOldWidthMeasureSpec
                || heightMeasureSpec != mOldHeightMeasureSpec;                      // 3.1 判断测量说明书是否有变化
        final boolean isSpecExactly = MeasureSpec.getMode(widthMeasureSpec) == MeasureSpec.EXACTLY
                && MeasureSpec.getMode(heightMeasureSpec) == MeasureSpec.EXACTLY;   // 3.2 判断是否是精确值测量
        final boolean matchesSpecSize = getMeasuredWidth() == MeasureSpec.getSize(widthMeasureSpec)
                && getMeasuredHeight() == MeasureSpec.getSize(heightMeasureSpec);   // 3.3 判断说明书中给予的值与当前 View 已经测量的宽高是否是一致的
        final boolean needsLayout = specChanged
                && (sAlwaysRemeasureExactly || !isSpecExactly || !matchesSpecSize); // 3.4 通过上述条件判断是否需要测量
        // 4. 进行测量操作
        if (forceLayout || needsLayout) {
            // 获取缓存的索引
            int cacheIndex = forceLayout ? -1 : mMeasureCache.indexOfKey(key);
            if (cacheIndex < 0 || sIgnoreMeasureCache) {
                // 4.1 执行测量操作
                onMeasure(widthMeasureSpec, heightMeasureSpec);
                ......
            } else {
                // 4.2 直接从缓存中读取数据, 并且重新赋值
                long value = mMeasureCache.valueAt(cacheIndex);
                setMeasuredDimensionRaw((int) (value >> 32), (int) value);
                ......
            }
            ......
        }
        // 5. 将最新的测量说明书更新到成员变量
        mOldWidthMeasureSpec = widthMeasureSpec;
        mOldHeightMeasureSpec = heightMeasureSpec;
        // 6. 将 key 与 val 存入缓存
        // val 的构建: 宽度测量说明书左移 32 位占用 long 类型的高 32 位
        //             高度测量说明书保存在 long 类型的低 32 位
        mMeasureCache.put(key, ((long) mMeasuredWidth) << 32 |
                (long) mMeasuredHeight & 0xffffffffL); // suppress sign extension
    }

            
}
```
可以看到 View 的 measure 中做了如下的操作
- 构建当前 View 测量说明书缓存的 key
  - mMeasureCache 未创建, 先进行创建
- 根据 Flag 判断是否要强制重新布局
- 根据一些条件判断是否需要重新测量
  - 若无缓存, 则使用 **onMeasure** 执行测量操作
  - 若存在缓存, 则直接使用缓存中的测量好的数据
- 将最新的测量说明书更新到成员变量
- 将 key 与测量的宽高存入缓存
  - 测量的宽高是将 int 类型合并到了 long 类型中进行存储 

好的, 因为 DecorView 是 FrameLayout 的容器, 我们就以 FrameLayout 举例, 看看一次测量的过程是到底是怎样的

## 五. FrameLayout 测量过程
ViewGroup 中是没有 onMeasure 实现的, 我们看看 FrameLayout 的实现
```
public class FrameLayout extends ViewGroup {
    
    @Override
    protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        ......
        int maxHeight = 0;
        int maxWidth = 0;
        for (int i = 0; i < count; i++) {
            final View child = getChildAt(i);
            if (mMeasureAllChildren || child.getVisibility() != GONE) {
                // 1. 测量子孩子
                measureChildWithMargins(child, widthMeasureSpec, 0, heightMeasureSpec, 0);
                final LayoutParams lp = (LayoutParams) child.getLayoutParams();
                // 记录测量过程中, 子孩子的最大宽度
                maxWidth = Math.max(maxWidth,
                        child.getMeasuredWidth() + lp.leftMargin + lp.rightMargin);
                // 记录测量过程中, 子孩子的最大高度
                maxHeight = Math.max(maxHeight,
                        child.getMeasuredHeight() + lp.topMargin + lp.bottomMargin);
                ......
            }
        }
        ......// 对测量出来的子视图大小做一些验证
        
        // 2. 根据子视图的大小来确定自己的大小
        setMeasuredDimension(
               resolveSizeAndState(maxWidth, widthMeasureSpec, childState),
                resolveSizeAndState(maxHeight, heightMeasureSpec, childState << MEASURED_HEIGHT_STATE_SHIFT)
        );
        ......
    }
    
}
```
可以看到 FrameLayout 容器的设计, **首先是调用了 measureChildWithMargins 来确定子视图的大小, 然后再来确定自身的大小**

### 一) 确定子 View 大小
我们看看这个确认子 View 大小的过程
```
public class ViewGroup extends View {

    protected void measureChildWithMargins(View child,
            int parentWidthMeasureSpec, int widthUsed,
            int parentHeightMeasureSpec, int heightUsed) {
        // 获取子 View 的 LP
        final MarginLayoutParams lp = (MarginLayoutParams) child.getLayoutParams();
        // 1. 构建子视图的测量说明书
        final int childWidthMeasureSpec = getChildMeasureSpec(parentWidthMeasureSpec,
                mPaddingLeft + mPaddingRight + lp.leftMargin + lp.rightMargin
                        + widthUsed, lp.width);
        final int childHeightMeasureSpec = getChildMeasureSpec(parentHeightMeasureSpec,
                mPaddingTop + mPaddingBottom + lp.topMargin + lp.bottomMargin
                        + heightUsed, lp.height);
        // 2. 将 measure 操作分发给子 View 
        child.measure(childWidthMeasureSpec, childHeightMeasureSpec);
    }
}
```
主要有两个操作, 一是构建子视图的测量说明书, 二是分发给子 View 内部去处理测量
#### 1. 构建子 View 测量说明书
```
public class View {

    public static int getChildMeasureSpec(int spec, int padding, int childDimension) {
        // 1. 解析当前容器的测量模式与大小
        int specMode = MeasureSpec.getMode(spec);
        int specSize = MeasureSpec.getSize(spec);
        // 2. 去除 padding, 即获取当前容器可供 View 使用的空间
        int size = Math.max(0, specSize - padding);
        // 3. 用于构建子 View 说明书
        int resultSize = 0;
        int resultMode = 0;
        switch (specMode) {
        // 3.1 当前容器是 match_parent 或者精确值
        case MeasureSpec.EXACTLY:
            if (childDimension >= 0) {
                resultSize = childDimension;
                resultMode = MeasureSpec.EXACTLY;
            } else if (childDimension == LayoutParams.MATCH_PARENT) {
                resultSize = size;
                resultMode = MeasureSpec.EXACTLY;
            } else if (childDimension == LayoutParams.WRAP_CONTENT) {
                resultSize = size;
                resultMode = MeasureSpec.AT_MOST;
            }
            break;
        // 3.2 当前容器是 wrap_content
        case MeasureSpec.AT_MOST:
            if (childDimension >= 0) {
                resultSize = childDimension;
                resultMode = MeasureSpec.EXACTLY;
            } else if (childDimension == LayoutParams.MATCH_PARENT) {
                resultSize = size;
                resultMode = MeasureSpec.AT_MOST;
            } else if (childDimension == LayoutParams.WRAP_CONTENT) {
                resultSize = size;
                resultMode = MeasureSpec.AT_MOST;
            }
            break;
        // 3.3 当前容器是尺寸是未指明的(子视图可以随意发挥)
        case MeasureSpec.UNSPECIFIED:
            if (childDimension >= 0) {
                resultSize = childDimension;
                resultMode = MeasureSpec.EXACTLY;
            } else if (childDimension == LayoutParams.MATCH_PARENT) {
                resultSize = View.sUseZeroUnspecifiedMeasureSpec ? 0 : size;
                resultMode = MeasureSpec.UNSPECIFIED;
            } else if (childDimension == LayoutParams.WRAP_CONTENT) {
                resultSize = View.sUseZeroUnspecifiedMeasureSpec ? 0 : size;
                resultMode = MeasureSpec.UNSPECIFIED;
            }
            break;
        }
        // 构建子视图说明书
        return MeasureSpec.makeMeasureSpec(resultSize, resultMode);
    }
    
}
```
好的可以看到测量子 View 的过程中, 主要有两个操作
- 第一个是构建子 View 的测量参考说明书
- 第二个便是将测量参考说明书下发给子 view, 让其内部去处理测量操作

上面终点介绍了子 View 说明书的构建过程, 接下来看看 View 拿到了自己的测量说明书之后, 坐了什么操作

#### 2. 测量子 View
```
public class View implements Drawable.Callback, KeyEvent.Callback,
        AccessibilityEventSource {

    protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        // 通过测量说明书来构建 View 的宽高, 以宽度举例
        // 1. 首先获取当前 View 建议的最小宽度
        // 2. 获取默认的大小
        setMeasuredDimension(getDefaultSize(getSuggestedMinimumWidth(), widthMeasureSpec),
                getDefaultSize(getSuggestedMinimumHeight(), heightMeasureSpec));
    }

    private Drawable mBackground;
    
    protected int getSuggestedMinimumWidth() {
        // 1. 若 View 没有设置背景, 则使用 mMinWidth, 它由解析 "android: minWidth" 标签获取到
        // 2. 若设置了背景, 则使用 mMinWidth 和背景宽度中最大的
        return (mBackground == null) ? mMinWidth : max(mMinWidth, mBackground.getMinimumWidth());
    }
    
    public static int getDefaultSize(int size, int measureSpec) {
        // 先将 result 置为最小的尺寸
        int result = size;
        int specMode = MeasureSpec.getMode(measureSpec);
        int specSize = MeasureSpec.getSize(measureSpec);
        switch (specMode) {
        case MeasureSpec.UNSPECIFIED:
            // UNSPECIFIED 该情况默认是 View 最小的尺寸
            result = size;
            break;
        case MeasureSpec.AT_MOST:
            // 从 ViewRootImpl.getRootMeasureSpec 方法可知, AT_MOST 对应 wrap_content
            // View 默认实现中, 并没有对 AT_MOST 做任何处理
        case MeasureSpec.EXACTLY:
            // 从 ViewRootImpl.getRootMeasureSpec 说明书构建的过程中可知
            result = specSize;
            break;
        }
        return result;
    }

    protected final void setMeasuredDimension(int measuredWidth, int measuredHeight) {
        ......
        setMeasuredDimensionRaw(measuredWidth, measuredHeight);
    }

    int mMeasuredWidth;
    int mMeasuredHeight;
    
    private void setMeasuredDimensionRaw(int measuredWidth, int measuredHeight) {
        // 至此便获得了 Measured 的宽高
        mMeasuredWidth = measuredWidth;
        mMeasuredHeight = measuredHeight;
        ......
    }
}
```
可以看到在 View 大小的确定中
- 首先**确定 View 的最小尺寸**
  - 选择 android:minWidth = "xx" 和 background 中最大的作为最小的尺寸
- 根据**测量说明书的类型, 确认 View 最终的大小**
  - **UNSPECIFIED**: 系统的实现为最小的尺寸
  - **AT_MOST**(wrap_content): View 中没有做任何处理, 它的效果与 EXACTLY 一致
    - 因此一个 View 想要实现 wrap_content 效果, 需要自己手动重写实现 
  - **EXACTLY**(match_parent 和 数值): 说明书中的值

至此子 View 的测量就完成了, 接下来看看容器根据子 View 确认自身大小的过程

### 二) 确认自身大小
先回顾一下 FrameLayout 的 onMeasure 中, 确定自身大小的代码
```
public class FrameLayout extends ViewGroup {
    
    @Override
    protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        // 记录子 View 的最大宽高
        int maxHeight = 0;
        int maxWidth = 0;
        
        ......// 进行子 View 的测量
        
        // 2. 根据子视图的大小来确定自己的大小
        setMeasuredDimension(
               resolveSizeAndState(maxWidth, widthMeasureSpec, childState),
                resolveSizeAndState(maxHeight, heightMeasureSpec, childState << MEASURED_HEIGHT_STATE_SHIFT)
        );
        ......
    }
    
}
```
好的, 测量完子 View 之后, 便知道他们之间最大的宽高了, 接下来我们看看 resolveSizeAndState 这个方法, 看看它是如何确认 FrameLayout 的尺寸的

```
public class View implements Drawable.Callback, KeyEvent.Callback,
        AccessibilityEventSource {
    
    public static int resolveSizeAndState(int size, int measureSpec, int childMeasuredState) {
        // 1. 获取父容器传递下来的 MeasureSpec
        // 若为 DecorView 则为 ViewRootImpl 中根据 Window 大小构建的说明书
        final int specMode = MeasureSpec.getMode(measureSpec);
        final int specSize = MeasureSpec.getSize(measureSpec);
        // 2. 确认自身大小
        final int result;
        switch (specMode) {
            // 若为 wrap_content, 则将子 View 的大小截断值说明书的最大值
            case MeasureSpec.AT_MOST:
                if (specSize < size) {
                    result = specSize | MEASURED_STATE_TOO_SMALL;
                } else {
                    result = size;
                }
                break;
            // 若为 match_parent 和精确值, 则直接使用说明书中的尺寸
            case MeasureSpec.EXACTLY:
                result = specSize;
                break;
            // 若为未确认的, 则忽略说明书中的值, 直接使用子 View 测量结果
            case MeasureSpec.UNSPECIFIED:
            default:
                result = size;
        }
        // 返回测量尺寸
        return result | (childMeasuredState & MEASURED_STATE_MASK);
    }
                
}
```
好的, 到这里容器的大小就确认好了, 之后便可以通过 getMeasuredWidth 和 getMeasuredHeight 获取测量的宽高了


## 六. UNSPECIFIED 的使用场景
在整个策略过程中, 我们发现了一个特殊的说明书类型, 它是 UNSPECIFIED, 其他的 AT_MOST 对应 wrap_content, EXACTLY 对应着 match_parent 和具体的数值, **这个 UNSPECIFIED 设计它的目的是什么, 它的作用又是什么呢?**

### 一) 问题描述
UNSPECIFIED 这个类型的说明书在 Android 使用的非常少, 有迹可寻的地方是在 ScrollView 中, 我们先看看一个特殊的现象
```
<?xml version="1.0" encoding="utf-8"?>
<ScrollView
    xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent">

    <TextView
        android:layout_width="match_parent"
        android:layout_height="30dp"
        android:gravity="center"
        android:text="TextView"
        />

</ScrollView>
```
这个 xml 文件我们设想的展示效果是 ScrollView 撑满整个屏幕, 然后 TextView 占用 300 dp 的高度, 并且文本是居中显示的, 然而实际上, TextView 仅仅只有一行文字的高度, 如下图所示

![产生的问题](https://i.loli.net/2019/10/23/mBLPe86yTNh7fJK.png)

这是为什么呢? 不可思议, 我们去 ScrollView 中看看它做了什么操作

### 二) 问题探究
#### 1. ScrollView 的测量策略
```
public class ScrollView extends FrameLayout {
    
    @Override
    protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        // 调用了 FrameLayout 的 onMeasure 对子 View 进行测量
        super.onMeasure(widthMeasureSpec, heightMeasureSpec);
        // 我们的 .xml 文件中, 没有使用 android:fillViewport = true, 因此这里会 return
        if (!mFillViewport) {
            return;
        }
        ......
    }
    
}
```
可以看到在没有开启 fillViewport 的情况下, 它会调用 FrameLayout 的 onMeasure, 很奇怪我们的 TextView 在 FrameLayout 中展示的效果是没有问题的, 为什么 ScrollView 中出现了偏差呢, 那是因为虽然它没有破坏 FrameLayout onMeasure 的流程, 但它重写了 measureChild 和 measureChildWithMargins, 影响了最终测量的结果

我们选择 measureChildWithMargins 看看它内部做了哪些操作
```
public class ScrollView extends FrameLayout {
    
    @Override
    protected void measureChildWithMargins(View child, int parentWidthMeasureSpec, int widthUsed,
            int parentHeightMeasureSpec, int heightUsed) {
        final MarginLayoutParams lp = (MarginLayoutParams) child.getLayoutParams();
        // 构建宽度测量说明书, 没有任何问题
        final int childWidthMeasureSpec = getChildMeasureSpec(parentWidthMeasureSpec,
                mPaddingLeft + mPaddingRight + lp.leftMargin + lp.rightMargin
                        + widthUsed, lp.width);
        // 构建高度测量说明书, 这里有趣了
        final int usedTotal = mPaddingTop + mPaddingBottom + lp.topMargin + lp.bottomMargin +
                heightUsed;
        // 可以看到他将子 View 的说明书类型, 强制传入了 UNSPECIFIED
        final int childHeightMeasureSpec = MeasureSpec.makeSafeMeasureSpec(
                Math.max(0, MeasureSpec.getSize(parentHeightMeasureSpec) - usedTotal),
                MeasureSpec.UNSPECIFIED);
        child.measure(childWidthMeasureSpec, childHeightMeasureSpec);
    }
    
}
```
很有意思, **ScrollView 在测量子孩子的时候, 将其测量说明书强行改写成了 UNSPECIFIED 类型, size 为其最大可用空间**, 因此我们得去 TextView 的 onMeasure 中寻找答案

#### 2. TextView 的测量策略
我们回顾一下在 View 测量的过程中, 在面对 UNSPECIFIED 策略说明书时, 它会使用自己最小的高度, 但 TextView 不同, 它重写了 onMeasure, 于是我们去看看 TextView 中做了哪些处理

```
public class TextView extends View {
    
    protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        int widthMode = MeasureSpec.getMode(widthMeasureSpec);
        int heightMode = MeasureSpec.getMode(heightMeasureSpec);
        int widthSize = MeasureSpec.getSize(widthMeasureSpec);
        int heightSize = MeasureSpec.getSize(heightMeasureSpec);
        
        int width;
        int height;
        
        ......
        // 这里是对高度的测量策略
        if (heightMode == MeasureSpec.EXACTLY) {
            // 若为 EXACTLY 的, 则使用策略说明书中分配的
            height = heightSize;
            mDesiredHeightAtMeasure = -1;
        } else {
            // 1. 若为 AT_MOST/UNSPECIFIED 类型
            // 获取期望的高度, 很显然是 wrap_content 的效果
            int desired = getDesiredHeight();
            height = desired;
            ......
        }
        ......
        setMeasuredDimension(width, height);
    }
    
}
```
可以看到, TextView 中对于 UNSPECIFIED 类型的高度说明书, 直接使用了 wrap_content 的效果, 这也就很好的解释了, 为什么会呈现出上图中的结果

### 三) 问题解析
**通过了对 ScrollView 和 TextView 组合的分析, 我们看到了 UNSPECIFIED 的使用, 不过似乎通过对它的使用, 我们的视图效果却变得更差了, 这个 UNSPECIFIED 存在有什么意义呢?**

我们知道 MeasureSpec 在整个 View 的测量过程中起到了一个非常重要的作用, 子 View 获取到父容器传递给其的策略说明书后, 就可以进行测量了, 它其中有三个模式, 分别是 EXACTLY, AT_MOST, 我们通常理解为: 
- EXACTLY: 对应 match_parent 和设置的 dp 值
- AT_MOST: 对应 wrap_content

于是 UNSPECIFIED 就找不到映射关系了, 从而产生了对这个类型的存在产生了怀疑, 但其实并非如此, **对于 MeasureSpec 中的 mode, 并非是为了映射 match_parent 和 wrap_content 等 xml 元素而存在的, 它的作用是用来规范子 View 的测量行为**
- EXACTLY: 指的是子 View 必须**严格按照说明书中 size 的值来设置自己的尺寸**
- AT_MOST: 指的是子 View 可以**在说明书中 size 之内来设置自己的尺寸**
- UNSPECIFIED: 指的是子 View 可以**忽略说明书中 size 的大小, 随意的设置自己的尺寸**

当我们正确的认识了 MeasureSpec 的作用之后, 一切就迎刃而解了, **之前理解的映射关系其实仅仅是 Android Framework 开发工程师遵守 MeasureSpec 中 mode 的约束而产生的**, 若是忽略 mode 的规范, 我们自然是可以随心所欲的吧 UNSPECIFIED 当成 match_parent 的映射, 但这样的自定义组件, 显然是不会受到大众认可的

**因此 UNSPECIFIED 存在的意义是为了应对 View 的大小不需要受到父容器限制的场景**, ScrollView 中对子 View 使用 UNSPECIFIED 策略模式, 是因为它是可以滚动的, 它不希望子 View 受到当前父容器尺寸的影响, 否则我们何来滚动的效果呢?

## 总结
View 的测量可见是一个非常漫长的过程, 回过头来看, 它主要分为如下几个阶段
- 在 ViewRootImpl 中通过窗体大小来构建顶层容器 DecorView 的测量说明书
- 将说明书分发到容器中时, 根据容器的特性来构建子 View 的测量说明书
- 子 View 根据测量说明书, 来确认自身的大小
- 当子 View 测量完成后, 会回到容器中, 进一步确认容器的尺寸

至此 View 的 measure 过程就完成了, 其中不乏有很多值得我们思考的细节, 代码中注释的也很详细