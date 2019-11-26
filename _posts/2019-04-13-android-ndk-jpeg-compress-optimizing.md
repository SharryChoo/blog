---
layout: article
title: "Android NDK —— JPEG 压缩的优化"
permalink: android-ndk/jpeg-compress-optimizing
key: android-ndk-jpeg-compress-optimizing
tags: NDK
---

## 前言
若想将一 RGB 的像素值封装成 JPEG 输出, 主要需要经过如下几步

色彩空间的转化 -> 离散余弦变换DCT -> 数据量化(Quantization) -> 压缩编码

为了优化 Android 的 JPEG 压缩率, 这里利用 libjpeg-turbo 探究一下 **不同压缩编码的性能表现**, 文章主体如下
- 编译 libjpeg-turbo
- JPEG 压缩编码性能比较
- Android JPEG 压缩的优化

## 一. 编译 libjpeg-turbo
### 操作系统 
MacOS Mojave version 10.14.5

### Libjpeg-turbo 版本
从 Github 上下载最新的源码即可<br>
[https://github.com/libjpeg-turbo/libjpeg-turbo](https://github.com/libjpeg-turbo/libjpeg-turbo)

### NDK 版本
NDK16

### cmake 版本
```
➜  ~ cmake -version
cmake version 3.14.5
```

### 注释版本号
为了方便使用, 我们需要先注释版本号
- 打开 libjpeg-turbo/sharedLibs/CMakeList.txt, 将设置版本号的位置注释
```
#set_target_properties(jpeg PROPERTIES SOVERSION ${SO_MAJOR_VERSION}
#  VERSION ${SO_MAJOR_VERSION}.${SO_AGE}.${SO_MINOR_VERSION})
```

### 脚本编写
Android 端脚本编写指南在 libjpeg-turbo 库中的 [BUILDING.md](https://github.com/libjpeg-turbo/libjpeg-turbo/blob/master/BUILDING.md) 中有说明
```
Building libjpeg-turbo for Android
----------------------------------

Building libjpeg-turbo for Android platforms requires v13b or later of the
[Android NDK](https://developer.android.com/tools/sdk/ndk).


### ARMv7 (32-bit)

The following is a general recipe script that can be modified for your specific
needs.

    # Set these variables to suit your needs
    NDK_PATH={full path to the NDK directory-- for example,
      /opt/android/android-ndk-r16b}
    TOOLCHAIN={"gcc" or "clang"-- "gcc" must be used with NDK r16b and earlier,
      and "clang" must be used with NDK r17c and later}
    ANDROID_VERSION={the minimum version of Android to support-- for example,
      "16", "19", etc.}

    cd {build_directory}
    cmake -G"Unix Makefiles" \
      -DANDROID_ABI=armeabi-v7a \
      -DANDROID_ARM_MODE=arm \
      -DANDROID_PLATFORM=android-${ANDROID_VERSION} \
      -DANDROID_TOOLCHAIN=${TOOLCHAIN} \
      -DCMAKE_ASM_FLAGS="--target=arm-linux-androideabi${ANDROID_VERSION}" \
      -DCMAKE_TOOLCHAIN_FILE=${NDK_PATH}/build/cmake/android.toolchain.cmake \
      [additional CMake flags] {source_directory}
    make
......
```
我们按照它的要求, 进行 shell 脚本的编写即可, 编写后的shell 脚本如下
```
# 定义变量
ARCH=arm
ANDROID_ARCH_ABI=armeabi-v7a
ANDROID_VERSION=19
NDK_PATH=/Users/sharrychoo/Library/Android/ndk/android-ndk-r16b
PREFIX=`pwd`/android/${ARCH}/${CPU}
CFALGS="-march=armv7-a -mfloat-abi=softfp -mfpu=neon"

# 使用 cmake 命令生成 Makefile
cmake -G"Unix Makefiles" \
	-DANDROID_ABI=${ANDROID_ARCH_ABI} \
	-DANDROID_ARM_MODE=${ARCH} \
	-DANDROID_PLATFORM=android-${ANDROID_VERSION} \
	-DANDROID_TOOLCHAIN=clang \
	-DCMAKE_TOOLCHAIN_FILE=${NDK_PATH}/build/cmake/android.toolchain.cmake \
	-DCMAKE_BUILD_TYPE=Release \
	-DANDROID_NDK=${NDK_PATH} \
	-DCMAKE_POSITION_INDEPENDENT_CODE=1 \
	-DCMAKE_INSTALL_PREFIX=${PREFIX} \
	-DANDROID_ARM_NEON=TRUE \
	-DANDROID_STL=c++_static \
	-DCMAKE_C_FLAGS="${CFALGS} -Os -Wall -pipe -fPIC" \
	-DCMAKE_CXX_FLAGS="${CFALGS} -Os -Wall -pipe -fPIC" \
	-DANDROID_CPP_FEATURES=rtti exceptions \
	-DWITH_JPEG8=1 \
	..

# 生成 so 库
make clean
make
make install
```
如此便可以获得 libjpeg 的 so 库, 将其和头文件拷贝到工程中即可使用

关于 libjpeg-turbo 的使用, 这里就不赘述了, 其官方提供好的 sample 如下
- [https://raw.githubusercontent.com/libjpeg-turbo/libjpeg-turbo/master/example.txt](https://raw.githubusercontent.com/libjpeg-turbo/libjpeg-turbo/master/example.txt)

得到了 libjpeg-turbo 的 so, 接下来便可以探究 JPEG 压缩编码的表现了

## 二. JPEG 压缩编码性能比较
libjpeg-turbo 支持的编码算法如下
- 霍夫曼编码
  - 优化的霍夫曼
  - 未优化的霍夫曼
- 算术编码

**我们将压缩质量设置 5, 看看他们在空间和时间两个个维度上的表现**

### 空间与时间
```
Origin file length is 8284kb
// 未优化的霍夫曼编码
libjpeg-turbo compressed file length is 433kb, cost time is 456ms
// 优化的霍夫曼编码
libjpeg-turbo compressed file length is 260kb, cost time is 499ms
// 算术编码
libjpeg-turbo compressed file length is 148kb, cost time is 459ms
```
- 压缩率上
  - **优化的霍夫曼编码比未优化的高 30%**
  - **算术编码比优化的霍夫曼编码高 40%**
- 时间消耗上, 优化的霍夫曼编码稍慢

### 问题探究
##### 为什么算术编码比优化的霍夫曼编码压缩率高? 
- 算术编码若想完成解码需求, **需要保存原始数据的编码表和长度大小, 以及编码后的一个浮点数值**
  - 编码表的大小与数据的重复度成正相关
- 霍夫曼编码若想完成解码需求, **需要保存霍夫曼编码树和编码后的整个数据**
  - 霍夫曼树的大小与数据的重复度成正相关 

算术编码从实现上就比霍夫曼更加优秀, 它编码后的结果为一个浮点数值, 而霍夫曼编码则需要保存编码后的整个数据, 这正是算术编码比霍夫曼编码的压缩率高 40% 的原因

##### 为什么优化的霍夫曼编码比未优化的压缩率高?
优化的霍夫曼编码, 即根据源数据的计算一个霍夫曼树, 并且按照这个霍夫曼树进行霍夫曼编码, 而未优化的霍夫曼算法是直接使用 Libjpeg 提供的默认的霍夫曼树进行编码
- 优势: 省去了构建霍夫曼树和霍夫曼树映射表的过程
- 劣势: **默认的霍夫曼树为了保证通用性, 势必要考虑所有的数值(假设为 0-255), 因此这个霍夫曼树要比根据数据源构建的要大一些**

了解优化的霍夫曼编码压缩率更高的原因, 其耗时更久的疑惑也同样得以解决了

## 三. Android JPEG 压缩的优化
### 源码实现
Android 的 2D 处理框架为 Skia, 在 JPEG 图像压缩上它链接了 libjpeg-turbo, 不同的是 Google 在不同的 Android 版本上的使用方式有所不同
- [Android 7.0 以下的设备未开启优化的霍夫曼编码](http://androidxref.com/6.0.1_r10/xref/external/skia/src/images/SkImageDecoder_libjpeg.cpp#onEncode)
```
jpeg_set_defaults(&cinfo);
// ... 此处未开启优化的霍夫曼压缩
#ifdef DCT_IFAST_SUPPORTED
    // 使用了快速的离散余弦变化, 丢失了精度
    cinfo.dct_method = JDCT_IFAST;
#endif
```
- [Android 7.0 以上的 skia 开启了优化的霍夫曼编码](http://androidxref.com/7.0.0_r1/xref/external/skia/src/images/SkImageDecoder_libjpeg.cpp#onEncode)
```
// 开启了优化的霍夫曼编码
cinfo.optimize_coding = TRUE;
// ...关闭了快速离散余弦, 使用了默认的离散余弦算法
```
了解了 Android 对 libjpeg 的使用思路, 接下来看看如何制定优化方案

### 优化方案
在 Android 7.0 以下的设备, 其 skia 的压缩实现是使用未优化的霍夫曼编码
- 使用优化的霍夫曼编码, 提升 30% 的压缩率
- 使用算术编码来, 提升 70% 的压缩率
  - 使用算术编码生成的 jpeg 可能存在兼容性问题
- 关闭快速离散余弦算法, 解决精度丢失的问题

在 Android 7.0 以上的设备, 其 skia 的压缩实现为使用优化的霍夫曼编码, 可以采用以下的方式优化
- 通过使用算术编码, 来提升 40% 的压缩率
  - 使用算术编码生成的 jpeg 可能存在兼容性问题
 
代码流程如下所示
```
// 在 Android 7.0 以上并且未开启算术编码, 使用 skia 实现
if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.N && !request.isArithmeticCoding) {
    skiaCompress(request, downsampledBitmap, outputFile);
} 
// 在 Android 7.0 以下, 或开启了算术编码时使用我们自己的 libjpeg-turbo 实现
else {
    libjpegTurboCompress(request, downsampledBitmap, outputFile);
}
```


## 总结
通过本次的学习与实践, 很好的利用了 libjpeg-turbo 解决了 Android JPEG 压缩率低的问题, 现如今手机的性能日益强劲, 采用了时间换空间的思路, 使用毫秒级的时间差异去换取更好的压缩率, 能够加快图片在网络上的传输, 个人认为还是非常值得的

## 参考文献
- [https://blog.csdn.net/yuxiatongzhi/article/details/81743249](https://blog.csdn.net/yuxiatongzhi/article/details/81743249)
- [https://blog.csdn.net/qq_36752072/article/details/77986159](https://blog.csdn.net/qq_36752072/article/details/77986159)