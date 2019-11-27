---
title: Android 系统架构 —— PKMS 的启动 之 应用目录的扫描
permalink: android-source/pkms-launch2
key: android-source-pkms-launch2
tags: AndroidFramework
---

## 前言
通过上一篇文章我们了解了 PKMS 是如何通过 **Settings.readLPw** 解析备份文件 package.xml 的

读取了上一次备份的信息之后, 下面便会调用  **PackageManagerService.scanDirTracedLI** 扫描指定目录下的应用程序, 我们就探讨一下它是如何进行扫描处理的

<!--more-->

## 扫描入口
```
public class PackageManagerService {

    private void scanDirTracedLI(File scanDir, final int parseFlags, int scanFlags, long currentTime) {
        try {
            scanDirLI(scanDir, parseFlags, scanFlags, currentTime);
        } finally {
            ......
        }
    }
    
    private void scanDirLI(File scanDir, int parseFlags, int scanFlags, long currentTime) {
        // 获取扫描文件目录 scanDir 下的所有文件集合
        final File[] files = scanDir.listFiles();
        if (ArrayUtils.isEmpty(files)) {
            // 目录里没有文件, 则直接 return
            return;
        }
        // 解析 scanDir 目录中的文件
        // 创建 ParallelPackageParser 对象, 可并发进行安装包解析的工具
        try (ParallelPackageParser parallelPackageParser = new ParallelPackageParser(
                mSeparateProcesses, mOnlyCore, mMetrics, mCacheDir,
                mParallelPackageParserCallback)) {
            int fileCount = 0;
            // 1. 提交解析任务
            for (File file : files) {
                // apk 文件或者文件目录
                final boolean isPackage = (isApkFile(file) || file.isDirectory())
                        && !PackageInstallerService.isStageName(file.getName());
                .....
                // 提交解析任务
                parallelPackageParser.submit(file, parseFlags);
                fileCount++;// 解析的文件数递增
            }

            // 2. 处理解析结果
            for (; fileCount > 0; fileCount--) {
                // 获取解析结果 ParseResult
                ParallelPackageParser.ParseResult parseResult = parallelPackageParser.take();
                Throwable throwable = parseResult.throwable;
                int errorCode = PackageManager.INSTALL_SUCCEEDED;
                if (throwable == null) {
                    ......
                    try {
                        // 若没有异常, 则调用 scanPackageChildLI 解析结果
                        if (errorCode == PackageManager.INSTALL_SUCCEEDED) {
                            scanPackageChildLI(parseResult.pkg, parseFlags, scanFlags,
                                    currentTime, null);
                        }
                    } catch (PackageManagerException e) {
                        ......
                    }
                }
                ......
            }
        }
    }
}
```
好的, 可以看到 PMS 的 scanDirLI 中的实现非常有意思, 为了加快扫描的速度, 它**使用了并发编程中生产者消费者**的思路
- **解析文件夹下的可安装文件**
  - 将找到的文件 通过 **ParallelPackageParser.submit** 到线程池中进行解析
  - 将解析的结果放置到 ParallelPackageParser 的阻塞队列中
- **处理解析结果**
  - 通过 ParallelPackageParser.take() 获取解析结果 ParseResult
  - 通过 **scanPackageChildLI** 方法处理扫描结果

这里有两个步骤需要我们关注
- **首先是 ParallelPackageParser.submit 线程池并发的扫描已安装的应用程序**
- **另一个是 scanPackageChildLI 将扫描到的应用数据发布到 PMS**

接下来我们一一查看

## 一. 扫描已安装的应用程序
```
class ParallelPackageParser implements AutoCloseable {

    // 用于并发进行 apk 解析安装的线程池
    private final ExecutorService mService = ConcurrentUtils.newFixedThreadPool(MAX_THREADS,
            "package-parsing-thread", Process.THREAD_PRIORITY_FOREGROUND);
    // 存储安装包解析结果的队列
    private final BlockingQueue<ParseResult> mQueue = new ArrayBlockingQueue<>(QUEUE_CAPACITY);

    public void submit(File scanFile, int parseFlags) {
        mService.submit(() -> {
            // 1. 构建一个解析结果的描述对象
            ParseResult pr = new ParseResult();
            try {
               // 2. 构建一个 安装包 解析器
                PackageParser pp = new PackageParser();
                ......// 2.1 给 pp 设置一些参数
                pr.scanFile = scanFile;
                // 3. 调用 parsePackage 方法真正执行安装包解析
                // 解析成功之后会返回一个 PackageParser.Package 对象, 写入到 ParseResult 的成员变量 pkg 中
                pr.pkg = parsePackage(pp, scanFile, parseFlags);
            } 
            ......
            try {
                // 4. 将解析结果添加到队列中
                // 通过 ParallelPackageParser.take 方法便可从队列中获取一个安装包的解析结果
                mQueue.put(pr);
            } catch (InterruptedException e) {
                ......
            }
        });
    }
    
    
    @VisibleForTesting
    protected PackageParser.Package parsePackage(PackageParser packageParser, File scanFile,
            int parseFlags) throws PackageParser.PackageParserException {
        // 3.1 将安装包解析的操作分发给了 PackageParser.parsePackage
        return packageParser.parsePackage(scanFile, parseFlags, true /* useCaches */);
    }

    
}
```
好的可以看到 ParallelPackageParser.submit 方法主要作用是创建一个 Runnable 投递到内部维护的线程池之中, 具体操作如下
- 创建 ParseResult 
- 创建 PackageParser, 调用 **PackageParser.parsePackage** 方法解析 apk
- 解析成功之后会返回一个 PackageParser.Package 对象, 写入到 ParseResult 的成员变量 pkg 中
- 写入工作队列

