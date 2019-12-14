---
title: Android IO 监控技术
tags: PerformanceOptimization
---

## 前言
了解了 IO 优化的相关思路, 这里主要探究一下 Android 端 IO 监控技术的实现

## 一. IO 监控的意义
- 在开发过程中, 记录 IO 耗时较长的场景, 有利于代码优化
- 在线上, 可以监控不同机型的 IO 情况, 便于根据设备机型去做针对性的优化

## 二. 如何实现 IO 监控
关于 IO 的监控, 主要的思路有 Java Hook 和 Native Hook

<!--more-->

Java Hook 主要有**插桩**和**动态代理**两种方式
- 插桩: 无法覆盖所有的 IO 操作, 大量系统代码也存在 IO 操作
- 动态代理: 
  - 不同的 Android 版本, Java 层的实现变更比较大, 有大量的版本适配性工作
  - 但很多系统级的 IO 操作, 是直接调用 native 层的函数, 因此只做 java 层的 hook, 无法覆盖所有场景

因此想要实现 IO 监控, 最好的方式是使用 Native 层的 hook

### Native Hook
Java 层的 IO 操作, 最终都会对应到 Linux 的 open/read/write/close 操作, 因此只需要 hook 这些函数就能够达到目的
```
int open(const char *pathname, int flags, mode_t mode);
ssize_t read(int fd, void *buf, size_t size);
ssize_t write(int fd, const void *buf, size_t size); write_cuk
int close(int fd);
```
这些函数的实现, 定义在 libopenjdkjvm.so, libjavacore.so 和 libopenjdk.so 中, 可以覆盖到所有的 java 层 IO 调用

这里以 Tencent 开源库 [Matrix-io-canary](https://github.com/Tencent/matrix/tree/master/matrix/matrix-android/matrix-io-canary) 中 IO 操作为例, 看看它是如何实现的

#### Matrix IO 监控
```
// io_canary_jin.cc
namespace iocanary {
    ......
    
    const static char* TARGET_MODULES[] = {
        "libopenjdkjvm.so",
        "libjavacore.so",
        "libopenjdk.so"
    };
    extern "C" {
        JNIEXPORT jboolean JNICALL
        Java_com_tencent_matrix_iocanary_core_IOCanaryJniBridge_doHook(JNIEnv *env, jclass type) {
            // for 循环要 hook 的动态库
            for (int i = 0; i < TARGET_MODULE_COUNT; ++i) {
                const char* so_name = TARGET_MODULES[i];
                // 1. 通过 elfhook_open 找到 so 库的信息, 并且使用 loaded_soinfo 描述
                loaded_soinfo* soinfo = elfhook_open(so_name);
                ......
                // elfhook_replace  有两个作用, 一是将库中 open 函数替换成 ProxyOpen 的函数
                // 二是将原来的函数实现保存在传出参数 original_open 中
                // 2. hook open 函数
                elfhook_replace(soinfo, "open", (void*)ProxyOpen, (void**)&original_open);
                ......
                if (is_libjavacore) {
                    // 3. hook read 函数
                    if (!elfhook_replace(soinfo, "read", (void*)ProxyRead, (void**)&original_read)) {
                        ......
                    }
                    // 4. hook write 函数
                    if (!elfhook_replace(soinfo, "write", (void*)ProxyWrite, (void**)&original_write)) {
                        .......
                    }
                }
                // 5. hook close 函数
                elfhook_replace(soinfo, "close", (void*)ProxyClose, (void**)&original_close);
                ......
            }
            return true;
        }
        ......
    }
    ......
}
```
好的, 可以看到 Matrix 的 hook 操作有两步
- 找到对应的 so 库的内存地址信息
- 替换 so 库中的函数实现

其 elfhook_xx 函数定义在 [matrix-android-commons/src/main/cpp/elf_hook](https://github.com/Tencent/matrix/tree/master/matrix/matrix-android/matrix-android-commons/src/main/cpp/elf_hook) 中, 感兴趣的可以点击查看

这里以 read 的 hook 函数实现为例

```
// io_canary_jin.cc
namespace iocanary {
    ......
    
    extern "C" {
    
        ......
        /**
         *  Proxy for read: callback to the java layer
         */
        ssize_t ProxyRead(int fd, void *buf, size_t size) {
            // 如果不是主线程, 则直接调用原始方法
            if(!IsMainThread()) {
                return original_read(fd, buf, size);
            }
            // 1. 记录开始时间
            int64_t start = GetTickCountMicros();
            // 2. 调用系统实现
            size_t ret = original_read(fd, buf, size);
            // 3. 记录结束时间
            long read_cost_μs = GetTickCountMicros() - start;
            // 4, 回调到 java 层
            iocanary::IOCanary::Get().OnRead(fd, buf, size, ret, read_cost_μs);
            return ret;
        }
        ......
    }
    
}
```
好的, 可以看到其对于 read 函数的 Hook 也非常的清晰, 整体流程如下图所示

![库函数替换技术](https://i.loli.net/2019/06/21/5d0ca5d2ee12578731.png)

可能是考虑到多线程同步的问题 这里只对主线程做了 hook 操作, 说明还是有优化空间的, 其 Github issue 如下 https://github.com/Tencent/matrix/issues/146

#### 优势
- Native 层的 API 变更比较小, 版本控制方便
- 能够获得比 Java 代码更高的性能

## 总结
通过对 Matrix IO 的监控实现的阅读, 对 Native 替换动态库的函数实现有了一定的了解, 感觉开辟了新的大陆, 原来 Native Hook 还能这么玩

关于 IO 监控的开源库还有 Facebook 的 [Atrace](https://github.com/facebookincubator/profilo/blob/master/cpp/atrace/Atrace.cpp#L172), 感兴趣可以点击查看