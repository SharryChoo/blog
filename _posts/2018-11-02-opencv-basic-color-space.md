---
layout: article
title: "OpenCV 基础篇 —— 色彩空间"
tags: OpenCV
---

## 一. RGB
三原色光模式（RGB color model）

### 表示方式
![RGB 颜色空间](https://i.loli.net/2019/07/26/5d3a69cd9f6f263496.jpg)

<!--more-->

- R(red): 红色
- G(green): 绿色
- B(blue): 蓝色

### 使用场景
摄影, 彩色电视, 彩色显示屏

## 二. HSV(HSL)
HSV(HSL) 是一种将 RGB 色彩模型中的点在圆柱坐标系中的表示法。

### 表示方式
![HSV 颜色空间](https://i.loli.net/2019/07/26/5d3a69ea25fc692604.png)

- **H(Hue)**: 色相(表示什么颜色)
  - 每个角度上都有不同的颜色
- **S(Saturation)**: 饱和度(表示颜色的深浅)
  - 色轮上的饱和度, 从圆心出发, 半径越大, 饱和度越高 
- **V(Value)/B(Light)**: 亮度值(表示颜色的明暗)
  - 从圆锥的顶点出发, 越往上, 亮度越大 

### 使用场景
HSV 模型通常用于计算机图形应用中

### 特点
HSV 因为它类似于人类感觉颜色的方式，具有较强的感知度。而HSV以人类更熟悉的方式封装了关于颜色的信息：“这是什么颜色？深浅如何？明暗如何？”
- 通过 S 分量, 调节饱和度更方便
- 通过 V 分量, 调节亮度更方便

## 三. YUV(YCbCr)
YUV 是北美 NTSC 系统和欧洲 PAL 系统中模拟电视信号编码的基础
- YCbCr 则是在世界数字组织视频标准研制过程中作为 ITU - R BT1601 建议的一部分, 其实是 YUV 经过缩放和偏移的翻版

### 一) 表示方式
![通道分离](https://i.loli.net/2019/07/26/5d3a69ff9b3c496889.jpg)

- Y: 亮度信号
- U(Cb): 蓝色色度分量
- V(Cr): 红色色度分量

### 二) 使用场景
YCbCr 常用于视频领域, 也是 JPEG 图像编码中采用的色彩空间
- 常用的 YUV 格式有 YUY2、YUYV、YVYU、UYVY、AYUV、Y41P、Y411、Y211、IF09、IYUV、YV12、YVU9、YUV411、YUV420

**其中 YUV420 的格式比较常用**

### 三) YUV420
YUV420 每四个 Y 共用一组 UV 分量
- 分为 **YUV420P** 和 **YUV420SP** 两种子模式

![YUV420](https://i.loli.net/2019/07/26/5d3a6a175970031696.png)

- 其中颜色相同的表示一组 YUV 的像素

#### 1. YUV420P
YUV420P 是 YUV 标准格式 YUV420, 主要分为: YU12和YV12
- **YU12(I420)**: 先存储 Y 值, 再存储 U 值, 最后存储所有 V 值
- **YV12**: 先存储 Y 值, 再存储 V 值, 最后存储所有 U 值

#### 2. YUV420SP
YUV420SP 先存储所有 Y 值, 然后是 UV/VU 交替存储, 一共两个平面, 其主要的子格式有 NV12 和 NV21
- **NV12**: 先存储 Y 值, 再存储 UV
- **NV21**: 先存储 Y 值, 再存储 VU

## 参考资料
- https://www.cnblogs.com/azraelly/archive/2013/01/01/2841269.html
- https://www.jianshu.com/p/533479615b2b