下面我们重点看看 PackageParser.parsePackage 是如何解析 Apk 的
```
public class PackageParser {
    
    public Package parsePackage(File packageFile, int flags, boolean useCaches)
            throws PackageParserException {
        // 1. 尝试从缓存获取 Package 安装包对象
        // 初次解析不存在缓存, 我们主要分析无缓存的情况
        Package parsed = useCaches ? getCachedResult(packageFile, flags) : null;
        if (parsed != null) {
            return parsed;
        }
        ......
        // 2. 执行解析操作
        // 2.1 解析安装目录
        if (packageFile.isDirectory()) {   
            parsed = parseClusterPackage(packageFile, flags);
        } 
        // 2.2 解析 ".apk" 安装包
        else {
            parsed = parseMonolithicPackage(packageFile, flags);
        }
        // 3. 将解析后的 Package 对象写入缓存
        cacheResult(packageFile, flags, parsed);
        ......
        return parsed;
    }
```
在 parsePackage 方法中我们发现了很有意思的信息, 它支持两种解析方式
- 解析安装目录
- 解析 ".apk" 安装包

**这里我们可能会觉得奇怪, 一个安装好的应用, 不是应该以安装包的形式存在吗? 为什么还有 apk 的情况呢?**
- 这是因为 parsePackage 方法不仅仅在扫描时会调用, 在安装一个应用程序时也会被调用, 我们在应用安装篇会看到相关的调用, 这里就不赘述了

一个安装好的应用文件目录如下

