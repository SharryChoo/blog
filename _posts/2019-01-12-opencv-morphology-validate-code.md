---
title: OpenCV 形态学 —— 验证码提取
tags: OpenCV
---

## 问题描述
去除图片中的干扰, 获取验证码信息

![参考图片](https://i.loli.net/2019/05/29/5cee1e61d673517139.png)

## 思路
形态学操作适合黑白图片, 因此需要二值化
- 图片转灰度并且取反
- 图片二值化
- 使用形态学闭操作过滤

<!--more-->

## 实现代码
```
#include<opencv2/opencv.hpp>
#include<iostream>

using namespace std;
using namespace cv;

void main() {
	Mat src = imread("F:/VisualStudioSpace/OpenCV/Resource/verifycode.jpg");
	imshow("Source", src);
	// 1. 转灰度并取反
	Mat gray;
	cvtColor(src, gray, COLOR_BGR2GRAY);
	Mat flip_gray = ~gray;
	imshow("flip_gray", flip_gray);
	// 2. 图片二值化
	Mat binary;
	threshold(flip_gray, binary, 40, 255, THRESH_BINARY);
	imshow("binary", binary);
	// 3. 开操作过滤
	Mat dst;
	// 定义形态
	Mat kernel = getStructuringElement(
		MORPH_RECT,          // 形态描述
		Size(7, 7)           // 定义卷积区域
	);
	morphologyEx(binary, dst, CV_MOP_OPEN, kernel);
	imshow("OPEN", dst);
	cvWaitKey(0);
}
```

![验证码过滤](https://i.loli.net/2019/05/29/5cee1e6ea723587147.png)