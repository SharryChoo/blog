---
layout: article
title: "Android 系统架构 —— 图形架构篇 之 ViewRootImpl 与 WMS"
key: "Android 系统架构 —— 图形架构篇 之 ViewRootImpl 与 WMS"
tags: AndroidFramework
aside:
  toc: true
---

## 前言
上一篇文章中, 我们理清了 Window 与 WindowManger 之间的关系, 我们常说 Window 是 View 的载体, 这里我们便探究一下, Window 是如何与我们的 View 产生关联的

<!--more-->

我们知道在 Activity 中调用 setContentView 之后, 页面便会展示我们布局中搭建的视图, 我们就以此为切入点, 看看 Window 是如何关联起 View 的
```
public class Activity extends ContextThemeWrapper { 
    
    public void setContentView(@LayoutRes int layoutResID) {
        // 获取 Activity.attach 中创建的 PhoneWindow 对象, 调用其 setContentView 
        getWindow().setContentView(layoutResID);
        ......
    }
    
}

public class PhoneWindow extends Window implements MenuBuilder.Callback {
    
    private LayoutInflater mLayoutInflater;
    ViewGroup mContentParent;
 
     @Override
    public void setContentView(int layoutResID) {
        if (mContentParent == null) {
            // 1. 给 Window 填充 DecorView 
            installDecor();
        } else if (!hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
            ......
        }

        if (hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
            ......// 处理 Sence 场景动画
        } else {
            // 2. 构建我们搭建的布局填充到 mContentParent 中
            mLayoutInflater.inflate(layoutResID, mContentParent);
        }
        ......
    }
    
}
```
好的, 可以看到在 PhoneWindow 的 setContentView 中, 做了两件事情
- 给 **Window 安装 DecorView**
- 将我们传入的布局填充到 mContentParent 中

接下来我们就看看 Window 是如何安装 DecorView 的

## 一. 安装 DecorView
```
public class PhoneWindow extends Window implements MenuBuilder.Callback {

    public static final int ID_ANDROID_CONTENT = com.android.internal.R.id.content;
    
    private void installDecor() {
        if (mDecor == null) {
            // 1. 创建了 DecorView 的对象
            mDecor = generateDecor(-1); // new DecorView(context, featureId, this, getAttributes())
        } else {
            // 若不为null, 则让 DecorView 直接与当前的 Window 对象绑定
            mDecor.setWindow(this);
        }
        // 2. 构建 mContentParent
        if (mContentParent == null) {
            mContentParent = generateLayout(mDecor);
        }
    }
    
    protected DecorView generateDecor(int featureId) {
        Context context;
        if (mUseDecorContext) {
            ...... // 创建 DecorView 的Context
        } else {
            context = getContext();
        }
        // 1.1 创建了 DecorView, 可以看到 this 为当前 Window 的实例
        return new DecorView(context, featureId, this, getAttributes());
    }

    protected ViewGroup generateLayout(DecorView decor) {
        ......
        // 2.1 找寻 Android 系统提供给 DecorView 的填充布局, 存入 layoutResource 中
        // 这些布局文件中均有一个 ID 为 ID_ANDROID_CONTENT 的 View
        int layoutResource;
        ......
        
        // 2.2 将 layoutResource 布局文件解析成 View 添加到了 DecorView 之中
        mDecor.onResourcesLoaded(mLayoutInflater, layoutResource);
        // 2.3 通过 findViewById 给 contentParent 赋值
        ViewGroup contentParent = (ViewGroup)findViewById(ID_ANDROID_CONTENT);
        
        ......
        return contentParent;
    }
    
    public <T extends View> T findViewById(@IdRes int id) {
        // 通过 DecorView 找寻视图
        return getDecorView().findViewById(id);
    }
}


```
可以看到在构建 DecorView 中主要做了如下的操作
- **通过 generateDecor 创建了 DecorView 的实例, 保存在 Window.mDecor 属性中**

- **通过 generateLayout 创建 View实例, 保存在 Window.mContentParent 属性中**
  - 找寻 Android 系统提供给 DecorView 的填充布局, 存入 layoutResource 中
    - 这些布局文件中均有一个 ID 为 ID_ANDROID_CONTENT 的 View
  - 调用 **DecorView.onResourcesLoaded** 将 layoutResource 加载到 DecorView 中
  - 调用 findViewById 获取 ID 为 ID_ANDROID_CONTENT 的 View, 保存到 mContentParent 中
     - 从 Window.mDecor 中查找

可以看到在 DecorView 创建之后, 会加载一个填充布局, 这个布局由系统提供, 然后便可从 DecorView 中获取一个 ID 为 ID_ANDROID_CONTENT 的 View 了, 这便是 mContentParent

接下来我们看看 DecorView 如何加载填充视图

### DecorView 加载填充布局
```
public class DecorView extends FrameLayout {

   // 若 Window 不需要这个 View, 则为 null
   DecorCaptionView mDecorCaptionView;
   ViewGroup mContentRoot;
   
   void onResourcesLoaded(LayoutInflater inflater, int layoutResource) {
        mDecorCaptionView = createDecorCaptionView(inflater);
        // 1. 根据 layoutResource 创建 root
        final View root = inflater.inflate(layoutResource, null);
        if (mDecorCaptionView != null) {
            ......
        } else {
            ......
            // 2. 添加到 DecorView 中, 作为其子容器
            addView(root, 0, new ViewGroup.LayoutParams(MATCH_PARENT, MATCH_PARENT));
        }
        // 3. 赋值给成员变量
        mContentRoot = (ViewGroup) root;
    }
    
}
```
可以看到 DecorView.onResourcesLoaded 中实例化了传入的布局文件将其添加到 DecorView 中, 并且保存在成员变量 mContentRoot 中

**至此 Window 中便可以通过 DecorView 的 findViewById 找到 ID 为 ID_ANDROID_CONTENT 的 View 了, 也就是 mContentParent**

## 二. Window 层级表
 容器 | childCount | child
---|-- |---
Window | 1 |DecorView
DecorView | 1 | mContentRoot <br> (根据feature拿到的layoutResource  例如: screen_simple.xml)
**mContentRoot**(以screen_simple.xml为例) | 2 | child1: ViewStub(@+id/action_mode_bar_stub) <br> **child2: FrameLayout(@android:id/content), 看到这个ID就放心了, 它就是mCotnentParent**

## 三. Window 层级图
![DecorView层级图.jpg](http://upload-images.jianshu.io/upload_images/4147272-00b5f60232885fac.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 四. 总结
通过本次分析可知, **Window 维护了一个 DecorView, 我们创建的视图都是添加在 DecorView 的下的, Window 通过 DecorView 便可以找到我们填充的视图了**