---
title: Android 系统架构 —— PKMS 的启动 之 备份文件的解析
permalink: android-source/pkms-launch1
key: android-source-pkms-launch1
tags: AndroidFramework
---

## 前言
在 SystemService 进程启动时, 会将 PackageManagerService 启动起来, 这是一个应用资源管理的服务
```
public class PackageManagerService {

    public static PackageManagerService main(Context context, Installer installer,
            boolean factoryTest, boolean onlyCore) {
        ......
        // 创建了 PackageManagerService 对象, 并且将它添加到 ServiceManager, 用于对外界提供功能支持
        PackageManagerService m = new PackageManagerService(context, installer,
                factoryTest, onlyCore);
        ......
        ServiceManager.addService("package", m);
        // 返回了一个 PackageManagerService 对象
        return m;
    }
    
}
```
因此我们从 PackageManagerService 的构造函数来探查 PKMS 的启动流程

<!--more-->

## PKMS 的构造
```
public class PackageManagerService {

    public static PackageManagerService main(Context context, Installer installer,
            boolean factoryTest, boolean onlyCore) {
        ......
        // 创建了 PackageManagerService 对象, 并且将它添加到 ServiceManager, 用于对外界提供功能支持
        PackageManagerService m = new PackageManagerService(context, installer,
                factoryTest, onlyCore);
        ......
        ServiceManager.addService("package", m);
        // 返回了一个 PackageManagerService 对象
        return m;
    }
    
    /**
     * 描述 /data/app 目录, 其中保存的是由用户自己安装的应用程序
     */
    private static final File sAppInstallDir =
            new File(Environment.getDataDirectory(), "app");
    /**
     * 描述 /data/app-private 目录, 其中保存的是受 DRM 保护的私有应用程序
     */
    private static final File sDrmAppPrivateInstallDir =
            new File(Environment.getDataDirectory(), "app-private");
    
    public PackageManagerService(Context context, Installer installer,
            boolean factoryTest, boolean onlyCore) {
        
        // 创建一个 Setting 对象, 用来管理应用程序的安装
        synchronized (mInstallLock) {
            synchronized (mPackages) {
                ......
                mSettings = new Settings(mPermissionManager.getPermissionSettings(), mPackages);
            }
        }
        ......
        synchronized (mInstallLock) {
            // writer
            synchronized (mPackages) {
                // 1. 读取上一次 App 的安装信息
                mFirstBoot = !mSettings.readLPw(sUserManager.getUsers(false));
                // 2. 扫描相关文件下的应用程序
                // 2.1 描述 /system/framework 目录, 其中保存的是资源型应用程序
                File frameworkDir = new File(Environment.getRootDirectory(), "framework");
                // 扫描 /system/framework 下的应用
                scanDirTracedLI(frameworkDir,
                        mDefParseFlags
                                | PackageParser.PARSE_IS_SYSTEM_DIR,
                        scanFlags
                                | SCAN_NO_DEX
                                | SCAN_AS_SYSTEM
                                | SCAN_AS_PRIVILEGED,
                        0);
                
                // 2.2 描述 /system/app 目录, 其中保存的是系统自带的应用
                final File systemAppDir = new File(Environment.getRootDirectory(), "app");
                // 扫描系统自带应用
                scanDirTracedLI(systemAppDir,
                        mDefParseFlags
                                | PackageParser.PARSE_IS_SYSTEM_DIR,
                        scanFlags
                                | SCAN_AS_SYSTEM,
                        0);
                        
                .......
                
                if (!mOnlyCore) {
                    // 2.3 扫描用户安装应用
                    scanDirTracedLI(sAppInstallDir, 0, scanFlags | SCAN_REQUIRE_KNOWN, 0);
                    // 2.4 受 DRM 保护的私有应用
                    scanDirTracedLI(sDrmAppPrivateInstallDir, mDefParseFlags
                                    | PackageParser.PARSE_FORWARD_LOCK,
                            scanFlags | SCAN_REQUIRE_KNOWN, 0);
                    ......
                }
                ......
                // 3. 将应用程序信息备份到配置文件中
                mSettings.writeLPr();
            }
            
        }
    }
    
}
```
好的, 可以看到在 PackageManagerService 中, 主要做了三件事情
- 通过调用 **Settings.readLPw** 获取上一次启动时记录的应用信息
- 调用  **PackageManagerService.scanDirTracedLI** 扫描指定目录下的应用程序
  - /system/framework: 系统资源型应用
  - /system/app: 系统自带的应用
  - /data/app: 用户自己安装的应用程序
  - /data/app-private: 受 DRM 保护的私有应用程序
