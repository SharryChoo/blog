---
title: OpenCV 滤波处理 —— 图像掩模
tags: OpenCV
---

## 一. 掩模操作原理
图像的掩模操作就是滤波器滤波操作
- **定义一个卷积核**
- **使用卷积核在图像上滑动**
  - 对应位置相乘, 计算的结果作用于指定的像素
- **最终图片呈现的效果, 由指定的卷积核决定**

## 二. 掩模的作用
使图片的轮廓更加清晰

## 三. 手动实现掩模
- 掩模卷积的公式描述
  - **dst_pixel = cur_pixel * 5 - left_pixel - right_pixel - top_pixel - bottom_pixel**

<!--more-->

```
#include<opencv2/opencv.hpp>

using namespace cv;

void main() {
	Mat src = imread("F:/VisualStudioSpace/OpenCV/Resource/Naruto.jpg");
	// 拷贝 src 的参数, 创建一个空的 dst;
	Mat dst = Mat::zeros(src.size(), src.type());
	// 进行掩模操作
	uchar *prev_pixel, *cur_pixel, *next_pixel, *dst_pixel;
	int channels = src.channels();
	int rows = src.rows;
	int cols = src.cols * channels;
	// 边缘部分不进行处理
	for (int i = 1; i < rows - 1; i++) {
		prev_pixel = src.ptr(i - 1);   // 上一行像素的首地址
		next_pixel = src.ptr(i + 1);   // 下一行像素的首地址
		cur_pixel = src.ptr(i);        // 当前行像素的首地址
		dst_pixel = dst.ptr(i);        // 目标输出 Mat 当前行的像素首地址
		// 边缘部分不进行处理
		for (int j = channels; j < cols - channels; j++) {
			// 自定义的掩模公式: dst_pixel = cur_pixel * 5 - left_pixel - right_pixel - top_pixel - bottom_pixel
			dst_pixel[j] = saturate_cast<uchar> (
				5 * cur_pixel[j]
				- cur_pixel[j - channels]  // 减去左边与其通道类型相同的值
				- cur_pixel[j + channels]  // 减去右边与其通道类型相同的值
				- prev_pixel[j]            // 减去上面与其通道类型相同的值
				- next_pixel[j]            // 减去下面与其通道类型相同的值
			);
		}
	}
	imshow("Origin", src);
	imshow("Mask", dst);
	cvWaitKey(0);
}
```

## 使用 OpenCV 实现掩模功能
- 定义一个用于掩模的卷积核
  - {   0,&nbsp; -1,&nbsp;  0,       <br>
&nbsp; -1,&nbsp;  5,&nbsp; -1,       <br>
&nbsp;  0,&nbsp; -1,&nbsp;  0 }
- 使用 **filter2D** 函数即可对图片的每一个像素点进行掩模操作

```
#include<opencv2/opencv.hpp>
#include<iostream>

using namespace cv;
using namespace std;

void main() {
	Mat src = imread("F:/VisualStudioSpace/OpenCV/Resource/Naruto.jpg");
	// 拷贝 src 的参数, 创建一个空的 dst;
	Mat dst = Mat::zeros(src.size(), src.type());
	// 定义掩模的卷积核
	// 即: dst_pixel = cur_pixel * 5 - left_pixel - right_pixel - top_pixel - bottom_pixel
	Mat kernel = (Mat_<char>(3, 3) <<
		0, -1, 0,
		-1, 5, -1,
		0, -1, 0
		);
	// 调用 OpenCV api 实现
	filter2D(
		src,
		dst,
		src.depth(),      // opencv 中的深度, 即图片 type 精度的程度
		kernel,           // 掩模的卷积核
		Point(-1, -1)     // 指定掩模计算之后, 最终作用的像素点, Point(-1, -1) 表示卷积核的中心对应的像素点
	);
	imshow("Origin", src);
	imshow("Mask", dst);
	cvWaitKey(0);
}
```

## 效果展示
- 通过 **dst_pixel = cur_pixel * 5 - left_pixel - right_pixel - top_pixel - bottom_pixel** 掩模过后的图片, 轮廓和线条会更加清晰硬朗了

![掩模效果展示](https://i.loli.net/2019/05/29/5cee15c309f6042482.png)