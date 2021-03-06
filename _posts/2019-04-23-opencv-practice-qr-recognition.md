---
title: OpenCV 实战 —— 二维码识别
tags: OpenCV
---

## 一. 特征分析
![示例图片](https://i.loli.net/2019/05/29/5cee39328d86229194.png)

- 二维码的区域都是方形的
- 矩形的 **左/右/下 区域有三个标识符**, 可以为圆形, 可以为方形

## 二. 实现思路
- 图片转灰度
- 二值化, 自动阈值
- 轮廓查找
- 轮廓的筛选
  - 考虑图片被旋转的问题 
  - 根据轮廓面积比例去判断
    - 二维码特征符, 夹层白色面积/中心黑色面积 约等于 2 

<!--more-->

## 三. 代码实现
```
#include<opencv2/opencv.hpp>
#include<vector>

using namespace cv;
using namespace std;

#define QR_FEATURE_NUMBER 3
#define PIXEL_BLACK 0
#define PIXEL_WHITE 255

/// 进行仿射变化
void warpTransform(const Mat &src, Mat &dst, const RotatedRect &rect) {
	int width = rect.size.width;
	int height = rect.size.height;
	// 源矩形位置
	vector<Point> srcPoints;
	Point2f pts[4];
	rect.points(pts);
	for (int i = 0; i < 4; i++)
	{
		srcPoints.push_back(pts[i]);
	}
	// 目标矩形位置
	vector<Point> dstPoints;
	dstPoints.push_back(Point(0, 0));
	dstPoints.push_back(Point(width, 0));
	dstPoints.push_back(Point(width, height));
	dstPoints.push_back(Point(0, height));
	// 进行仿射变化
	Mat matrix = findHomography(srcPoints, dstPoints);
	dst.create(Size(width, height), src.type());
	warpPerspective(src, dst, matrix, dst.size());
}

/// 查看绘制效果
void drawContours(const Mat &src, const vector<vector<Point> > &contours) {
	for (int i = 0; i < contours.size(); i++) {
		// 绘制轮廓
		drawContours(
			src,                      // 画布
			contours,                 // 轮廓集合
			i,                        // 轮廓索引
			Scalar(0, 0, 255),        // 绘制颜色
			1,
			LINE_AA
		);
	}
}

bool verifyContour(const Mat &binary, vector<Point> &contour) {
	// 1. 利用仿射, 获取旋转后的像素数据, 并写入 cur_mat
	Mat cur_mat;
	warpTransform(binary, cur_mat, minAreaRect(contour));
	// 2. 计算中心黑色区域的宽高
	int width = cur_mat.size().width, height = cur_mat.size().height;
	int c_x = width >> 1, c_y = height >> 1;
	int in_l = c_x, in_r = c_x, in_t = c_y, in_b = c_y;
	while (in_l > 0 && cur_mat.at<uchar>(c_y, in_l--) == PIXEL_BLACK);
	while (in_t > 0 && cur_mat.at<uchar>(in_t--, c_x) == PIXEL_BLACK);
	while (in_r < width && cur_mat.at<uchar>(c_y, in_r++) == PIXEL_BLACK);
	while (in_b < height && cur_mat.at<uchar>(in_b++, c_x) == PIXEL_BLACK);
	int in_w = in_r - in_l, in_h = in_b - in_t;
	// 2.1 确保中心黑色区域为正方形
	float scale;
	if ((scale = (in_w / (float)in_h)) > 1.2f || scale < 0.8f) {
		return false;
	}
	// 3. 计算中间白色夹层的宽高
	int mid_l = in_l, mid_r = in_r, mid_t = in_t, mid_b = in_b;
	while (mid_l > 0 && cur_mat.at<uchar>(c_y, mid_l--) == PIXEL_WHITE);
	while (mid_t > 0 && cur_mat.at<uchar>(mid_t--, c_x) == PIXEL_WHITE);
	while (mid_r < width && cur_mat.at<uchar>(c_y, mid_r++) == PIXEL_WHITE);
	while (mid_b < height && cur_mat.at<uchar>(mid_b++, c_x) == PIXEL_WHITE);
	int mid_w = mid_r - mid_l, mid_h = mid_b - mid_t;
	// 3.1 确保夹层的宽高比为正方形
	if ((scale = (mid_w / (float)mid_h)) > 1.2f || scale < 0.8f) {
		return false;
	}
	// 4. 验证轮廓的正确性
	// 计算 中间白色区域 与 中心黑色区域 比例
	cout << mid_w / (float)in_w << endl;
	return mid_w / (float)width <= 0.95f && (scale = (mid_w / (float)in_w)) >= 1.2f && scale <= 2.5f;
}

bool detectQrCode(const Mat &mat) {
	// 转灰度
	Mat gray;
	cvtColor(mat, gray, COLOR_BGR2GRAY);
	// 二值化
	Mat binary;
	threshold(gray, binary, 0, 255, THRESH_BINARY | THRESH_OTSU);
	// 边缘检测
	Mat candy;
	Canny(binary, candy, 100, 200, 3);
	// 找寻轮廓
	vector<Vec4i> hierarchy;
	vector<vector<Point> > contours;
	findContours(candy, contours, hierarchy, CV_RETR_TREE, CHAIN_APPROX_NONE, Point(0, 0));
	// 找寻特征轮廓
	vector<vector<Point> > candidates;
	int p_index = -1, pp_index = -1;
	for (int i = 0; i < contours.size(); i++) {
		// QR 码特征: 一个大轮廓, 嵌套两个小轮廓
		// 1. 确保存在 -> 内环(父轮廓即内环)
		p_index = hierarchy[i][2];
		if (p_index == -1) {
			continue;
		}
		// 2. 确保存在 -> 内环的内环
		pp_index = hierarchy[p_index][2];
		if (pp_index == -1) {
			continue;
		}
		// 3. 验轮廓是否满足二维码的特征
		if (verifyContour(binary, contours[i])) {
			candidates.push_back(contours[i]);
		}
	}
	// 绘制轮廓查看结果
	drawContours(mat, candidates);
	return candidates.size() >= QR_FEATURE_NUMBER;
}

void main() {
	Mat src = imread("F:/VisualStudioSpace/OpenCV/Resource/qrcode3.jpg");
	detectQrCode(src);
	imshow("src", src);
	waitKey(0);
	getchar();
}
```

## 识别效果展示
![识别结果1](https://i.loli.net/2019/05/29/5cee3948939c013654.png)

![识别结果2](https://i.loli.net/2019/05/29/5cee395505df568332.png)