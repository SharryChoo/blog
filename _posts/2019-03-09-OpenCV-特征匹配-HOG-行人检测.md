---
layout: article
title: "OpenCV 特征匹配 —— HOG 行人检测"
date: 2019-03-09
categories: OpenCV
tags: OpenCV Hist
img: https://i.loli.net/2019/05/29/5cee10b4a534832047.jpg
describe: HOG 应用在哪些领域? 它是如何提取特征值的呢?
---

## 什么是 HOG
方向梯度直方图（Histogram of Oriented Gradient, HOG）特征是一种在计算机视觉和图像处理中用来进行物体检测的特征描述子。它通过计算和统计图像局部区域的梯度方向直方图来构成特征。

## 使用领域
**HOG 特征结合 SVM 分类器已经被广泛应用于图像识别中，尤其在行人检测中**。而如今虽然有很多行人检测算法不断提出，但基本都是以 HOG + SVM 的思路为主。

## HOG 特征提取步骤
- 灰度化（将图像看做一个x,y,z（灰度）的三维图像）
- 采用Gamma校正法对输入图像进行颜色空间的标准化（归一化）
  - 目的是调节图像的对比度，降低图像局部的阴影和光照变化所造成的影响，同时可以抑制噪音的干扰；
- 计算图像每个像素的梯度（包括大小和方向）
  - 主要是为了捕获轮廓信息，同时进一步弱化光照的干扰
- 将图像划分成小 cells（例如 16 * 16 像素/cell）
- 统计每个 cell 的梯度直方图（不同梯度的个数）
  - 形成每个 cell 的 descriptor
- 将每几个 cell 组成一个 block（例如 2 * 2 个 cell/block）, 每次合并有 1/2 的重叠区
- 将图像内的所有 block 的 HOG 特征 descriptor 串联起来

```
于 64 * 128 的图像而言，每 16 * 16 的像素组成一个cell，每 2 * 2 个 cell 组成一个 block

因为每个 cell 有 9 个特征，所以每个块内有 4 * 9 = 36 个特征
以 8 个像素为步长，那么，水平方向将有 7 个扫描窗口，垂直方向将有 15 个扫描窗口。

也就是说，64 * 128 的图片，总共有 36 * 7 * 15 = 3780 个特征。
```

## HOG 提取特征值
```
#include<opencv2/opencv.hpp>
#include<vector>

using namespace cv;
using namespace std;

void main() {
	Mat pattern = imread("F:/VisualStudioSpace/OpenCV/Resource/Naruto_head.jpg");

	// 1. 将特征分成 8 x 8 的网格, 求取直方图
	// 2. 对 ceil 中 4 x 4 的块合并, 每次必须要有 1/2 的重叠区域
	// 3. 获取直方图的数据

	/*
	 CV_WRAP HOGDescriptor() :
		winSize(64,128),                                   // 检测窗口大小，这里即完整的图
		blockSize(16,16),                                  // 块大小
		blockStride(8,8),                                  // 步进    
		cellSize(8,8),                                     // 细胞块大小
		nbins(9),                                          // 9 个bin
		derivAperture(1),
		winSigma(-1),
		histogramNormType(HOGDescriptor::L2Hys),
		L2HysThreshold(0.2),
		gammaCorrection(true),
		free_coef(-1.f),
		nlevels(HOGDescriptor::DEFAULT_NLEVELS),
		signedGradient(false)
	 {}
	 */
	HOGDescriptor descriptor;
	/*
	 CV_WRAP virtual void compute(InputArray img,
						 CV_OUT std::vector<float>& descriptors,
						 Size winStride = Size(), Size padding = Size(),
						 const std::vector<Point>& locations = std::vector<Point>()) const;
	 */
	cvtColor(pattern, pattern, COLOR_BGR2GRAY);
	resize(pattern, pattern, Size(64, 128));
	// 保存提取到的特征
	vector<float> descriptors;
	descriptor.compute(
		pattern,
		descriptors,
		Size(64, 64),  // 滑动步进
		Size()
	);
	cout << descriptors.size() << endl;

	waitKey(0);
	getchar();
}
```
通过 OpenCV 提供的 API, 使用 HOG 进行特征值的提取

## HOG 匹配行人
```
#include<opencv2/opencv.hpp>
#include<vector>

using namespace cv;
using namespace std;

void main() {
	Mat src = imread("F:/VisualStudioSpace/OpenCV/Resource/people.jpg");
	// 加载 HOG 的训练样本
	HOGDescriptor descriptor;
	descriptor.setSVMDetector(descriptor.getDefaultPeopleDetector());
	// 找寻行人
	vector<Rect> foundLocations;
	descriptor.detectMultiScale(src, foundLocations, 0 , Size(8, 8));
	// 绘制行人
	for (int i = 0; i < foundLocations.size(); i++)
	{
		rectangle(src, foundLocations[i], Scalar(255, 0, 0), 2, LINE_AA);
	}
	imshow("src", src);
	waitKey(0);
	getchar();
}
```
![HOG 匹配行人](https://i.loli.net/2019/05/29/5cee3339f25e469517.png)