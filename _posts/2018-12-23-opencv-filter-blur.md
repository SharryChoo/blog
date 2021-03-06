---
title: OpenCV 滤波处理 —— 均值模糊与高斯模糊
tags: OpenCV
---

## 一. 均值模糊
### 卷积核定义
- **每个像素点都取其周边像素点之和的平均值**

![image](https://i.loli.net/2019/05/29/5cee16f909c4866287.png) 

- 目标像素的值: **为周边像素之和的平均值**

![image](https://i.loli.net/2019/05/29/5cee170c0884674302.png)

<!--more-->

### 算法实现
```
#include<opencv2/opencv.hpp>

using namespace cv;

void main() {
	Mat src = imread("F:/VisualStudioSpace/OpenCV/Resource/Naruto.jpg");
	// 拷贝 src 的参数, 创建一个空的 dst;
	Mat dst = Mat::zeros(src.size(), src.type());
	// 进行掩模操作
	uchar *top_pixel, *cur_pixel, *bottom_pixel, *dst_pixel;
	int channels = src.channels();
	int rows = src.rows;
	int cols = src.cols * channels;
	// 边缘部分不进行处理
	for (int i = 1; i < rows - 1; i++) {
		top_pixel = src.ptr(i - 1);      // 上一行像素的首地址
		bottom_pixel = src.ptr(i + 1);   // 下一行像素的首地址
		cur_pixel = src.ptr(i);          // 当前行像素的首地址
		dst_pixel = dst.ptr(i);          // 目标输出 Mat 当前行的像素首地址
		// 边缘部分不进行处理
		for (int j = channels; j < cols - channels; j++) {
			/* 均值模糊掩模测试 3 x 3: 
			   dst_pixel = (
			       top_left + top_center + top_right + 
				   cur_left + cur_right + 
   				   bottom_left + bottom_center + botton_right
				) / 8
			*/
			dst_pixel[j] = (
				top_pixel[j - channels] + top_pixel[j] + top_pixel[j + channels] +
				cur_pixel[j - channels] + cur_pixel[j + channels] + 
				bottom_pixel[j - channels] + bottom_pixel[j] + bottom_pixel[j + channels]
			) / 8;
		}
	}
	imshow("Origin", src);
	imshow("Mask", dst);
	cvWaitKey(0);
}
```

### OpenCV 使用方式
```
#include<opencv2/opencv.hpp>

using namespace cv;

void main() {
	Mat src = imread("F:/VisualStudioSpace/OpenCV/Resource/Naruto.jpg");
	Mat dst;
	// 均值模糊
	blur(
		src,
		dst, 
		Size(3, 3),      // 卷积核大小(需为奇数, 否则卷积相乘之后, 找不到中心点)
		Point(-1, -1)    // 指定掩模计算之后, 最终作用的像素点, Point(-1, -1) 表示卷积核的中心对应的像素点
	);
	imshow("blur", dst);
	cvWaitKey(0);
}
```

### 效果分析
- 模糊效果很差, 没有毛玻璃效果
- 有像素块, 比较粗糙

![均值模糊效果](https://i.loli.net/2019/05/29/5cee16b8bbcab63128.png)

## 二. 高斯模糊
### 卷积核定义
- **1. 根据正太分布计算权重矩阵, 权重之和为 1**

![image](https://i.loli.net/2019/05/29/5cee1756d9d9051958.jpg)

- **2. 像素点周边的像素与权重矩阵相乘**

![image](https://i.loli.net/2019/05/29/5cee17653531f15707.jpg)

- **3. 计算结果之和即为高斯模糊的像素值**

### OpenCV 使用方式
- sigmaX: 用于控制高斯模糊动态分布上升的曲率
  -  趋近于 0 时, 曲线上升趋势比较陡峭, 模糊效果不明显
  -  趋近于卷积核大小时, 曲线上升趋势平缓, 模糊效果退化成均值模糊
  -  传 0 时, 通过 **sigmaX = 0.3 * ((kenelSize - 1) * 0.5 - 1) + 0.8** 公式计算一个较为合理的值

```
#include<opencv2/opencv.hpp>

using namespace cv;

void main() {
	Mat src = imread("F:/VisualStudioSpace/OpenCV/Resource/Naruto.jpg");
	Mat dst;
	// 高斯模糊
	GaussianBlur(
		src,
		dst, 
		Size(15, 15),    // 卷积核大小(需为奇数, 否则卷积相乘之后, 找不到中心点)
		0,               // sigmaX: 传 0 会自己计算: sigmaX = 0.3 * ((kenelSize - 1) * 0.5 - 1) + 0.8
		0                // sigmaY: 不传代表和 sigmaX 值相同
	);
	imshow("Gaussian", dst);
	cvWaitKey(0);
}
```

### 效果分析
- 保留了一些轮廓, 过渡平滑
- 没有像素块
- 效果较好

![高斯模糊效果](https://i.loli.net/2019/05/29/5cee17b89982272530.jpg)

## 总结
- 均值模糊
  - 滤波矩阵实现简单, 只需相加取平均值, 算法运行速度快
  - 实现效果会有明显的色块, 过渡不自然

- 高斯模糊
  - 滤波矩阵需要根据正太分布, 进行计算, 算法运行效率低
  - 实现效果边缘过渡自然, 有毛玻璃效果