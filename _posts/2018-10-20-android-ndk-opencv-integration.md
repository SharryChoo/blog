---
title: OpenCV 的集成
tags: OpenCV
---

## OpenCV
https://opencv.org/

## OpenCV 源码
https://github.com/opencv/opencv/releases
- 下载 opencv-3.4.4-android-sdk.zip

<!--more-->

## 拷贝头文件和 so 到 Android 工程
- E:\OpenCV\opencv-3.4.4-android-sdk\sdk\native\libs\armeabi
- E:\OpenCV\opencv-3.4.4-android-sdk\sdk\native\jni\include

![OpenCV 目录结构](https://i.loli.net/2019/05/29/5cee3e9367a1084305.png)

## Gradle 的配置
在当前的 Module 下
- 指定 CPU 的架构类型
- 添加 ndk 文件的查找目录

```
android {
    ......
    externalNativeBuild {
        cmake {
            path "CMakeLists.txt"
        }
    }
    defaultConfig {
        multiDexEnabled true
        ndk {
            // 只生成 armeabi 的 CPU 架构的 .so
            abiFilters "armeabi"
        }
    }
    sourceSets {
        main {
            jniLibs.srcDirs = ['libs']
            // 添加 ndk 文件查找路径
            jniLibs.srcDirs = ['src/main/jniLibs']
        }
    }
}
```

## CMake 的配置
```
# For more information about using CMake with Android Studio, read the
# documentation: https://d.android.com/studio/projects/add-native-code.html

# Sets the minimum version of CMake required to build the native library.
cmake_minimum_required(VERSION 3.4.1)

# 判断编译器类型,如果是gcc编译器,则在编译选项中加入c++11支持
#set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=gnu++11")
if (CMAKE_COMPILER_IS_GNUCXX)
    set(CMAKE_CXX_FLAGS "-std=c++11 ${CMAKE_CXX_FLAGS}")
    message(STATUS "optional:-std=c++11")
endif (CMAKE_COMPILER_IS_GNUCXX)

# CPP 中需要引入的头文件, 以这个配置的目录为基准, 配置后才可以使用相对路径
include_directories(src/main/jniLibs/include)

# OpenCV 配置目录
set(distribution_DIR ${CMAKE_SOURCE_DIR}/../../../../src/main/jniLibs)

# 添加 opencv 的 so 库
add_library(
        opencv_java3
        SHARED
        IMPORTED
)
set_target_properties(
        opencv_java3
        PROPERTIES IMPORTED_LOCATION
        ../../../../src/main/jniLibs/armeabi/libopencv_java3.so
)

# 添加我们自己写的 cpp
add_library(
        opencv-lib
        SHARED
        src/main/cpp/opencv-lib.cpp
        src/main/cpp/impl/opencv_util.cpp
        src/main/cpp/impl/card_ocr.cpp
        src/main/cpp/impl/face_detection.cpp
)

# 链接的库
target_link_libraries(
        opencv-lib
        opencv_java3
        jnigraphics
        ${log-lib}
)

find_library(
        log-lib
        log
)
```

## 编写一个工具类
于处理 OpenCV 的 Mat 与 Android 的 Bitmap 之间的相互转化
- Bitmap 中的色彩空间与 Mat 的对应关系
   - **RGBA_8888** 对应  **CV_8UC4**
   - **RGBA_565**  对应  **CV_8UC2**
   - **灰度图**    对应  **CV_8UC1**

- 头文件(opencv_util.h)

```
//
// Created by Frank on 2018/6/5.
// OpenCV 工具类的头文件
//
#ifndef OPENCV_UTILS_H_
#define OPENCV_UTILS_H_

#include <jni.h>
#include <android/bitmap.h>
#include "opencv2/opencv.hpp"

using namespace cv;
using namespace std;

class OpenCVUtil {

public:

    OpenCVUtil() {}

    ~OpenCVUtil() {}

    /**
     * Bitmap 转为 Mat
     *
     * @param bitmap 需要操作的 bitmap
     * @param destMat 用于存储的 mat
     */
    static void bitmapToMat(JNIEnv *env, jobject &bitmap, Mat &destMat);

    /**
     * Mat 转为 Bitmap
     *
     * @param mat 需要操作的 mat
     * @param destBitmap  被作用的 bitmap
     */
    static void matToBitmap(JNIEnv *env, Mat &mat, jobject &destBitmap);

    /**
     * 在 bitmap 上标记矩形框
     */
    static void markRectOnBitmap(JNIEnv *env, jobject &bitmap, vector<Rect> &rects);

    /**
     * 抛出 Java 异常
     */
    static void throwRuntimeException(JNIEnv *env, const char *message);
};

#endif
```

- 实现代码(opencv_util.cpp)

```
//
// Created by Frank on 2018/6/5.
//

#include "../header/opencv_util.h"

/**
 * bitmap 与 mat 之间的转换
 */
void OpenCVUtil::bitmapToMat(JNIEnv *env, jobject &bitmap, Mat &destMat) {
    // 锁定画布
    void *pixels = 0;
    AndroidBitmap_lockPixels(env, bitmap, &pixels);

    // 获取 bitmap 的信息
    AndroidBitmapInfo info;
    AndroidBitmap_getInfo(env, bitmap, &info);

    // 创建 Mat
    // RGBA_888 <---->  CV_8UC4
    // RGB565   <---->  CV_8UC2
    // GRAY     <---->  CV_8UC1
    destMat.create(info.height, info.width, CV_8UC4);// 创建一个 8UC4 的 Mat, 用来接收 bitmap
    if (info.format == ANDROID_BITMAP_FORMAT_RGBA_8888) {// rgba_888 对应 openCV 的 CV_8UC4
        Mat temp(info.height, info.width, CV_8UC4, pixels);
        temp.copyTo(destMat);
    } else if (info.format == ANDROID_BITMAP_FORMAT_RGB_565) {
        Mat temp(info.height, info.width, CV_8UC2, pixels);
        temp.copyTo(destMat, COLOR_BGR5652RGBA);
    } else {
        // TODO: Bitmap 的色彩模式为其他情况, 另行处理
    }

    // 解锁画布
    AndroidBitmap_unlockPixels(env, bitmap);
}

/**
 * mat 与 bitmap 之间的转换
 */
void OpenCVUtil::matToBitmap(JNIEnv *env, Mat &mat, jobject &destBitmap) {
    // 锁定画布
    void *pixels;
    AndroidBitmap_lockPixels(env, destBitmap, &pixels);
    // 获取 bitmap 的信息
    AndroidBitmapInfo info;
    AndroidBitmap_getInfo(env, destBitmap, &info);
    // 创建 Mat
    // RGBA_888 <---->  CV_8UC4
    // RGB565   <---->  CV_8UC2
    // GRAY     <---->  CV_8UC1
    if (info.format == ANDROID_BITMAP_FORMAT_RGBA_8888) {// rgba_888 对应 openCV 的 CV_8UC4
        Mat temp(info.height, info.width, CV_8UC4, pixels);
        if (mat.type() == CV_8UC4) {
            mat.copyTo(temp);
        } else if (mat.type() == CV_8UC2) {
            cvtColor(mat, temp, COLOR_BGR5652RGBA);
        } else if (mat.type() == CV_8UC1) {
            cvtColor(mat, temp, COLOR_GRAY2RGBA);
        } else {
            // TODO: Mat 为其他情况, 另行处理
        }
    } else if (info.format == ANDROID_BITMAP_FORMAT_RGB_565) {
        Mat temp(info.height, info.width, CV_8UC2, pixels);
        if (mat.type() == CV_8UC2) {
            mat.copyTo(temp);
        } else if (mat.type() == CV_8UC4) {
            cvtColor(mat, temp, COLOR_RGBA2BGR565);
        } else if (mat.type() == CV_8UC1) {
            cvtColor(mat, temp, COLOR_GRAY2BGR565);
        } else {
            // TODO: Mat 为其他情况, 另行处理
        }
    } else {
        // TODO: Bitmap 的色彩模式为其他情况, 另行处理
    }
    // 解锁画布
    AndroidBitmap_unlockPixels(env, destBitmap);
}

/**
 * 标记 bitmap 上的矩形框
 */
void OpenCVUtil::markRectOnBitmap(JNIEnv *env, jobject &bitmap, vector<Rect> &rects) {
    // 将 bitmap 转为 mat
    Mat mat;
    bitmapToMat(env, bitmap, mat);
    int i = 0;
    for (; i < rects.size(); i++) {
        // 在原始矩阵上将人脸绘制出来
        rectangle(mat, rects[i], Scalar(255, 155, 155), 4);
    }
    // 将 origin_mat 应用到 bitmap 上去
    matToBitmap(env, mat, bitmap);
}

/**
 * 抛出 runtime 异常
 */
void OpenCVUtil::throwRuntimeException(JNIEnv *env, const char *message) {
    if (message == NULL) return;
    jclass je = env->FindClass("java/lang/Exception");
    env->ThrowNew(je, message);
}
```