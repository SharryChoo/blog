---
layout: article
title: "Android 端 Libjpeg-turbo 编译与集成"
tags: NDK
aside:
  toc: true
sidebar:
  nav: layouts
---

## 前言
Android 提供的 JPEG 压缩, 是由外部链接库中的 libjpeg 实现的, 但 Google 考虑到 Android 设备性能的瓶颈, **在 Skia 调用中的三方链接库 libjpeg 时, 多处进行了阉割处理**, 这样带来的好处就是压缩的速度更快了, 但细节丢失严重, 压缩后甚至有偏绿的情况, 下面的代码便是 Android 执行 JPEG 压缩的关键
```
/**
 * SkImageDecoder_libjpeg.cpp
 */
class SkJPEGImageEncoder : public SkImageEncoder {
protected:
    virtual bool onEncode(SkWStream* stream, const SkBitmap& bm, int quality) {
        ......
        // 1. 初始化 libjpeg
        jpeg_create_compress(&cinfo);
        // 设置一些参数
        cinfo.dest = &sk_wstream;
        cinfo.image_width = bm.width();
        cinfo.image_height = bm.height();
        cinfo.input_components = 3;
        // FIXME: Can we take advantage of other in_color_spaces in libjpeg-turbo?
        cinfo.in_color_space = JCS_RGB;
        // The gamma value is ignored by libjpeg-turbo.
        cinfo.input_gamma = 1;
        jpeg_set_defaults(&cinfo);
        // 这个标志用于控制是否使用优化的哈夫曼表
        cinfo.optimize_coding = TRUE;
        jpeg_set_quality(&cinfo, quality, TRUE /* limit to baseline-JPEG values */);
        
        // 2. 开始压缩
        jpeg_start_compress(&cinfo, TRUE);

        const int       width = bm.width();
        uint8_t*        oneRowP = oneRow.reset(width * 3);

        const SkPMColor* colors = bm.getColorTable() ? bm.getColorTable()->readColors() : nullptr;
        const void*      srcRow = bm.getPixels();
        
        while (cinfo.next_scanline < cinfo.image_height) {
            JSAMPROW row_pointer[1];    /* pointer to JSAMPLE row[s] */
            writer(oneRowP, srcRow, width, colors);
            row_pointer[0] = oneRowP;
            (void) jpeg_write_scanlines(&cinfo, row_pointer, 1);
            srcRow = (const void*)((const char*)srcRow + bm.rowBytes());
        }
        
        // 3. 结束压缩
        jpeg_finish_compress(&cinfo);
        
        // 4. 释放内存
        jpeg_destroy_compress(&cinfo);

        return true;
    }
};
```
从上面的代码中, 我们定位到 cinfo.optimize_coding 这个参数
- Android7.0 之后, 这个参数为 true
  - 在图片压缩的时候, 会根据图片去计算其对应的哈夫曼表, 图片质量更高, 但是图片占用的磁盘空间也相应更高
- Android7.0 之前, 这个参数为 false
  - 使用默认的哈夫曼表, 不会去根据图片进行特定的计算, 经 Google 测试, 图片质量比使用哈夫曼低两倍左右

除此之外早期的 Android 版本, 同样考虑到性能问题, skia 引擎写了一个函数替代了原来 libjpeg 的转换函数, 好处是提高了编码速度, 坏处就是牺牲了每一个像素的精度

为了实现更快速更高质量的 JPEG 有损压缩, 因此笔者选择编译 libjpeg-turbo, 来处理项目中的图片压缩, 据官方介绍, 得益于它高度优化的哈夫曼算法, 它比 libjpeg 要快上 2-6 倍, 接下来我们来一步一步的将它集成到项目中

## 一. 环境
### 操作系统 
MacOS Mojave version 10.14.5

### Libjpeg-turbo 版本
从 Github 上下载最新的源码即可<br>
https://github.com/libjpeg-turbo/libjpeg-turbo

### NDK 版本
NDK16

### cmake 版本
```
➜  ~ cmake -version
cmake version 3.14.5
```

## 二. 准备
### 注释版本号
为了方便使用, 我们需要先注释版本号
- 打开 libjpeg-turbo/sharedLibs/CMakeList.txt, 将设置版本号的位置注释
```
#set_target_properties(jpeg PROPERTIES SOVERSION ${SO_MAJOR_VERSION}
#  VERSION ${SO_MAJOR_VERSION}.${SO_AGE}.${SO_MINOR_VERSION})
```

## 三. 编译
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
### 结果展示

