---
title: Android 系统架构 —— 资源管理的创建
permalink: android-source/resources-manager
key: android-source-resources-manager
tags: AndroidFramework
---

## 前言
前面我们学习 App 资源的打包流程, 知晓了 resources.arsc 的结构组成, 我们在开发过程中想要访问资源需要通过如下的方式进行

```java
Resources res = getContext().getResources();
res.getString(R.string.xxx);
```

我们在应用开发的过程中, 通常通过 Resources 对象来访问 app 的资源文件, 通过 aapt 编译的过程我们知道, R.string.xxx 是资源的 ID, 想通过 ID 获取到对应的资源, 只能够通过资源索引表 resources.arsc 

我们在 Application 的 onCreate 中就可以通过 Resources 访问 app 的资源了, 因此本篇文章从 Application 的创建方法 makeApplication 为起点来探索一下资源管理类的创建流程

<!--more-->

```java
public final class LoadedApk {
    
    public Application makeApplication(boolean forceDefaultAppClass,
            Instrumentation instrumentation) {   
            
        // 构建 Application 对象
        if (mApplication != null) {
            return mApplication;
        }
        Application app = null;

        String appClass = mApplicationInfo.className;
        if (forceDefaultAppClass || (appClass == null)) {
            appClass = "android.app.Application";
        }

        try {
            java.lang.ClassLoader cl = getClassLoader();
            if (!mPackageName.equals("android")) {
                Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER,
                        "initializeJavaContextClassLoader");
                initializeJavaContextClassLoader();
                Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
            }
            // 1. 创建 Application 的 ContextImpl
            ContextImpl appContext = ContextImpl.createAppContext(mActivityThread, this);
            
            // 实例化 Application 对象
            app = mActivityThread.mInstrumentation.newApplication(
                    cl, appClass, appContext);
            appContext.setOuterContext(app);
        } catch (Exception e) {
            ......
        }
        .....// 回调 onCreate

        // 2. 重写 R 文件的 ID 值为运行时 ID
        SparseArray<String> packageIdentifiers = getAssets().getAssignedPackageIdentifiers();
        final int N = packageIdentifiers.size();
        for (int i = 0; i < N; i++) {
            final int id = packageIdentifiers.keyAt(i);
            if (id == 0x01 || id == 0x7f) {
                continue;
            }

            rewriteRValues(getClassLoader(), packageIdentifiers.valueAt(i), id);
        }
        return app;
    }
    
}
```
Application 创建的代码, 我们已经看过很多次了, 这里我们主要关注一下与资源相关的初始化操作
- 首先会创建 Application 的 ContextImpl
- 调用了 rewriteRValues 重写 R 文件的资源 ID 值

这里我们从 ContextImpl 的创建来学习资源管理的创建流程, 关于 R 文件中的资源 ID 为什么需要被重写, 我们到文章的结束再进行揭秘

## 一. 资源管理的创建
我们在 Activity 启动流程中知道, 在创建 ContextImpl 之后, 紧接着会为它注入 Resource 对象, 如此一来 ContextImpl 才具有资源访问的能力, 这里以创建 Application 的 ContextImpl 为例
```java
class ContextImpl extends Context {
    
    static ContextImpl createAppContext(ActivityThread mainThread, LoadedApk packageInfo) {
        ......
        // 1. 创建 ContextImpl 对象
        ContextImpl context = new ContextImpl(null, mainThread, packageInfo, null, null, null, 0,
                null);
        // 2. 通过 LoadedApk.getResources 来获取 Resource 对象
        // 3. 将 Resource 注入到 ContextImpl 中
        context.setResources(packageInfo.getResources());
        return context;
    }

}
```
从这里可以看到 Resource 对象是通过 LoadedApk 的 getResources 获取到的, 这个 LoadedApk 我们在 Activity 启动篇中也分析过, 它用来描述一个 apk 的资源信息

接下来我们看看 LoadedApk 的 getResource 方法是如何获取 Apk 对象信息的

```java
public final class LoadedApk {
    
    Resources mResources;
    
    public Resources getResources() {
        if (mResources == null) {
            final String[] splitPaths;
            try {
                splitPaths = getSplitPaths(null);
            } catch (NameNotFoundException e) {
                ......
            }
            // 通过 ResourceManager 来获取这个 apk 对应的资源
            mResources = ResourcesManager.getInstance().getResources(null, mResDir,
                    splitPaths, mOverlayDirs, mApplicationInfo.sharedLibraryFiles,
                    Display.DEFAULT_DISPLAY, null, getCompatibilityInfo(),
                    getClassLoader());
        }
        return mResources;
    }
    
}
```
可以看到每一个 Apk 都有一个 Resource 对象来描述它包内的资源, 而这个 Resource 对象是通过 ResourcesManager 的 getResource 获取到的
- 我们一个应用可能会存在多个 apk, 比如说一个主 base.apk, 然后又动态下载了一些换肤包的 apk 问题, 每一个 apk 都会用 LoadedApk 来描述

下面我们看看 ResourceManager 是如何获取当前 Apk 的资源信息的

