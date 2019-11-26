---
layout: article
title: "Android NDK —— Libyuv 编译与使用"
tags: NDK
---

## 前言
在 Android 系统上, Camera 输出的图像一般为 NV21(YUV420SP 系列) 格式, 当我们想进行录像处理时, 会面临两个问题

#### 问题 1
**图像的旋转问题**
- 后置镜头: 需要旋转 90°
- 前置镜头: 需要旋转 270° 然后再进行镜像处理

#### 问题 2
处理好镜头的旋转后, 当我们尝试使用 MediaCodec 进行 H.264 的硬编时, 便会发现偏色的问题

<!--more-->

这是因为 MediaCodec 的 COLOR_FormatYUV420SemiPlanar 格式为 NV12, 并非是 NV21, 虽然都是 YUV420SP 系列, 但他们的排列不同, **都是先存储 Y 的数据, NV21 是 vu 交替存储, NV12 是 uv 交替存储**
```
- NV21: yyyy yyyy vu vu
- NV12: yyyy yyyy uv uv
```

为了解决这个问题, 对于这个问题网上有很多的解决思路, 我们可以在 Java 层使用进行数据操作, 不过经过测试之后发现, 在 Samsung S7 Edge 上, 录制 1080p
- 旋转与镜像: 20ms
- NV21 转 NV12: 16ms

消耗时长约为 40ms, 这也仅仅是勉强能够进行 25 帧的录制, 在使用 opencv 进行人脸识别或滤镜处理时, 能够感觉到明显的卡顿感

**libyuv 便是 google 为了解决移动端 NV21 数据处理不便所提供的开源库, 它提供了旋转, 裁剪, 镜像, 缩放等功能**

接下来看看 libyuv 的编译与使用

## 一. 环境
### 操作系统 
MacOS Mojave version 10.14.5

