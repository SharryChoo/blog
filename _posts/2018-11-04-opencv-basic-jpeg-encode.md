---
layout: article
title: OpenCV 基础篇 —— JPEG 编码
tags: OpenCV
---

## 前言
我们常见的图片封装格式有 jpeg, png 和 webp, 其中 jpeg 和 webp 为有损封装格式
 
这里主要介绍一下 jpeg 的编码流程

## 一. 色彩空间的转化
- 将图片分割成为 8x8 的小方格(即 64 个像素的小方格), 每个小方格都会被单独处理
- 将 RGB 颜色空间对应的 3 个 8x8 的矩阵 ->  YCbCr 模型的 3 个 8x8 的矩阵

![色彩空间的转化](https://i.loli.net/2019/05/29/5cee0e7cc282499705.png)

<!--more-->

## 二. 离散余弦变换DCT(Discrete Cosine Transform)
对每个矩阵进行 DCT 操作
  - 矩阵的左上角的直流分量保存了一个大的数值, 其他分量都接近于 0 

![离散余弦变换DCT](https://i.loli.net/2019/05/29/5cee0e9732af970035.png)

## 三. 数据量化(Quantization)
数据量化即损失浮点数的精度, 换取更少的空间去存储浮点数

### JPEG 压缩的量化系数矩阵
![量化系数矩阵](https://i.loli.net/2019/05/29/5cee0ead8676246343.png)

### 量化操作后的矩阵
![量化操作后的矩阵](https://i.loli.net/2019/05/29/5cee0eca60b5777423.png)

可见大部分数据变成了 0, 这有利于后续的压缩算法

### 转为一维数组
- 目的: 这么做的目的只有一个, 就是尽可能把 0 放在一起, 由于 0 大部分集中在右下角, 所以才去这种由左上角到右下角的顺序, 经过这种顺序变换, 最终矩阵变成一个整数数组, 以方便后面的霍夫曼压缩
- 取值顺序
  - ![取值顺序](https://i.loli.net/2019/05/29/5cee0eebc210524671.png)
- 取值结果: -26,-3,0,-3,-2,-6,2,-4,1,-3,0,1,5,,1,2,-1,1,-1,2,0,0,0,0,0,-1,-1,0,0,0,0,…,0,0

## 四. 哈夫曼编码
对数据量化后获取的一维数组进行[哈夫曼编码](https://zh.wikipedia.org/wiki/%E9%9C%8D%E5%A4%AB%E6%9B%BC%E7%BC%96%E7%A0%81), 至此 JPEG 压缩算法就完成了

## 参考文献
- [https://thecodeway.com/blog/?p=69](https://thecodeway.com/blog/?p=69)
- [https://thecodeway.com/blog/?p=353](https://thecodeway.com/blog/?p=353)
- [https://thecodeway.com/blog/?p=480](https://thecodeway.com/blog/?p=480)
- [https://thecodeway.com/blog/?p=522](https://thecodeway.com/blog/?p=522)
