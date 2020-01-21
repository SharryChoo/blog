---
title: Android 系统架构 —— View 的测量与布局
permalink: android-source/graphic-view-traversals-measure-layout
key: android-source-graphic-view-traversals-measure-layout
tags: AndroidFramework
sidebar:
  nav: android-source
---
## 前言
通过图像渲染的准备的学习, 我们知道在 onResume 之后我们的 ViewRootImpl 会通过 addToDisplay 与 WMS 建立联系, 之后就会执行 requestLayout 发送的消息执行 View 的遍历操作

这里我们就 requestLayout 入手分析 View 的变量

<!--more-->

## 一. requestLayout
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
            // 2. 标记为请求遍历的状态
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
从这里我们可以看到, 我们的 Traversals 操作是添加到 Choreographer 的 CallbackQueue 中统一调度的, 当 VSYNC 到来时会自动执行

最有趣的是, 它为了保证 traversal 操作的优先执行, 插入了一个同步屏障

接下来我们看看 ViewRootImpl 是如何遍历 View 的

###  ViewRootImpl.performTraversals
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
        // 使用局部变量保存, 防止变更
        final View host = mView;
        
        WindowManager.LayoutParams lp = mWindowAttributes;   // 获取这个 DecorView 的布局参数
        boolean windowSizeMayChange = false;                 // 用于判断窗体大小是否有可能发生改变
        
        // 1. View 的测量
        int desiredWindowWidth;
        int desiredWindowHeight;
        Rect frame = mWinFrame;
        // 若为第一次 Traversals
        if (mFirst) {
            ......
            mLayoutRequested = true;
            final Configuration config = mContext.getResources().getConfiguration();
            // 使用屏幕物理宽高作为窗体期望尺寸
            if (shouldUseDisplaySize(lp)) {
                Point size = new Point();
                mDisplay.getRealSize(size);
                desiredWindowWidth = size.x;
                desiredWindowHeight = size.y;
            } else {
                // 使用 mWindFrame 中的宽高作为窗体期望尺寸
                desiredWindowWidth = mWinFrame.width();
                desiredWindowHeight = mWinFrame.height();
                ......
            }
            ......
        } else {
            // 若非第一次进行 Traversals, 那么窗体期望尺寸直接使用 frame(mWinFrame) 中记录的宽高
            desiredWindowWidth = frame.width();
            desiredWindowHeight = frame.height();
            ......
        }
        ......
        boolean layoutRequested = mLayoutRequested && (!mStopped || mReportNextDraw);
        if (layoutRequested) {
            final Resources res = mView.getContext().getResources();
            ......
             if (mFirst) {
                ......// 第一次的尺寸已经在上面设置过了, 这里不会进行窗体期望尺寸的设置操作
            } else {
                ......
                // 若 LayoutParams 为 WRAP_CONTENT, 那么会再次进行窗体期望尺寸的判断
                if (lp.width == ViewGroup.LayoutParams.WRAP_CONTENT
                        || lp.height == ViewGroup.LayoutParams.WRAP_CONTENT) {
                    windowSizeMayChange = true;
                    // 使用屏幕物理宽高作为窗体期望尺寸
                    if (shouldUseDisplaySize(lp)) {
                        Point size = new Point();
                        mDisplay.getRealSize(size);
                        desiredWindowWidth = size.x;
                        desiredWindowHeight = size.y;
                    } else { 
                        // 使用屏幕可用宽高作为窗体期望尺寸
                        Configuration config = res.getConfiguration();
                        desiredWindowWidth = dipToPx(config.screenWidthDp);
                        desiredWindowHeight = dipToPx(config.screenHeightDp);
                    }
                }
            }
            // 1.1 调用 measureHierarchy 方法, 为了确定其布局位置, 先确定它的大小
            windowSizeMayChange |= measureHierarchy(host, lp, res,
                    desiredWindowWidth, desiredWindowHeight);
        }
        
        ......
        
        // 1.2 若测量尺寸变更, 则重置 Surface
        ......
        if (mFirst || windowShouldResize || insetsChanged ||
                viewVisibilityChanged || params != null || mForceNextWindowRelayout) {
            ......
            boolean hadSurface = mSurface.isValid();
            try {
                ......
                relayoutResult = relayoutWindow(params, viewVisibility, insetsPending);
                ......
            }
        }
        ......
        if (didLayout) {
            // 2. View 的布局
            performLayout(lp, mWidth, mHeight);
            ......
        }
        // 3. View 的绘制
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
- **View 的测量**
  - **调用 measureHierarchy 进行 View 测量**
  - **测量之后窗体变更则调用 relayoutWindow 重置 Surface**