- 调用 Settings.writeLPr 将应用程序的信息备份到配置文件中
  - 以便下次再安装应用程序时, 可以将需要保持一致的信息恢复过来

关于 PMS 的启动, 我们主要关注 **读取备份信息** 和 **扫描应用程序** 的操作, 由于它们都比较复杂, 因此我们分成两篇文章来阐述

**这篇我们主要关注读取备份信息的操作**

## 一. 读取备份信息
```
package com.android.server.pm;

public final class Settings {

    private final File mSettingsFilename;
    private final File mBackupSettingsFilename;
    
    // 构造函数
    Settings(File dataDir, PermissionSettings permission, Object lock) {
        ......
        mSystemDir = new File(dataDir, "system");
        mSystemDir.mkdirs();
        ......
        // 描述 /data/system/packages.xml, 用于保存上一次应用程序安装信息
        mSettingsFilename = new File(mSystemDir, "packages.xml");
        // 描述 /data/system/packages-backup.xml, 用于备份 packages.xml 文件
        mBackupSettingsFilename = new File(mSystemDir, "packages-backup.xml");
        ......
    }
```
可以看到在 Settings 的构造函数中, 首先初始化了两个文件对象
- /data/system/**packages.xml**, 用于保存上一次应用程序安装信息
- /data/system/**packages-backup.xml**, 用于备份 packages.xml 文件

### 一) package.xml 文件结构
下面是笔者从已经 root 的手机中拷贝出的 package.xml 文件代码, 它的组成结构如下
```
<packages>
    <version sdkVersion="26" databaseVersion="3" fingerprint="samsung/a9y18qltezc/a9y18qltechn:8.0.0/R16NW/A9200ZCU1ARKK:user/release-keys" />
    <version volumeUuid="primary_physical" sdkVersion="0" databaseVersion="0" />
    <permission-trees>
        <item name="com.google.android.googleapps.permission.GOOGLE_AUTH" package="com.google.android.gsf" />
        <item name="com.oppo.com.eg.android.AlipayGphone" package="com.eg.android.AlipayGphone" />
    </permission-trees>
    <permissions>
        <item name="com.samsung.knox.securefolder.switcher.knoxusage.knoxusage.KNOX_USAGE_PROVIDER_READ" package="com.samsung.knox.securefolder" protection="18" />
        <item name="android.permission.REAL_GET_TASKS" package="android" protection="18" />
        ......
    </permissions>
    ......
    <!--普通的应用-->
    <package name="com.sharry.media.app" 
        codePath="/data/app/com.sharry.media.app-ScNNpBlTPRKrwccwzNU4LQ=="
        nativeLibraryPath="/data/app/com.sharry.media.app-ScNNpBlTPRKrwccwzNU4LQ==/lib"
        primaryCpuAbi="armeabi-v7a" 
        ......
        version="0"
        userId="10240">
        <sigs count="1">
            <cert index="35" key="......" />
        </sigs>
        <perms>
            <item name="android.permission.INTERNET" granted="true" flags="0" />
            <item name="android.permission.ACCESS_NETWORK_STATE" granted="true" flags="0" />
        </perms>
        <proper-signing-keyset identifier="92" />
    </package>
    ......
    <!--sharedUserId 的应用-->
    <package name="com.android.systemui" 
        codePath="/system/priv-app/SystemUI" 
        nativeLibraryPath="/system/priv-app/SystemUI/lib" 
        primaryCpuAbi="arm64-v8a" 
        ......
        version="20171030" 
        sharedUserId="10002" 
        isOrphaned="true">
        <sigs count="1">
            <cert index="0" />
        </sigs>
        <proper-signing-keyset identifier="2" />
    </package>
    <shared-user name="android.uid.systemui" userId="10002">
        <sigs count="1">
            <cert index="0" />
        </sigs>
        <perms>......</perms>
    </shared-user>
    
</packages>
```
可以看到备份的内容还是十分丰富的, 我们需要关注的标签如下
- <packag>: 描述一个已安装的应用
  - name: 包名 
  - userId: 应用的 Linux Id
  - sharedUserId: 应用的共享 Linux Id
- <shared-user>: 描述一个共享的 Linux Id
   - **如果两个应用配置了相同的 sharedUserId, 他们之间可以访问彼此沙箱中的数据**
   - 这个属性笔者在开发过程中解除的比较少, 关于更详细的介绍可以查阅[这篇文章](https://blog.csdn.net/jiangwei0910410003/article/details/51316688)
- <perms>: 描述应用所需的权限

了解 package.xml 的组成结构, 下面我们看看 Settings.readLPw 是如何解析 package 的

### 二) package.xml 的解析
```
public final class Settings {

    boolean readLPw(@NonNull List<UserInfo> users) {
        FileInputStream str = null;
        // 1. 创建备份文件的输入流
        // 检查备份的文件是否存在, 若存在则直接创建输入流
        if (mBackupSettingsFilename.exists()) {
            try {
                str = new FileInputStream(mBackupSettingsFilename);
                .......
            } catch (java.io.IOException e) {
                // We'll try for the normal settings file.
            }
        }
        ......
         try {
            // 通过 mSettingsFilename 创建文夹输入流
            if (str == null) {
                ......
                str = new FileInputStream(mSettingsFilename);
            }
            // 2. 用于解析 XML 文件
            XmlPullParser parser = Xml.newPullParser();
            parser.setInput(str, StandardCharsets.UTF_8.name());
            
            // 找寻 XML 的起始标签
            int type;
            while ((type = parser.next()) != XmlPullParser.START_TAG
                    && type != XmlPullParser.END_DOCUMENT) {
                ;
            }
            ......
            // 解析 XML 文件
            int outerDepth = parser.getDepth();
            while ((type = parser.next()) != XmlPullParser.END_DOCUMENT
                    && (type != XmlPullParser.END_TAG || parser.getDepth() > outerDepth)) {
                ......
                // 2.1 解析 package 标签
                String tagName = parser.getName();
                if (tagName.equals("package")) {
                    readPackageLPw(parser);
                } 
                .......
                // 2.2 解析 shared-user 元素
                } else if (tagName.equals("shared-user")) {
                    readSharedUserLPw(parser);
                } 
                .......
            }
            str.close();
        }
        
        ......// 处理共享应用程序
        final int N = mPendingPackages.size();
        for (int i = 0; i < N; i++) {
            // 3.1 获取共享应用的描述
            final PackageSetting p = mPendingPackages.get(i);
            
            // 3.2 获取目标 ID 对应的应用描述 
            final int sharedUserId = p.getSharedUserId();
            
            // 3.3 根据 sharedUserId 从 mUserId 缓存中 SharedUserSetting
            final Object idObj = getUserIdLPr(sharedUserId);
            if (idObj instanceof SharedUserSetting) {
                // 3.3.1 将 userId 保存到共享应用的 appId 中
                final SharedUserSetting sharedUser = (SharedUserSetting) idObj;
                p.sharedUser = sharedUser;
                p.appId = sharedUser.userId;
                
                // 3.3.2 调用 addPackageSettingLPw 将共享应用与共享 ID 关联
                addPackageSettingLPw(p, sharedUser);
            } 
            ......
        }
        mPendingPackages.clear();
    }
    
}
```
Settings.readLPw 这个文件中, 主要做了如下几件事情
- 通过 mBackupSettingsFilename 或者 mSettingsFilename 建立文件输入流
- 解析 xml 流数据
  - **readPackageLPw**: 解析 <package> 标签
  - **readSharedUserLPw**: 解析 <shared-user> 标签
- 缓存共享应用信息

接下来我们一一查看

先看看解析 <package> 标签的操作

## 二. 解析 <package> 标签
该方法用于解析 package 标签
```
public final class Settings {

    public static final String ATTR_NAME = "name";
    private final ArrayList<PackageSetting> mPendingPackages = new ArrayList<>();

    // 解析 <package/> 标签
    private void readPackageLPw(XmlPullParser parser) throws XmlPullParserException, IOException {
        String name = null;
        ......
        String idStr = null;
        String sharedIdStr = null;
        ......
        try {
            // 1. 解析相关元素
            // 1.1 解析 "name" 元素, 用于描述上一次安装应用的包名
            name = parser.getAttributeValue(null, ATTR_NAME);
            ......
            // 1.2 解析 "userId" 元素, 描述这个应用程序使用的独立 Linux 用户 ID
            idStr = parser.getAttributeValue(null, "userId");
            // 1.3 解析 "sharedUserId" 元素, 描述这个应用使用的共享 Linux 用户 ID
            sharedIdStr = parser.getAttributeValue(null, "sharedUserId");
            ......
            // 2. 处理解析到的元素
            // 将 idStr 转为整型 userId
            final int userId = idStr != null ? Integer.parseInt(idStr) : 0;
            // 将 sharedIdStr 转为整型 sharedUserId
            final int sharedUserId = sharedIdStr != null ? Integer.parseInt(sharedIdStr) : 0;
            // 2.1 处理 name 不存在的情况
            if (name == null) {
                ......// 此处扔出异常
            } 
            ......
            else if (userId > 0) {
                // 2.2 调用 addPackageLPw 缓存解析到的独立应用的信息
                packageSetting = addPackageLPw(name.intern(), realName, new File(codePathStr),
                        new File(resourcePathStr), legacyNativeLibraryPathStr, primaryCpuAbiString,
                        secondaryCpuAbiString, cpuAbiOverrideString, userId, versionCode, pkgFlags,
                        pkgPrivateFlags, parentPackageName, null /*childPackageNames*/,
                        null /*usesStaticLibraries*/, null /*usesStaticLibraryVersions*/);
                ......
            } else if (sharedIdStr != null) {
                if (sharedUserId > 0) {
                    //  2.3 构建共享应用的 PackageSetting 对象, 并且添加到 mPendingPackages 中缓存
                    packageSetting = new PackageSetting(name.intern(), realName, new File(
                            codePathStr), new File(resourcePathStr), legacyNativeLibraryPathStr,
                            primaryCpuAbiString, secondaryCpuAbiString, cpuAbiOverrideString,
                            versionCode, pkgFlags, pkgPrivateFlags, parentPackageName,
                            null /*childPackageNames*/, sharedUserId,
                            null /*usesStaticLibraries*/, null /*usesStaticLibraryVersions*/);
                    ......
                    // 缓存到 mPendingPackages
                    mPendingPackages.add(packageSetting);
                    ......
                } 
                ......
            } else {
                ......
            }
        } catch (NumberFormatException e) {
            ......
        }
        ......
    }
    ......
}
```
Settings.readLPw 中主要做了如下几件事情
- **解析 <package> 标签中的元素**
  -  解析 "name" 元素, 用于应用的包名
  -  解析 "userId" 元素, 描述这个应用程序使用的独立 Linux 用户 ID
  -  解析 "sharedUserId" 元素, 描述这个应用使用的共享 Linux 用户 ID
- **缓存解析到的数据**
  - 若为独立应用: 调用 addPackageLPw 保存应用信息
  - 若为共享应用: 创建 PackageSetting 对象, 并缓存到 mPendingPackages, 表示延时处理
    
可以看到, 这里只对独立的应用程序进行操作, 对于共享应用, 则只是添加到了 mPendingPackages 中

我们从 package.xml 的结构中可以知道, 一个共享应用程序的 id 会使用一个 <shared-user> 标签描述, 但此时我们只解析了 <package> 标签, 并未解析 <shared-user> 标签, 因此关于 mPendingPackages 中共享应用的缓存我们到后面再去分析

这里先看看独立应用信息的缓存过程

### 一) 缓存独立应用信息
```
public final class Settings {

