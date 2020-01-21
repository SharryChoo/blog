---
title: Android 系统架构 —— 资源管理框架导读
permalink: android-source/resources-overview
key: android-source-resources-overview
tags: AndroidFramework
sidebar:
  nav: android-source
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
InputStream is = res.openRawResource(R.raw.filename);
```

## 二. App 资源编译打包
![编译方式](https://i.loli.net/2019/12/01/tWCwNDVe3dAg6ZY.png)

Android 的 aapt 的资源打包还是比较复杂的, 这里主要介绍了资源打包的流程, 并没有深究其每一个细节, 其主要流程如下
- **ResoureTable 资源表结构**
  - Pacakge: 描述资源包
    - 一个应用程序至少有两个资源包, 一个是 app 自身的资源
    - 另一个是引用的 framwork-res.apk 中的资源 
  - AaptAssets: 描述资源文件夹
- **资源的编译**
  - 收集 AaptAssets 中资源文件夹的信息到 ResoureTable 的 Package 中
  - **编译 Xml 文件为二进制 Xml 文件**
    - 写入 Xml 头信息
    - 写入 Xml 中采集到的字符串
    - 写入 Xml 中采集到的 ID 值
    - 把所有的字符串替换成采集后的索引值
- **资源的打包**
  - **遍历 Package 中的资源项, 为其分配 ID**
    - 资源项 ID = PacakgeId + TypeId + 资源序号
    - 如 0x7f 01 0001
  - **生成资源索引表 resources.arsc**
    - 收集数据
    - 生成 Package 数据块
    - 写入 resources.arsc
  - 生成其他文件
    - 编译 AndroidManifest 文件为二进制 Xml
    - **将 resources.arsc 的 ID 导出, 生成 R.java 文件**
  - **打包为 APK 文件**

## 三. 资源管理器

![资源管理整体依赖图](https://i.loli.net/2019/12/01/8ehpGSx9wWjfBLc.png)

构建 Application 的 ContextImpl 中通过 ResourcesManager 获取 Resources 的步骤如下
- 构建 ResourcesKey
- 根据 ResourcesKey 从缓存 mResourceImpls 中获取 ResourcesImpl 对象
  - 缓存不存在则构建 ResourcesImpl 对象, 创建流程如下
    - **获取 AssetManager**
      - **构建 ApkAsset**
        - **创建 Native 的 ApkAsset, 解析  apk 中的 resources.arsc 数据到内存**
        - **保存 resources.arsc 中的资源项值的字符串资源池 StringBlock**
      - **构建 AssetManager**
        - 合并 framework-res.apk 的系统资源 ApkAsset 集合
        - 构建 Native 层的 AssetManager2
        - 将合并后的资源集合保存在 AssetManager2 中
          - 保存资源集合
          - 重新构建引用表
          - 重新构建缓存集合
          - 刷新缓存 
    - **创建 ResourcesImpl 对象**
      - 内部持有 AssetManager
- 优先从 mResourceReferences 缓存中获取 Resources 对象
  - 不存在则创建新的 Resources 对象
    - 将 ClassLoader 和 ResourcesImpl 注入 

当 Application 创建完成之后, 会通过 rewriteRValues 来重新复写 R 文件中的 ID 为运行时 ID

## 四. 资源的查找
Android 获取 layout 资源的流程主要有资源的查找和构建解析器两个步骤, 具体的流程如下

### 查找资源
- **从资源表 resource.arsc 中查找资源**
  - String 类型的资源需要到 Java 的 StringBlock 中二次查找
- **从 ApkAsset 的 StringBlock 字符串资源池获取资源路径**
  - 优先从 Java 缓存池中获取
    - 若缓存不存在, 则构建缓存
  - 到 Native 层根据 idx 查找数据
  - 投放到 Java 缓存池

### 构建解析器
有了资源的路径, 从 base.apk 压缩包中将我们要查找的资源提取出来
- AssetManager 打开资源文件
  - **获取 XML 文件资源的 Asset 对象**
    - 从 .apk 压缩包中查找我们的资源文件的数据项 ZipEntry
    - **从 .apk 中将我们的所需的文件资源提取出来**
      - 对于压缩文件
        - 通过 FileMap 将资源文件 mmap 到内存
        - 通过 Asset::createFromCompressedMap 创建 Asset 对象
      - 对于未压缩文件
        - 通过 FileMap 将资源文件 mmap 到内存
        - 通过 Asset::createFromUncompressedMap 创建 Asset 对象
  - **将 Asset 数据注入 ResXMLTree**
    - 保存二进制 XML 数据到  mOwnedData
    - 获取 XML 的 String 池
    - 获取 XML 的 ID 池
  - **将 ResXMLTree 保存到 XmlBlock 中**
- 构建 XmlResourceParser

## 参考文献
- https://blog.csdn.net/luoshengyang/article/details/8738877
- https://developer.android.com/guide/topics/resources/providing-resources