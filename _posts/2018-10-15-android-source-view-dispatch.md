---
title: Android 系统架构 —— View 的事件分发
permalink: android-source/view-dispatch
key: android-source-view-dispatch
tags: AndroidFramework
---

## 前言
View 的事件分发是我们在自定义 View 必须要掌握的技术点, Android 是支持多指触控的, 一个完整的事件序列如下
```
DOWN ... MOVE ... POINTER_DOWN ... MOVE .... POINTER_UP ... MOVE .... UP
```
可以看到一个事件序列从 Down -> UP 的过程中存在多个连续的 MOVE 事件, 若有其他手指参与还会存在 POINTER_DOWN 和 POINTER_UP 事件
- 一个 DOWN/POINTER_DOWN 称之为焦点事件, 在 MotionEvent 中用 PointerId 描述

好的了解了这个基础知识之后, 我们从 ViewGroup 的 dispatchTouchEvent 开始看看一次事件分发的流程

<!--more-->

## 一. ViewGroup 的事件分发
```java
public abstract class ViewGroup extends View implements ViewParent, ViewManager {
    
    // TouchTarget 描述一个正在响应事件的 View
    // 一个 View 可以响应多个焦点事件
    // mFirstTouchTarget 为链表头
    private TouchTarget mFirstTouchTarget;
    
    @Override
    public boolean dispatchTouchEvent(MotionEvent ev) {
        ......
        boolean handled = false;
        // 过滤不安全的事件
        if (onFilterTouchEventForSecurity(ev)) {
            // 获取事件的类型
            final int action = ev.getAction();
            final int actionMasked = action & MotionEvent.ACTION_MASK;
            // 若为 ACTION_DOWN, 则说明是一个全新的事件序列
            if (actionMasked == MotionEvent.ACTION_DOWN) {
                cancelAndClearTouchTargets(ev);   // 清空所有 TouchTarget
                resetTouchState();                // 清空所有触摸状态
            }
            // 1. 处理事件拦截
            final boolean intercepted;
            // 1.1 若为初始事件 或 当前容器存在子 View 正在消费事件, 则尝试拦截
            if (actionMasked == MotionEvent.ACTION_DOWN || mFirstTouchTarget != null) {
                // 1.1.1 判断当前 ViewGroup 的 Flag 中是否设置了 不允许拦截事件
                // 通过 ViewGroup.requestDisallowInterceptTouchEvent 进行设置, 常用于内部拦截法, 由子 View 控制容器的行为
                final boolean disallowIntercept = (mGroupFlags & FLAG_DISALLOW_INTERCEPT) != 0;
                if (!disallowIntercept) {
                    // 若允许拦截, 则调用 onInterceptTouchEvent, 尝试进行拦截
                    intercepted = onInterceptTouchEvent(ev);
                    ev.setAction(action);
                } else {
                    // 不允许拦截, 则直接将标记为置为 false
                    intercepted = false;
                }
            } else {
                // 1.2 若非初始事件, 并且没有子 View 响应事件中的焦点, 则自己拦截下来由自己处理
                // 自己处理不代表一定能消费, 这点要区分开来
                intercepted = true;
            }
            ......
            // 判断当前容器是否被取消了响应事件
            final boolean canceled = resetCancelNextUpFlag(this)
                    || actionMasked == MotionEvent.ACTION_CANCEL;
            ......
            // 判断是否需要拆分 MotionEvent
            final boolean split = (mGroupFlags & FLAG_SPLIT_MOTION_EVENTS) != 0;
            TouchTarget newTouchTarget = null;                // 描述响应的目标
            boolean alreadyDispatchedToNewTouchTarget = false;// 描述这个事件是否分发给了新的 TouchTarget
            if (!canceled && !intercepted) {
                ......
                // 2. 若为 ACTION_DOWN 或者 ACTION_POINTER_DOWN, 则说明当前事件序列出现了一个新的焦点, 则找寻该焦点的处理者
                if (actionMasked == MotionEvent.ACTION_DOWN
                        || (split && actionMasked == MotionEvent.ACTION_POINTER_DOWN)
                        || actionMasked == MotionEvent.ACTION_HOVER_MOVE) { 
                    // 2.1 获取焦点的索引(第一个手指的 down 为 0, 第二个手指的 down 为 1)
                    final int actionIndex = ev.getActionIndex(); // always 0 for down
                    // 将焦点的索引序号映射成二进制位, 用于后续保存在 TouchTarget 的 pointerIdBits 中
                    // 0 -> 1, 1 -> 10, 2 -> 100
                    final int idBitsToAssign = split ? 1 << ev.getPointerId(actionIndex) : TouchTarget.ALL_POINTER_IDS;
                    // 清理 TouchTarget 中对历史遗留的 idBitsToAssign 的缓存
                    removePointersFromTouchTargets(idBitsToAssign);
                    final int childrenCount = mChildrenCount;
                    if (newTouchTarget == null && childrenCount != 0) {
                        // 2.2 获取事件在当前容器中的相对位置
                        final float x = ev.getX(actionIndex);
                        final float y = ev.getY(actionIndex);
                        // 获取当前容器前序遍历的 View 序列(即层级从高到低排序)
                        final ArrayList<View> preorderedList = buildTouchDispatchChildList();
                        // 2.3 遍历子 View, 找寻可以响应事件的目标
                        final boolean customOrder = preorderedList == null
                                && isChildrenDrawingOrderEnabled();
                        final View[] children = mChildren;
                        for (int i = childrenCount - 1; i >= 0; i--) {
                            // 通过索引找到子孩子实例
                            final int childIndex = getAndVerifyPreorderedIndex(
                                    childrenCount, i, customOrder);
                            final View child = getAndVerifyPreorderedView(
                                    preorderedList, children, childIndex);
                            ......
                            // 2.3.1 判断这个子 View 是否在响应事件的区域
                            if (!canViewReceivePointerEvents(child)
                                    || !isTransformedTouchPointInView(x, y, child, null)) {
                                ......
                                // 若不在可响应的区域, 则继续遍历下一个子 View
                                continue;
                            }
                            // 2.3.2 判断获取这个子 View, 是否已经响应了序列中的一个焦点
                            newTouchTarget = getTouchTarget(child);
                            if (newTouchTarget != null) {
                                // 若这个 View 已经响应了一个焦点, 则在它的 TouchTarget.pointerIdBits 添加新焦点的索引
                                newTouchTarget.pointerIdBits |= idBitsToAssign;
                                // 则直接结束查找, 直接进行后续的分发操作
                                break;
                            }
                            ......
                            // 2.3.3 调用 dispatchTransformedTouchEvent 尝试将这个焦点分发给这个 View 处理
                            if (dispatchTransformedTouchEvent(ev, false, child, idBitsToAssign)) {
                                ......
                                // 子 View 成功的消费了这个事件, 则将这个 View 封装成 TouchTarget 链到表头
                                newTouchTarget = addTouchTarget(child, idBitsToAssign);
                                // 表示这个事件在找寻新的响应目标时已经消费了
                                alreadyDispatchedToNewTouchTarget = true;
                                break;
                            }
                            ......
                        }
                        ......
                    }
                    // 2.4 若没有找到可以响应的子 View, 则交由最早的 TouchTarget 处理
                    if (newTouchTarget == null && mFirstTouchTarget != null) {
                        newTouchTarget = mFirstTouchTarget;
                        while (newTouchTarget.next != null) {
                            newTouchTarget = newTouchTarget.next;
                        }
                        // 在其 pointerIdBits 保存焦点的 ID 
                        newTouchTarget.pointerIdBits |= idBitsToAssign;
                    }
                }
            }
            // 3. 执行事件分发
            // 3.1 没有任何子 View 可以响应事件序列, 则交由自己处理
            if (mFirstTouchTarget == null) {
                handled = dispatchTransformedTouchEvent(ev, canceled, null,
                        TouchTarget.ALL_POINTER_IDS);
            } else {
                // 3.2 若存在子 View 响应事件序列, 则将这个事件分发下去
                TouchTarget predecessor = null;
                TouchTarget target = mFirstTouchTarget;
                while (target != null) {
                    final TouchTarget next = target.next;
                    // 3.2.1 alreadyDispatchedToNewTouchTarget 为 true 并且 target 为我们上面新找到的响应目标时, 跳过这次分发
                    // 因为在查找焦点处理者的过程中, 已经分发给这个 View 了
                    if (alreadyDispatchedToNewTouchTarget && target == newTouchTarget) {
                        handled = true;
                    } else {
                        // 3.2.2 处理还未消耗这个事件的子 View
                        final boolean cancelChild = resetCancelNextUpFlag(target.child)
                                || intercepted;// 判断是否需要给 View 发送 CANCEL 事件, view 设置了 PFLAG_CANCEL_NEXT_UP_EVENT 这个 Flag, 或者这个事件被该容器拦截了, 那么将会给子 View 发送 Cancel 事件
                        // 3.2.2 调用 dispatchTransformedTouchEvent 将事件分发给子 View 
                        if (dispatchTransformedTouchEvent(ev, cancelChild,
                                target.child, target.pointerIdBits)) {
                            handled = true;
                        }
                        // 3.2.3 若给这个 View 分发了 cancel 事件, 则说明它已经无法响应事件序列的焦点了, 因此将它从响应链表中移除
                        if (cancelChild) {
                            if (predecessor == null) {
                                mFirstTouchTarget = next;
                            } else {
                                predecessor.next = next;
                            }
                            target.recycle();
                            target = next;
                            continue;
                        }
                    }
                    predecessor = target;
                    target = next;
                }
            } 
            // 4. 清理失效的 TouchTarget
            // 4.1 若事件序列取消 或 整个事件 UP 了, 则移除所有的 TouchTarget
            if (canceled || actionMasked == MotionEvent.ACTION_UP || actionMasked == MotionEvent.ACTION_HOVER_MOVE) {
                resetTouchState();
            } else if (split && actionMasked == MotionEvent.ACTION_POINTER_UP) {
                // 4.2 若事件序列的一个焦点 UP 了, 则移除响应这个焦点的 TouchTarget
                final int actionIndex = ev.getActionIndex();
                final int idBitsToRemove = 1 << ev.getPointerId(actionIndex);
                removePointersFromTouchTargets(idBitsToRemove);
            }
        }
        ......
        return handled;
    }
    
}
```
好的, 可以看到上述的步骤非常复杂的, 其注释已经非常详细了, 总结下来其实就三点
- **拦截事件**
  - 若为 **起始事件 ACTION_DOWN/ACTION_POINTER_DOWN** 或 **存在子 View 正在消费事件**, 则调用 onInterceptTouchEvent 尝试拦截
    - 若设置了 FLAG_DISALLOW_INTERCEPT 则不进行拦截操作, 直接分发给子 View
  - 若非起始事件, 并且无子 View 响应事件, 则进行拦截操作
