---
layout: article
title: Android 系统架构 —— View 的布局
permalink: android-source/graphic-producer6
key: android-source-graphic-producer6
tags: AndroidFramework
---

## 前言
在 View 的测量过程中, 我们确定了整个 View 树的大小, 这里我们接着看看它是如何进行布局操作的

<!--more-->

```
public final class ViewRootImpl implements ViewParent,
        View.AttachInfo.Callbacks, ThreadedRenderer.DrawCallbacks {
    
    private void performTraversals() {
        ......
        // 3. 进行视图的摆放
        final boolean didLayout = layoutRequested && (!mStopped || mReportNextDraw);
        if (didLayout) {
            performLayout(lp, mWidth, mHeight);
            ......
        }
        ......
    }
}
```
我们这里就从 performLayout 来观测 View 的布局过程

## performLayout
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

## View 的 layout 过程
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
- 保存当前 View 的坐标值
- 更新子视图的位置

### 一) 更新自身坐标值
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

### 二) 确定子视图的位置
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

## 总结
layout 的过程比起 measure 是简单很多的, 它主要是根据当前容器的特性, 来确定子 View 在其中的位置, 然后将其向下分发

好的, 到这里 View 绘制的三大流程已经走了两步了, 后面要分析的就是比较困难的 View 的渲染了, 因为 Android 的图形引擎为 Skia 和 OpenGL, 下一篇文章我们看看 Skia 图形引擎