![生成结果](https://user-gold-cdn.xitu.io/2019/6/13/16b4e6ac29a66941?w=1186&h=392&f=png&s=100222)

## 四. 集成
### 一) 添加
将我们上面编译好的 so 和头文件拷贝到我们的项目中

![添加](https://user-gold-cdn.xitu.io/2019/6/13/16b4e6ac29bc4564?w=339&h=255&f=png&s=2623)

### 二) CMake 链接
在 CMake 中将我们的动态了添加进去
```
# 链接头文件
include_directories(${source_dir}/jniLibs/include)

# libjpeg-turbo
add_library(libjpeg SHARED IMPORTED)
set_target_properties(
        libjpeg
        PROPERTIES
        IMPORTED_LOCATION
        ${source_dir}/jniLibs/armeabi/libjpeg.so
)

# 将打包的 so 链接到项目中
target_link_libraries(
        ......
        libjpeg
        ......
)
```

### 三)  build.gradle 
因为我们只编译了 armeabi 架构的 so, 因此我们需要再 gradle 中添加 filters
```
android {
    compileSdkVersion 28
    defaultConfig {
        minSdkVersion 16
        targetSdkVersion 28
        versionCode 1
        versionName "1.0"
        externalNativeBuild {
            ndk {
                abiFilters "armeabi-v7a"
            }
        }
    }
}
```
好的, 至此我们的集成就完成了, 接下来提供一些简单的用法

## 五. 代码的编写与测试
我们编译 libjpeg-turbo 的主要目的就是为了进行 JPEG 的高质量压缩, 关于 libjpeg-turbo 的使用, 这里就不赘述了, 其官方提供好的 sample 如下<br>
https://raw.githubusercontent.com/libjpeg-turbo/libjpeg-turbo/master/example.txt

**简单的来说, 就是将 Bitmap 的颜色通道转为 BGR, 然后传给 libjpeg-turbo API 即可**, 代码还是非常简单的
```
extern "C"
JNIEXPORT jint JNICALL
Java_com_sharry_libscompressor_Core_nativeCompress(JNIEnv *env, jclass type, jobject bitmap,
                                                   jint quality, jstring destPath_) {
    // 1. 获取 bitmap 信息
    AndroidBitmapInfo info;
    AndroidBitmap_getInfo(env, bitmap, &info);
    int cols = info.width;
    int rows = info.height;
    int format = info.format;
    LOGI("Bitmap width is %d, height is %d", cols, rows);
    // 若不为 ARGB8888, 则不给予压缩
    if (format != ANDROID_BITMAP_FORMAT_RGBA_8888) {
        LOGE("Unsupported Bitmap channels, Please ensure channels is ARGB_8888.");
        return false;
    }
    // 2. 解析数据
    LOGI("Parse bitmap pixels");
    // 锁定画布
    uchar *pixels = NULL;
    AndroidBitmap_lockPixels(env, bitmap, (void **) &pixels);
    if (pixels == NULL) {
        LOGE("Fetch Bitmap data failed.");
        return false;
    }
    // 创建存储数组
    uchar *data = (uchar *) malloc(static_cast<size_t>(cols * rows * 3));
    uchar *data_header_pointer = data;// 临时保存 data 的首地址, 用于后续释放内存
    uchar r, g, b;
    int row = 0, col = 0, pixel;
    for (row = 0; row < rows; ++row) {
        for (col = 0; col < cols; ++col) {
            // 2.1 获取像素值
            pixel = *((int *) pixels);
            // ...                                              // 忽略 A 通道值
            r = static_cast<uchar>((pixel & 0x00FF0000) >> 16); // 获取 R 通道值
            g = static_cast<uchar>((pixel & 0x0000FF00) >> 8);  // 获取 G 通道值
            b = static_cast<uchar>((pixel & 0x000000FF));       // 获取 B 通道值
            pixels += 4;
            // 2.2 为 Data 填充数据
            *(data++) = b;
            *(data++) = g;
            *(data++) = r;
        }
    }
    // 解锁画布
    AndroidBitmap_unlockPixels(env, bitmap);

    // 3. 使用 libjpeg 进行图片质量压缩
    LOGI("Lib jpeg turbo do compress");
    char *output_filename = (char *) (env)->GetStringUTFChars(destPath_, NULL);
    int result = LibJpegTurboUtils::write_JPEG_file(data_header_pointer, rows, cols,
                                                    output_filename,
                                                    quality);
    // 4. 释放资源
    LOGI("Release memory");
    free((void *) data_header_pointer);
    env->ReleaseStringUTFChars(destPath_, output_filename);
    return result;
}
```
### 效果展示

![效果展示](https://user-gold-cdn.xitu.io/2019/6/13/16b4e6ac29f94bea?w=410&h=817&f=png&s=295622)

```
I/Core: Request{inputSourceType = String, outputSourceType = Bitmap, quality = 70, destWidth = -1, destHeight = -1}
// 采样压缩之后
E/Core_native: ->> Bitmap width is 1512, height is 2016
E/Core_native: ->> Parse bitmap pixels
E/Core_native: ->> Lib jpeg turbo do compress
E/Core_native: ->> Release memory
I/Core: ->> output file is: /data/user/0/com.sharry.scompressor/cache/1555157510264.jpg
// 质量压缩之后
I/Core: ->> Output file length is 196kb
```
可以看到 1512 x 2016 的图片,  在 quality 为 70 的情况下压缩之后, 为 196kb, 当然他的依旧是非常清晰的

## 总结
到这里我们的编译与集成就完成了, 整体的过程还是比较简单的, 其效果也非常的 nice, 而且不会受到 Android SDK 版本的困扰, 感兴趣的同学可以按照上述的方式试试看。

## 参考文献
[https://blog.csdn.net/yuxiatongzhi/article/details/81743249](https://blog.csdn.net/yuxiatongzhi/article/details/81743249)