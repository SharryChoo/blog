---
layout: article
title: Page - Single
permalink: /page/single.html
cover: /docs/assets/images/axure/page-single.jpg
---

## 前言
在前面的分析中, 我们分别了解了 SurfaceFlinger 显示设备的创建和 Vsync 垂直同步的处理

这篇文章我们主要探究一下 SurfaceFlinger 对渲染数据的处理, 正好可以将前面所学的知识进行一次汇总

在一个 Vsync 信号到来之后, 最终会触发 SurfaceFlinger 的 onMessageReceived 的函数

A post with a single column.

<!--more-->

**front matter:**

    ---
    layout: article
    title: Page - Single
    ---