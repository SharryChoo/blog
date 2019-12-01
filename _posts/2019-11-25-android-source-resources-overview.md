---
title: Android 系统架构 —— 资源管理框架导读
permalink: android-source/resources-overview
key: android-source-resources-overview
tags: AndroidFramework
---

## 前言
Android 应用程序作为移动客户端应用程序, 它采用将代码逻辑与应用界面分离的逻辑实现, 在设计层面上进行了一次解耦的操作

这里我们看看 Android 应用程序资源的组织和管理的整体架构

<!--more-->

## 一. 资源分类
Android 的资源主要可以分为两种类型, 分别为 assets 和 res

### assets
assets 类资源放在 module 根目录的 assets 文件夹下, 它里面保存的是一些原始的文件, 可以以任何方式来进行组织

这些文件最终会被原装不动地打包在 apk 文件中, 文件访问方式如下
```
AssetManager am = getAssets();  
InputStream is = assset.open("filename");
```

### res
res 文件目录是我们开发中最常打交道的, 它当前 moudle 的 res 文件夹下, 其内部保存的文件大多都会被编译, 并赋予资源的 ID, 其内容结构如下
- animator: 存放属性动画
- anim: 存放补间动画(属性动画也可保存在此目录中, 但为了区分这两种类型, 属性动画首选 animator/ 目录)
- drawable: 存放资源图片
- mipmap: 存在 app 图标
- layout: 存放资源布局
- menu: 描述应用程序的菜单
- values: 用来存放一些简单的数据
  - theme.xml
  - styles.xml
  - strings.xml
  - colors.xml
  - dimens.xml
  - arrays.xml
- xml: 用来存放应用的配置信息, 如 7.0 之后需要的 FileProvider 等
- raw: 与 assets 的文件夹类似, 它内部的资源也会原封不动的打包在 apk 中, 不同的是它在打包的过程中会被赋予资源 ID, 我们可以通过 ID 快捷的访问它
```
Resources res = getResources();
InputStream is = res .openRawResource(R.raw.filename);
```

## 二. App 资源编译打包
![编译方式](https://i.loli.net/2019/12/01/tWCwNDVe3dAg6ZY.png)

- assets 文件
  - 直接存入 apk 内部
- res/raw 文件
  - 建立索引 ID 
  - 将 values 和 raw 之外的 Xml 文件平压成二进制 Xml, 存入 apk 内部
- 其他资源会使用 aapt 编译打包生成如下的文件
  - **将文本格式的 XML 文件平压成二进制的 XML 文件**
  - **resource.arsc: 资源索引表**
  - **R.java: 资源 ID 值**

## 三. 学习目标
- [资源打包流程](https://sharrychoo.github.io/blog/android-source/resources-package.md)
- [资源的管理者的创建](https://sharrychoo.github.io/blog/android-source/resources-manager.md)
- [资源的查找与打开](https://sharrychoo.github.io/blog/android-source/resources-find-and-open.md)

## 参考文献
- https://blog.csdn.net/luoshengyang/article/details/8738877
- https://developer.android.com/guide/topics/resources/providing-resources