    final ArrayMap<String, PackageSetting> mPackages = new ArrayMap<>();
      
    PackageSetting addPackageLPw(String name, String realName, File codePath, File resourcePath, .... int uid, ......) {
        // 1. 尝试从缓存中读取 name 对应的 PackageSetting 信息
        PackageSetting p = mPackages.get(name);
        if (p != null) {
            // 1.1 若这个对象的 appId 与 uid 相等, 则说明这个缓存是有效的
            if (p.appId == uid) {
                return p;
            }
            // 1.2 不相等说明缓存无效......
            return null;
        }
        // 2. 构建对象
        p = new PackageSetting(name, realName, codePath, resourcePath,......, 0 /*userId*/......);
        // 2.1 给这个 appId 赋值
        p.appId = uid;
        // 2.2 调用 addUserIdLPw, 通知系统将这个 uid  分配给包名为 name 的应用
        if (addUserIdLPw(uid, p, name)) {
            // 2.3 添加到缓存中
            mPackages.put(name, p);
            return p;
        }
        return null;
    }
    
}
```
addPackageLPw 优先从 mPackage 缓存中根据包名读取应用的缓存对象 PackageSetting, 若不存在则进行如下操作
- 首先通过 addUserIdLPw 向系统申请从 <package> 中解析到的 uid
- 然后再将这个 PackageSetting 对象保存到 mPackages 中

流程上清晰了, 下面我们再了解一下 addUserIdLPw 申请 uid 的

### 二) 申请 uid
```
public final class Settings {