- **找寻起始事件的处理者**
  - 通过 canViewReceivePointerEvents 和 isTransformedTouchPointInView 找寻能区域内的子 View
    - 若子 View 已经响应了一个焦点事件序列, 则将这个事件序列的 id 添加到这个 View 对应的 TouchTarget 的 pointerIdBits 中
    - 若子 View 当前未响应事件, 则调用 dispatchTransformedTouchEvent 尝试让子 View 处理
      - 处理成功, 则将这个 View 构建成 TouchTarget 保存起来 
  - 若无处理者, 则交由 mFirstTouchTarget 处理
- **执行事件分发**
  - 若无响应目标, 则 ViewGroup 自行处理
  - 若存在响应目标, 则遍历 TouchTarget 链表, 将事件分发给所有的 TouchTarget

以上过程有两个疑问
- **当事件的新焦点到来时, 它是如何找寻能够响应它的子 View 的, 仅仅根据坐标这么简单吗?**
- **在事件分发时, 我们看到 ViewGroup 遍历了所有的 TouchTarget 对它们逐一进行事件分发, 直接分发给这事件对应焦点的处理者不就可以了吗, 为什么要分发给事件序列所有焦点的处理者呢?**

我们先看看第一个问题

### 一) 找寻事件的响应目标
从 ViewGroup 的 dispatchTouchEvent 中, 我们知道判断一个 View 是否可以响应事件主要有两个方法分别是 canViewReceivePointerEvents 和 isTransformedTouchPointInView, 这里我们一个一个探究
```java
public abstract class ViewGroup extends View implements ViewParent, ViewManager {
    
    private static boolean canViewReceivePointerEvents(@NonNull View child) {
        // Condition1: View 为 Visible
        // Condition2: 当前 View 设置了动画
        return (child.mViewFlags & VISIBILITY_MASK) == VISIBLE
                || child.getAnimation() != null;
    }
    
}
```
可以看到 canViewReceivePointerEvents 中, 若 View 是可见的 或 View 存在动画, 则说明它可以接收事件, 可以看到这个方法是根据 View 的状态来判断是否可以接收事件的, 具体是否能够接收事件还是得看后面一个方法

