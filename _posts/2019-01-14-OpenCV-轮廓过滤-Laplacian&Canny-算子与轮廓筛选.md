---
layout: post
title: "OpenCV 轮廓过滤 —— Laplacian & Canny 算子与轮廓筛选"
date: 2019-01-14
categories: OpenCV
tags: OpenCV Filter
img: https://i.loli.net/2019/05/29/5cee10b4a534832047.jpg
describe: Laplacian 与 Canny 如何实现? 如何进行轮廓筛选?
---

## 一. Laplacian
### 原理
- 对像素曲线的二阶求导

### 算子的卷积核
- [&nbsp;&nbsp;    0,&nbsp;      -1,&nbsp;&nbsp;  0, <br>
&nbsp;&nbsp;      -1,&nbsp;&nbsp; 4,             -1, <br>
&nbsp;&nbsp;&nbsp; 0,&nbsp;      -1,&nbsp;&nbsp;  0]

### 代码实现
```
#include<opencv2/opencv.hpp>

using namespace cv;

void main() {
	Mat src = imread("F:/VisualStudioSpace/OpenCV/Resource/card.jpg");
	imshow("src", src);
	// 降噪
	Mat gussian;
	GaussianBlur(src, gussian, Size(3, 3), 0);
	// 灰度
	Mat gray;
	cvtColor(gussian, gray, COLOR_BGR2GRAY);
	// 拉普拉斯算法
	Mat dst;
	Mat lpls = (Mat_<char>(3, 3) <<
		0, -1, 0,
		-1, 4, -1,
		0, -1, 0
		);
	filter2D(
		gray,
		dst,
		gray.depth(),
		lpls,
		Point(-1, -1)
	);
	imshow("lpls", dst);
	cvWaitKey(0);
}
```

### OpenCV API
```
#include<opencv2/opencv.hpp>

using namespace cv;

void main() {
	Mat src = imread("F:/VisualStudioSpace/OpenCV/Resource/card.jpg");
	imshow("src", src);
	// 降噪
	Mat gussian;
	GaussianBlur(src, gussian, Size(3, 3), 0);
	// 灰度
	Mat gray;
	cvtColor(gussian, gray, COLOR_BGR2GRAY);
	// 拉普拉斯算法
	Mat lpls;
	Laplacian(
		gray, 
		lpls,   
		CV_16S, // 卷积核的精度
		5       // 卷积核大小
	);
	convertScaleAbs(lpls, lpls);
	imshow("lpls", lpls);
	cvWaitKey(0);
}
```
![image](https://i.loli.net/2019/05/29/5cee215704b3239109.png)

## 二. Canny
### 原理
- 高斯去噪声
- 灰度转换
- 计算梯度(Sobel/Scharr)
- 非最大信号抑制(轮廓上留下最大的像素)
  - 可能会出现不连贯的情况 
- 高低阈值输出二值化图像
  - 在高低阈值区间内取 255, 反之取 0 
  - 一般高低值比例为: 1:2, 1:3

### 实现
```
#include<opencv2/opencv.hpp>

using namespace cv;

void main() {
	Mat src = imread("F:/VisualStudioSpace/OpenCV/Resource/card.jpg");
	imshow("src", src);
	// canny 算法
	Mat canny;
	// L1gradient: x,y 梯度绝对值求和
	// L2gradient: x,y 求平方根
	Canny(
		src,
		canny,
		50,     // 高低阈值1
		150,    // 高低阈值2, 在 [高低阈值1, 高低阈值2] 之间取 255, 否则取 0
		3,      // 卷积核大小
		false   // 梯度合并类型, L1gradient is false, L2gradient is true
	);
	imshow("canny", canny);
	cvWaitKey(0);
}
```
![image](https://i.loli.net/2019/05/29/5cee21424771474985.png)

## 三. 利用 Canny 实现轮廓查找
- 首先使用 Canny 边缘检测去除冗杂信息, 保留边缘信息
- 使用 findContours 进行轮廓查找
- 筛选对查找到的轮廓进行

### OpenCV 代码实现
```
#include<opencv2/opencv.hpp>
#include<vector>

using namespace cv;
using namespace std;

void main() {
	Mat src = imread("F:/VisualStudioSpace/OpenCV/Resource/card.jpg");
	imshow("src", src);

	// 1. Canny 边缘检测
	Mat binary;
	Canny(src, binary, 50, 150);
	imshow("binary", binary);

	// 2. 轮廓查询
	vector<vector<Point>> contours;
	findContours(
		binary,
		contours,
		RETR_LIST,
		CHAIN_APPROX_SIMPLE
	);

	// 3. 绘制轮廓
	Mat contour_mat = Mat::zeros(src.size(), CV_8UC3);
	for (int i = 0; i < contours.size(); i++) {
		// 筛选出大于画布一半的轮廓绘制出来
		Rect rect = boundingRect(contours[i]);
		if (rect.width > contour_mat.cols >> 1 && rect.height > contour_mat.rows >> 1) {
			// 绘制轮廓
			drawContours(
				contour_mat,              // 画布
				contours,                 // 轮廓集合
				i,                        // 轮廓索引
				Scalar(0, 255, 0)         // 绘制颜色
			);
			// 绘制矩形框
			rectangle(contour_mat,
				Point(rect.x, rect.y),
				Point(rect.x + rect.width, rect.y + rect.height), 
				Scalar(255, 255, 255)
			);
			break;
		}
	}
	imshow("contour_mat", contour_mat);
	cvWaitKey(0);
	getchar();
}
```
### 效果展示
![查找到的轮廓](https://i.loli.net/2019/05/29/5cee318d586a416480.png)