- **View 的布局**
  - 调用 **performLayout** 进行 View 的摆放
- **View 的绘制**
  - 调用 **performDraw** 进行 View 的绘制

由于篇幅限制, 这里我们主要学习 View 的测量和布局, 关于 Surface 的重置和 View 的绘制流程都较为复杂, 我们放到后面进行分析

## 二. View 的测量
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
measureHierarchy 中主要有两步
- 首先是构建根 View 的测量说明书
- 然后调用 performMeasure 进行测量分发

其说明书构建过程如下
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

### 二) View 树的测量分发
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
- **根据一些条件判断是否需要重新测量**
  - 若无缓存, 则使用 **onMeasure** 执行测量操作
  - 若存在缓存, 则直接使用缓存中的测量好的数据
- 将最新的测量说明书更新到成员变量
- 将 key 与测量的宽高存入缓存
  - 测量的宽高是将 int 类型合并到了 long 类型中进行存储 

好的, 因为 DecorView 是 FrameLayout 的容器, 我们就以 FrameLayout 举例, 看看一次测量的过程是到底是怎样的

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
可以看到 FrameLayout 容器的 onMeasyre 设计如下
- 首先是调用了 measureChildWithMargins 测量子孩子
- 然后再来确定自身的大小

#### 1. 测量子 View
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

好的可以看到测量子 View 的过程中, 主要有两个操作
- 第一个是构建子 View 的测量参考说明书
- 第二个便是将测量参考说明书下发给子 view, 让其内部去处理测量操作

下面我们一一查看

##### 1) 构建子 View 测量说明书
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
上面终点介绍了子 View 说明书的构建过程, 接下来看看 View 拿到了自己的测量说明书之后, 做了什么操作

##### 2) 测量子 View
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

#### 2. 测量容器自身
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

### 回顾
View 的测量流程如下
- 构建 DecorView 的测量说明书
- 遍历 View 树进行测量(以 FrameLayout 为例)
  - **测量子 View**
    - 构建子 View 测量说明书
    - 分发到子 View 执行测量操作
      - 确定 View 的最小尺寸
      - 根据测量说明书的类型, 确认 View 最终的大小
  - **测量容器自身**
     - 根据容器的特性进行测量即可    

