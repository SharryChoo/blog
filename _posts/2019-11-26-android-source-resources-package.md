---
title: Android 系统架构 —— 资源的打包流程
permalink: android-source/resources-package
key: android-source-resources-package
tags: AndroidFramework
sidebar:
  nav: android-source
---

## 前言

![编译方式](https://i.loli.net/2019/12/01/tWCwNDVe3dAg6ZY.png)

aapt 在将我们应用的资源编译到 apk 中主要有如下几步操作
- 创建一个资源表 ResourceTable
- 编译应用资源文件
- 资源的打包

接下来我们就从这几个步骤来分析 app 的资源打包流程

<!--more-->

## 一. 资源表创建
资源表 [ResourceTable](http://androidxref.com/9.0.0_r3/xref/frameworks/base/tools/aapt/ResourceTable.h) 用于描述 aapt 打包过程中资源信息, 它的定义如下

```
// frameworks/base/tools/aapt/ResourceTable.h
class ResourceTable : public ResTable::Accessor
{
......

private:    
    // 表示当前正在编译的资源的包名称
    String16 mAssetsPackage;
    
    // 表示当前编译的资源目录
    sp<AaptAssets> mAssets;
    
    // 表示当前正在编译的资源包, 每一个包都用一个 Package 对象来描述
    DefaultKeyedVector<String16, sp<Package> > mPackages;
    ......
    
}
```
可以看到 ResourceTable 中定义了对打包资源的描述, 其中比较重要的如下
- **mPackages**: 描述正在编译的资源包集合
  - key: 包名
  - value: 资源包描述
- **mAssets**: 描述正在编译的资源目录

这里我们可能会有疑问, 编译我们当前的 app, 使用一个 Package 描述不就可以了吗, 使用集合来描述正在编译的资源包是何用意?
- 因为我们的 app 内部是会引用 Android 系统的资源的, 所以编译一个 app 至少需要两个 Package 共同工作
- 类似 "android:id" 这类的字段, 定义在 "framework-res.apk" 中

下面我们看看资源包 Pacakge 的定义

### 一) 资源包 Package 的定义
```
// frameworks/base/tools/aapt/ResourceTable.h
class ResourceTable : public ResTable::Accessor
{
    class Package : public RefBase {
    private:
        status_t setStrings(const sp<AaptFile>& data,
                            ResStringPool* strings,
                            DefaultKeyedVector<String16, uint32_t>* mappings);
        // 表示包的名称
        const String16 mName;
        
        // 表示包含的资源类型
        DefaultKeyedVector<String16, sp<Type> > mTypes;
        
        ......
    };
    
}
```
这里可以看到 Package 中持有了一个资源类型集合 mTypes
- key: **"drawable"**
- value: 资源类型描述 **Type**

#### 1. 资源类型 Type 的定义
```
// frameworks/base/tools/aapt/ResourceTable.h
class ResourceTable : public ResTable::Accessor
{
    class Type : public RefBase {
    private:
        // 资源类型名
        String16 mName;
        ......
        // 包含的资源配置项列表，每一个配置项列表都包含了一系列同名的资源
        DefaultKeyedVector<String16, sp<ConfigList> > mConfigs;
        ......
    };
}
```
资源类型 Type 中持有了一个资源配置项列表 mConfigs
- Key: 配置名称
- Value: 配置描述 ConfigList

#### 2. 资源类型的配置列表 ConfigList
```
// frameworks/base/tools/aapt/ResourceTable.h
class ResourceTable : public ResTable::Accessor
{
    
    class ConfigList : public RefBase {
    private:
        // 配置名称
        const String16 mName;
        // 配置项
        DefaultKeyedVector<ConfigDescription, sp<Entry> > mEntries;
    };
    
}
```
资源配置描述中有一个配置项的列表 mEntries
- key: 配置项描述
- value: 配置项信息 Entry

#### 3. 资源配置信息 Entry
```
// frameworks/base/tools/aapt/ResourceTable.h
class ResourceTable : public ResTable::Accessor
{
    
    class Entry : public RefBase {
    private:
        // 配置项描述
        String16 mName;
        ......
        // 资源数据
        Item mItem;
        ......
    };
    
    class Item {
        
        // 资源原始值
        String16                                value;
        // 解析后得到的结构化的资源值
        Res_value                               parsedValue;
        
    }
    
}
```
好的可以看到资源配置信息 Entry 中存在一个资源数据的描述对象 Item, 这个 Item 真正存储了该项的信息

到这里资源包相关对象的定义就查看结束了, 接下来看看正在编译的资源目录 AaptAssets 的描述

### 二) 资源目录 [AaptAssets](http://androidxref.com/9.0.0_r3/xref/frameworks/base/tools/aapt/AaptAssets.h) 的定义
```
// frameworks/base/tools/aapt/AaptAssets.h
class AaptAssets : public AaptDir
{

private:
    // 表示当前正在编译的资源的包名
    String8 mPackage;
    
    // 表示是否有引用包
    bool mHaveIncludedAssets;
    
    // 表示当前正在编译的资源的重叠包
    sp<AaptAssets> mOverlay;
    
    // 指向的是一个AssetManager, 用来解析引用包
    // 引用包都是一些预编译好的资源包, 它们需要通过 AssetManager 来解析
    AssetManager mIncludedAssets;
    
    // 表示所包含的资源类型集
    // 每一个资源类型都使用一个ResourceTypeSet来描述
    // 并且以 Type Name 为 Key 保存在一个 KeyedVector 中
    KeyedVector<String8, sp<ResourceTypeSet> >* mRes;
    
}
```
其中比较难理解的是 mOverlay 和 ResourceTypeSet
- mOverlay: 表示重叠包
  - 假设我们正在编译的是Package-1, 这时候我们可以设置另外一个Package-2, 用来告诉aapt, 如果 Package-2 定义有和 Package-1 一样的资源, 那么就用定义在 Package-2 的资源来替换掉定义在 Package-1 的资源
  - 通过这种 Overlay 机制, 我们就可以对资源进行定制, 而又不失一般性, 有些类似我们的换肤框架
- mRes: 描述资源目录下的资源类型集合
  - key: 资源类型, 如 drawable 类型
  - ResourceTypeSet: 资源类型的数据集合, 如 drawable 中所有的文件信息
    - 每一个文件信息使用 AaptFile 对象来描述

下面我们看看一个资源文件 AaptFile 的描述

#### AaptFile 的定义
```
// frameworks/base/tools/aapt/AaptAssets.h
class AaptFile : public RefBase
{
private:
    
    // 表示资源文件路径
    String8 mPath;
    
    // 表示资源文件对应的配置信息
    AaptGroupEntry mGroupEntry;
    
    // 表示资源类型名称
    String8 mResourceType;
    
    // 表示资源文件编译后得到的二进制数据
    void* mData;
    
    // 表示资源文件编译后得到的二进制数据的大小
    size_t mDataSize;
};
```
一个资源文件的描述如上所示, 其中 mData 这个属性是 XML 文件平压成二进制文件后的数据

资源目录和文件的定义就看到这里, 下面看看资源包 Package 的定义

### 三) 回顾
一个资源表 ResourceTable 中主要包含两个部分的数据, 分别是资源包 Pacakge 和资源目录 AaptAssets

#### 资源包 Pacakge
Pacakge 与 resource.arsc 的对应关系如下

![资源包 Pacakge](https://i.loli.net/2019/12/01/75DNxh4QwLBK6dz.png)

#### 资源目录 AaptAssets
![资源目录 AaptAssets](https://i.loli.net/2019/12/01/ZHoPBusY4m6g3nR.png)

了解 ResourceTable 中 Pacakge 和 AaptAssets 的实现之后, 我们看看应用资源的编译过程

## 二. 应用资源的编译
应用资源编译打包, 即将资源文件中 AaptAssets 的数据编译注入到 Pacakge 中缓存起来, 以至于后面来生成 R.java 和 resource.arsc 文件, 主要有三部流程
- 资源的收集
- 编译 XML 文件

### 一) 资源的收集
- **解析 AndroidManifest.xml**
  - 解析 AndroidManifest.xml 是为了获得要编译资源的应用程序的包名称, 用于创建一个 ResourceTable 对象 
- **添加被引用的资源包**
  - Android系统定义了一套通用资源, 这些资源可以被应用程序引用
    - Android 系统资源包为 "out/target/common/obj/APPS/framework-res_intermediates/package-export.apk" 
  - 我们在XML布局文件中指定一个 LinearLayout 的 android:orientation 属性的值为 "vertical" 时, 这个 "vertical" 实际上就是在系统资源包里面定义的一个值 
- **收集资源文件**
  - 收集 res 目录下的资源文件, 创建 AaptAssets 对象 
- **将非 values 资源增加到 ResourceTable 的 Package 中**
  -  values 资源比较特殊, 它们要经过编译之后, 才可以添加到资源表中去
- **编译 values 类资源**
  - 对 values 进行合并, 合并到一个 values.xml 中 
  - 将 values.xml 中资源添加到 ResourceTable 的 Package 中
- **给 Bag 资源分配 ID**
  - 类型为 values 的资源除了是 string 之外, 还有其它很多类型的资源, 其中有一些比较特殊, 如 bag/style/plurals/array 类的资源
  - 这些资源会给自己定义一些专用的值, 这些带有专用值的资源就统称为 Bag 资源
    - 如 android:orientation 属性的取值范围为 {"vertical" "horizontal"}, 就相当于是定义了 vertical 和 horizontal 两个 Bag

到这里资源的收集就完成了, 下面看看对 XML 的压缩过程

### 二) 编译 XML 文件
**除了 values 类型的资源文件, 其它所有的 Xml 资源文件都需要进行平压操作, 将其转为二进制, 压缩的意义如下**
- **文件占用更小**
  - 假设在原来的文本格式的XML文件中, 有四个地方使用的都是同一个字符串, 那么在最终编译出来的二进制格式的 XML 文件中, 字符串资源池只有一份字符串值, 而引用它的四个地方只占用一个整数值
- **解析速度更快**
  - 由于在二进制格式的 XML 文件中, 所有的 XML 元素标签和属性等值都是使用整数来描述的, 因此在解析的过程中, 就不再需要进行字符串解析, 这样就可以提高解析速度


XML 的编译流程如下所示

![XML 的编译流程](https://i.loli.net/2019/12/01/H4XSakeE9N2VtpK.png)

- **解析 Xml 文件**
- 为**属性名字符串赋予 ID**
  - 如为 "andorid:gravity" 赋予资源 ID
- **解析属性值**
  - 若 "@+id/xxxxx" 的 id 不存在, 则到 Package 的 ID 中分配一个 ID 值
- **平压 XML**
  - 收集收集所有属性字符串
    - ![收集字符串](https://i.loli.net/2019/12/01/t7I4ZSvj2Eezn9c.png)
  - 收集其它字符串: 控件名称, 命名空间...
    - ![收集其它字符串](https://i.loli.net/2019/12/01/BkINXdFAom2QLMy.png)
  - 写入 Xml 文件头
    - 最终编译出来的 Xml 二进制文件是一系列的 chunk 组成的, 每一个 chunk 都有一个头部, 用来描述 chunk 的元信息
    - chunk 的头部类型为 **[ResXMLTree_header](http://androidxref.com/9.0.0_r3/xref/frameworks/base/libs/androidfw/include/androidfw/ResourceTypes.h)**
  - 将 XML 中采集的字符串全部写入到当前文件的 String 常量池
    - 严格按照顺序写入字符串常量池
    - chunk 的头部类型为 **[ResStringPool_header](http://androidxref.com/9.0.0_r3/xref/frameworks/base/libs/androidfw/include/androidfw/ResourceTypes.h)**
  - 写入资源 ID
    - 在前面收集资源 ID 和属性名称过程中, 我们把属性的资源ID都收集起来了
    - 这些收集起来的资源 ID 会作为一个单独的 chunk 写入到最终的二进制 Xml 文件中去
    - 这个 chunk 位于字符串资源池的后面, 它的头部使用 **[ResChunk_header](http://androidxref.com/9.0.0_r3/xref/frameworks/base/libs/androidfw/include/androidfw/ResourceTypes.h)** 来描述
  - 压平 Xml 文件
    - 把所有的字符串替换成 String 常量池中的索引

通过 AS 查看平压后的 xml 文件, 其呈现方式如下所示, 可以发现 value 全部都被替换成了资源 ID
```
   ......
        <ImageView
            android:id="@ref/0x7f08007a"
            android:layout_width="dimension(20481)"
            android:layout_height="dimension(20481)"
            android:layout_marginLeft="dimension(7681)"
            app:layout_constraintBottom_toBottomOf="0"
            app:layout_constraintLeft_toRightOf="@ref/0x7f080078"
            app:layout_constraintRight_toRightOf="0"
            app:layout_constraintTop_toTopOf="0"
            app:srcCompat="@ref/0x7f07006f" />
    ......
```

## 三. 资源打包
### 一) 生成资源符号 ID
- 遍历资源表 Package 中每一个 Type, 取出每一个 ConfigList 的名字
- 根据这个 ConfigList 在自己 Type 中出现的次序得到它的资源 ID
- 资源符号 ID = PackgeId + TypeId + 出现则次序


![生成资源符号 ID](https://i.loli.net/2019/12/01/KqNSTaj4PUWbgsE.png)

如上图所示
- **应用程序的 PacakgeId 为 0x7f**
- **anim 的 typeId 为 0x01**

### 二) 生成资源索引表 resource.arsc
#### 1. 收集数据
- 收集**类型字符串**
  - 即 Package 中 Type 的名称
  - 如: "layout", "drawable", "anim"...
- 收集**资源项目名称字符串**
  - 即 ConfigList 的 Name
- 收集**资源项值的字符串**
  - 如 "res/drawable-xxhdpi/icon.png", "res/drawable-xxxhdpi/icon.png"...

![收集资源表的数据](https://i.loli.net/2019/12/01/dLtNYcVMoTAJ4G9.png)

#### 2. 生成 Package 数据块
- 写入 Pacakge 头
  - **ResTable_package**
- 写入**类型字符串**资源池
- 写入**资源项名称字符串**资源池
- 写入类型规范数据块
  - **ResTable_typeSpec**
- 写入类型资源项数据块
  - **ResTable_type**

#### 3. 将数据写入 resources.arsc
- 写入资源索引表头部
  - ResTable_header
- 写入**资源项值**的字符串资源池
  - 紧跟在索引表头后面
- 将生成的 Packge 数据块写入
  - 紧跟在全局字符资源池后面

而后生成的 resources.arsc 文件格式如下所示

![resources.arsc结构](https://i.loli.net/2019/12/01/9dTVoLOMSsyntuH.png)

### 三) 其他
- 编译 AndroidManifest 文件
- 将 resources.arsc 的 ID 导出, 生成 R.java 文件
- 打包为 APK 文件

## 总结
Android 的 aapt 的资源打包还是比较复杂的, 这里主要介绍了资源打包的流程, 并没有深究其每一个细节, 其主要流程如下
- **ResoureTable 资源表结构**
  - Pacakge: 描述资源包
    - 一个应用程序至少有两个资源包, 一个是 app 自身的资源
    - 另一个是引用的 framwork-res.apk 中的资源 
  - AaptAssets: 描述资源文件夹
- **资源的编译**
  - 收集 AaptAssets 中资源文件夹的信息到 ResoureTable 的 Package 中
  - 编译 Xml 文件为二进制 Xml 文件
    - 写入 Xml 头信息
    - 写入 Xml 中采集到的字符串
    - 写入 Xml 中采集到的 ID 值
    - 把所有的字符串替换成采集后的索引值
- **资源的打包**
  - 遍历 Package 中的资源项, 为其分配 ID
    - 资源项 ID = PacakgeId + TypeId + 资源序号
    - 如 0x7f 01 0001
  - 生成资源索引表 resources.arsc
    - 收集数据
    - 生成 Package 数据块
    - 写入 resources.arsc
  - 生成其他文件
    - 编译 AndroidManifest 文件为二进制 Xml
    - 将 resources.arsc 的 ID 导出, 生成 R.java 文件
  - 打包为 APK 文件 

## 参考文献
- https://blog.csdn.net/luoshengyang/article/details/8744683
- https://www.jianshu.com/p/817a787910f2