---
layout: article
title: "Android 存储优化 —— MMKV 集成与原理"
key: "Android 存储优化 —— MMKV 集成与原理"
tags: PerformanceOptimization
aside:
  toc: true
---

## 前言
APP 的性能优化之路是永无止境的, 这里学习一个**腾讯开源用于提升本地存储效率的轻量级存储框架** [MMKV](https://github.com/Tencent/MMKV)

目前项目中在轻量级存储上使用的是 SharedPreferences, 虽然 SP 兼容性极好, 但 SP 的低性能一直被诟病, 线上也出现了一些因为 SP 导致的 ANR

网上有很多针对 SP 的优化方案, 这里笔者使用的是通过 Hook SP 在 Application 中的创建, 将其替换成自定义的 SP 的方式来增强性能, 但 SDK 28 以后禁止反射 QueuedWork.getHandler 接口, 这个方式就失效了

因此需要一种替代的轻量级存储方案, MMKV 便是这样的一个框架

## 一. MMKV 使用方式
以下介绍简单的使用方式, 更多详情请查看 [Wiki](https://github.com/Tencent/MMKV/wiki/android_setup_cn)

<!--more-->

### 依赖注入
在 App 模块的 build.gradle 文件里添加:
```
dependencies {
    implementation 'com.tencent:mmkv:1.0.22'
    // replace "1.0.22" with any available version
}
```

### 初始化
```
// 设置初始化的根目录
String dir = getFilesDir().getAbsolutePath() + "/mmkv_2";
String rootDir = MMKV.initialize(dir);
Log.i("MMKV", "mmkv root: " + rootDir);
```

### 获取实例
```
// 获取默认的全局实例
MMKV kv = MMKV.defaultMMKV();

// 根据业务区别存储, 附带一个自己的 ID
MMKV kv = MMKV.mmkvWithID("MyID");

// 多进程同步支持
MMKV kv = MMKV.mmkvWithID("MyID", MMKV.MULTI_PROCESS_MODE);
```

### CURD
```
// 添加/更新数据
kv.encode(key, value);

// 获取数据
int tmp = kv.decodeInt(key);

// 删除数据
kv.removeValueForKey(key);
```

### SP 的迁移
```
private void testImportSharedPreferences() {
    MMKV mmkv = MMKV.mmkvWithID("myData");
    SharedPreferences old_man = getSharedPreferences("myData", MODE_PRIVATE);
    // 迁移旧数据
    mmkv.importFromSharedPreferences(old_man);
    // 清空旧数据
    old_man.edit().clear().commit();
    ......
}
```

## 二. 性能对比
```
// MMKV
MMKV: MMKV write int: loop[1000]: 12 ms
MMKV: MMKV read int: loop[1000]: 3 ms

MMKV: MMKV write String: loop[1000]: 7 ms
MMKV: MMKV read String: loop[1000]: 4 ms

// SharedPreferences
MMKV: SharedPreferences write int: loop[1000]: 119 ms
MMKV: SharedPreferences read int: loop[1000]: 3 ms

MMKV: SharedPreferences write String: loop[1000]: 187
MMKV: SharedPreferences read String: loop[1000]: 2 ms

// SQLite
MMKV: sqlite write int: loop[1000]: 101 ms
MMKV: sqlite read int: loop[1000]: 136 ms

MMKV: sqlite write String: loop[1000]: 29 ms
MMKV: sqlite read String: loop[1000]: 93 ms
```
可以看到 MMKV 无论是对比 SP 还是 SQLite, 在性能上都有非常大的优势

![单进程读写性能对比](https://i.loli.net/2019/08/15/xIgAq2fEDvlZ4dc.png)

更详细的性能测试见 [wiki](https://github.com/Tencent/MMKV/wiki/android_benchmark_cn)

## 三. 原理分析
前面分析了 MMKV 的使用方式和其性能分析, 这里我们剖析一下它的实现原理, 看看它是如何将性能做到这个地步的

这里对主要对 MMKV 的基本操作进行剖析
- 初始化 
- 实例化
- encode
- decode
- 进程读写的同步

我们从初始化的流程开始分析

### 一) 初始化
```
public class MMKV implements SharedPreferences, SharedPreferences.Editor {
    
    // call on program start
    public static String initialize(Context context) {
        String root = context.getFilesDir().getAbsolutePath() + "/mmkv";
        return initialize(root, null);
    }

    static private String rootDir = null;

    public static String initialize(String rootDir, LibLoader loader) {
        ...... // 省略库文件加载器相关代码
        // 保存根目录
        MMKV.rootDir = rootDir;
        // Native 层初始化
        jniInitialize(MMKV.rootDir);
        return rootDir;
    }
    
    private static native void jniInitialize(String rootDir);
    
}
```
MMKV 的初始化, 主要是将根目录通过 jniInitialize 传入了 Native 层, 接下来看看 Native 的初始化操作

```
// native-bridge.cpp
namespace mmkv {
    
MMKV_JNI void jniInitialize(JNIEnv *env, jobject obj, jstring rootDir) {
    if (!rootDir) {
        return;
    }
    const char *kstr = env->GetStringUTFChars(rootDir, nullptr);
    if (kstr) {
        MMKV::initializeMMKV(kstr);
        env->ReleaseStringUTFChars(rootDir, kstr);
    }
}
    
}

// MMKV.cpp

static unordered_map<std::string, MMKV *> *g_instanceDic;
static ThreadLock g_instanceLock;
static std::string g_rootDir;

void initialize() {
    // 1.1 获取一个 unordered_map, 类似于 Java 中的 HashMap
    g_instanceDic = new unordered_map<std::string, MMKV *>;
    // 1.2 初始化线程锁
    g_instanceLock = ThreadLock();
    ......
}

void MMKV::initializeMMKV(const std::string &rootDir) {
    // 由 Linux Thread 互斥锁和条件变量保证 initialize 函数在一个进程内只会执行一次
    // https://blog.csdn.net/zhangxiao93/article/details/51910043
    static pthread_once_t once_control = PTHREAD_ONCE_INIT;
    // 1. 进行初始化操作
    pthread_once(&once_control, initialize);
    // 2. 将根目录保存到全局变量
    g_rootDir = rootDir;
    // 拷贝字符串
    char *path = strdup(g_rootDir.c_str());
    if (path) {
        // 3. 根据路径, 生成目标地址的目录
        mkPath(path);
        // 释放内存
        free(path);
    }
}
```
可以看到 initializeMMKV 中主要任务是初始化数据, 以及创建根目录
- pthread_once_t: 类似于 Java 的单例, 其 initialize 方法在进程内只会执行一次
  - 创建 MMKV 对象的缓存散列表 g_instanceDic
  - 创建一个线程锁 g_instanceLock
- mkPath: 根据字符串创建文件目录

接下来我们看看这个目录创建的过程

#### 目录的创建
```
// MmapedFile.cpp
bool mkPath(char *path) {
    // 定义 stat 结构体用于描述文件的属性
    struct stat sb = {};
    bool done = false;
    // 指向字符串起始地址
    char *slash = path;
    while (!done) {
        // 移动到第一个非 "/" 的下标处
        slash += strspn(slash, "/");
        // 移动到第一个 "/" 下标出处
        slash += strcspn(slash, "/");

        done = (*slash == '\0');
        *slash = '\0';

        if (stat(path, &sb) != 0) {
            // 执行创建文件夹的操作, C 中无 mkdirs 的操作, 需要一个一个文件夹的创建
            if (errno != ENOENT || mkdir(path, 0777) != 0) {
                MMKVWarning("%s : %s", path, strerror(errno));
                return false;
            }
        }
        // 若非文件夹, 则说明为非法路径
        else if (!S_ISDIR(sb.st_mode)) {
            MMKVWarning("%s: %s", path, strerror(ENOTDIR));
            return false;
        }

        *slash = '/';
    }
    return true;
}
```
以上是 Native 层创建文件路径的通用代码, 逻辑很清晰

好的, 文件目录创建好了之后, Native 层的初始化操作便结束了, 接下来看看 MMKV 实例构建的过程

### 二) 实例化
```
public class MMKV implements SharedPreferences, SharedPreferences.Editor {

    @Nullable
    public static MMKV mmkvWithID(String mmapID, int mode, String cryptKey, String relativePath) {
        ......
        // 执行 Native 初始化, 获取句柄值
        long handle = getMMKVWithID(mmapID, mode, cryptKey, relativePath);
        if (handle == 0) {
            return null;
        }
        // 构建一个 Java 的壳对象
        return new MMKV(handle);
    }
    
    private native static long
    getMMKVWithID(String mmapID, int mode, String cryptKey, String relativePath);
    
    // jni
    private long nativeHandle;

    private MMKV(long handle) {
        nativeHandle = handle;
    }
}
```
可以看到 MMKV 实例构建的主要逻辑通过 getMMKVWithID 方法实现, 看它内部做了什么
```
// native-bridge.cpp
namespace mmkv {

MMKV_JNI jlong getMMKVWithID(
    JNIEnv *env, jobject, jstring mmapID, jint mode, jstring cryptKey, jstring relativePath) {
    MMKV *kv = nullptr;
    if (!mmapID) {
        return (jlong) kv;
    }
    // 获取独立存储 id
    string str = jstring2string(env, mmapID);

    bool done = false;
    if (cryptKey) {
        // 获取秘钥
        string crypt = jstring2string(env, cryptKey);
        if (crypt.length() > 0) {
            if (relativePath) {
                // 获取相对路径
                string path = jstring2string(env, relativePath);
                // 通过 mmkvWithID 函数获取一个 MMKV 的对象
                kv = MMKV::mmkvWithID(str, DEFAULT_MMAP_SIZE, (MMKVMode) mode, &crypt, &path);
            } else {
                kv = MMKV::mmkvWithID(str, DEFAULT_MMAP_SIZE, (MMKVMode) mode, &crypt, nullptr);
            }
            done = true;
        }
    }
    ......
    // 强转成句柄, 返回到 Java
    return (jlong) kv;
}

}
```
可以看到最终通过  MMKV::mmkvWithID 函数获取到 MMKV 的对象
```
// MMKV.cpp
MMKV *MMKV::mmkvWithID(
    const std::string &mmapID, int size, MMKVMode mode, string *cryptKey, string *relativePath) {

    if (mmapID.empty()) {
        return nullptr;
    }
    SCOPEDLOCK(g_instanceLock);
    // 1. 通过 mmapID 和 relativePath, 组成最终的 mmap 文件路径的 key
    auto mmapKey = mmapedKVKey(mmapID, relativePath);
    // 2. 从全局缓存中查找
    auto itr = g_instanceDic->find(mmapKey);
    if (itr != g_instanceDic->end()) {
        MMKV *kv = itr->second;
        return kv;
    }
    // 3. 创建缓存文件
    if (relativePath) {
        // 根据 mappedKVPathWithID 获取 mmap 的最终文件路径
        // mmapID 使用 md5 加密
        auto filePath = mappedKVPathWithID(mmapID, mode, relativePath);
        // 不存在则创建一个文件
        if (!isFileExist(filePath)) {
            if (!createFile(filePath)) {
                return nullptr;
            }
        }
        ......
    }
    // 4. 创建实例对象
    auto kv = new MMKV(mmapID, size, mode, cryptKey, relativePath);
    // 5. 缓存这个 mmapKey
    (*g_instanceDic)[mmapKey] = kv;
    return kv;
}
```
mmkvWithID 函数的实现流程非常的清晰, 这里我们主要关注一下实例对象的创建流程
```
// MMKV.cpp
MMKV::MMKV(
    const std::string &mmapID, int size, MMKVMode mode, string *cryptKey, string *relativePath)
    : m_mmapID(mmapedKVKey(mmapID, relativePath)) 
    // 拼装文件的路径
    , m_path(mappedKVPathWithID(m_mmapID, mode, relativePath))
    // 拼装 .crc 文件路径
    , m_crcPath(crcPathWithID(m_mmapID, mode, relativePath))
    // 1. 将文件映射到内存
    , m_metaFile(m_crcPath, DEFAULT_MMAP_SIZE, (mode & MMKV_ASHMEM) ? MMAP_ASHMEM : MMAP_FILE)
    ......
    , m_sharedProcessLock(&m_fileLock, SharedLockType)
    ......
    , m_isAshmem((mode & MMKV_ASHMEM) != 0) {
    ......
    // 判断是否为 Ashmem 跨进程匿名共享内存
    if (m_isAshmem) {
        // 创共享内存的文件
        m_ashmemFile = new MmapedFile(m_mmapID, static_cast<size_t>(size), MMAP_ASHMEM);
        m_fd = m_ashmemFile->getFd();
    } else {
        m_ashmemFile = nullptr;
    }
    // 根据 cryptKey 创建 AES 加解密的引擎
    if (cryptKey && cryptKey->length() > 0) {
        m_crypter = new AESCrypt((const unsigned char *) cryptKey->data(), cryptKey->length());
    }
    ......
    // sensitive zone
    {
        SCOPEDLOCK(m_sharedProcessLock);
        // 2. 根据 m_mmapID 来加载文件中的数据
        loadFromFile();
    }
}
```
可以从 MMKV 的构造函数中看到很多有趣的信息, **MMKV 是支持 Ashmem 共享内存的, 这意味着即使是跨进程大数据的传输, 它也能够提供很好的性能支持**

不过这里我们主要关注两个关键点
- m_metaFile 文件的映射
- loadFromFile 数据的载入

接下来我们先看看, 文件的映射

#### 1. 文件映射到内存
```
// MmapedFile.cpp
MmapedFile::MmapedFile(const std::string &path, size_t size, bool fileType)
    : m_name(path), m_fd(-1), m_segmentPtr(nullptr), m_segmentSize(0), m_fileType(fileType) {
    // 用于内存映射的文件
    if (m_fileType == MMAP_FILE) {
        // 1. 打开文件
        m_fd = open(m_name.c_str(), O_RDWR | O_CREAT, S_IRWXU);
        if (m_fd < 0) {
            MMKVError("fail to open:%s, %s", m_name.c_str(), strerror(errno));
        } else {
            // 2. 创建文件锁
            FileLock fileLock(m_fd);
            InterProcessLock lock(&fileLock, ExclusiveLockType);
            SCOPEDLOCK(lock);
            // 获取文件的信息
            struct stat st = {};
            if (fstat(m_fd, &st) != -1) {
                // 获取文件大小
                m_segmentSize = static_cast<size_t>(st.st_size);
            }
            // 3. 验证文件的大小是否小于一个内存页, 一般为 4kb
            if (m_segmentSize < DEFAULT_MMAP_SIZE) {
                m_segmentSize = static_cast<size_t>(DEFAULT_MMAP_SIZE);
                // 3.1 通过 ftruncate 将文件大小对其到内存页
                // 3.2 通过 zeroFillFile 将文件对其后的空白部分用 0 填充
                if (ftruncate(m_fd, m_segmentSize) != 0 || !zeroFillFile(m_fd, 0, m_segmentSize)) {
                    // 说明文件拓展失败了, 移除这个文件
                    close(m_fd);
                    m_fd = -1;
                    removeFile(m_name);
                    return;
                }
            }
            // 4. 通过 mmap 将文件映射到内存, 获取内存首地址
            m_segmentPtr =
                (char *) mmap(nullptr, m_segmentSize, PROT_READ | PROT_WRITE, MAP_SHARED, m_fd, 0);
            if (m_segmentPtr == MAP_FAILED) {
                MMKVError("fail to mmap [%s], %s", m_name.c_str(), strerror(errno));
                close(m_fd);
                m_fd = -1;
                m_segmentPtr = nullptr;
            }
        }
    }
    // 用于共享内存的文件
    else {
        ......
    }
}
```
MmapedFile 的构造函数处理的事务如下
- 打开指定的文件
- 创建这个文件锁
- 修正文件大小, 最小为 4kb
  - 前 4kb 用于统计数据总大小 
- 通过 mmap 将文件映射到内存

好的, 通过 MmapedFile 的构造函数, 我们便能够获取到映射后的内存首地址了, 操作这块内存时 Linux 内核会负责将内存中的数据同步到文件中

比起 SP 的数据同步, mmap 显然是要优雅的多, **即使进程意外死亡, 也能够通过 Linux 内核的保护机制, 将进行了文件映射的内存数据刷入到文件中, 提升了数据写入的可靠性**

结下来看看数据的载入

#### 2. 数据的载入
```
// MMKV.cpp
void MMKV::loadFromFile() {
    
    ......// 忽略匿名共享内存相关代码
    
    // 若已经进行了文件映射
    if (m_metaFile.isFileValid()) {
        // 则获取相关数据
        m_metaInfo.read(m_metaFile.getMemory());
    }
    // 获取文件描述符
    m_fd = open(m_path.c_str(), O_RDWR | O_CREAT, S_IRWXU);
    if (m_fd < 0) {
        MMKVError("fail to open:%s, %s", m_path.c_str(), strerror(errno));
    } else {
        // 1. 获取文件大小
        m_size = 0;
        struct stat st = {0};
        if (fstat(m_fd, &st) != -1) {
            m_size = static_cast<size_t>(st.st_size);
        }
        // 1.1 将文件大小对其到内存页的整数倍
        if (m_size < DEFAULT_MMAP_SIZE || (m_size % DEFAULT_MMAP_SIZE != 0)) {
            ......
        }
        // 2. 获取文件映射后的内存地址
        m_ptr = (char *) mmap(nullptr, m_size, PROT_READ | PROT_WRITE, MAP_SHARED, m_fd, 0);
        if (m_ptr == MAP_FAILED) {
            ......
        } else {
            // 3. 读取内存文件的前 32 位, 获取存储数据的真实大小
            memcpy(&m_actualSize, m_ptr, Fixed32Size);
            ......
            bool loadFromFile = false, needFullWriteback = false;
            if (m_actualSize > 0) {
                // 4. 验证文件的长度
                if (m_actualSize < m_size && m_actualSize + Fixed32Size <= m_size) {
                    // 5. 验证文件 CRC 的正确性
                    if (checkFileCRCValid()) {
                        loadFromFile = true;
                    } else {
                        // 若不正确, 则回调异常 CRC 异常
                        auto strategic = mmkv::onMMKVCRCCheckFail(m_mmapID);
                        if (strategic == OnErrorRecover) {
                            loadFromFile = true;
                            needFullWriteback = true;
                        }
                    }
                } else {
                    // 回调文件长度异常
                    auto strategic = mmkv::onMMKVFileLengthError(m_mmapID);
                    if (strategic == OnErrorRecover) {
                        writeAcutalSize(m_size - Fixed32Size);
                        loadFromFile = true;
                        needFullWriteback = true;
                    }
                }
            }
            // 6. 需要从文件获取数据
            if (loadFromFile) {
                ......
                // 构建输入缓存
                MMBuffer inputBuffer(m_ptr + Fixed32Size, m_actualSize, MMBufferNoCopy);
                if (m_crypter) {
                    // 解密输入缓冲中的数据
                    decryptBuffer(*m_crypter, inputBuffer);
                }
                // 从输入缓冲中将数据读入 m_dic
                m_dic.clear();
                MiniPBCoder::decodeMap(m_dic, inputBuffer);
                // 构建输出数据
                m_output = new CodedOutputData(m_ptr + Fixed32Size + m_actualSize,
                                               m_size - Fixed32Size - m_actualSize);
                // 进行重整回写, 剔除重复的数据
                if (needFullWriteback) {
                    fullWriteback();
                }
            } 
            // 7. 说明文件中没有数据, 或者校验失败了
            else {
                SCOPEDLOCK(m_exclusiveProcessLock);
                // 清空文件中的数据
                if (m_actualSize > 0) {
                    writeAcutalSize(0);
                }
                m_output = new CodedOutputData(m_ptr + Fixed32Size, m_size - Fixed32Size);
                // 重新计算 CRC
                recaculateCRCDigest();
            }
            ......
        }
    }
    
    ......

    m_needLoadFromFile = false;
}
```
好的, 可以看到 loadFromFile 中对于 CRC 验证通过的文件, 会将文件中的数据读入到 m_dic 中缓存, 否则则会清空文件
- 因此用户恶意修改文件之后, 会破坏 CRC 的值, 这个存储数据便会被作废, 这一点要尤为注意
- **从文件中读取数据到 m_dic 之后, 会将 mdic 回写到文件中**, 其重写的目的是为了剔除重复的数据
  - 关于为什么会出现重复的数据, 在后面 encode 操作中再分析

#### 3. 回顾
到这里 MMKV 实例的构建就完成了, 有了 m_dic 这个内存缓存, 我们进行数据查询的效率就大大提升了

从最终的结果来看它与 SP 是一致的, 都是初次加载时会将文件中所有的数据加载到散列表中, 不过 MMKV 多了一步数据回写的操作, 因此当数据量比较大时, 对实例构建的速度有一定的影响

```
// 写入 1000 条数据之后, MMVK 和 SharedPreferences 实例化的时间对比
E/TAG: create MMKV instance time is 4 ms
E/TAG: create SharedPreferences instance time is 1 ms
```

从结果上来看, MMVK 的确在实例构造速度上有一定的劣势, 不过得益于是将 m_dic 中的数据写入到 mmap 的内存, 其真正进行文件写入的时机由 Linux 内核决定, 再加上文件的页缓存机制, 所以速度上虽有劣势, 但不至于无法接受

### 三) encode
关于 **encode 即数据的添加与更新**的流程, 这里以 encodeString 为例
```
public class MMKV implements SharedPreferences, SharedPreferences.Editor {

    public boolean encode(String key, String value) {
        return encodeString(nativeHandle, key, value);
    }
    
    private native boolean encodeString(long handle, String key, String value);

}
```
看看 native 层的实现
```
// native-bridge.cpp
namespace mmkv {

MMKV_JNI jboolean encodeString(JNIEnv *env, jobject, jlong handle, jstring oKey, jstring oValue) {
    MMKV *kv = reinterpret_cast<MMKV *>(handle);
    if (kv && oKey) {
        string key = jstring2string(env, oKey);
        // 若是 value 非 NULL
        if (oValue) {
            // 通过 setStringForKey 函数, 将数据存入
            string value = jstring2string(env, oValue);
            return (jboolean) kv->setStringForKey(value, key);
        } 
        // 若是 value 为 NULL, 则移除 key 对应的 value 值
        else {
            kv->removeValueForKey(key);
            return (jboolean) true;
        }
    }
    return (jboolean) false;
}

}
```
这里我们主要分析一下 setStringForKey 这个函数

```
// MMKV.cpp
bool MMKV::setStringForKey(const std::string &value, const std::string &key) {
    if (key.empty()) {
        return false;
    }
    // 1. 将数据编码成 ProtocolBuffer
    auto data = MiniPBCoder::encodeDataWithObject(value);
    // 2. 更新键值对
    return setDataForKey(std::move(data), key);
}
```
这里主要分为两步操作
- 数据编码
- 更新键值对

#### 1. 数据的编码
MMKV 采用的是 ProtocolBuffer 编码方式, 这里就不做过多介绍了, 具体请查看 [Google 官方文档](https://developers.google.com/protocol-buffers/docs/encoding)
```
// MiniPBCoder.cpp
MMBuffer MiniPBCoder::getEncodeData(const string &str) {
    // 1. 创建编码条目的集合
    m_encodeItems = new vector<PBEncodeItem>();
    // 2. 为集合填充数据
    size_t index = prepareObjectForEncode(str);
    PBEncodeItem *oItem = (index < m_encodeItems->size()) ? &(*m_encodeItems)[index] : nullptr;
    if (oItem && oItem->compiledSize > 0) {
        // 3. 开辟一个内存缓冲区, 用于存放编码后的数据
        m_outputBuffer = new MMBuffer(oItem->compiledSize);
        // 4. 创建一个编码操作对象
        m_outputData = new CodedOutputData(m_outputBuffer->getPtr(), m_outputBuffer->length());
        // 执行 protocolbuffer 编码, 并输出到缓冲区
        writeRootObject();
    }
    // 调用移动构造函数, 重新创建实例返回
    return move(*m_outputBuffer);
}

size_t MiniPBCoder::prepareObjectForEncode(const string &str) {
    // 2.1 创建 PBEncodeItem 对象用来描述待编码的条目, 并添加到 vector 集合
    m_encodeItems->push_back(PBEncodeItem());
    // 2.2 获取 PBEncodeItem 对象
    PBEncodeItem *encodeItem = &(m_encodeItems->back());
    // 2.3 记录索引位置
    size_t index = m_encodeItems->size() - 1;
    {   
        // 2.4 填充编码类型
        encodeItem->type = PBEncodeItemType_String;
        // 2.5 填充要编码的数据
        encodeItem->value.strValue = &str;
        // 2.6 填充数据大小
        encodeItem->valueSize = static_cast<int32_t>(str.size());
    }
    // 2.7 计算编码后的大小
    encodeItem->compiledSize = pbRawVarint32Size(encodeItem->valueSize) + encodeItem->valueSize;
    return index;
}
```
可以看到, 再未进行编码操作之前, 编码后的数据大小就已经确定好了, 并且将它保存在了 encodeItem->compiledSize 中, 接下来我们看看执行数据编码并输出到缓冲区的操作流程

```
// MiniPBCoder.cpp
void MiniPBCoder::writeRootObject() {
    for (size_t index = 0, total = m_encodeItems->size(); index < total; index++) {
        PBEncodeItem *encodeItem = &(*m_encodeItems)[index];
        switch (encodeItem->type) {
            // 主要关心编码 String
            case PBEncodeItemType_String: {
                m_outputData->writeString(*(encodeItem->value.strValue));
                break;
            }
            ......
        }
    }
}

// CodedOutputData.cpp
void CodedOutputData::writeString(const string &value) {
    size_t numberOfBytes = value.size();
    ......
    // 1. 按照 varint 方式编码字符串长度, 会改变 m_position 的值
    this->writeRawVarint32((int32_t) numberOfBytes);
    // 2. 将字符串的数据拷贝到编码好的长度后面
    memcpy(m_ptr + m_position, ((uint8_t *) value.data()), numberOfBytes);
    // 更新 position 的值
    m_position += numberOfBytes;
}
```
可以看到 CodedOutputData 的 writeString 中按照 protocol buffer 进行了字符串的编码操作

其中 m_ptr 是上面开辟的内存缓冲区的地址, 也就是说 writeString 执行结束之后, 数据就已经被写入缓冲区了

有了编码好的数据缓冲区, 接下来看看更新键值对的操作

#### 2. 更新键值对
```
// MMKV.cpp
bool MMKV::setStringForKey(const std::string &value, const std::string &key) {
    // 编码数据获取存放数据的缓冲区
    auto data = MiniPBCoder::encodeDataWithObject(value);
    // 更新键值对
    return setDataForKey(std::move(data), key);
}

bool MMKV::setDataForKey(MMBuffer &&data, const std::string &key) {
    ......
    // 将键值对写入 mmap 文件映射的内存中
    auto ret = appendDataWithKey(data, key);
    // 写入成功, 更新散列数据
    if (ret) {
        m_dic[key] = std::move(data);
        m_hasFullWriteback = false;
    }
    return ret;
}

bool MMKV::appendDataWithKey(const MMBuffer &data, const std::string &key) {
    // 1. 计算 key + value 的 ProtocolBuffer 编码后的长度
    size_t keyLength = key.length();
    size_t size = keyLength + pbRawVarint32Size((int32_t) keyLength);
    size += data.length() + pbRawVarint32Size((int32_t) data.length());
    SCOPEDLOCK(m_exclusiveProcessLock);
    
    // 2. 验证是否有足够的空间, 不足则进行数据重整与扩容操作
    bool hasEnoughSize = ensureMemorySize(size);
    if (!hasEnoughSize || !isFileValid()) {
        return false;
    }
    
    // 3. 更新文件头的数据总大小
    writeAcutalSize(m_actualSize + size);
    
    // 4. 将 key 和编码后的 value 写入到文件映射的内存
    m_output->writeString(key);
    m_output->writeData(data);
    
    // 5. 获取文件映射内存当前 <key, value> 的起始位置
    auto ptr = (uint8_t *) m_ptr + Fixed32Size + m_actualSize - size;
    if (m_crypter) {
        // 加密这块区域
        m_crypter->encrypt(ptr, ptr, size);
    }
    
    // 6. 更新 CRC
    updateCRCDigest(ptr, size, KeepSequence);
    return true;
}
```
好的, 可以看到更新键值对的操作还是比较复杂的, 首先将键值对数据写入到文件映射的内存中, 写入成功之后更新散列数据

关于写入到文件映射的过程, 上面代码中的注释也非常的清晰, 接下来我们 ensureMemorySize 是如何进行数据的重整与扩容的

##### 数据的重整与扩容
```
// MMKV.cpp
bool MMKV::ensureMemorySize(size_t newSize) {
    ......
    // 计算新键值对的大小
    constexpr size_t ItemSizeHolderSize = 4;
    if (m_dic.empty()) {
        newSize += ItemSizeHolderSize;
    }
    // 数据重写: 
    // 1. 文件剩余空闲空间少于新的键值对
    // 2. 散列为空
    if (newSize >= m_output->spaceLeft() || m_dic.empty()) {
        // 计算所需的数据空间
        static const int offset = pbFixed32Size(0);
        MMBuffer data = MiniPBCoder::encodeDataWithObject(m_dic);
        size_t lenNeeded = data.length() + offset + newSize;
        if (m_isAshmem) {
            ......
        } else {
            // 
            // 计算每个键值对的平均大小
            size_t avgItemSize = lenNeeded / std::max<size_t>(1, m_dic.size());
            // 计算未来可能会使用的大小(类似于 1.5 倍)
            size_t futureUsage = avgItemSize * std::max<size_t>(8, (m_dic.size() + 1) / 2);
            // 1. 所需空间 >= 当前文件总大小
            // 2. 所需空间的 1.5 倍 >= 当前文件总大小
            if (lenNeeded >= m_size || (lenNeeded + futureUsage) >= m_size) {
                // 扩容为 2 倍
                size_t oldSize = m_size;
                do {
                    m_size *= 2;
                } while (lenNeeded + futureUsage >= m_size);
                .......
            }
        }
        ......
        // 进行数据的重写
        writeAcutalSize(data.length());
        ......
    }
    return true;
}
```
从上面的代码我们可以了解到
- **数据的重写时机**
  - 文件剩余空间少于新的键值对大小
  - 散列为空
- **文件扩容时机**
  - 所需空间的 1.5 倍超过了当前文件的总大小时, 扩容为之前的两倍

#### 3. 回顾
至此 encode 的流程我们就走完了, 回顾一下整个 encode 的流程
- 使用 ProtocolBuffer 编码 value
- 将 **key** 和 **编码后的 value** 使用 ProtocolBuffer 的格式 append 到文件映射区内存的尾部
  - 文件空间不足
    - 判断是否需要扩容
    - 进行数据的回写
  - 即在文件后进行追加 
- 对这个键值对区域进行统一的加密
- 更新 CRC 的值
- 将 key 和 value 对应的 ProtocolBuffer 编码内存区域, 更新到散列表 m_dic 中

通过 encode 的分析, 我们得知 MMKV 文件的存储方式如下

![MMKV 文件存储格式](https://i.loli.net/2019/08/15/Y3xknw8GSXPbQVK.png)

接下来看看 decode 的流程

### 四) decode
decode 的过程同样以 decodeString 为例
```
// native-bridge.cpp
MMKV_JNI jstring
decodeString(JNIEnv *env, jobject obj, jlong handle, jstring oKey, jstring oDefaultValue) {
    MMKV *kv = reinterpret_cast<MMKV *>(handle);
    if (kv && oKey) {
        string key = jstring2string(env, oKey);
        // 通过 getStringForKey, 将数据输出到传出参数中 value 中
        string value;
        bool hasValue = kv->getStringForKey(key, value);
        if (hasValue) {
            return string2jstring(env, value);
        }
    }
    return oDefaultValue;
}

// MMKV.cpp
bool MMKV::getStringForKey(const std::string &key, std::string &result) {
    if (key.empty()) {
        return false;
    }
    SCOPEDLOCK(m_lock);
    // 1. 从内存缓存中获取数据
    auto &data = getDataForKey(key);
    if (data.length() > 0) {
        // 2. 解析 data 对应的 ProtocolBuffer 数据
        result = MiniPBCoder::decodeString(data);
        return true;
    }
    return false;
}

const MMBuffer &MMKV::getDataForKey(const std::string &key) {
    // 从散列表中获取 key 对应的 value
    auto itr = m_dic.find(key);
    if (itr != m_dic.end()) {
        return itr->second;
    }
    static MMBuffer nan(0);
    return nan;
}
```
好的可以看到 decode 的流程比较简单, 先从内存缓存中获取 key 对应的 value 的 ProtocolBuffer 内存区域, 再解析这块内存区域, 从中获取真正的 value 值

#### 思考
看到这里可能会有一个疑问, **为什么 m_dic 不直接存储 key 和 value 原始数据呢, 这样查询效率不是更快吗?**
- 如此一来查询效率的确会更快, 因为少了 ProtocolBuffer 解码的过程

![读取性能对比](https://i.loli.net/2019/08/15/8gpChT6l2soamzD.png)

从图上的结果可以看出, MMKV 的读取性能时略低于 SharedPreferences 的, 这里笔者给出自己的思考
- **m_dic 在数据重整中也起到了非常重要的作用, 需要依靠 m_dic 将数据写入到 mmap 的文件映射区**, 这个过程是非常耗时的, 若是原始的 value, 则需要对所有的 value 再进行一次 ProtocolBuffer 编码操作, 尤其是当数据量比较庞大时, 其带来的性能损耗更是无法忽略的

既然 m_dic 还承担着方便数据复写的功能, 那**能否再添加一个内存缓存专门用于存储原始的 value 呢?**
- 当然可以, 这样 MMKV 的读取定是能够达到 SharedPreferences 的水平, 不过 value 的内存消耗则会加倍, **MMKV 作为一个轻量级缓存的框架, 查询时时间的提升幅度还不足以用内存加倍的代价去换取**, 我想这是 Tencent 在进行多方面权衡之后, 得到的一个比较合理的解决方案

### 五) 进程读写的同步
说起进程间读写同步, 我们很自然的想到 Linux 的共享内存配合信号量使用的案例, 但是这种方式有一个弊端, 那就是**当持有锁的进程意外死亡的时候, 并不会释放其拥有的信号量, 若多进程之间存在竞争, 那么阻塞的进程将不会被唤醒**, 这是非常危险的

MMKV 是采用 **文件锁** 的方式来进行进程间的同步操作
- **LOCK_SH(共享锁)**: 多个进程可以使用同一把锁, 常被用作读共享锁
- **LOCK_EX(排他锁)**: 同时只允许一个进程使用, 常被用作写锁
- **LOCK_UN**: 释放锁

接下来我看看 MMKV 加解锁的操作

#### 1. 文件共享锁
```
MMKV::MMKV(
    const std::string &mmapID, int size, MMKVMode mode, string *cryptKey, string *relativePath)
    : m_mmapID(mmapedKVKey(mmapID, relativePath))
    // 创建文件锁的描述
    , m_fileLock(m_metaFile.getFd())
    // 描述共享锁
    , m_sharedProcessLock(&m_fileLock, SharedLockType)
    // 描述排它锁
    , m_exclusiveProcessLock(&m_fileLock, ExclusiveLockType)
    // 判读是否为进程间通信
    , m_isInterProcess((mode & MMKV_MULTI_PROCESS) != 0 || (mode & CONTEXT_MODE_MULTI_PROCESS) != 0)
    , m_isAshmem((mode & MMKV_ASHMEM) != 0) {
    ......
    // 根据是否跨进程操作判断共享锁和排它锁的开关
    m_sharedProcessLock.m_enable = m_isInterProcess;
    m_exclusiveProcessLock.m_enable = m_isInterProcess;

    // sensitive zone
    {
        // 文件读操作, 启用了文件共享锁
        SCOPEDLOCK(m_sharedProcessLock);
        loadFromFile();
    }
}
```
可以看到在我们前面分析过的构造函数中, MMKV 对文件锁进行了初始化, 并且创建了共享锁和排它锁, 并在跨进程操作时开启, 当进行读操作时, 启动了共享锁

#### 2. 文件排它锁
```
bool MMKV::fullWriteback() {
    ......
    auto allData = MiniPBCoder::encodeDataWithObject(m_dic);
    // 启动了排它锁
    SCOPEDLOCK(m_exclusiveProcessLock);
    if (allData.length() > 0) {
        if (allData.length() + Fixed32Size <= m_size) {
            if (m_crypter) {
                m_crypter->reset();
                auto ptr = (unsigned char *) allData.getPtr();
                m_crypter->encrypt(ptr, ptr, allData.length());
            }
            writeAcutalSize(allData.length());
            delete m_output;
            m_output = new CodedOutputData(m_ptr + Fixed32Size, m_size - Fixed32Size);
            m_output->writeRawData(allData); // note: don't write size of data
            recaculateCRCDigest();
            m_hasFullWriteback = true;
            return true;
        } else {
            // ensureMemorySize will extend file & full rewrite, no need to write back again
            return ensureMemorySize(allData.length() + Fixed32Size - m_size);
        }
    }
    return false;
}
```
在进行数据回写的函数中, 启动了排它锁

#### 3. 读写效率表现
其进程同步读写的性能表现如下

![进程同步读写表现](https://i.loli.net/2019/08/15/cH6GlXVobQBWDfZ.png)

可以看到进程同步读写的效率也是非常 nice 的

关于跨进程同步就介绍到这里, 当然 MMKV 的文件锁并没有表面上那么简单, 因为文件锁为状态锁, 无论加了多少次锁, 一个解锁操作就全解除, 显然无法应对子函数嵌套调用的问题, **MMKV 内部通过了自行实现计数器来实现锁的可重入性**, 更多的细节可以查看 [wiki](https://github.com/Tencent/MMKV/wiki/android_ipc)

## 总结
通过上面的分析, 我们对 MMKV 有了一个整体上的把控, 其具体的表现如下所示

项目 | 评价 | 描述
---|---|---
正确性 | 优 | 支持多进程安全, 使用 mmap, 由操作系统保证数据回写的正确性
时间开销 | 优 | 使用 mmap 实现, 减少了用户空间数据到内核空间的拷贝
空间开销 | 中 | 使用 protocl buffer 存储数据, 同样的数据会比 xml 和 json 消耗空间小 <br> 使用的是数据追加到末尾的方式, 只有到达一定阈值之后才会触发键值合并, 不合并之前会导致同一个 key 存在多份
安全 | 中 | 使用 crc 校验, 甄别文件系统和操作系统不稳定导致的异常数据
开发成本 | 优 | 使用方式较为简单
兼容性 | 优 | 各个安卓版本都前后兼容

虽然 MMKV 一些场景下比 SP 稍慢(如: 首次实例化会进行数据的复写剔除重复数据, 比 SP 稍慢, 查询数据时存在 ProtocolBuffer 解码, 比 SP 稍慢), 但其**逆天的数据写入速度、mmap Linux 内核保证数据的同步, 以及 ProtocolBuffer 编码带来的更小的本地存储空间占用等都是非常棒的闪光点**

在分析 MMKV 的代码的过程中, 从中学习到了很多知识, 非常感谢 Tencent 为开源社区做出的贡献

## 参考文献
- https://github.com/Tencent/MMKV/wiki/android_setup_cn
- https://developers.google.com/protocol-buffers/docs/encoding
- https://time.geekbang.org/column/article/76677
- https://www.cnblogs.com/kex1n/p/7100107.html