接下来我们看看 isTransformedTouchPointInView 这个方法做了哪些判断
```java
public abstract class ViewGroup extends View implements ViewParent, ViewManager {
    
    protected boolean isTransformedTouchPointInView(float x, float y, View child,
            PointF outLocalPoint) {
        // 1. 获取一个坐标数组, 数据为事件在当前 ViewGroup 中的相对坐标 x, y 值
        final float[] point = getTempPoint();
        point[0] = x;
        point[1] = y;
        // 2. 调用了 transformPointToViewLocal, 将坐标转为 Child 的相对坐标
        transformPointToViewLocal(point, child);
        // 3. 调用了 View.pointInView 判断坐标是否落在了 View 中
        final boolean isInView = child.pointInView(point[0], point[1]);
        // 若在子 View 中, 则尝试输出到 outLocalPoint 中, dispatchToucEvent 中传入的为 null
        if (isInView && outLocalPoint != null) {
            outLocalPoint.set(point[0], point[1]);
        }
        return isInView;
    }
    
     public void transformPointToViewLocal(float[] point, View child) {
        // 2.1 将 point 转为 View 的相对坐标
        point[0] += mScrollX - child.mLeft;
        point[1] += mScrollY - child.mTop;
        // 2.2 若 View 设置了 Matrix 变化, 则通过 Matrix 来映射这个坐标
        if (!child.hasIdentityMatrix()) {
            child.getInverseMatrix().mapPoints(point);
        }
    }
    
}

public class View implements Drawable.Callback, KeyEvent.Callback,
        AccessibilityEventSource {
    
    final boolean pointInView(float localX, float localY) {
        return pointInView(localX, localY, 0);
    }
    
    public boolean pointInView(float localX, float localY, float slop) {
        // 很简单, 即判断是否落在 View 的区域内, 小于 View 的宽度和高度
        // slop 为当前 Android 系统能够识别出来的手指区域的大小
        return localX >= -slop && localY >= -slop && localX < ((mRight - mLeft) + slop) &&
                localY < ((mBottom - mTop) + slop);
    }
    
}
```
好的, 可以看到 isTransformedTouchPointInView 中判断其是否可以响应事件的依据是判断事件的相对坐标是否落在了子 View 的宽高之内