    // 表示应用程序可分配的 Linux ID 的范围 [10000, 19999]
    public static final int Process.FIRST_APPLICATION_UID = 10000;
    public static final int Process.LAST_APPLICATION_UID = 19999;
    
    // 缓存分配给应用程序使用的 Linux ID
    private final ArrayList<Object> mUserIds = new ArrayList<Object>();
    // 缓存分配给 特权用户使用的 Linux 用户 ID
    private final SparseArray<Object> mOtherUserIds =
            new SparseArray<Object>();

    private boolean addUserIdLPw(int uid, Object obj, Object name) {
        // Case1: 若这个 uid 大于最后一个可用的应用 ID 则直接操作失败
        if (uid > Process.LAST_APPLICATION_UID) {
            return false;
        }
        // Case2: uid 比第一个可用的应用 ID 大
        if (uid >= Process.FIRST_APPLICATION_UID) {
            int N = mUserIds.size();
            // 2.1 计算这个 uid 要插入的位置
            final int index = uid - Process.FIRST_APPLICATION_UID;
            // 2.2 比 mUserId 缓存的数据大, 则让缓存自增到可以插入 index 的状态
            while (index >= N) {
                mUserIds.add(null);
                N++;
            }
            // 2.3 表示这个 uid 已经分配给它应用了, 分配失败
            if (mUserIds.get(index) != null) {
                // ....... 汇报错误信息
                return false;
            }
            // 2.4 添加到缓存中
            mUserIds.set(index, obj);
        } else {
            // Case3: uid 比第一个可用的应用 ID 小
            // 该 uid 已经被分配过了
            if (mOtherUserIds.get(uid) != null) {
                // ....汇报异常
                return false;
            }
            // 添加到 mOtherUserIds 的缓存中
            mOtherUserIds.put(uid, obj);
        }
        return true;
    }
}
```
可以看到有两个阈值, FIRST_APPLICATION_UID 和 LAST_APPLICATION_UID, 它表示应用程序的 Linux ID 的范围 [10000, 19999]
- 若超过分配的 uid 超过了 19999, 则直接分配失败
- 在 10000 和 19999 之间, 则插入到 mUserIds 中
  - 若已经被分配过, 则返回失败 
- 比 10000 小, 则插入到 mOtherUserIds 中
  - 若已经被分配过, 则返回失败

到这里 package.xml 中解析并缓存 <package/> 标签信息就结束了, 下面我们看看 <shared-user> 标签的解析过程

## 三. 解析 <shared-user> 标签
先回顾一下 <shared-user> 标签的定义

```
......
    <shared-user 
        name="android.uid.systemui" 
        userId="10002">
        ......
    </shared-user>