## 三. View 的布局
```
public final class ViewRootImpl implements ViewParent,
        View.AttachInfo.Callbacks, ThreadedRenderer.DrawCallbacks {
   
    private void performLayout(WindowManager.LayoutParams lp, int desiredWindowWidth,
            int desiredWindowHeight) {
        // 这里已经开始执行布局请求了, 因此置为 false
        mLayoutRequested = false;
        // 1. 第一阶段, 执行第一次 Layout 操作
        // 表示当前正在执行 layout 操作
        mInLayout = true;
        final View host = mView;
        ......
        try {
            // 1.1  执行 DecorView 的 layout
            host.layout(0, 0, host.getMeasuredWidth(), host.getMeasuredHeight());
            // layout 完毕了, 标记为 false
            mInLayout = false;
            
            // 2. 第二阶段, 处理在第一次 layout 的过程中, 请求的调用了 requestLayout 的 view 集合
            int numViewsRequestingLayout = mLayoutRequesters.size();
            if (numViewsRequestingLayout > 0) {
                // 获取有效的请求者
                ArrayList<View> validLayoutRequesters = getValidLayoutRequesters(mLayoutRequesters, false);
                if (validLayoutRequesters != null) {
                    // 表示正在处理第一次 layout 过程中调用 requestLayout 的 view  
                    mHandlingLayoutInLayoutRequest = true;
                    int numValidRequests = validLayoutRequesters.size();
                    // 2.1 调用再次请求 view 的 reqeustLayout 操作, 重新发起 Traversals
                    for (int i = 0; i < numValidRequests; ++i) {
                        final View view = validLayoutRequesters.get(i);
                        view.requestLayout();
                    }
                    // 2.2 进行测量操作
                    measureHierarchy(host, lp, mView.getContext().getResources(),
                            desiredWindowWidth, desiredWindowHeight);
                    // 2.3 进行布局操作
                    mInLayout = true;
                    host.layout(0, 0, host.getMeasuredWidth(), host.getMeasuredHeight());
                    // 表示第一次 layout 过程中调用 requestLayout 的 view 已经处理结束了
                    mHandlingLayoutInLayoutRequest = false;

                    // 3. 第三阶段: 处理在第二次 layout 过程中, 调用了 requestLayout 的 view 集合
                    validLayoutRequesters = getValidLayoutRequesters(mLayoutRequesters, true);
                    if (validLayoutRequesters != null) {
                        final ArrayList<View> finalRequesters = validLayoutRequesters;
                        getRunQueue().post(new Runnable() {
                            @Override
                            public void run() {
                                // 若在第二次 layout 时, 又有新的请求添加进来, 那么直接 post 到下一帧去执行
                                int numValidRequests = finalRequesters.size();
                                for (int i = 0; i < numValidRequests; ++i) {
                                    final View view = finalRequesters.get(i);
                                    view.requestLayout();
                                }
                            }
                        });
                    }
                }

            }
        } finally {
            ......
        }
        mInLayout = false;
    }
    
}
```
好的, 可以看到 ViewRootImpl.performLayout 主要三个阶段, 注释中也标注的很清楚

接下来我们就分析一下 View 的 layout 过程

```
public class View {
    
    public void layout(int l, int t, int r, int b) {
        int oldL = mLeft;
        int oldT = mTop;
        int oldB = mBottom;
        int oldR = mRight;
        // 1. 调用 setFrame 保存其新的坐标位置 
        boolean changed = isLayoutModeOptical(mParent) ?
                setOpticalFrame(l, t, r, b) : setFrame(l, t, r, b);
        // 2. 若新的位置比起之前有所变更, 则调用 onLayout 去确定其子 View 的布局
        if (changed || (mPrivateFlags & PFLAG_LAYOUT_REQUIRED) == PFLAG_LAYOUT_REQUIRED) {
            // 调用了 onLayout 操作
            onLayout(changed, l, t, r, b);
            ......
        }
        ......
    }
    
}
```
可以看到主要做了两个操作
- 更新当前 View 的位置
- 更新子 View 的位置