不过**这里有一个细微的操作, 它会调用 child.getInverseMatrix().mapPoints(point); 来映射一次坐标, 这是为什么呢?**
- 因为我们在执行属性动画的时候, 有的时候会进行 View 的 transition, scale 等操作, 这些操作并不会改变 View 的原始坐标, 但会改变它内部的 RenderNode 的 Matrix, **进行坐标映射, 是为了让 View 在变化后的区域依旧可以响应事件流, 这就是为什么属性动画作用后的 View 依旧可以响应点击事件的原因**

好的, 分析完了如何判断子 View 是否可以响应事件之后, 接下来看一看 ViewGroup 事件流是如何分发的

### 二) 将事件分发给子 View
在 ViewGroup.dispatchTouchEvent 中, 我们知道最后它会遍历 TouchTarget 链表, 逐个调用 dispatchTransformedTouchEvent 方法, 将一个事件, 分发给该事件所在序列的所有焦点处理者, 接下来我们就看看它是如何做的
```java
public abstract class ViewGroup extends View implements ViewParent, ViewManager {
    
    private boolean dispatchTransformedTouchEvent(MotionEvent event, boolean cancel,
            View child, int desiredPointerIdBits) {
        final boolean handled;
        // 获取事件的动作
        final int oldAction = event.getAction();
        // 1. 处理 Cancel 操作
        if (cancel || oldAction == MotionEvent.ACTION_CANCEL) {
            // 1.1 将事件的 Action 强行改为 ACTION_CANCEL
            event.setAction(MotionEvent.ACTION_CANCEL);
            // 1.2 ACTION_CANCEL 的分发操作
            if (child == null) {
                // 1.2.1 自己处理
                handled = super.dispatchTouchEvent(event);
            } else {
                // 1.2.2 分发给子 View 
                handled = child.dispatchTouchEvent(event);
            }
            event.setAction(oldAction);
            return handled;
        }
        
        // 2. 判断这个 child 是否可以响应该事件
        // 2.1 获取这个事件所在序列的所有焦点
        final int oldPointerIdBits = event.getPointerIdBits();
        // 2.2 desiredPointerIdBits 描述这个 child 能够处理的焦点
        final int newPointerIdBits = oldPointerIdBits & desiredPointerIdBits; 
        // 2.3 若他们之间没有交集, 则说明出现了异常情况, 这个 child 所响应的并非是这个事件序列中的焦点
        if (newPointerIdBits == 0) {
            return false;
        }
        
        // 3. 将事件分发给子 View 
        final MotionEvent transformedEvent;
        // 3.1 若这个子 View 能够处理这个事件序列的所有焦点, 则直接进行分发操作
        if (newPointerIdBits == oldPointerIdBits) {
            // 若不存子 View, 或者子 View 设置 Matrix 变幻, 则走下面的分支
            if (child == null || child.hasIdentityMatrix()) {
                if (child == null) {
                    // 3.1.1 自己处理
                    handled = super.dispatchTouchEvent(event);
                } else {
                    // 3.1.2 分发给子 View
                    ......
                    handled = child.dispatchTouchEvent(event);
                    ......
                }
                return handled;
            }
            transformedEvent = MotionEvent.obtain(event);
        } else {
            // 3.2 若这个子 View 只能够处理事件序列中部分的焦点, 则调用 MotionEvent.split 进行焦点分割
            transformedEvent = event.split(newPointerIdBits);
        }

        if (child == null) {
            // 3.2.1 不存在子 View 则自己处理
            handled = super.dispatchTouchEvent(transformedEvent);
        } else {
            // 3.2.2 将进行焦点分割后的事件, 分发给子 View
            ......
            handled = child.dispatchTouchEvent(transformedEvent);
        }
        ......
        return handled;
    }
    
}
```
可以看到, **只要 View 能够处理当前 MotionEvent 所在事件序列中的一个焦点, 便会分发给它**, 分发之前会调用 MotionEvent.split 将事件分割成为 View 能够处理的焦点事件, 并将其分发下去