![应用安装目录](https://i.loli.net/2019/11/15/ad8CQ3tIMwZvxnb.png)

接下来我们看看 **PackageParser.parseClusterPackage** 是如何扫描这个文件夹下的数据的
```
public class PackageParser {

   private Package parseClusterPackage(File packageDir, int flags) throws PackageParserException {
        // 1. 获取应用摘要信息
        final PackageLite lite = parseClusterPackageLite(packageDir, 0);
        ......
        // 2. 创建资源管理者
        // 2.1 构建资源加载器
        SparseArray<int[]> splitDependencies = null;
        final SplitAssetLoader assetLoader;
        if (lite.isolatedSplits && !ArrayUtils.isEmpty(lite.splitNames)) {
            .....// 是否是独立分离的应用? 这里没看懂
        } else {
            // 创建 DefaultSplitAssetLoader 资源加载器
            assetLoader = new DefaultSplitAssetLoader(lite, flags);
        }
        try {
            // 2.2 获取资源管理器 AssetManager
            final AssetManager assets = assetLoader.getBaseAssetManager();
            
            // 3. 解析 bask.apk
            // 获取 "bask.apk" 的路径
            final File baseApk = new File(lite.baseCodePath);
            // 解析 bask.apk 中的信息
            final Package pkg = parseBaseApk(baseApk, assets, flags);
            ......
            // 存储代码路径
            pkg.setCodePath(packageDir.getCanonicalPath());
            ......
            return pkg;
        } catch (IOException e) {
            throw new PackageParserException(INSTALL_PARSE_FAILED_UNEXPECTED_EXCEPTION,
                    "Failed to get path: " + lite.baseCodePath, e);
        } finally {
            IoUtils.closeQuietly(assetLoader);
        }
    }
    
}
```
PackageParser.parseMonolithicPackage 解析安装包的操作如下
- parseClusterPackageLite: **获取应用摘要信息**
- **创建资源管理者 AssetManager**
  - 创建 DefaultSplitAssetLoader 
  - 创建 AssetManager
- parseBaseApk: **解析 "base.apk" 信息**

接下来我们从这三个方面切入, 来分析解析一个应用的过程

### 一) 获取应用摘要信息
```
 static PackageLite parseClusterPackageLite(File packageDir, int flags)
            throws PackageParserException {
        // 获取路径下的文件集合
        final File[] files = packageDir.listFiles();

        String packageName = null;
        int versionCode = 0;
        final ArrayMap<String, ApkLite> apks = new ArrayMap<>();
        for (File file : files) {
            if (isApkFile(file)) {
                // 解析 apk 文件的摘要信息
                final ApkLite lite = parseApkLite(file, flags);
                ......
                // 环境解析到的 apk 文件
                if (apks.put(lite.splitName, lite) != null) {
                    ......
                }
            }
        }
        // 获取 baseApk 的
        final ApkLite baseApk = apks.remove(null);
        ......
        // 代码路径
        final String codePath = packageDir.getAbsolutePath();
        // 将数据封装到 PackageLite 中
        return new PackageLite(codePath, baseApk, splitNames, isFeatureSplits, usesSplitNames,
                configForSplits, splitCodePaths, splitRevisionCodes);
    }
    
}
```
可以看到 parseClusterPackageLite 方法中, 主要调用了 parseApkLite 来解析 apk 的摘要信息, 然后将相关数据封装到 PackageLite 中返回到上层

下面我们看看 **parseApkLite** 的实现过程
```
public class PackageParser {
    
    public static ApkLite parseApkLite(File apkFile, int flags)
            throws PackageParserException {
        return parseApkLiteInner(apkFile, null, null, flags);
    }
    
    public static final String ANDROID_MANIFEST_FILENAME = "AndroidManifest.xml";
  
    private static ApkLite parseApkLiteInner(File apkFile, FileDescriptor fd, String debugPathName,
            int flags) throws PackageParserException {
        final String apkPath = fd != null ? debugPathName : apkFile.getAbsolutePath();
        // 1. 创建 AndroidManifest.xml 文件的解析器
        XmlResourceParser parser = null;
        try {
            // 1.1 创建 Apk 的资源信息
            final ApkAssets apkAssets;
            try {
                apkAssets = fd != null
                        ? ApkAssets.loadFromFd(fd, debugPathName, false, false)
                        : ApkAssets.loadFromPath(apkPath);
            } catch (IOException e) {
                throw new PackageParserException(INSTALL_PARSE_FAILED_NOT_APK,
                        "Failed to parse " + apkPath);
            }

            // 1.2 创建一个 AndroidManifest.xml 文件的解析器
            parser = apkAssets.openXml(ANDROID_MANIFEST_FILENAME);

            final SigningDetails signingDetails;
            ..... 获取签名信息
            final AttributeSet attrs = parser;
            // 3. 解析包的轻量级信息
            return parseApkLite(apkPath, parser, attrs, signingDetails);
        } catch (XmlPullParserException | IOException | RuntimeException e) {
            ......
        } finally {
            ......
        }
    }
}
```
PackageParser.parseMonolithicPackageLite 会调用到 parseApkLiteInner 中去, 这个方法中主要做了如下的工作
- 创建 AndroidManifest.xml 的解析器
  - 创建 Apk 的资源信息 ApkAssets
  - 创建 AndroidManifest.xml 的解析器
- 解析 apk 的轻量级信息

下面我们一一查看

#### 1. AndroidManifest.xml 解析器的创建
```
public final class ApkAssets {
    
    public static @NonNull ApkAssets loadFromPath(@NonNull String path) throws IOException {
        return new ApkAssets(path, false /*system*/, false /*forceSharedLib*/, false /*overlay*/);
    }
    
    @GuardedBy("this") private final long mNativePtr;
  
    private ApkAssets(@NonNull String path, boolean system, boolean forceSharedLib, boolean overlay)
            throws IOException {
        Preconditions.checkNotNull(path, "path");
        // 加载 APK 中的资源信息
        mNativePtr = nativeLoad(path, system, forceSharedLib, overlay);
        // 检索 APK 中的字符串信息
        mStringBlock = new StringBlock(nativeGetStringBlock(mNativePtr), true /*useSparse*/);
    }
    
    private static native long nativeLoad(
            @NonNull String path, boolean system, boolean forceSharedLib, boolean overlay)
            throws IOException;
            
    private static native long nativeGetStringBlock(long ptr);

    public @NonNull XmlResourceParser openXml(@NonNull String fileName) throws IOException {
        synchronized (this) {
            // 获取 xml 解析器句柄值
            long nativeXmlPtr = nativeOpenXml(mNativePtr, fileName);
            // 创建 XmlBlock
            try (XmlBlock block = new XmlBlock(null, nativeXmlPtr)) {
                // 创建解析器
                XmlResourceParser parser = block.newParser();
                ......
                return parser;
            }
        }
    }

    private static native long nativeOpenXml(long ptr, @NonNull String fileName) throws IOException;

}
```
ApkAssets 的构造函数中主要做了两件事情
- 通过 nativeLoad 获取 Apk 中的资源信息
- 通过 nativeGetStringBlock 获取 Apk 中的资源字符串
- 通过 nativeOpenXml 创建 xml 解析器的 C++ 实现

这便是 Resource 资源管理的知识点了, 这里我们先放置一边, 到后面的章节再具体分析, 下面我们主要看看, parseApkLite 如何解析 apk 的轻量级数据

#### 2. 解析 base.apk 的摘要信息
```
public class PackageParser {

    private static ApkLite parseApkLite(String codePath, XmlPullParser parser, AttributeSet attrs,
            SigningDetails signingDetails)
            throws IOException, XmlPullParserException, PackageParserException {
        final Pair<String, String> packageSplit = parsePackageSplitNames(parser, attrs);
        ......
        
        final List<VerifierInfo> verifiers = new ArrayList<VerifierInfo>();
        while ((type = parser.next()) != XmlPullParser.END_DOCUMENT
                && (type != XmlPullParser.END_TAG || parser.getDepth() >= searchDepth)) {
            ......
            // 解析 <package-verifier>
            if (TAG_PACKAGE_VERIFIER.equals(parser.getName())) {
                final VerifierInfo verifier = parseVerifier(attrs);
                if (verifier != null) {
                    verifiers.add(verifier);
                }
            }
            // 解析 <application> 的概要
            else if (TAG_APPLICATION.equals(parser.getName())) {
                for (int i = 0; i < attrs.getAttributeCount(); ++i) {
                    final String attr = attrs.getAttributeName(i);
                    if ("debuggable".equals(attr)) {
                        debuggable = attrs.getAttributeBooleanValue(i, false);
                    }
                    if ("multiArch".equals(attr)) {
                        multiArch = attrs.getAttributeBooleanValue(i, false);
                    }
                    if ("use32bitAbi".equals(attr)) {
                        use32bitAbi = attrs.getAttributeBooleanValue(i, false);
                    }
                    if ("extractNativeLibs".equals(attr)) {
                        extractNativeLibs = attrs.getAttributeBooleanValue(i, true);
                    }
                }
            } 
            // 解析  <uses-split>
            else if (TAG_USES_SPLIT.equals(parser.getName())) {
                ......
                usesSplitName = attrs.getAttributeValue(ANDROID_RESOURCES, "name");
                ......
            }
        }

        return new ApkLite(codePath, packageSplit.first, packageSplit.second, isFeatureSplit,
                configForSplit, usesSplitName, versionCode, versionCodeMajor, revisionCode,
                installLocation, verifiers, signingDetails, coreApp, debuggable,
                multiArch, use32bitAbi, extractNativeLibs, isolatedSplits);
    }
    
}
```
可以看到 PackageParser 的 parseApkLite 重载方法, 主要解析了以下的摘要
- <package-verifier> 标签
- <application> 标签
  - debuggable
  - multiArch
  - use32bitAbi
  - extractNativeLibs
- <uses-split> 标签

最终会将这些摘要信息封装到 **ApkLite** 中暂存, 到这里一个 apk 的轻量级信息就创建好了, 最终会构建一个 **PackageLite** 返回到上层

接下来我们回到 **parseClusterPackage** 方法看看 AssetManager 的创建过程

### 二) 创建应用的资源管理者 AssetManager
在 parseClusterPackage 方法中我们知道, 在获取 AssetManager 之前, 会先创建一个 DefaultSplitAssetLoader 对象, 它的构造如下
```
public class DefaultSplitAssetLoader implements SplitAssetLoader {
    
    private final String mBaseCodePath;
    private final String[] mSplitCodePaths;
    
    public DefaultSplitAssetLoader(PackageParser.PackageLite pkg, @ParseFlags int flags) {
       // 即 apk 文件路径
        mBaseCodePath = pkg.baseCodePath;
        // 由 PackageLite 的构造可知, 这个属性为 null
        mSplitCodePaths = pkg.splitCodePaths;
        mFlags = flags;
    }
    
}
```
DefaultSplitAssetLoader 构建流程比较简单, 即为成员变量赋值, 下面看看它获取 AssetManager 的操作
```
public class DefaultSplitAssetLoader implements SplitAssetLoader {
    
    @Override
    public AssetManager getBaseAssetManager() throws PackageParserException {
        if (mCachedAssetManager != null) {
            return mCachedAssetManager;
        }
        // 这里为 null
        ApkAssets[] apkAssets = new ApkAssets[(mSplitCodePaths != null
                ? mSplitCodePaths.length : 0) + 1];
        // Load the base.
        int splitIdx = 0;
        
        // 加载 App 资源
        apkAssets[splitIdx++] = loadApkAssets(mBaseCodePath, mFlags);
        ......
        
        // 创建 AssetManager 对象
        AssetManager assets = new AssetManager();
        ....
        // 注入 App 资源
        assets.setApkAssets(apkAssets, false /*invalidateCaches*/);
        // 获取这个 AssetManager
        mCachedAssetManager = assets;
        return mCachedAssetManager;
    }
    
    private static ApkAssets loadApkAssets(String path, @ParseFlags int flags)
            throws PackageParserException {
        .......
        try {
            // 调用 ApkAssets 的 loadFromPath 获取 app 的资源
            return ApkAssets.loadFromPath(path);
        } catch (IOException e) {
            ......
        }
    }
}
```
DefaultSplitAssetLoader 的 getBaseAssetManager 函数
- 调用 ApkAssets.loadFromPath 根据 apk 路径, 获取 apk 的资源 ApkAssets
- 将这个资源 ApkAssets 作为形参注入到创建的 AssetManager 对象中

关于 ApkAssets.loadFromPath 在解析 apk 的轻量级信息中我们已经看到过了, 它的实现在 native 中, 不是我们本篇文章讨论的重点

现在有了 AssetManager 对象, 下面就看看 PackageParser.parseBaseApk 如何解析 apk 完整的信息

### 三) 解析 base.apk 信息
```
public class PackageParser {

    public static final String ANDROID_MANIFEST_FILENAME = "AndroidManifest.xml";

    private Package parseBaseApk(File apkFile, AssetManager assets, int flags)
            throws PackageParserException {
        // 1. 获取安装包路径
        final String apkPath = apkFile.getAbsolutePath();
        ......
        // 2. 将要解析的安装包路径保存到成员变量 mArchiveSourcePath 中
        mArchiveSourcePath = apkFile.getAbsolutePath();
        // 3. 构建一个 AndroidManifest.xml 的 XML 文件解析器
        XmlResourceParser parser = null;
        try {
            final int cookie = assets.findCookieForPath(apkPath);
            ......
            parser = assets.openXmlResourceParser(cookie, ANDROID_MANIFEST_FILENAME);
            // 构建一个资源对象
            final Resources res = new Resources(assets, mMetrics, null);
            ......
            // 4. 调用重载方法 parseBaseApk 继续进行解析操作
            final Package pkg = parseBaseApk(apkPath, res, parser, flags, outError);
            ......
            return pkg;
        } catch (PackageParserException e) {
            ......
        }
    }
    
    private Package parseBaseApk(String apkPath, Resources res, XmlResourceParser parser, int flags,
            String[] outError) throws XmlPullParserException, IOException {
        ......
        // 解析包名
        final String splitName;
        final String pkgName;
        try {
            Pair<String, String> packageSplit = parsePackageSplitNames(parser, parser);
            pkgName = packageSplit.first;
            splitName = packageSplit.second;
            ......
        } catch (PackageParserException e) {
            ......
        }
        ......
        // 构建了一个 Package 描述一个应用的安装包信息
        final Package pkg = new Package(pkgName);
        // 获取 AndroidManifest.xml 的属性数组
        TypedArray sa = res.obtainAttributes(parser,
                com.android.internal.R.styleable.AndroidManifest);
        // ......解析版本号相关数据
        sa.recycle();
        // 解析基本信息
        return parseBaseApkCommon(pkg, null, res, parser, flags, outError);
    }

    private Package parseBaseApkCommon(Package pkg, Set<String> acceptedTags, Resources res,
            XmlResourceParser parser, int flags, String[] outError) throws XmlPullParserException,
            IOException {
        // 获取 AndroidManifest.xml 的属性数组
        TypedArray sa = res.obtainAttributes(parser,
                com.android.internal.R.styleable.AndroidManifest);
        // 1. 解析 sharedUserId 属性
        String str = sa.getNonConfigurationString(
                com.android.internal.R.styleable.AndroidManifest_sharedUserId, 0);
        if (str != null && str.length() > 0) {
            ......
            // 提取属性值, 保存在 pkg 中的成员变量中
            pkg.mSharedUserId = str.intern();
        }
        int outerDepth = parser.getDepth();
        while ((type = parser.next()) != XmlPullParser.END_DOCUMENT
                && (type != XmlPullParser.END_TAG || parser.getDepth() > outerDepth)) {
            if (type == XmlPullParser.END_TAG || type == XmlPullParser.TEXT) {
                continue;
            }
            String tagName = parser.getName();
            if (tagName.equals(TAG_APPLICATION)) {
                ......
                // 2. 解析 <application> ... </application> 标签
                if (!parseBaseApplication(pkg, res, parser, flags, outError)) {
                    return null;
                }
            } 
            ......
            else if (tagName.equals(TAG_USES_PERMISSION)) {
                // 3. 解析<users-permission> 标签
                if (!parseUsesPermission(pkg, res, parser)) {
                    return null;
                }
            }
        }
    }
}
```
好的, 可以看到解析 APK 安装包, 本质是即解析 AndroidManifest.xml 文件中的数据, 主要如下
- 构建 **Package** 对象, 用于存储解析到的信息
- 解析 sharedUserId 属性, 写入到 Package 的成员变量 mSharedUserId 中
- 调用 **PackageParser.parseBaseApplication** 解析 <application> 标签
- 调用 PackageParser.parseUsesPermission 解析 <users-permission>  标签

我们都知道, 一个 APP 中所有组件信息都会在 <application> 标签中声明, 可见其分量之重

这里接着分析 PackageParser.parseBaseApplication 方法, 看看它对 <application> 标签做了哪些操作
```
public class PackageParser {
    
    private boolean parseBaseApplication(Package owner, Resources res,
            XmlResourceParser parser, int flags, String[] outError)
        throws XmlPullParserException, IOException {
        ......
        int type;
        ......
        while ((type = parser.next()) != XmlPullParser.END_DOCUMENT
                && (type != XmlPullParser.END_TAG || parser.getDepth() > innerDepth)) {
            if (type == XmlPullParser.END_TAG || type == XmlPullParser.TEXT) {
                continue;
            }      
            String tagName = parser.getName();
            // 1. 解析 <activity/> 标签
            if (tagName.equals("activity")) {
                Activity a = parseActivity(owner, res, parser, flags, outError, cachedArgs, false,
                        owner.baseHardwareAccelerated);
                ......
                owner.activities.add(a);
            // 2. 解析 <receiver/> 标签
            } else if (tagName.equals("receiver")) {
                Activity a = parseActivity(owner, res, parser, flags, outError, cachedArgs,
                        true, false);
                ......
                owner.receivers.add(a);
            // 3. 解析 <Service/> 标签
            } else if (tagName.equals("service")) {
                Service s = parseService(owner, res, parser, flags, outError, cachedArgs);
                ......
                owner.services.add(s);
            // 4. 解析 <provider/> 标签
            } else if (tagName.equals("provider")) {
                Provider p = parseProvider(owner, res, parser, flags, outError, cachedArgs);
                ......
                owner.providers.add(p);
            }
            ......
        }
        ......
        return true;
    }
    
}
```
在 PackageParser.parseBaseApplication 方法中, 主要是对我们解析我们的 <application/> 的标签, 有我们常见的四大组件, 解析之后将它封装到我们的 Package 对象的缓存中去, 后续就可以进行快速的匹配了
- 这里的 PackageParser.parseActivity 获取到的 Activity 对象, 其实就是一个用于做用于做简单的信息存储的 Activity 对象

好的, 至此一个 Apk 中 AndroidManifest 的信息都已经解析封装到 Package 对象中了, 至此解析操作便完成了

### 回顾
扫描一个已安装的应用程序的流程如下
- **构建应用摘要对象 PackageLite**
  - 构建 base.apk 摘要对象 ApkLite 
    - 创建 AndroidManifest.xml 解析器
    - 解析 AndroidMainfest 中的概要, 封装到 ApkLite 中
    - 将 ApkLite 存入 PackageLite 返回
- **获取应用的 AssetManager**
  - 创建描述 apk 资源的 ApkAsset 对象
  - 创建 AssetManager 
  - 将 ApkAsset 存入 AssetManager
- **构建整个应用信息的对象 Package**
  - 获取 AndroidManifest.xml 的解析器
  - 解析标签内四大组件的信息
  - 将解析到的数据封装到 Package 中

安装包的数据解析完毕之后, 就可以通过 Package 来获取应用程序的数据了, 下面我们还需要将扫描的数据发布到 PMS 中, 整个工作才算完成

接下来我们回到 PMS 的 scanPackageChildLI 中看看 PMS 是如何处理发布应用信息的

## 二. 发布应用信息
```
public class PackageManagerService {
    
    final ArrayMap<String, PackageParser.Package> mPackages =
            new ArrayMap<String, PackageParser.Package>();
    
    private PackageParser.Package scanPackageChildLI(PackageParser.Package pkg,
                                                     final @ParseFlags int parseFlags, @ScanFlags int scanFlags, long currentTime,
                                                     @Nullable UserHandle user)
            throws PackageManagerException {
        ......
        if (scanSystemPartition && isSystemPkgUpdated && !isSystemPkgBetter) {
            ......
            synchronized (mPackages) {
                // 1. 将我们的 Package 对象添加到缓存
                if (!mPackages.containsKey(pkg.packageName)) {
                    mPackages.put(pkg.packageName, pkg);
                }
            }
            ......
        }
        // 调用 addForInitLI 方法, 处理安装包信息
        PackageParser.Package scannedPkg = addForInitLI(pkg, parseFlags,
                scanFlags, currentTime, user);
        ......
        return scannedPkg;
    }
    
    private PackageParser.Package addForInitLI(PackageParser.Package pkg,
                                               @ParseFlags int parseFlags, @ScanFlags int scanFlags, long currentTime,
                                               @Nullable UserHandle user)
            throws PackageManagerException {
        ......
        // 将 pkg 传递给了 scanPackageNewLI 处理后续操作
        final PackageParser.Package scannedPkg = scanPackageNewLI(pkg, parseFlags, scanFlags
                | SCAN_UPDATE_SIGNATURE, currentTime, user);
        ......
        return scannedPkg;  
    }
    
    private PackageParser.Package scanPackageNewLI(@NonNull PackageParser.Package pkg,
                                                   final @ParseFlags int parseFlags, @ScanFlags int scanFlags, long currentTime,
                                                   @Nullable UserHandle user) throws PackageManagerException {
        ......
        synchronized (mPackages) {
            SharedUserSetting sharedUserSetting = null;
            // 2.1 若该应用为共享应用, 则获取共享应用 ID 的描述
            if (pkg.mSharedUserId != null) {
                sharedUserSetting = mSettings.getSharedUserLPw(
                        pkg.mSharedUserId, 0 /*pkgFlags*/, 0 /*pkgPrivateFlags*/, true /*create*/);
                ......
            }
            try {
                // 2.2 将我们的 pkg 封装到 ScanRequest 中
                 final ScanRequest request = new ScanRequest(pkg, sharedUserSetting,
                        pkgSetting == null ? null : pkgSetting.pkg, pkgSetting, disabledPkgSetting,
                        originalPkgSetting, realPkgName, parseFlags, scanFlags,
                        (pkg == mPlatformPackage), user);
                // 2.1 对这个请求进行扫描
                final ScanResult result = scanPackageOnlyLI(request, mFactoryTest, currentTime);
                
                // 2.2 若执行成功了, 则调用 commitScanResultsLocked
                if (result.success) {
                    commitScanResultsLocked(request, result);
                }
                ......
            } finally {
                .......
            }
        }
        return pkg;
    }
    
}
```
可以看到首先调用了 PackageManagerService.scanPackageChildLI
- 将 安装包信息 添加到 **mPackages** 中缓存
- 经过一系列分发, 传到了 scanPackageNewLI 方法
  - 若为共享应用, 调用 getSharedUserLPw 构建共享用户 ID 的描述 SharedUserSetting
    - 与解析备份文件中创建流程一致 
  - 将 Package 封装成为 ScanRequest 对象, 等待扫描
     - 调用 scanPackageOnlyLI 扫描这个 ScanRequest, 同时获取扫描结果 ScanResult
     - 调用 commitScanResultsLocked 处理扫描结果

关于创建共享用户描述我们在解析备份信息中已经分析过了, 这里就不再赘述了, 下面我们直接看看 PMS.scanPackageOnlyLI 扫描安装包信息的过程

### 一) 扫描安装包信息
```
public class PackageManagerService {
    
    ScanResult scanPackageOnlyLI(@NonNull ScanRequest request,
                                 boolean isUnderFactoryTest, long currentTime){
        // 创建应用的描述  PackageSetting
        PackageSetting pkgSetting = request.pkgSetting; 
        final SharedUserSetting sharedUserSetting = request.sharedUserSetting;
        final boolean createNewPackage = (pkgSetting == null);
        if (createNewPackage) {
            final String parentPackageName = (pkg.parentPackage != null)
                    ? pkg.parentPackage.packageName : null;
            ......
            // 调用了 createNewSetting 创建了应用的描述
            pkgSetting = Settings.createNewSetting(......);
        }
        
        // 将应用数据对象封装成 ScanResult 返回出去
        return new ScanResult(true, pkgSetting, changedAbiCodePath);
    }
    
}
```
PackageManagerService.scanPackageOnlyLI 方法主要是为了创建应用的描述对象 PackageSetting, 与上一篇文章中解析备份数据生成应用信息的方式类似, 这里就不再赘述了

创建好应用信息 PackageSetting 之后, 将它封装成 ScanResult 返回出去, 即扫描完成

接下来看看如何处理扫描的结果

### 二) 处理扫描结果
```
public class PackageManagerService {

    private void commitScanResultsLocked(@NonNull ScanRequest request, @NonNull ScanResult result)
            throws PackageManagerException {
        final PackageSetting pkgSetting = result.pkgSetting;
        ......
        // 我们知道, 若是新安装的应用, 在上面的扫描过程中会创建一个 PackageSetting 对象并且赋值给 result, 此时 request 中时没有这个对象的
        final boolean newPkgSettingCreated = (result.pkgSetting != request.pkgSetting);
        // 若是新的应用
        if (newPkgSettingCreated) {
            ......
            // 1. 调用 addUserToSettingLPw 为独立应用分配 ID
            mSettings.addUserToSettingLPw(pkgSetting);
            ......
        }
        if ((scanFlags & SCAN_CHECK_ONLY) != 0) {
            ......
        } else {
            // 2. 转发到该方法
            commitPackageSettings(pkg, oldPkg, pkgSetting, user, scanFlags,
                    (parseFlags & PackageParser.PARSE_CHATTY) != 0 /*chatty*/);
            ......
        }
    }
    
    // 缓存 APK 中解析出来的 Activity
    final ActivityIntentResolver mActivities =
            new ActivityIntentResolver();

    // 缓存 APK 中解析出来的 BroadcastReceiver
    final ActivityIntentResolver mReceivers =
            new ActivityIntentResolver();

    // 缓存 APK 中解析出来的 Service
    final ServiceIntentResolver mServices = new ServiceIntentResolver();

    // 缓存 APK 中解析出来的 ContentProvider
    final ProviderIntentResolver mProviders = new ProviderIntentResolver();
    
    private void commitPackageSettings(PackageParser.Package pkg,
                                       @Nullable PackageParser.Package oldPkg, PackageSetting pkgSetting, UserHandle user,
                                       final @ScanFlags int scanFlags, boolean chatty) {
        final String pkgName = pkg.packageName;
        ......
        synchronized (mPackages) {
            // 1. 将 ContentProvider 加入缓存
            int N = pkg.providers.size();
            int i;
            for (i = 0; i < N; i++) {
                PackageParser.Provider p = pkg.providers.get(i);
                ......
                mProviders.addProvider(p);
                ......
            }
            ......
            // 2. 将 Service 加入缓存
            N = pkg.services.size();
            for (i = 0; i < N; i++) {
                PackageParser.Service s = pkg.services.get(i);
                ......
                mServices.addService(s);
                .......
            }
            // 3. 静态注册的 BroadcastReceiver 加入缓存
            N = pkg.receivers.size();
            for (i = 0; i < N; i++) {
                PackageParser.Activity a = pkg.receivers.get(i);
                mReceivers.addActivity(a, "receiver");
            }
            // 4. 将 Activity 加入缓存
            N = pkg.activities.size();
            for (i = 0; i < N; i++) {
                PackageParser.Activity a = pkg.activities.get(i);
                mActivities.addActivity(a, "activity");
            }
         }
    }
    
}
```
上面我们看到了为共享应用分配 UserID, 但是独立应用还没有处理, 所以 PMS 的 commitScanResultsLocked 方法首先为独立应用分配了 UserID, 然后调用了 commitPackageSettings 将安装包内的四大组件信息发布到 PMS 的缓存中

至此这个应用便发布到 PMS 中了

### 回顾
将安装好的应用发布到 PMS 的流程如下
- 创建共享应用描述 SharedUserSetting, 缓存到 mPackages 中
- 扫描安装包信息
  - 创建应用描述 PackageSetting, 缓存到 mPackages 中
- 提交扫描结果
  - 将安装包内的 Android 四大组件发布到 PMS

## 总结
**扫描应用程序安装目录**
- **构建应用摘要对象 PackageLite**
  - 构建 base.apk 摘要对象 ApkLite 
    - 创建 AndroidManifest.xml 解析器
    - 解析 AndroidMainfest 中的概要, 封装到 ApkLite 中
    - 将 ApkLite 存入 PackageLite 返回
- **获取应用的 AssetManager**
  - 创建描述 apk 资源的 ApkAsset 对象
  - 创建 AssetManager 
  - 将 ApkAsset 存入 AssetManager
- **构建整个应用信息的对象 Package**
  - 获取 AndroidManifest.xml 的解析器
  - 解析标签内四大组件的信息
  - 将解析到的数据封装到 Package 中

**发布应用数据**
- 创建共享应用描述 SharedUserSetting, 缓存到 mPackages 中
- 扫描安装包信息
  - 创建应用描述 PackageSetting, 缓存到 mPackages 中
- 提交扫描结果
  - 将安装包内的 Android 四大组件发布到 PMS

PKMS 走到这里, 我们已经安装的程序就读到内存中了, PKMS 也就可以很好的向外界提供 App 信息了