......
```

下面我们看看 Settings.readSharedUserLPw 是如何解析它的

```
public final class Settings {
    
    // 解析 <shared-user>...</shared-user>
    private void readSharedUserLPw(XmlPullParser parser) throws XmlPullParserException,IOException {
        String name = null;
        String idStr = null;
        int pkgFlags = 0;
        SharedUserSetting su = null;
        try {
            // 1. 解析共享 Linux 用户的包名
            name = parser.getAttributeValue(null, ATTR_NAME);
            // 2. 解析共享 Linux 用户的 userId.
            idStr = parser.getAttributeValue(null, "userId");
            int userId = idStr != null ? Integer.parseInt(idStr) : 0;
            // 3. "system" 元素描述这个共享的 Linux 用户的 ID 是否是分配给一个系统类型的应用程序使用
            if ("true".equals(parser.getAttributeValue(null, "system"))) {
                pkgFlags |= ApplicationInfo.FLAG_SYSTEM;
            }
            if (name == null) {
                ......
            } else if (userId == 0) {
                ......
            } else {
                // 4. 将这个 Linux 共享的 userId 与该应用关联起来
                if ((su = addSharedUserLPw(name.intern(), userId, pkgFlags, pkgPrivateFlags))
                        == null) {
                    // ...... 通知保存失败了
                }
            }
        } catch (NumberFormatException e) {
            .......
        }
        .......
    }
}
```
可以看到, 上面的解析操作与解析 <package> 标签很类似, 下面我们看看 addSharedUserLPw 方法做了哪些操作
```
public final class Settings {