```
E/TAG: --------------------dispatch begin----------------------
E/TAG: ViewGroup PointerId is: 0, eventRawX 531.7157, eventRawY 747.5201
E/TAG: Child PointerId is: 1, eventRawX 467.7539, eventRawY 1387.9688
E/TAG: Child PointerId is: 0, eventRawX 531.7157, eventRawY 747.5201
E/TAG: --------------------dispatch begin----------------------
E/TAG: ViewGroup PointerId is: 0, eventRawX 534.09283, eventRawY 744.18854
E/TAG: Child PointerId is: 1, eventRawX 467.7539, eventRawY 1387.9688
E/TAG: Child PointerId is: 0, eventRawX 534.09283, eventRawY 744.18854
```
从测试的打印日志可以看出, **ViewGroup 中的一个事件的确会分发给这个事件序列所有的焦点处理者, 可以看到 MotionEvent.split 方法, 不仅仅分割了焦点, 并且还巧妙的将触摸事件转换到了焦点处理 View 对应的区域, 这样一来就没有任何违和感了, Google 如此处理的原因可能是为了保证触摸过程中事件的连续性, 以实现更好的交互效果**

好的, 接下来我们看看 View 中是如何处理事件的

## 二. View 的事件分发
```java
public class View implements Drawable.Callback, KeyEvent.Callback,
        AccessibilityEventSource {
    
    public boolean dispatchTouchEvent(MotionEvent event) {
        ......
        boolean result = false;
        ......
        // 获取 Mask 后的 action
        final int actionMasked = event.getActionMasked();
        // 若为 DOWN, 则停止 Scroll 操作
        if (actionMasked == MotionEvent.ACTION_DOWN) {
            // Defensive cleanup for new gesture
            stopNestedScroll();
        }
        // 处理事件
        if (onFilterTouchEventForSecurity(event)) {
            // 1. 若是拖拽滚动条, 则优先将事件交给 handleScrollBarDragging 方法消耗
            if ((mViewFlags & ENABLED_MASK) == ENABLED && handleScrollBarDragging(event)) {
                result = true;
            }
            // 2. 若 ListenerInfo 中设置了 OnTouchListener, 则尝试将其交给 mOnTouchListener 消耗
            ListenerInfo li = mListenerInfo;
            if (li != null && li.mOnTouchListener != null
                    && (mViewFlags & ENABLED_MASK) == ENABLED
                    && li.mOnTouchListener.onTouch(this, event)) {
                result = true;
            }
            // 3. 若上面的操作没有将 result 位置 true, 则交给 onTouchEvent 消耗
            if (!result && onTouchEvent(event)) {
                result = true;
            }
        }
        ......
        return result;
    }
               
}
```
好的, 可以看到主要有三个操作
- 优先将事件交给 ScrollBar 处理
  - 成功消费则将 result 置为 true