### Libyuv
https://chromium.googlesource.com/libyuv/libyuv/
```
git clone https://chromium.googlesource.com/libyuv/libyuv
```
![libyuv 源码](https://user-gold-cdn.xitu.io/2019/7/25/16c299bea0772d57?w=1142&h=706&f=png&s=213127)

### NDK 版本
NDK16

### cmake 版本
```
➜  ~ cmake -version
cmake version 3.14.5
```

## 二. 编译脚本
从 libyuv 的源码中, 可以看到 libyuv 已经提供了 CMakeLists.txt, 因此我们可以直接通过 cmake 生成 Makefile, 然后通过 make 对 Makefile 进行编译
```
ARCH=arm
ANDROID_ARCH_ABI=armeabi-v7a
NDK_PATH=/Users/sharrychoo/Library/Android/ndk/android-ndk-r16b
PREFIX=`pwd`/android/${ARCH}/${CPU}

# cmake 传参
cmake -G"Unix Makefiles" \
	-DANDROID_NDK=${NDK_PATH} \
    -DCMAKE_TOOLCHAIN_FILE=${NDK_PATH}/build/cmake/android.toolchain.cmake \
    -DANDROID_ABI=${ANDROID_ARCH_ABI} \
    -DANDROID_NATIVE_API_LEVE=16 \
    -DCMAKE_INSTALL_PREFIX=${PREFIX} \
	-DANDROID_ARM_NEON=TRUE \
    ..
    
# 生成动态库
make 
make install
```
![编译结果](https://user-gold-cdn.xitu.io/2019/7/25/16c299becb6e8d9e?w=1075&h=639&f=png&s=225356)

### 输出的 so 库
![so 库](https://user-gold-cdn.xitu.io/2019/7/25/16c299be9fdfef25?w=1069&h=637&f=png&s=148109)

## 三. 代码编写
我们将 so 库和头文件拷贝到 AS 中, 便可以进行代码的编写了, 这里编写一个 Libyuv 的工具类, 方便后续使用

### 一) Java 代码
这里以 NV21 转 I420 为例
```
/**
 * 处理 YUV 的工具类
 *
 * @author Sharry <a href="sharrychoochn@gmail.com">Contact me.</a>
 * @version 1.0
 * @since 2019-07-23
 */
public class LibyuvUtil {

    static {
        System.loadLibrary("smedia-camera");
    }

    /**
     * 将 NV21 转 I420
     */
    public static native void convertNV21ToI420(byte[] src, byte[] dst, int width, int height);
    
    ......
}
```

### 二) native 实现
这里以将 NV21 转 I420 为例
```
namespace libyuv_util {

    void convertI420ToNV12(JNIEnv *env, jclass, jbyteArray i420_src, jbyteArray nv12_dst, int width,
                           int height) {
        jbyte *src = env->GetByteArrayElements(i420_src, NULL);
        jbyte *dst = env->GetByteArrayElements(nv12_dst, NULL);
        // 执行转换 I420 -> NV12 的转换
        LibyuvUtil::I420ToNV12(src, dst, width, height);
        // 释放资源
        env->ReleaseByteArrayElements(i420_src, src, 0);
        env->ReleaseByteArrayElements(nv12_dst, dst, 0);
    }
    
}

void LibyuvUtil::NV21ToI420(jbyte *src, jbyte *dst, int width, int height) {
    // NV21 参数
    jint src_y_size = width * height;
    jbyte *src_y = src;
    jbyte *src_vu = src + src_y_size;
    // I420 参数
    jint dst_y_size = width * height;
    jint dst_u_size = dst_y_size >> 2;
    jbyte *dst_y = dst;
    jbyte *dst_u = dst + dst_y_size;
    jbyte *dst_v = dst + dst_y_size + dst_u_size;
    /**
    * <pre>
    * int NV21ToI420(const uint8_t* src_y,
    *          int src_stride_y,
    *          const uint8_t* src_vu,
    *          int src_stride_vu,
    *          uint8_t* dst_y,
    *          int dst_stride_y,
    *          uint8_t* dst_u,
    *          int dst_stride_u,
    *          uint8_t* dst_v,
    *          int dst_stride_v,
    *          int width,
    *          int height);
    * </pre>
    * <p>
    * stride 为颜色分量的跨距: 它描述一行像素中, 该颜色分量所占的 byte 数目, YUV 每个通道均为 1byte(8bit)
    * <p>
    * stride_y: Y 是最全的, 一行中有 width 个像素, 也就有 width 个 Y
    * stride_u: YUV420 的采样为 Y:U:V = 4:1:1, 从整体的存储来看, 一个 Y 分量的数目为 U/V 的四倍
    * 但从一行上来看, width 个 Y, 它会用到 width/2 个 U
    * stride_v: 同 stride_u 的分析方式
    */
    libyuv::NV21ToI420(
            (uint8_t *) src_y, width,
            (uint8_t *) src_vu, width,
            (uint8_t *) dst_y, width,
            (uint8_t *) dst_u, width >> 1,
            (uint8_t *) dst_v, width >> 1,
            width, height
    );
}
```
可以看到方法的调用也非常的简单, 只需要传入相关参数即可, 其中有个非常重要的参数, **stride 跨距**, 它描述一行像素中, 该颜色分量所占的 byte 数目
- YUV420 系列
  - **NV21**
    - Y: 跨距为 width
    - VU: 跨距为 width  
  - **I420P(YU12)**:
    - Y: 跨距为 width
    - U: 跨距为 width/2
    - V: 跨距为 width/2
- ABGR: 跨距为 4 *width

若对常用的色彩空间不熟悉, [请点击这里查看](https://sharrychoo.github.io/blog/2018/11/02/OpenCV-%E5%9F%BA%E7%A1%80%E7%AF%87-%E8%89%B2%E5%BD%A9%E7%A9%BA%E9%97%B4%E8%A1%A8%E7%A4%BA.html)

## 总结
通过 libyuv 进行旋转镜像转码等操作, 其时长如下
- 旋转镜像: 5~8ms
- NV21 转 NV12: 0~3ms

可以看到比起 java 代码, 几乎快了 3 倍, 这已经能够满足流畅录制的需求了

笔者将常用的 YUV 操作整理成了[demo 点击查看](https://github.com/SharryChoo/LibyuvSample), 如有需要可以将代码直接拷走使用

## 参考文献
- https://www.jianshu.com/p/533479615b2b