### 一) 通过 ResourcesManager 获取 Resources
```java
public class ResourcesManager {
  
    private static ResourcesManager sResourcesManager;
    
    public static ResourcesManager getInstance() {
        synchronized (ResourcesManager.class) {
            if (sResourcesManager == null) {
                sResourcesManager = new ResourcesManager();
            }
            return sResourcesManager;
        }
    }
   
    public @Nullable Resources getResources(@Nullable IBinder activityToken,
            @Nullable String resDir,
            @Nullable String[] splitResDirs,
            @Nullable String[] overlayDirs,
            @Nullable String[] libDirs,
            int displayId,
            @Nullable Configuration overrideConfig,
            @NonNull CompatibilityInfo compatInfo,
            @Nullable ClassLoader classLoader) {
        try {
            ...// systrace 埋点
            // 将请求的资源信息构建成 ResourceKey
            final ResourcesKey key = new ResourcesKey(
                    resDir,
                    splitResDirs,
                    overlayDirs,
                    libDirs,
                    displayId,
                    overrideConfig != null ? new Configuration(overrideConfig) : null, // Copy
                    compatInfo);
            // 获取 LoadedApk 传递过来的 ClassLoader
            classLoader = classLoader != null ? classLoader : ClassLoader.getSystemClassLoader();
            // 调用 getOrCreateResources 获取资源信息
            return getOrCreateResources(activityToken, key, classLoader);
        } finally {
            ...// systrace 埋点
        }
    }
    
}
```
ResourcesManager 的 getResource 方法实现如下
- 构建 **ResourcesKey**
- 获取 ClassLoader
  - 优先使用 LoadedApk 中的 ClassLoader
  - 不存在使用系统的 ClassLoader, 这是一个进程单例的 PathClassLoader
- 调用 getOrCreateResources 根据 ResourcesKey 获取 Resource 对象

下面我们看看 getOrCreateResources 的实现

```java
public class ResourcesManager {

    private @Nullable Resources getOrCreateResources(@Nullable IBinder activityToken,
            @NonNull ResourcesKey key, @NonNull ClassLoader classLoader) {
        synchronized (this) {
            ......
            // 1. 从缓存中查找 ResourceImpl
            // 1.1 在构建 Activity 的 ContextImpl 的 Resource 是走这里
            if (activityToken != null) {
                final ActivityResources activityResources =
                        getOrCreateActivityResourcesStructLocked(activityToken);

                ......// 清除一些失效的引用
                
                // 根据 key 从缓存中查找 ResourceImpl 对象
                ResourcesImpl resourcesImpl = findResourcesImplForKeyLocked(key);
                if (resourcesImpl != null) {
                    ......
                    return getOrCreateResourcesForActivityLocked(activityToken, classLoader,
                            resourcesImpl, key.mCompatInfo);
                }
            } 
            // 1.2 在构建其他 ContextImpl 的 Resource 时走这里, 我们的 Application 会走这里
            else {
                
                ......// 清除一些失效的引用
                
                // 根据 key 从缓存中查找 ResourceImpl 对象
                ResourcesImpl resourcesImpl = findResourcesImplForKeyLocked(key);
                if (resourcesImpl != null) {
                    ......
                    // 获取 Resource 对象
                    return getOrCreateResourcesLocked(classLoader, resourcesImpl, key.mCompatInfo);
                }
            }

            // 2. 构建新的 ResourceImpl
            ResourcesImpl resourcesImpl = createResourcesImpl(key);
            if (resourcesImpl == null) {
                return null;
            }
            
            // 3. 添加到缓存
            mResourceImpls.put(key, new WeakReference<>(resourcesImpl));
            
            // 4. 创建 Resources
            final Resources resources;
            if (activityToken != null) {
                // 4.1 创建 Activity 的 Resources
                resources = getOrCreateResourcesForActivityLocked(activityToken, classLoader,
                        resourcesImpl, key.mCompatInfo);
            } else {
                // 4.2 创建其他的 Resource
                resources = getOrCreateResourcesLocked(classLoader, resourcesImpl, key.mCompatInfo);
            }
            return resources;
        }
    }
    
}
```
ResourcesManager 创建 Resources 的流程如下
- **获取 ResourcesImpl**
  - 从缓存中查找 ResourcesKey 对应的 ResourcesImpl
  - 不存在则创建一个新的 ResourcesImpl 加入缓存
- 调用 getOrCreateResourcesLocked **通过 ResourcesImpl 构建 Resources 对象**

从 ResourcesImpl 的命名上上就可以看出它是 Resources 的真正实现, 我们先看看 ResourcesImpl 的构建过程

### 二) 获取 ResourcesImpl
获取 ResourcesImpl 有两种方式, 优先从缓存中获取 ResourcesKey 对应的 ResourceImpl 对象, 次优先创建新对象, 这里我们先看看从缓存中获取的流程
```java
public class ResourcesManager {

    private final ArrayMap<ResourcesKey, WeakReference<ResourcesImpl>> mResourceImpls =
            new ArrayMap<>();

    private @Nullable ResourcesImpl findResourcesImplForKeyLocked(@NonNull ResourcesKey key) {
        // 从缓存中获取 ResourcesImpl
        WeakReference<ResourcesImpl> weakImplRef = mResourceImpls.get(key);
        ResourcesImpl impl = weakImplRef != null ? weakImplRef.get() : null;
        if (impl != null && impl.getAssets().isUpToDate()) {
            return impl;
        }
        return null;
    }    
    
}
```
可以看到 ResourcesManager 维护了一个 ResourcesKey 和 ResourcesImpl 弱引用对象的缓存, 若弱引用的对象存在, 则返回这个 ResourcesImpl