- 次优先将事件交给 OnTouchListener 处理
  - 成功消费则将 result 置为 true
- 最后将事件交给 onTouchEvent 处理

好的, 可以看到这里 View 的事件分发比 ViewGroup 的要简单太多太多, 接下来我们看看 View 的事件处理

### onTouchEvent 处理事件
```java
public class View implements Drawable.Callback, KeyEvent.Callback,
        AccessibilityEventSource {
            
    public boolean onTouchEvent(MotionEvent event) {
        final float x = event.getX();
        final float y = event.getY();
        final int viewFlags = mViewFlags;
        final int action = event.getAction();
        // 判断当前 View 是否是可点击的
        final boolean clickable = ((viewFlags & CLICKABLE) == CLICKABLE
                || (viewFlags & LONG_CLICKABLE) == LONG_CLICKABLE)
                || (viewFlags & CONTEXT_CLICKABLE) == CONTEXT_CLICKABLE;

        ......
        // 若设置了代理, 则交由代理处理
        if (mTouchDelegate != null) {
            if (mTouchDelegate.onTouchEvent(event)) {
                return true;
            }
        }
        // 执行事件处理
        if (clickable || (viewFlags & TOOLTIP) == TOOLTIP) {
            switch (action) {
                // 1. 处理按压事件
                case MotionEvent.ACTION_DOWN:
                    ......
                    // 处理 View 在可滑动容器中的按压事件
                    if (isInScrollingContainer) {
                        ......
                    } else {
                        // 处理在非滑动容器中的按压事件
                        // 1.1 设置为 Pressed 状态
                        setPressed(true, x, y);
                        // 1.2 尝试添加一个长按事件
                        checkForLongClick(0, x, y);
                    }
                    break;
                // 2. 处理移动事件
                case MotionEvent.ACTION_MOVE:
                    ......
                    // 处理若移动到了 View 之外的情况
                    if (!pointInView(x, y, mTouchSlop)) {
                        // 移除 TapCallback
                        removeTapCallback();
                        // 移除长按的 Callback
                        removeLongPressCallback();
                        // 清除按压状态
                        if ((mPrivateFlags & PFLAG_PRESSED) != 0) {
                            setPressed(false);
                        }
                        ......
                    }
                    break;
                // 3. 处理 UP 事件
                case MotionEvent.ACTION_UP:
                    // 3.1 若置为不可点击了, 则移除相关回调, 重置相关 Flag
                    if (!clickable) {
                        removeTapCallback();
                        removeLongPressCallback();
                        ......
                        break;
                    }
                    // 3.2 判断 UP 时, Viwe 是否处于按压状态
                    boolean prepressed = (mPrivateFlags & PFLAG_PREPRESSED) != 0;
                    if ((mPrivateFlags & PFLAG_PRESSED) != 0 || prepressed) {
                        ......
                        // 3.2.1 若没有触发长按事件, 则处理点击事件
                        if (!mHasPerformedLongPress && !mIgnoreNextUpEvent) {
                            removeLongPressCallback();// 移除长按回调
                            // 处理点击事件
                            if (!focusTaken) {
                                // 创建一个事件处理器
                                if (mPerformClick == null) {
                                    mPerformClick = new PerformClick();
                                }
                                // 发送到 MessageQueue 中执行
                                if (!post(mPerformClick)) {
                                    performClickInternal();
                                }
                            }
                        }
                        // 清除按压状态
                        ......
                    }
                    ......
                    break;
                // 4. 处理取消事件
                case MotionEvent.ACTION_CANCEL:
                    // 移除回调, 重置标记位
                    if (clickable) {
                        setPressed(false);
                    }
                    removeTapCallback();
                    removeLongPressCallback();
                    mInContextButtonPress = false;
                    mHasPerformedLongPress = false;
                    mIgnoreNextUpEvent = false;
                    break;
            }
            return true;
        }
        return false;
    }
                
}
```
好的, 可以看到 onTouchEvent 中也做了非常多的操作, 这里省略了一些代码, 我们主要看看它如何处理不同事件的
- ACTION_DOWN
  - 将 View 设置为按压状态 
  - 添加了一个长按监听器 