### 一) 更新当前 View 的位置
```
public class View {

     protected boolean setFrame(int left, int top, int right, int bottom) {
        boolean changed = false;
        // 1.1 判断其新的位置与老的位置是否有变更
        if (mLeft != left || mRight != right || mTop != top || mBottom != bottom) {
            changed = true;
            // 1.2 计算之前窗体的尺寸
            int oldWidth = mRight - mLeft;
            int oldHeight = mBottom - mTop;
            int newWidth = right - left;
            int newHeight = bottom - top;
            // 判断尺寸是否变更了
            boolean sizeChanged = (newWidth != oldWidth) || (newHeight != oldHeight);
            invalidate(sizeChanged);// 若尺寸变更了, 则说明之前绘制的区域一定无效了, 因此调用该方法重新绘制
            // 1.3 更新当前成员变量的值
            mLeft = left;
            mTop = top;
            mRight = right;
            mBottom = bottom;
            // 确定当前 View 渲染器结点的大小(在 View 渲染的过程中介绍)
            mRenderNode.setLeftTopRightBottom(mLeft, mTop, mRight, mBottom);
            ......
        }
        return changed;
    }
    
    protected void onLayout(boolean changed, int left, int top, int right, int bottom) {
    }
    
}
```
好的, 可以看到在 layout 中主要做了两个操作
- 调用 setFrame 来更新当前 View 的坐标值
- 调用了 onLayout 操作去确定其子视图的位置
  - 这个方法在 View 中个空实现, 因为 View 的摆放的位置, 是由与其容器的特性相关的

### 二) 更新子 View 的位置
DecorView 是 FrameLayout, 这里以它来举例, 看看它 onLayout 的实现
```
public class FrameLayout extends ViewGroup {
    
    @Override
    protected void onLayout(boolean changed, int left, int top, int right, int bottom) {
        layoutChildren(left, top, right, bottom, false /* no force left gravity */);
    }
    
    void layoutChildren(int left, int top, int right, int bottom, boolean forceLeftGravity) {
        // 1. 获取子孩子的数量
        final int count = getChildCount();
        // 2. 计算当前容器的可用空间
        final int parentLeft = getPaddingLeftWithForeground();
        final int parentRight = right - left - getPaddingRightWithForeground();
        final int parentTop = getPaddingTopWithForeground();
        final int parentBottom = bottom - top - getPaddingBottomWithForeground();
        // 3. 循环操作子 View
        for (int i = 0; i < count; i++) {
            final View child = getChildAt(i);
            if (child.getVisibility() != GONE) {
                final LayoutParams lp = (LayoutParams) child.getLayoutParams();
                // 3.1 获取子 View 的宽高
                final int width = child.getMeasuredWidth();
                final int height = child.getMeasuredHeight();
                // 3.2 根据 gravity 参数来设置确定子 View 的 left, top 坐标
                int childLeft;
                int childTop;
                ......// 省略了计算坐标的代码
                // 3.3 确定了子 View 的坐标之后, 将其向下分发
                child.layout(childLeft, childTop, childLeft + width, childTop + height);
            }
        }
    }
    
}
```
可以看到, FrameLayout 计算好子视图位置之后, 调用了其 layout 方法, 将新的坐标位置传递给它

### 三) 回顾
layout 的过程比起 measure 是简单很多的
- 调用 setFrame 来更新当前 View 的坐标值
- 调用了 onLayout 操作去确定其子视图的位置

## 总结
View 的测量流程如下
- 构建 DecorView 的测量说明书
- 遍历 View 树进行测量(以 FrameLayout 为例)
  - **测量子 View**
    - 构建子 View 测量说明书
    - 分发到子 View 执行测量操作
      - 确定 View 的最小尺寸
      - 根据测量说明书的类型, 确认 View 最终的大小
   - **测量容器自身**
     - 根据容器的特性进行测量即可

View 的布局方式如下
- 调用 setFrame 来更新当前 View 的坐标值
- 调用了 onLayout 操作去确定其子视图的位置

至此 View 的 measure 过程就完成了, 其中不乏有很多值得我们思考的细节, 代码中注释的也很详细

## 思考
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
        android:layout_height="300dp"
        android:gravity="center"
        android:text="TextView"
        />

</ScrollView>
```
这个 xml 文件我们设想的展示效果是 ScrollView 撑满整个屏幕, 然后 TextView 占用 300 dp 的高度, 并且文本是居中显示的, 然而实际上, TextView 仅仅只有一行文字的高度, 如下图所示

![image](https://i.loli.net/2019/12/13/f8XqrKVHWB2Zxu6.png)

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
