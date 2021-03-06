---
title: OpenCV 形态学 —— 腐蚀与膨胀
tags: OpenCV
---

## 前言
形态学，即数学形态学(mathematical Morphology），是图像处理中应用最为广泛的技术之一，主要用于从图像中提取对表达和描绘区域形状有意义的图像分量, 这里主要介绍腐蚀与膨胀两种操作

## 图像的腐蚀
在形态区域内用**低像素 替代 高像素**

### 操作原理
- 定义一个形态区域
- 在形态区域内, **取颜色最小的作用于目标像素**

<!--more-->

### 操作代码
```
#include<opencv2/opencv.hpp>
#include<iostream>

using namespace std;
using namespace cv;

Mat src;
int elementSize = 1;
int maxSize = 21;
const char* winname = "OuputWindow";
const char* trackbarname = "Trackbar";

void trackbarCallback(int pos, void* userdata) {
	int size = pos * 2 + 1;
	Mat dst;
	// 定义形态
	Mat kernel = getStructuringElement(
		MORPH_ELLIPSE,       // 形态描述
		Size(size, size)     // 定义卷积区域
	);
	// 图像的腐蚀
	erode(src, dst, kernel);
	imshow(winname, dst);
}

void main() {
	src = imread("F:/VisualStudioSpace/OpenCV/Resource/Naruto.jpg");
	// 创建一个 Window, 名字为唯一标识符
	namedWindow(winname);
	// 创建进度条
	createTrackbar(
		trackbarname,       
		winname,
		&elementSize,
		maxSize,
		trackbarCallback
	);
	cvWaitKey(0);
}
```
### 效果展示
- 腐蚀前

![image](https://i.loli.net/2019/05/29/5cee1d9f3295a26905.png)

- 腐蚀后

![image](https://i.loli.net/2019/05/29/5cee1daa61fff23558.png)

**可以看到, 腐蚀后的图像有效的将图片中的白色噪声去除了**

## 图像的膨胀
在形态区域内用**高像素 替代 低像素**

### 操作原理
- 定义一个卷积矩阵
- 在矩阵区域内, **取颜色最大值的作用于目标像素上**

### 操作代码
```
#include<opencv2/opencv.hpp>
#include<iostream>

using namespace std;
using namespace cv;

Mat src;
int elementSize = 1;
int maxSize = 21;
const char* winname = "OuputWindow";
const char* trackbarname = "Trackbar";

void trackbarCallback(int pos, void* userdata) {
	int size = pos * 2 + 1;
	Mat dst;
	// 定义形态
	Mat kernel = getStructuringElement(
		MORPH_ELLIPSE,       // 形态描述
		Size(size, size)     // 定义卷积区域
	);
	// 图像的膨胀
	dilate(src, dst, kernel);
	imshow(winname, dst);
}

void main() {
	src = imread("F:/VisualStudioSpace/OpenCV/Resource/Naruto.jpg");
	namedWindow(winname);
	createTrackbar(
		trackbarname,       
		winname,
		&elementSize,
		maxSize,
		trackbarCallback
	);
	cvWaitKey(0);
}
```
### 效果展示
- 膨胀前

![image](https://i.loli.net/2019/05/29/5cee1ddb976f394645.png)

- 膨胀后

![image](https://i.loli.net/2019/05/29/5cee1df8ebebe53340.png)