- ACTION_MOVE
  - 若坐标在 View 的范围之外了, 则移除相关回调, 清除按压的状态 
- ACTION_UP
  - 若长按事件没有响应, 则处理 View 的点击事件
  - 移除按压状态
- ACTION_CANCEL
  - 移除相关回调, 清除按压状态

好的, 至此 View 对事件的处理就分析完了

## 总结
### ViewGroup 的事件分发
- **拦截事件**
  - 若为 **起始事件 ACTION_DOWN/ACTION_POINTER_DOWN** 或 **存在子 View 正在消费事件**, 则调用 onInterceptTouchEvent 尝试拦截
    - 若设置了 FLAG_DISALLOW_INTERCEPT 则不进行拦截操作, 直接分发给子 View
  - 若非起始事件, 并且无子 View 响应事件, 则进行拦截操作
- **找寻起始事件的处理者**
  - 通过 canViewReceivePointerEvents 和 isTransformedTouchPointInView 找寻能区域内的子 View
    - 若子 View 已经响应了一个焦点事件序列, 则将这个事件序列的 id 添加到这个 View 对应的 TouchTarget 的 pointerIdBits 中
    - 若子 View 当前未响应事件, 则调用 dispatchTransformedTouchEvent 尝试让子 View 处理
      - 处理成功, 则将这个 View 构建成 TouchTarget 保存起来 
  - 若无处理者, 则交由 mFirstTouchTarget 处理
- **执行事件分发**
  - 若无响应目标, 则 ViewGroup 自行处理
  - 若存在响应目标, 则遍历 TouchTarget 链表, 将事件分发给所有的 TouchTarget

### View 的事件分发
- 优先将事件交给 ScrollBar 处理
  - 成功消费则将 result 置为 true
- 次优先将事件交给 OnTouchListener 处理
  - 成功消费则将 result 置为 true
- 最后将事件交给 onTouchEvent 处理

### 疑问解答
- **为什么属性动画操作之后的 View 仍然可以响应事件流?**
  - 属性动画不会改变 View 真正的位置, 它改变的是 View 硬件渲染结点 mRenderNode 的矩阵数据
    - RenderNode 是 View 硬件渲染机制中用来捕获 View 渲染动作的, 关于其具体的实现请[查阅这篇文章](https://sharrychoo.github.io/blog/2019/08/14/android-source-graphic-producer8.html) 
  - ViewGroup 的 dispatchTouchTarget 在找寻事件位置所在区域的子 View 时, 会计算 View 矩阵映射后的坐标, 因此事件分发天然就是支持属性动画变换的

- **为什么 ViewGroup 会将一个事件会分发给所有的 TouchTarget, 只分发给响应位置的 TouchTarget 不行吗?**
  - 当一个手指按在屏幕上进行滑动, 另一个手指也按了上去, 此时他们都是连续的触摸事件
  - 但事件分发一次只能分发一个事件, 为了保证所有 View 响应的连续性, 分发时会调用 MotionEvent.split 方法, 巧妙的将触摸事件转换到了焦点处理 View 对应的区域, 以实现更好的交互效果