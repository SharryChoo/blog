---
layout: article
title: "Android Native 异常捕获"
key: "Android Native 异常捕获" 
tags: PerformanceOptimization
aside:
  toc: true
---

## 前言
Native 崩溃的捕获, 一直是一个比较令人头疼的问题, 即使是开发阶段的 Native 崩溃也是非常的难以定位, 这里的解决方案是使用 google 的 [Breakpad](https://github.com/google/breakpad)

## 一. 捕获 Native 异常
想要捕获 Native 异常, 我们需要编译 google 提供的 breakpad 库, 这个库中有 android 运行时库 liblog.so 的依赖, 因此在 Android Studio 中直接编译是最好的选择

<!--more-->

### 一) 编译动态库
将 src 中的源码拷贝到 AS 工程目录下, 其 CMakeLists 的编写方式如下
```
cmake_minimum_required(VERSION 3.4.1)

# 添加 cmake 参数
set(ENABLE_INPROCESS ON)
set(ENABLE_OUTOFPROCESS ON)
set(ENABLE_LIBCORKSCREW ON)
set(ENABLE_LIBUNWIND ON)
set(ENABLE_LIBUNWINDSTACK ON)
set(ENABLE_CXXABI ON)
set(ENABLE_STACKSCAN ON)
set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -Werror=implicit-function-declaration")
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++11 ")
if (${ENABLE_INPROCESS})
    add_definitions(-DENABLE_INPROCESS)
endif ()
if (${ENABLE_OUTOFPROCESS})
    add_definitions(-DENABLE_OUTOFPROCESS)
endif ()

# 添加使用的头文件
include_directories(
        ${CMAKE_CURRENT_SOURCE_DIR}
        ${CMAKE_CURRENT_SOURCE_DIR}/common/android/include
)

# 需要编译的文件列表
file(
        GLOB
        BREAKPAD_SOURCES_LIST
        ${CMAKE_CURRENT_SOURCE_DIR}/client/linux/crash_generation/crash_generation_client.cc
        ${CMAKE_CURRENT_SOURCE_DIR}/client/linux/dump_writer_common/thread_info.cc
        ${CMAKE_CURRENT_SOURCE_DIR}/client/linux/dump_writer_common/ucontext_reader.cc
        ${CMAKE_CURRENT_SOURCE_DIR}/client/linux/handler/exception_handler.cc
        ${CMAKE_CURRENT_SOURCE_DIR}/client/linux/handler/minidump_descriptor.cc
        ${CMAKE_CURRENT_SOURCE_DIR}/client/linux/log/log.cc
        ${CMAKE_CURRENT_SOURCE_DIR}/client/linux/microdump_writer/microdump_writer.cc
        ${CMAKE_CURRENT_SOURCE_DIR}/client/linux/minidump_writer/linux_dumper.cc
        ${CMAKE_CURRENT_SOURCE_DIR}/client/linux/minidump_writer/linux_ptrace_dumper.cc
        ${CMAKE_CURRENT_SOURCE_DIR}/client/linux/minidump_writer/minidump_writer.cc
        ${CMAKE_CURRENT_SOURCE_DIR}/client/minidump_file_writer.cc
        ${CMAKE_CURRENT_SOURCE_DIR}/common/convert_UTF.c
        ${CMAKE_CURRENT_SOURCE_DIR}/common/md5.cc
        ${CMAKE_CURRENT_SOURCE_DIR}/common/string_conversion.cc
        ${CMAKE_CURRENT_SOURCE_DIR}/common/linux/elfutils.cc
        ${CMAKE_CURRENT_SOURCE_DIR}/common/linux/file_id.cc
        ${CMAKE_CURRENT_SOURCE_DIR}/common/linux/guid_creator.cc
        ${CMAKE_CURRENT_SOURCE_DIR}/common/linux/linux_libc_support.cc
        ${CMAKE_CURRENT_SOURCE_DIR}/common/linux/memory_mapped_file.cc
        ${CMAKE_CURRENT_SOURCE_DIR}/common/linux/safe_readlink.cc
)

# 汇编文件列表
file(
        GLOB
        BREAKPAD_ASM_SOURCE_LIST
        ${CMAKE_CURRENT_SOURCE_DIR}/common/android/breakpad_getcontext.S
)

set_source_files_properties(${BREAKPAD_ASM_SOURCE_LIST} PROPERTIES LANGUAGE C)

# 生成 libbreakpad.so 库
add_library(
        breakpad
        SHARED
        ${BREAKPAD_SOURCES_LIST}
        ${BREAKPAD_ASM_SOURCE_LIST}
)

# 为 breakpad 添加链接库
target_link_libraries(
        breakpad
        log
)
```
好的, 如此一来, 我们便可以获得 libbreakpad.so 动态库, 就可以利用它愉快的进行 bug 捕获了

### native crash 捕获
```
#include <stdio.h>
#include <jni.h>
#include <android/log.h>

#include "client/linux/handler/exception_handler.h"
#include "client/linux/handler/minidump_descriptor.h"

#define LOG_TAG "Native-Crash"

#define ALOGE(...) __android_log_print(ANDROID_LOG_ERROR, LOG_TAG, __VA_ARGS__)


bool DumpCallback(const google_breakpad::MinidumpDescriptor &descriptor,
                  void *context,
                  bool succeeded) {
    ALOGE("===============Native-Cash================");
    ALOGE("Dump path: %s\n", descriptor.path());
    return succeeded;
}

extern "C"
JNIEXPORT void JNICALL
Java_com_sharry_lib_nativecrashcather_NativeCrashCatcherManager_nativeInit(JNIEnv *env,
                                                                                   jclass type,
                                                                                   jstring path_) {
    const char *path = env->GetStringUTFChars(path_, 0);
    // 注册 native 捕获器
    google_breakpad::MinidumpDescriptor descriptor(path);
    static google_breakpad::ExceptionHandler eh(descriptor, NULL, DumpCallback, NULL, true, -1);
    env->ReleaseStringUTFChars(path_, path);
}

```
一旦检测到 native 的 crash 便会将 crash 数据 dump 到磁盘
```
E/google-breakpad: ==========init handlers_installed==========
E/Native-Crash: ===============Native-Cash================
E/Native-Crash: Dump path: /storage/emulated/0/crashDump/33a9d33b-3f53-4274-4fb82182-abc09c3a.dmp
```
.dmp 文件是不可读的, 需要通过 google 提供的工具进行解析

## 二. 解析 dmp 文件
dmp 的解析工具为 minidump_stackwalk, 需要编译后才能使用

### minidump_stackwalk 的编译
#### 编译平台
MacOS

#### 编译方式
```
// 1. 在源码目录下输入
./configure 
// 2. 执行 Makefile
make 
// 3. 安装到执行目录
make install 
```
执行完成之后, 在 src/processor/ 下便可以找到编译好的 minidump_stackwalk 文件了

![minidump_stackwalk](https://i.loli.net/2019/06/20/5d0b45fb5dde151445.jpg)

### 命令行解析 .dmp 文件
```
➜  ./minidump_stackwalk crash.dmp >crashLog.txt
```
crashLog.txt 这就是 dmp 的可读文件了
```
Operating system: Android
                  0.0.0 Linux 3.18.71-14176914 #1 SMP PREEMPT Wed Mar 13 16:22:55 KST 2019 armv8l
CPU: arm
     ARMv1 Qualcomm part(0x51002150) features: half,thumb,fastmult,vfpv2,edsp,neon,vfpv3,tls,vfpv4,idiva,idivt
     4 CPUs

GPU: UNKNOWN

Crash reason:  SIGSEGV /SEGV_MAPERR
Crash address: 0x0
Process uptime: not available

Thread 0 (crashed)
...
```

## 总结
native 异常捕获的成本比起 java 异常要困难的多, 并且 dmp 的文件还需要通过工具反编译之后才能看, 更是增加了 native 异常的监管难度