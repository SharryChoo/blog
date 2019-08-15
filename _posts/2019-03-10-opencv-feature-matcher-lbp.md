---
layout: article
title: "OpenCV 特征匹配 —— LBP 提取纹理"
key:  "OpenCV 特征匹配 —— LBP 提取纹理"
tags: OpenCV
aside:
  toc: true
---

## 什么是 LBP?
LBP（Local Binary Pattern，局部二值模式）是一种**用来描述图像局部纹理特征的算子**
- 它具有旋转不变性, 和灰度不变性

## LBP 使用领域
**提取图像的局部的纹理特征**

<!--more-->

## LBP 特征提取思路
- 将图像分成 3 x 3 的窗口
- 以中心像素为阈值, 将相邻的 8 个像素的灰度值进行比较
  - 大于中心像素, 标记为 1
  - 小于等于中心像素, 标记为 0

如此一来, **3 * 3 邻域内的点可产生 8 位二进制数, 即为中心像素的 LBP 值, 并利用这个值来反映区域的纹理信息**

## LBP 特征提取代码实现
```
#include<opencv2/opencv.hpp>
#include<vector>

using namespace cv;
using namespace std;

void main() {
	Mat pattern = imread("F:/VisualStudioSpace/OpenCV/Resource/Naruto_head.jpg");

	// 1. 转灰度图
	Mat gray;
	cvtColor(pattern, gray, COLOR_BGR2GRAY);

	// 2. 创建存储 LBP 纹理信息的 Mat
	Mat result = Mat::zeros(Size(pattern.cols - 2, pattern.rows - 2), CV_8UC1);

	// 3. 卷积操作
	for (int row = 1; row < gray.rows - 1; ++row) {
		for (int col = 1; col < gray.cols - 1; ++col) {
		    // 获取中心像素
			uchar pixel = gray.at<uchar>(row, col);
			int r_pixel = 0;
			// 上面三个像素
			r_pixel |= (pixel >= gray.at<uchar>(row - 1, col - 1)) << 0;
			r_pixel |= (pixel >= gray.at<uchar>(row - 1, col)) << 1;
			r_pixel |= (pixel >= gray.at<uchar>(row - 1, col)) << 2;
			// 相邻两个像素
			r_pixel |= (pixel >= gray.at<uchar>(row, col - 1)) << 7;
			r_pixel |= (pixel >= gray.at<uchar>(row, col + 1)) << 3;
			// 下面三个像素
			r_pixel |= (pixel >= gray.at<uchar>(row + 1, col - 1)) << 6;
			r_pixel |= (pixel >= gray.at<uchar>(row + 1, col)) << 5;
			r_pixel |= (pixel >= gray.at<uchar>(row + 1, col + 1)) << 4;
			// 给纹理信息赋值
			result.at<uchar>(row - 1, col - 1) = r_pixel;
		}
	}
	// 4. 展示纹理
	imshow("lbp", result);
	waitKey(0);
	getchar();
}
```
![LBP 纹理信息](https://i.loli.net/2019/05/29/5cee33a8c1c0a73534.png)