下面我们看看缓存中不存在, ResourceImpl 的创建过程
```
public class ResourcesManager {
    
    private @Nullable ResourcesImpl createResourcesImpl(@NonNull ResourcesKey key) {
        final DisplayAdjustments daj = new DisplayAdjustments(key.mOverrideConfiguration);
        daj.setCompatibilityInfo(key.mCompatInfo);
        // 1. 创建 AssetManager
        final AssetManager assets = createAssetManager(key);
        if (assets == null) {
            return null;
        }
        // 获取显示屏幕的相关信息
        final DisplayMetrics dm = getDisplayMetrics(key.mDisplayId, daj);
        final Configuration config = generateConfig(key, dm);
        // 2. 创建 ResourcesImpl 
        final ResourcesImpl impl = new ResourcesImpl(assets, dm, config, daj);
        ......
        return impl;
    }
    
}

public class ResourcesImpl {
    
    public ResourcesImpl(@NonNull AssetManager assets, @Nullable DisplayMetrics metrics,
            @Nullable Configuration config, @NonNull DisplayAdjustments displayAdjustments) {
        // 将 AssetManager 保存到成员变量中
        mAssets = assets;
        mMetrics.setToDefaults();
        mDisplayAdjustments = displayAdjustments;
        mConfiguration.setToDefaults();
        updateConfiguration(config, metrics, displayAdjustments.getCompatibilityInfo());
    }
    
}
```
好的, 可以看到 createResourcesImpl 中主要有两步操作
- 创建 AssetManager
- 创建 ResourcesImpl 对象

**这里 ResourceImpl 是一个门面类, 其资源相关的具体实现是由 AssetManager 来完成的, 它也是整个 Android 资源管理框架的核心, 我们先将上层的 Resource 创建分析完之后到后面重点看看 AssetManager 的构建流程**

接下来我们看看有了 ResourceImpl 之后, 如何构建 Resource

### 三) 创建 Resources 对象
```java
public class ResourcesManager {

    private final ArrayList<WeakReference<Resources>> mResourceReferences = new ArrayList<>();

    private @NonNull Resources getOrCreateResourcesLocked(@NonNull ClassLoader classLoader,
            @NonNull ResourcesImpl impl, @NonNull CompatibilityInfo compatInfo) {
        // 1. 遍历 mResourceReferences 集合, 找寻与传入的 ResourcesImpl 和 ClassLoader 都相同的 Resources
        final int refCount = mResourceReferences.size();
        for (int i = 0; i < refCount; i++) {
            WeakReference<Resources> weakResourceRef = mResourceReferences.get(i);
            Resources resources = weakResourceRef.get();
            // ClassLoader 和 impl 都相同
            if (resources != null && Objects.equals(resources.getClassLoader(), classLoader) &&
                    resources.getImpl() == impl) {
                .......
                return resources;
            }
        }

        // 2. 创建新的 Resources 对象
        Resources resources = compatInfo.needsCompatResources() ? new CompatResources(classLoader)
                : new Resources(classLoader);
        // 2.1 注入 ResourcesImpl
        resources.setImpl(impl);
        // 添加到缓存集合
        mResourceReferences.add(new WeakReference<>(resources));
        ......
        return resources;
    }

}
```
getOrCreateResourcesLocked 中实现的逻辑也非常的简单
- 优先从引用集合中找寻目标的 Resources
  - ResourcesImpl 和 ClassLoader 都相同
- 若缓存中不存在, 则创建一个新的 Resources 对象
  - 将 ResourcesImpl 注入 

到这里我们一个 Resources 就创建好了

### 四) 回顾
ResourcesManager 获取 Resources 的步骤如下
- **构建 ResourcesKey**
- **根据 ResourcesKey 从缓存 mResourceImpls 中获取 ResourcesImpl 对象**
  - 缓存不存在则构建 ResourcesImpl 对象, 创建流程如下
    - 获取 AssetManager
    - 创建 ResourcesImpl 对象
      - 内部持有 AseetManager
- **优先从 mResourceReferences 缓存中获取 Resources 对象**
  - 不存在则创建新的 Resources 对象
    - 将 ClassLoader 和 ResourcesImpl 注入

他们之间的依赖关系如下