    final ArrayMap<String, SharedUserSetting> mSharedUsers =
            new ArrayMap<String, SharedUserSetting>();
    
    SharedUserSetting addSharedUserLPw(String name, int uid, int pkgFlags, int pkgPrivateFlags) {
        // 尝试从 mSharedUsers 的缓存中, 获取实例对象
        SharedUserSetting s = mSharedUsers.get(name);
        if (s != null) {
            // 若 userId 与当前要关联的 uid 相等, 则说明这个 name 已经与这个 Linux 共享 ID 关联起来了
            if (s.userId == uid) {
                return s;
            }
            // ..... 处理关联失败
            return null;
        }
        // 创建新的对象
        s = new SharedUserSetting(name, pkgFlags, pkgPrivateFlags);
        s.userId = uid;
        // 将这个对象保存到系统中
        if (addUserIdLPw(uid, s, name)) {
            // 保存成功则添加缓存
            mSharedUsers.put(name, s);
            return s;
        }
        return null;
    }
}
```
可以看到, Settings.addSharedUserLPw 中的操作与 Settings.addPackageLPw 基本一致, 不同的是**共享用户 ID 使用 SharedUserSetting 来描述**, userId 申请成功之后, 会添加到 mSharedUser 散列表中缓存

好的, 至此 <shared-user> 标签中的数据解析就完成了

在对 <package> 标签解析的过程中, 我们知道它会将使用共享应用的信息添加到 mPendingPackages 暂存, 并没有真正的将其数据缓存起来, 现在 <shared-user> 标签已经解析好了, 接下来我们便回到 Settings.readLPw 中看看它对共享应用的缓存操作

## 四. 缓存共享应用信息
```
package com.android.server.pm;

public final class Settings {

    private final ArrayList<PackageSetting> mPendingPackages = new ArrayList<>();

    boolean readLPw(@NonNull List<UserInfo> users) {
        // 1. 创建输入流, 已分析
        ......
        // 2. 解析 package.xml/package-backup.xm 文件中的信息, 已分析
        ......
        // 3. 缓存共享应用信息
        final int N = mPendingPackages.size();
        for (int i = 0; i < N; i++) {
            // 3.1 获取共享应用的描述
            final PackageSetting p = mPendingPackages.get(i);
            
            // 3.2 获取目标 ID 对应的应用描述 
            final int sharedUserId = p.getSharedUserId();
            
            // 3.3 根据 sharedUserId 从 mUserId 缓存中 SharedUserSetting
            final Object idObj = getUserIdLPr(sharedUserId);
            if (idObj instanceof SharedUserSetting) {
                // 3.3.1 将 userId 保存到共享应用的 appId 中
                final SharedUserSetting sharedUser = (SharedUserSetting) idObj;
                p.sharedUser = sharedUser;
                p.appId = sharedUser.userId;
                
                // 3.3.2 调用 addPackageSettingLPw 将共享应用与共享 ID 关联
                addPackageSettingLPw(p, sharedUser);
            } 
            ......
        }
        mPendingPackages.clear();

        ......
    }
    
    
}
```
好的, 这里可以看到在 readLPw 中, 遍历了 mPendingSettings 集合, 从 mUserId 中根据 sharedUserId 找到对应的 SharedUserSetting 对象, 然后调用 addPackageSettingLPw 将这个 PackageSetting 添加到 SharedUserSetting 中缓存

下面我们看看他们关联的过程
```
public final class Settings {
    final ArrayMap<String, PackageSetting> mPackages = new ArrayMap<>();
    
    private void addPackageSettingLPw(PackageSetting p, SharedUserSetting sharedUser) {
        // 1. 将 共享应用 添加到 mPackages 缓存中
        mPackages.put(p.name, p);
        // 2. 将 共享应用 添加到 共享应用 ID 的链表中
        if (sharedUser != null) {
            if (p.sharedUser != null && p.sharedUser != sharedUser) {
                ......
                // 2.1 从原先的共享用户 Id 维护的共享应用列表中, 将自身移除
                p.sharedUser.removePackage(p);
            } else if (p.appId != sharedUser.userId) {
                .....
            }
            // 2.2 在 sharedUser 共享列表中添加这个 PackageSetting
            sharedUser.addPackage(p);
            p.sharedUser = sharedUser;
            p.appId = sharedUser.userId;
        }
        .......
    }
    
}
```
好的, 可以看到经过解析 package.xml 文件的 <shared-user> 标签之后, 支持共享的 Linux Id 用户会被缓存起来, 因此在 Settings.readLPw 中的操作便如下
- 遍历需要与其他应用程序共享一个 Linux Id 的应用 p
- 调用 getUserIdLPr 从缓存中获取 p 这个用户目标的共享应用 idObj
- 若 idObj 为 SharedUserSetting, 则让 p 与这个 idObj 相互关联

## 总结
至此获取上一次启动时的应用程序信息的所有流程就已经结束了, 这里做个总结
- 解析备份的文件 package.xml/package-backup.xml
- 解析文件的 <package> 标签
  - 缓存独立应用
    - 将独立应用添加到 mUserIds/mOtherUserIds 中
    - 将独立应用缓存到 mPackages 中
  - 将需要使用共享 Linux Id 的应用添加到 mPendingPackages 中缓存
- 解析文件的 <shared-user> 标签
  - 缓存共享用户 ID: 将共享应用 ID 缓存到 mUserIds/mOtherUserIds 中
- 缓存共享应用
  - 将共享应用缓存到 mPackages 中
  - 将共享应用添加到共享应用 ID 的链表中

![PMS 备份数据缓存结构](https://i.loli.net/2019/11/15/JapznNI9AMqrFKx.png)

## 参考文献
- https://blog.csdn.net/jiangwei0910410003/article/details/51316688