![资源之间的依赖关系](https://i.loli.net/2019/12/01/ZWBJxcOEUaGPty3.png)

## 二. AssetManager 的构建
```java
public class ResourcesManager {
    
    protected @Nullable AssetManager createAssetManager(@NonNull final ResourcesKey key) {
        final AssetManager.Builder builder = new AssetManager.Builder();
        
        // 1. 获取 mResDir 的 ApkAssets 对象, 添加到 AssetManager.Builder 中
        if (key.mResDir != null) {
            try {
                
                builder.addApkAssets(loadApkAssets(key.mResDir, false /*sharedLib*/,
                        false /*overlay*/));
            } catch (IOException e) {
                ......
            }
        }

        // 2. 获取 mSplitResDirs 的 ApkAssets 对象, 添加到 AssetManager.Builder 中
        if (key.mSplitResDirs != null) {
            for (final String splitResDir : key.mSplitResDirs) {
                try {
                    builder.addApkAssets(loadApkAssets(splitResDir, false /*sharedLib*/,
                            false /*overlay*/));
                } catch (IOException e) {
                    ......
                }
            }
        }

        // 3. 获取 mOverlayDirs 的 ApkAssets 对象, 添加到 AssetManager.Builder 中
        if (key.mOverlayDirs != null) {
            for (final String idmapPath : key.mOverlayDirs) {
                try {
                    builder.addApkAssets(loadApkAssets(idmapPath, false /*sharedLib*/,
                            true /*overlay*/));
                } catch (IOException e) {
                    ......
                }
            }
        }
        
        // 4. 获取 mLibDirs 的 ApkAssets 对象, 添加到 AssetManager.Builder 中
        if (key.mLibDirs != null) {
            for (final String libDir : key.mLibDirs) {
                if (libDir.endsWith(".apk")) {
                    ......
                    try {
                        builder.addApkAssets(loadApkAssets(libDir, true /*sharedLib*/,
                                false /*overlay*/));
                    } catch (IOException e) {
                        ......
                    }
                }
            }
        }
        // 5. 构建 AssetManager
        return builder.build();
    }
    
}
```
可以看到 AssetManager 的构建主要有如下的步骤
- 通过 loadApkAssets 获取 ApkAssets 对象, 用于描述一个 apk 中的资源数据
- 调用 AssetManager.Builder.addApkAssets 将所有构件的资源包添加到构造者中
- 通过 AssetManager.Builder.build 构建 AssetManager 对象

从这个方法中可知, AssetManager 主要负责管理的是 ApkAssets, 下面我们先看看 ApkAsset 的构建过程, 然后再看看 AssetManager 的构建

### 一) 构建 ApkAsset
```
public class ResourcesManager {

    /**
     * 活跃缓存
     */
    private final LruCache<ApkKey, ApkAssets> mLoadedApkAssets = new LruCache<>(3);

    /**
     * 不活跃缓存
     */
    private final ArrayMap<ApkKey, WeakReference<ApkAssets>> mCachedApkAssets = new ArrayMap<>();

    private @NonNull ApkAssets loadApkAssets(String path, boolean sharedLib, boolean overlay)
            throws IOException {
        // 1. 构建一个 ApkKey 对象
        final ApkKey newKey = new ApkKey(path, sharedLib, overlay);
        // 2. 从 mLoadedApkAssets 缓存中获取 ApkAssets
        ApkAssets apkAssets = mLoadedApkAssets.get(newKey);
        if (apkAssets != null) {
            return apkAssets;
        }

        // 3. 从 mCachedApkAssets 缓存中获取 ApkAssets 对象
        final WeakReference<ApkAssets> apkAssetsRef = mCachedApkAssets.get(newKey);
        if (apkAssetsRef != null) {
            apkAssets = apkAssetsRef.get();
            if (apkAssets != null) {
                // 添加到活跃缓存
                mLoadedApkAssets.put(newKey, apkAssets);
                return apkAssets;
            } else {
                // 移除失效的缓存对象
                mCachedApkAssets.remove(newKey);
            }
        }

        // 4. 从磁盘中获取 ApkAssets 对象
        if (overlay) {
            apkAssets = ApkAssets.loadOverlayFromPath(overlayPathToIdmapPath(path),
                    false /*system*/);
        } else {
            // 读取资源包
            apkAssets = ApkAssets.loadFromPath(path, false /*system*/, sharedLib);
        }
        // 5. 加入缓存
        mLoadedApkAssets.put(newKey, apkAssets);
        mCachedApkAssets.put(newKey, new WeakReference<>(apkAssets));
        // 返回 ApkAssets
        return apkAssets;
    }
    
}
```
loadApkAssets 获取 ApkAssets 的过程还是比较有趣的, 从这里可以看到 ResourcesManager 中维护了两个内存缓存, 其获取流程如下
- 构建 ApkKey
- 从活跃缓存 mLoadedApkAssets 中获取
- 从不活跃缓存 mCachedApkAssets 中获取
- 从磁盘中读取到内存缓存

这里我们主要看看 ApkAssets.loadFromPath 是如何将 apk 的资源包读到内存的

```java
public final class ApkAssets {
    
    @GuardedBy("this") private final long mNativePtr;
    @GuardedBy("this") private StringBlock mStringBlock;
    
    public static @NonNull ApkAssets loadFromPath(@NonNull String path, boolean system,
            boolean forceSharedLibrary) throws IOException {
        // 创建了 ApkAssets 实例
        return new ApkAssets(path, system, forceSharedLibrary, false /*overlay*/);
    }
    
    private ApkAssets(@NonNull String path, boolean system, boolean forceSharedLib, boolean overlay)
            throws IOException {
        // 1. 调用 nativeLoad 到 Native 层加载资源
        mNativePtr = nativeLoad(path, system, forceSharedLib, overlay);
        // 2. 调用 nativeGetStringBlock 获取资源包中的字符串常量
        // 3. 构件 StringBlock 对象
        mStringBlock = new StringBlock(nativeGetStringBlock(mNativePtr), true /*useSparse*/);
    }    
    
}
```
从这里可以看到构件 ApkAssets 的构造中主要有三步操作,
- 构件 Native 层的 ApkAssets
- 获取字符常量

ApkAssets 的 Native 实现类为 [android_content_res_ApkAssets.cpp](http://androidxref.com/9.0.0_r3/xref/frameworks/base/core/jni/android_content_res_ApkAssets.cpp)

下面我们一一分析

#### 1. nativeLoad
```
// frameworks/base/core/jni/android_content_res_ApkAssets.cpp
static jlong NativeLoad(JNIEnv* env, jclass /*clazz*/, jstring java_path, jboolean system,
                        jboolean force_shared_lib, jboolean overlay) {
  // 获取资源路径
  ScopedUtfChars path(env, java_path);
  ......
  // 创建 Native 的 ApkAssets 对象
  std::unique_ptr<const ApkAssets> apk_assets;
  if (overlay) {
    apk_assets = ApkAssets::LoadOverlay(path.c_str(), system);
  } else if (force_shared_lib) {
    apk_assets = ApkAssets::LoadAsSharedLibrary(path.c_str(), system);
  } else {
    apk_assets = ApkAssets::Load(path.c_str(), system);
  }
  ......
  // 转为句柄值返回给 Java 层
  return reinterpret_cast<jlong>(apk_assets.release());
}
```
可以看到 ApkAssets 的 NativeLoad 实现中通过 ApkAssets::load 创建了一个 Native 的 [ApkAssets](http://androidxref.com/9.0.0_r3/xref/frameworks/base/libs/androidfw/ApkAssets.cpp) 对象, 下面我们就看看它的创建流程

```
// frameworks/base/libs/androidfw/ApkAssets.cpp
std::unique_ptr<const ApkAssets> ApkAssets::Load(const std::string& path, bool system) {
  return LoadImpl({} /*fd*/, path, nullptr, nullptr, system, false /*load_as_shared_library*/);
}

std::unique_ptr<const ApkAssets> ApkAssets::LoadImpl(
    unique_fd fd, const std::string& path, std::unique_ptr<Asset> idmap_asset,
    std::unique_ptr<const LoadedIdmap> loaded_idmap, bool system, bool load_as_shared_library) {
  
  // 1. 打开 .apk 文件, 使用 unmanaged_handle 描述文件句柄
  ::ZipArchiveHandle unmanaged_handle;
  int32_t result;
  if (fd >= 0) {
    result = ::OpenArchiveFd(fd.release(), path.c_str(), &unmanaged_handle, true /*assume_ownership*/);
  } else {
    // 调用 OpenArchive 打开资源目录
    result = ::OpenArchive(path.c_str(), &unmanaged_handle);
  }
  .......

  // 2. 创建一个 Native 的 ApkAssets 对象
  std::unique_ptr<ApkAssets> loaded_apk(new ApkAssets(unmanaged_handle, path));

  // 3. 从 .apk 中找寻资源表 resources.arsc
  ::ZipString entry_name(kResourcesArsc.c_str());
  ::ZipEntry entry;
  result = ::FindEntry(loaded_apk->zip_handle_.get(), entry_name, &entry);
  if (result != 0) {
    // 不存在 resources.arsc, 创建一个空的 LoadedArsc 对象
    loaded_apk->loaded_arsc_ = LoadedArsc::CreateEmpty();
    return std::move(loaded_apk);
  }
  ......
  // 4. 将 apk 中的 resources.arsc 文件 mmap 到内存
  // resources_asset_ 为 Asset 对象
  loaded_apk->resources_asset_ = loaded_apk->Open(kResourcesArsc, Asset::AccessMode::ACCESS_BUFFER);
  
  ......
  
  // 5. 反序列化 resources.arsc 数据到 LoadedArsc 对象中
  // 5.1 获取 resources.arsc 文件数据 Buffer
  const StringPiece data(
      reinterpret_cast<const char*>(loaded_apk->resources_asset_->getBuffer(true /*wordAligned*/)),
      loaded_apk->resources_asset_->getLength());
  
  // 5.2 调用 LoadedArsc::Load 反序列化 resources.arsc 文件, 构建成 LoadedArsc 对象
  loaded_apk->loaded_arsc_ =
      LoadedArsc::Load(data, loaded_idmap.get(), system, load_as_shared_library);
      
  ......
  // 返回 ApkAssets
  return std::move(loaded_apk);
}
```
ApkAssets::LoadImpl 中的流程还是非常清晰的
- 获取 apk 文件的 fd, 保存在 ApkAssets 对象中
- 从 apk 文件中找寻 resources.arsc 资源表
- 将 resources.arsc 文件 mmap 到内存
  - mmap 之后, 即可通过内存来访问文件中的数据, 使用这种方式减少与磁盘的直接 IO, 提升文件操作读取性能
- 反序列化 resources.arsc 数据到 **LoadedArsc** 对象中

ApkAssets 有了 [LoadedArsc](http://androidxref.com/9.0.0_r3/xref/frameworks/base/libs/androidfw/include/androidfw/LoadedArsc.h) 对象之后就具有资源查找的能力了, 其内部结构如下

![ApkAssets 存储结构](https://i.loli.net/2019/12/01/FVWo6CR7NTG1fuS.png)

关于 ApkAssets 反序列化 resources.arsc 的过程由于篇幅原因这里就不赘述了, 感兴趣可以打开源码查看

下面我们看看 nativeGetStringBlock 的实现

#### 2. nativeGetStringBlock
```
// frameworks/base/core/jni/android_content_res_ApkAssets.cpp
static jlong NativeGetStringBlock(JNIEnv* /*env*/, jclass /*clazz*/, jlong ptr) {
  // 获取 ApkAssets 对象
  const ApkAssets* apk_assets = reinterpret_cast<const ApkAssets*>(ptr);
  // 调用 LoadedArsc 的 GetStringPool 获取字符串资源池
  return reinterpret_cast<jlong>(apk_assets->GetLoadedArsc()->GetStringPool());
}

// frameworks/base/libs/androidfw/include/androidfw/LoadedArsc.h
inline const ResStringPool* GetStringPool() const {
  return &global_string_pool_;
}
```
NativeGetStringBlock 中主要是获取到了 LoadedArsc 中的 global_string_pool_

由 App 资源打包的知识可知, global_string_pool_ 存储的是**资源项值的字符串资源池**, 如下图所示

![资源项值的字符串资源池](https://i.loli.net/2019/12/01/iqDdISkt4Ol3uMW.png)

到这里 ApkAssets 就构建完成了, 下面我们看看 AssetsManager 的构建过程

### 二) AssetsManager 的构建
```java
public final class AssetManager implements AutoCloseable {

    public static class Builder {
    
        private ArrayList<ApkAssets> mUserApkAssets = new ArrayList<>();
        
        public AssetManager build() {
            // 1. 合并系统 Apk 资源数据 和 刚刚解析的 Apk 资源数据
            
            // 1.1 获取系统 Apk 资源数据集合
            final ApkAssets[] systemApkAssets = getSystem().getApkAssets();
            
            // 1.2 合并数据到 apkAssets数组
            final int totalApkAssetCount = systemApkAssets.length + mUserApkAssets.size();
            final ApkAssets[] apkAssets = new ApkAssets[totalApkAssetCount];
            System.arraycopy(systemApkAssets, 0, apkAssets, 0, systemApkAssets.length);
            final int userApkAssetCount = mUserApkAssets.size();
            for (int i = 0; i < userApkAssetCount; i++) {
                apkAssets[i + systemApkAssets.length] = mUserApkAssets.get(i);
            }
            
            // 2. 构建 AssetManaager
            final AssetManager assetManager = new AssetManager(false /*sentinel*/);
            
            // 3. 将资源数组 apkAssets 保存到 Java 的 AssetManaager
            assetManager.mApkAssets = apkAssets;
            
            // 4. 将资源数组 apkAssets 保存到 Native 的 AssetManaager
            AssetManager.nativeSetApkAssets(assetManager.mObject, apkAssets,
                    false /*invalidateCaches*/);
            return assetManager;
        }
    }
    
    @GuardedBy("this") private long mObject;

    private AssetManager(boolean sentinel) {
        // 2.1 创建 Native 层的 AssetManager
        mObject = nativeCreate();
        ......
    }
    
    private static native long nativeCreate();
    
    private static native void nativeSetApkAssets(long ptr, @NonNull ApkAssets[] apkAssets,
            boolean invalidateCaches);
}
```
AssetsManager 的构建流程如下
- 合并 **系统 Apk 的资源数据** 和 **上面解析的资源数据** 到 apkAssets 数组
  - 获取系统的 AssetManager, 进而获取系统资源数据数组
  - 进行合并操作
- 创建 Java 的 AssetsManager
  - 调用 nativeCreate 创建 Native 的 AssetsManager
- 将 apkAssets 数组保存到 Java 的 AssetsManager
- 将 apkAssets 数组保存到 Native 的 AssetsManager

接下来我们先看看获取系统 Apk 资源数据的过程

#### 1. 获取系统资源
```java
public final class AssetManager implements AutoCloseable {

    public static AssetManager getSystem() {
        synchronized (sSync) {
            // 获取系统的 AssetManager
            createSystemAssetsInZygoteLocked();
            return sSystem;
        }
    }
    
    private static final String FRAMEWORK_APK_PATH = "/system/framework/framework-res.apk";
    
    @GuardedBy("sSync") static AssetManager sSystem = null;

    
    /**
     * This must be called from Zygote so that system assets are shared by all applications.
     */
    @GuardedBy("sSync")
    private static void createSystemAssetsInZygoteLocked() {
    
        if (sSystem != null) {
            return;
        }
        
        ......

        try {
            final ArrayList<ApkAssets> apkAssets = new ArrayList<>();
            // 1. 加载系统资源 framework-res.apk 的数据
            apkAssets.add(ApkAssets.loadFromPath(FRAMEWORK_APK_PATH, true /*system*/));
            loadStaticRuntimeOverlays(apkAssets);
            sSystemApkAssetsSet = new ArraySet<>(apkAssets);
            sSystemApkAssets = apkAssets.toArray(new ApkAssets[apkAssets.size()]);
            
            // 2. 构建系统资源的 AssetManager
            sSystem = new AssetManager(true /*sentinel*/);
            
            // 3. 将数据保存到系统的 AssetManager
            sSystem.setApkAssets(sSystemApkAssets, false /*invalidateCaches*/);
        } catch (IOException e) {
            ......
        }
    }
    
}
```
从这里可以看到系统资源是以进程间单例 sSystem 的形式存在的, **它的 AssetManager 在 Zygote 进程启动时会进行加载, 也就是说我们应用进程 fork 之后就自带了系统的资源**

```
ps: 这里我们可以想到一个性能优化的思路, 我们在创建不需要使用系统资源的后台进程时, 可以通过释放系统资源的数据, 来减少后台服务进程的的内存
```
下面我们看看 Native 层 AssetsManager 的创建流程

#### 2. 创建 Native 层的 AssetsManager
nativeCreate 的 JNI 接口定义在 [android_util_AssetManager.cpp](http://androidxref.com/9.0.0_r3/xref/frameworks/base/core/jni/android_util_AssetManager.cpp) 中, 下面看看它的实现
```
// frameworks/base/core/jni/android_util_AssetManager.cpp
static jlong NativeCreate(JNIEnv* /*env*/, jclass /*clazz*/) {
  // 创建了一个 GuardedAssetManager 对象
  return reinterpret_cast<jlong>(new GuardedAssetManager());
}

struct GuardedAssetManager : public ::AAssetManager {
  // 内部持有 AssetManager2 对象
  Guarded<AssetManager2> guarded_assetmanager;
};
```
Native 层 AssetsManager 的构建过程比较简单, 最终构建了一个 [AssetManager2](http://androidxref.com/9.0.0_r3/xref/frameworks/base/libs/androidfw/AssetManager2.cpp) 对象, 这才是 Native 的 AssetManager 的真正实现

下面看看 将 ApkAsset 资源数组注入 Native 的过程

#### 3. 将 ApkAsset 资源数组注入 Native
```
// frameworks/base/core/jni/android_util_AssetManager.cpp
static void NativeSetApkAssets(JNIEnv* env, jclass /*clazz*/, jlong ptr,
                               jobjectArray apk_assets_array, jboolean invalidate_caches) {
  // 1. 将 Java 的 ApkAssets 数组构建成 C++ 的 vetor 集合
  const jsize apk_assets_len = env->GetArrayLength(apk_assets_array);
  std::vector<const ApkAssets*> apk_assets;
  apk_assets.reserve(apk_assets_len);
  for (jsize i = 0; i < apk_assets_len; i++) {
    jobject obj = env->GetObjectArrayElement(apk_assets_array, i);
    if (obj == nullptr) {
      std::string msg = StringPrintf("ApkAssets at index %d is null", i);
      jniThrowNullPointerException(env, msg.c_str());
      return;
    }
    jlong apk_assets_native_ptr = env->GetLongField(obj, gApkAssetsFields.native_ptr);
    if (env->ExceptionCheck()) {
      return;
    }
    apk_assets.push_back(reinterpret_cast<const ApkAssets*>(apk_assets_native_ptr));
  }
  // 2. 注入 Native 层的 AssetManager2 中
  ScopedLock<AssetManager2> assetmanager(AssetManagerFromLong(ptr));
  assetmanager->SetApkAssets(apk_assets, invalidate_caches);
}

// frameworks/base/libs/androidfw/AssetManager2.cpp
bool AssetManager2::SetApkAssets(const std::vector<const ApkAssets*>& apk_assets,
                                 bool invalidate_caches) {
  // 2.1 保存资源集合
  apk_assets_ = apk_assets;
  // 2.2 动态构建引用表
  BuildDynamicRefTable();
  // 2.3 重新构建缓存集合
  RebuildFilterList();
  // 2.4 刷新缓存
  if (invalidate_caches) {
    InvalidateCaches(static_cast<uint32_t>(-1));
  }
  return true;
}
```
可以看到将资源集合 ApkAssets 注入 AssetManager2 的过程中做了如下的操作
- 保存资源集合
- 构建动态引用表
- 重新构建缓存集合
- 清空缓存

这里我们主要关注一下 **动态引用表的构建过程**

```
// frameworks/base/libs/androidfw/include/androidfw/AssetManager2.h
std::vector<PackageGroup> package_groups_;
// 将包 ID 映射到索引到 package_groups 的数组
std::array<uint8_t, std::numeric_limits<uint8_t>::max() + 1> package_ids_;

// frameworks/base/libs/androidfw/AssetManager2.cpp
void AssetManager2::BuildDynamicRefTable() {
  // 清空 PackageGroup 集合
  package_groups_.clear();
  
  // 填充 packageId 数组
  package_ids_.fill(0xff);

  // 0x01 保留给 framework-res.apk 中的 android 资源包使用
  int next_package_id = 0x02;
  const size_t apk_assets_count = apk_assets_.size();
  
  // 1. 遍历 ApkAssets 资源包, 构建 PacakgeGroup
  for (size_t i = 0; i < apk_assets_count; i++) {
  
    const LoadedArsc* loaded_arsc = apk_assets_[i]->GetLoadedArsc();
    
    // 遍历 LoadedArsc 资源包中的 LoadedPackage 集合
    for (const std::unique_ptr<const LoadedPackage>& package : loaded_arsc->GetPackages()) {
    
      // 为共享库动态分配运行时 ID
      // 系统的资源为 0x01, 我们 App 的为 0x7f, 他们不需要动态分配, 运行时 ID 和编译时一致
      int package_id;
      // 判断是否需要动态分配 ID
      if (package->IsDynamic()) {
        // 需要动态分配
        package_id = next_package_id++;
      } else {
        // 不需要动态分配
        package_id = package->GetPackageId();
      }

      // 获取 运行时 ID 对应的 PacakgeGroup 对象在 package_groups_ 中的索引
      uint8_t idx = package_ids_[package_id];
      
      // 获取运行时 id 对应的 PacakgeGroup
      // idx 为 0xff, 说明当前运行时 ID 还未创建对应的 PackgeGroup 
      if (idx == 0xff) {
        // 获取索引值并保存
        package_ids_[package_id] = idx = static_cast<uint8_t>(package_groups_.size());
        // 添加一个 PacakgeGroup 对象
        package_groups_.push_back({});
        // 新添加的 PacakgeGroup 的资源表 DynamicRefTable
        DynamicRefTable& ref_table = package_groups_.back().dynamic_ref_table;
        // 记录运行时 ID
        ref_table.mAssignedPackageId = package_id;
        // 判断是否为应用程序的资源包, app 资源包 id 一般以 0x7f 开头
        ref_table.mAppAsLib = package->IsDynamic() && package->GetPackageId() == 0x7f;
      }
      // 获取 package_groups_ 在 idx 处的 PackageGroup 对象
      PackageGroup* package_group = &package_groups_[idx];
      
      // 将当前资源包 LoadedPackage 添加到 PackageGroup 中的 packages_ 缓存中
      // 因为运行时 ID 是递增的, 所以 PacakgeGroup 与 LoadedPackage 是一一对应的关系
      // 也就是说 packages_ 只有一条数据
      package_group->packages_.push_back(ConfiguredPackage{package.get(), {}});
      package_group->cookies_.push_back(static_cast<ApkAssetsCookie>(i));

      // 将 LoadedPacakge 的中所有的 包名 与 编译时 ID 映射关系保存在 PackageGroup 的 mEntries 中
      for (const DynamicPackageEntry& entry : package->GetDynamicPackageMap()) {
        // 获取当前 pacakge 的包名
        String16 package_name(entry.package_name.c_str(), entry.package_name.size());
        // 添加 包名 和 编译时 ID 的键值对
        // key: package_name
        // value: LoadedPackage 的静态 ID      
        package_group->dynamic_ref_table.mEntries.replaceValueFor(
            package_name, static_cast<uint8_t>(entry.package_id));
      }
    }
  }

  // 2. 为所有的 PackageGroup 动态引用表 DynamicRefTable 注入运行时 ID
  const auto package_groups_end = package_groups_.end();
  for (auto iter = package_groups_.begin(); iter != package_groups_end; ++iter) {
  
    // 2.1 获取 iter 中 LoadedPacakge 的包名
    // 上面提到过, 一个 PackageGroup 只存在一个 LoadedPacakge, 只取第 0 个元素即可
    const std::string& package_name = iter->packages_[0].loaded_package_->GetPackageName();
    
    // 2.2 为引用了 iter 资源的所有 PacakgeGroup.DynamicRefTable 引用表中注入 iter 的运行时 ID mAssignedPackageId
    for (auto iter2 = package_groups_.begin(); iter2 != package_groups_end; ++iter2) {
      iter2->dynamic_ref_table.addMapping(String16(package_name.c_str(), package_name.size()),
                                          iter->dynamic_ref_table.mAssignedPackageId);
    }
  }
  
}

// frameworks/base/libs/androidfw/ResourceTypes.cpp
status_t DynamicRefTable::addMapping(const String16& packageName, uint8_t packageId)
{
    // 2.2.1 获取 packageName 对应的编译时 ID
    ssize_t index = mEntries.indexOfKey(packageName);
    if (index < 0) {
        return UNKNOWN_ERROR;
    }
    // 2.2.2 将运行时 ID packageId 存储在, 以编译时 ID 为索引的槽内
    mLookupTable[mEntries.valueAt(index)] = packageId;
    return NO_ERROR;
}
```
动态引用表的构建是相当复杂的, 代码中的注释也非常的详细, 总结下来主要有两步
- 为共享资源库分配运行时 ID
  - framework-res.apk 为 0x01, 我们的 App 为 0x7f, 他们的运行时 ID 与编译时 ID 一致
- 为所有的 LoadedPackage 创建对应的 PacakgeGroup
  - 生成运行时 ID, 保存在 PacakgeGroup 的 mAssignedPackageId 字段中
  - 将 LoadedPackage 的编译时 ID, 注入到 PacakgeGroup 动态引用表的 mEntry 中
- 为所有的 PackageGroup 动态引用表 DynamicRefTable 注入运行时 ID

动态引用表的构建, **是为了解决共享资源库在我们 resources.arsc 中的编译时 ID 和运行时动态分配的 ID 不一致的问题**

### 三) 回顾
AssetManager 的构建算是资源管理的核心内容了, 它主要处理的事务如下
- 构建 ApkAsset
  - **创建 Native 的 ApkAsset, 解析  apk 中的 resources.arsc 数据到 LoadedArsc**
  - **保存 resources.arsc 中的资源项值的字符串资源池**
- 构建 AssetManager
  - 合并 framework-res.apk 的系统资源 ApkAsset 集合
  - 构建 Native 层的 AssetManager2
  - 将合并后的资源集合保存在 AssetManager2 中
    - 保存资源集合
    - **重新构建引用表**
    - 重新构建缓存集合
    - 清空缓存

**需要注意的是, 虽然我们应用 apk 内在编译时只会生成一个 resource.arsc 文件, 但是这个文件里至少存在两个 LoadedPackage, 一个是我们自己的资源生成的 LoadedPackage, 还有有一个是我们引用的 android 资源, 即 framework-res.apk 中的资源**

整个编译时 ID 与运行时 ID 的映射关系如下所示

![编译时 ID 与运行时 ID 的映射](https://i.loli.net/2019/12/01/1fAFdg8XwItCpoh.png)


```
到这里我们就清楚为什么资源管理类初始化完成之后, 需要进行 R 文件资源 ID 的重写了

为了在运行时能够正确的定位资源, 我们需要将 R 文件中所有的 ID 的 packageId 替换成运行时 ID, 这样才能够通过 package_ids_ 表正确的定位到最终的目标资源包
```

## 总结

![资源管理整体依赖图](https://i.loli.net/2019/12/01/8ehpGSx9wWjfBLc.png)

构建 Application 的 ContextImpl 中通过 ResourcesManager 获取 Resources 的步骤如下
- 构建 ResourcesKey
- 根据 ResourcesKey 从缓存 mResourceImpls 中获取 ResourcesImpl 对象
  - 缓存不存在则构建 ResourcesImpl 对象, 创建流程如下
    - 获取 AssetManager
      - 构建 ApkAsset
        - **创建 Native 的 ApkAsset, 解析  apk 中的 resources.arsc 数据到 LoadedArsc**
        - **保存 resources.arsc 中的资源项值的字符串资源池**
      - 构建 AssetManager
        - 合并 framework-res.apk 的系统资源 ApkAsset 集合
        - 构建 Native 层的 AssetManager2
        - 将合并后的资源集合保存在 AssetManager2 中
          - 保存资源集合
          - 重新构建引用表
          - 重新构建缓存集合
          - 刷新缓存 
    - 创建 ResourcesImpl 对象
      - 内部持有 AseetManager
- 优先从 mResourceReferences 缓存中获取 Resources 对象
  - 不存在则创建新的 Resources 对象
    - 将 ClassLoader 和 ResourcesImpl 注入 

当 Application 创建完成之后, 会通过 rewriteRValues 来重新复写 R 文件中的 ID 为运行时 ID

资源管理的创建我们就分析到这里, 后面一篇文章我们着重分析一下资源的查找流程, 看看我们是如何通过资源表找到对应的资源的

## 参考文献
- [Android 重学系列 资源的查找](https://www.jianshu.com/p/b153d63d60b3)
- [Dynamic Reference](https://blog.csdn.net/dayong198866/article/details/95635910)
- [共享资源库](https://blog.csdn.net/dayong198866/article/details/95315337)