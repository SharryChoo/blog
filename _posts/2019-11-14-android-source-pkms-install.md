---
title: Android 系统架构 —— PKMS 之 应用的安装流程
permalink: android-source/pkms-install
key: android-source-pkms-install
tags: AndroidFramework
sidebar:
  nav: android-source
---

## 前言
通过前面两篇文章的学习, 我们知晓了 PKMS 的启动过程
- 解析备份文件
- 扫描已安装的应用程序

不过 PKMS 除了负责管理已安装的应用程序之外, 还负责应用程序的安装操作, 我们 Android 端, 整个应用程序的安装主要有两种方式:
- **通过 Session 发起(adb, 手动点击安装)**
  - 提前将安装包拷贝到 "data/app/vmdl${sessionId}.tmp/" 目录下 
  - 再通过 Session commit, 通知 PKMS 的 PKMS.installStage 进行安装
- **通过系统的软件商店自动发起**
  - 直接通过 PKMS 的 PKMS.installStage 进行安装

也就是说通过**非系统软件商店的安装要多一步通过 Session 提前拷贝的过程**, 这里我们为了更全面了解应用安装, 选择有 Session 通信的进行分析

<!--more-->

## 一. Session 发起
当我们下载了一个 apk, 点击进行安装时, 会跳转到 [PackageInstallerActivity](http://androidxref.com/9.0.0_r3/xref/packages/apps/PackageInstaller/src/com/android/packageinstaller/PackageInstallerActivity.java) 这个 Activity, 厂商可以为这个 Activity 进行定制和修改, 不过万变不离其宗, 我们看看它是如何安装一个 apk 的

```
public class PackageInstallerActivity extends OverlayTouchActivity implements OnClickListener {
    
    private Uri mPackageURI;
  
    @Override
    protected void onCreate(Bundle icicle) {
        super.onCreate(null);
        final Uri packageUri;
        if (PackageInstaller.ACTION_CONFIRM_PERMISSIONS.equals(intent.getAction())) {
            ......
        } else {
            mSessionId = -1;
            // 1. 获取发起者传来的要待安装文件的 URI
            packageUri = intent.getData();
            ......
        }
        ......
        boolean wasSetUp = processPackageUri(packageUri);
        // 初始化视图
        bindUi(R.layout.install_confirm, false);
    }
    
    private boolean processPackageUri(final Uri packageUri) {
        mPackageURI = packageUri;
        ......
        return true;
    }
    
    private Button mOk;
    
    private void bindUi(int layout, boolean enableOk) {
        setContentView(layout);
        ......
        mOk = (Button) findViewById(R.id.ok_button);
        ......
        mOk.setEnabled(enableOk);
        ......
    }

    public void onClick(View v) {
        if (v == mOk) {
            if (mOk.isEnabled()) {
                if (mOkCanInstall || mScrollView == null) {
                    if (mSessionId != -1) {
                        ......
                    } else {
                        // 2. 点击确认启动安装
                        startInstall();
                    }
                } else {
                    ......
                }
            }
        } else if (v == mCancel) {
            ......
        }
    }
    
    private void startInstall() {
        // 3. 跳转到应用安装页面
        Intent newIntent = new Intent();
        newIntent.putExtra(PackageUtil.INTENT_ATTR_APPLICATION_INFO,
                mPkgInfo.applicationInfo);
        // 注入安装包 uri
        newIntent.setData(mPackageURI);
        // 跳转到 InstallInstalling 页面执行应用安装操作
        newIntent.setClass(this, InstallInstalling.class);
        ......
        startActivity(newIntent);
        finish();
    }
    
}
```
可以看到 PackageInstallerActivity 在 onCreate 的方法中获取了应用安装包的 URI, 当我们点击确定的时候, 调用了 startInstall 这个方法, 它将安装包 URI 作为参数注入 intent 跳转到了 InstallInstalling 页面

接下来我们看看 [InstallInstalling](http://androidxref.com/9.0.0_r3/xref/packages/apps/PackageInstaller/src/com/android/packageinstaller/InstallInstalling.java) 又是如何执行安装操作的
```
public class InstallInstalling extends Activity {
    
    /** URI of package to install */
    private Uri mPackageURI;
    
    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.install_installing);
        ......
        mPackageURI = getIntent().getData();
        // 若 URI 为 package 开头, 意为更新应用
        if ("package".equals(mPackageURI.getScheme())) {
            ......
        } 
        // 安装应用
        else {
            final File sourceFile = new File(mPackageURI.getPath());
            ......
            if (savedInstanceState != null) {
                ......
            } else {
                // 1. 获取 SessionId
                // 1.1 构建请求参数
                PackageInstaller.SessionParams params = new PackageInstaller.SessionParams(
                        PackageInstaller.SessionParams.MODE_FULL_INSTALL);
                ......
                try {
                    // 1.2 将 params 传入, 创建一个与 PKMS 安装服务交互的 SessionId
                    mSessionId = getPackageManager().getPackageInstaller().createSession(params);
                } catch (IOException e) {
                    ......
                }
            }
            ......
            mSessionCallback = new InstallSessionCallback();
        }
    }
     
    ......

    private InstallingAsyncTask mInstallingTask;
    
    @Override
    protected void onResume() {
        super.onResume();
        // 
        if (mInstallingTask == null) {
            // 获取 PackageInstaller
            PackageInstaller installer = getPackageManager().getPackageInstaller();
            // 获取会话信息
            PackageInstaller.SessionInfo sessionInfo = installer.getSessionInfo(mSessionId);
            // 2. 通过 InstallingAsyncTask 将文件发送给 PackageInstallerSession
            if (sessionInfo != null && !sessionInfo.isActive()) {
                mInstallingTask = new InstallingAsyncTask();
                mInstallingTask.execute();
            } else {
                ......
            }
        }
    }
    
}
```
好的, 可以看到 InstallInstalling 的生命周期回调中所做的事情如下
- 通过 getPackageManager().getPackageInstaller().createSession(...) 获取一个 SessionId
- 通过 InstallingAsyncTask 将安装包发送给 PackageInstallerSession

首先我们看看获取 SessionId 的动作, **PackageInstaller 是 PackageInstallerService 在客户端的 Binder 代理对象**, PackageInstallerService 用于维护整个系统的应用安装的任务, 我们直接看看它的实现类 PackageInstallerService 的实现逻辑

### 一) 获取 SessionId
```
public class PackageInstallerService extends IPackageInstaller.Stub {
    
     @Override
    public int createSession(SessionParams params, String installerPackageName, int userId) {
        try {
            return createSessionInternal(params, installerPackageName, userId);
        } catch (IOException e) {
            ......
        }
    }

    // 描述一个应用安装任务集合
    private final SparseArray<PackageInstallerSession> mSessions = new SparseArray<>();
    /** Upper bound on number of active sessions for a UID */
    private static final long MAX_ACTIVE_SESSIONS = 1024;
    /** Upper bound on number of historical sessions for a UID */
    private static final long MAX_HISTORICAL_SESSIONS = 1048576;
    
    private int createSessionInternal(SessionParams params, String installerPackageName, int userId)
            throws IOException {
        // 描述应用安装发起进程
        final int callingUid = Binder.getCallingUid();
        
        ......// 执行一堆验参
        
        // 创建 SessionId
        final int sessionId;
        final PackageInstallerSession session;
        synchronized (mSessions) {
            // 一个客户端正在安装的应用不能超过 1024
            final int activeCount = getSessionCount(mSessions, callingUid);
            if (activeCount >= MAX_ACTIVE_SESSIONS) {
                throw new IllegalStateException(
                        "Too many active sessions for UID " + callingUid);
            }
            // 一个客户端应用安装历史不能炒作 1048576
            final int historicalCount = mHistoricalSessionsByInstaller.get(callingUid);
            if (historicalCount >= MAX_HISTORICAL_SESSIONS) {
                ......
            }
            // 1. 为客户端发起的安装任务, 分配一个 SessionId
            sessionId = allocateSessionIdLocked();
        }
        ......
        // 2. 创建一个 stageDir, 用于接收客户端要安装的 apk
        File stageDir = null;
        ......
        if ((params.installFlags & PackageManager.INSTALL_INTERNAL) != 0) {
            // 安装到内部存储区, 创建一个路径, 用于存储客户端的 apk
            final boolean isInstant = (params.installFlags & PackageManager.INSTALL_INSTANT_APP) != 0;
            // 构建一个路径
            stageDir = buildStageDir(params.volumeUuid, sessionId, isInstant);
        } else {
            ......
        }
        // 3. 创建一个 PackageInstallerSession, 描述安装任务
        session = new PackageInstallerSession(mInternalCallback, mContext, mPm,
                mInstallThread.getLooper(), sessionId, userId, installerPackageName, callingUid,
                params, createdMillis, stageDir, stageCid, false, false);
        // 投入缓存
        synchronized (mSessions) {
            mSessions.put(sessionId, session);
        }
        ......
        return sessionId;
    }
    
    
    private File buildStagingDir(String volumeUuid, boolean isEphemeral) {
        return Environment.getDataAppDirectory(volumeUuid);
    }

    private File buildStageDir(String volumeUuid, int sessionId, boolean isEphemeral) {
        final File stagingDir = buildStagingDir(volumeUuid, isEphemeral);
        return new File(stagingDir, "vmdl" + sessionId + ".tmp");
    }

}
```
从 PackageInstallerService 的 openSession 实现中还是能够看到很多有意思的信息, 比如一个客户端能够发起安装的最大数量为 1024, 它的历史安装任务不能超过 1048576 等, 不过其中我们需要重点关注的事情如下
- 分配 SessionId
- 创建 stageDir 用于后续接收客户端要安装的 apk
  - "data/app/vmdl${sessionId}.tmp/"
- 创建一个 PackageInstallerSession 对象, 描述一个安装任务
  - 它也是一个 Binder 代理对象

**客户端有了 SessionId, 就可以找到服务端对应的 PackageInstallerSession 与之进行交互了**

上面我们看到, 在创建 PackageInstallerSession 的过程中, 创建一个了一个 stageDir, 这个文件路径就是用来接收要安装的 apk 文件的, 接下来我们看看 InstallingAsyncTask 发送安装文件的过程

### 二) 发送安装文件
```
public class InstallInstalling extends Activity {
    
    private final class InstallingAsyncTask extends AsyncTask<Void, Void,
            PackageInstaller.Session> {
        volatile boolean isDone;

        @Override
        protected PackageInstaller.Session doInBackground(Void... params) {
            // 1. 获取 PackageInstallerSession 在客户端的代理对象
            PackageInstaller.Session session;
            try {
                session = getPackageManager().getPackageInstaller().openSession(mSessionId);
            } catch (IOException e) {
                return null;
            }

            session.setStagingProgress(0);
            // 2. 将安装包发送给 Pacakge Installer
            try {
                File file = new File(mPackageURI.getPath());
                // 打开待安装文件的输入流
                try (InputStream in = new FileInputStream(file)) {
                    long sizeBytes = file.length();
                    // 打开 暂存位置 的输出流
                    try (OutputStream out = session
                            .openWrite("PackageInstaller", 0, sizeBytes)) {
                        byte[] buffer = new byte[1024 * 1024];
                        while (true) {
                            int numRead = in.read(buffer);
                            ......
                            out.write(buffer, 0, numRead);
                            ......
                        }
                    }
                }
                return session;
            }
            .....
        }

        @Override
        protected void onPostExecute(PackageInstaller.Session session) {
            if (session != null) {
                ......
                // 2. 提交安装请求
                session.commit(pendingIntent.getIntentSender());
                ......
            } else {
                ......// 回调安装失败
            }
        }
    }
    
}
```
InstallingAsyncTask 中做了如下的事务
- 首选根据上面打开的 mSessionId 获取一个 PackageInstallerSession 在客户端的 Binder 代理对象
- 然后通过这个 Binder 代理对象获取 暂存位置的 输出流, 并将安装包拷贝到其中
  - 暂存位置为 "data/app/vmdl${sessionId}.tmp/PackageInstaller"
- 文件发送成功之后, 通过 Session 提交一个应用安装的请求

**InstallingAsyncTask 执行完毕之后, 我们的文件就拷贝到 stageDir 中了, 接下来我们去系统服务进程中看看 PackageInstallerSession 如何提交一个应用安装请求**

### 三) 提交应用安装请求
```
public class PackageInstallerSession extends IPackageInstallerSession.Stub {
    
    @Override
    public void commit(@NonNull IntentSender statusReceiver, boolean forTransfer) {
        ......
        final boolean wasSealed;
        synchronized (mLock) {
            wasSealed = mSealed;
            if (!mSealed) {
                try {
                    // 1. 将 PackageInstaller 重命名为 "base.apk"
                    sealAndValidateLocked();
                } catch (IOException e) {
                    ......
                }
            }
            ......
            mCommitted = true;
            // 发送一个 MSG_COMMIT 到 Handler 的消息队列中执行
            mHandler.obtainMessage(MSG_COMMIT).sendToTarget();
        }
        ......
    }

    private final Handler.Callback mHandlerCallback = new Handler.Callback() {
        @Override
        public boolean handleMessage(Message msg) {
            switch (msg.what) {
                ......
                case MSG_COMMIT:
                    synchronized (mLock) {
                        try {
                            commitLocked();
                        } catch (PackageManagerException e) {
                            ......
                        }
                    }
                    break;
            }
        }
    }
    
    private final PackageManagerService mPm;
    
    @GuardedBy("mLock")
    private void commitLocked()
            throws PackageManagerException {
            
        // 2. 解压 Native 库到 "data/app/vmdl${sessionId}.tmp/lib/" 目录下暂存
        extractNativeLibraries(mResolvedStageDir, params.abiOverride, mayInheritNativeLibs());
        ......
        // 3. 请求 PKMS.installStage 执行安装操作
        mPm.installStage(mPackageName, stageDir, localObserver, params,
                mInstallerPackageName, mInstallerUid, user, mSigningDetails);
    }
}
```
可以看到 PackageInstallerSession 的 commit 操作如下
- 调用 sealAndValidateLocked 对拷贝过来的安装包进行验证
  - 将  "data/app/vmdl${sessionId}.tmp/PackageInstaller" 重命名为 "data/app/vmdl${sessionId}.tmp/base.apk"
- 解压 Native 库
- 调用 PKMS.installStage 发起这次应用安装的请求

到这里应用安装前的准备就执行完毕了, 下面做个简单的回顾

### 四) 回顾
应用安装前的准备工作如下
- 获取 SessionId 描述一个安装任务
  - 分配 SessionId
  - 创建临时目录 stageDir
    - "data/app/vmdl{sessionId}.tmp"
  - 创建服务进程的 PackageInstallerSession 对象
- 将安装包拷贝到 "data/app/vmdl${sessionId}.tmp/PackageInstaller" 位置下暂存
- 提交安装任务
  - 调用 sealAndValidateLocked 对拷贝过来的安装包进行验证
    - **将 "data/app/vmdl${sessionId}.tmp/PackageInstaller" 重命名为 "data/app/vmdl${sessionId}.tmp/base.apk"**
  - 解压 Native 库
  - 调用 PKMS.installStage 发起这次应用安装的请求

## 二. PKMS 安装应用
```
public class PackageManagerService extends IPackageManager.Stub
        implements PackageSender {
            
    void installStage(String packageName, File stagedDir,
            IPackageInstallObserver2 observer, PackageInstaller.SessionParams sessionParams,
            String installerPackageName, int installerUid, UserHandle user,
            PackageParser.SigningDetails signingDetails) {
        ......
        // 构建一个 INIT_COPY 的 Message
        final Message msg = mHandler.obtainMessage(INIT_COPY);
        ......
        // 1. 创建安装参数
        final InstallParams params = new InstallParams(origin, null, observer,
                sessionParams.installFlags, installerPackageName, sessionParams.volumeUuid,
                verificationInfo, user, sessionParams.abiOverride,
                sessionParams.grantedRuntimePermissions, signingDetails, installReason);
        ......
        msg.obj = params;
        ......
        mHandler.sendMessage(msg);
    }
    
    class PackageHandler extends Handler {
          
        void doHandleMessage(Message msg) {
            switch (msg.what) {
                ......
                case INIT_COPY: {
                    HandlerParams params = (HandlerParams) msg.obj;
                    int idx = mPendingInstalls.size();
                    .....
                    // 若为开启应用安装服务, 则先执行绑定操作
                    if (!mBound) {
                        // 2. 尝试取绑定应用安装服务
                        if (!connectToService()) {
                            ......
                            return;
                        } else {
                            // 3. 绑定成功, 则添加到 mPendingInstalls, 等待执行
                            mPendingInstalls.add(idx, params);
                        }
                    } else {
                        // 已经绑定了安装服务, 则直接添加到缓存中, 会自动取数据执行
                        mPendingInstalls.add(idx, params);
                        // 若 idx 为 0, 则需要手动触发一次, 来执行 mPendingInstalls 中的任务
                        if (idx == 0) {
                            mHandler.sendEmptyMessage(MCS_BOUND);
                        }
                    }
                    break;
                }
                ......
            }
        }  
    }
}
```
当 PackageInstallerSession 提交了安装任务之后, **PKMS 会构建一个 InstallParams 描述待安装任务的参数, 然后发送一条应用拷贝的消息**

应用拷贝的操作是在应用安装服务中进行的, 因此 PKMS 还需要绑定应用安装服务, 绑定成功之后将这个任务添加到 mPendingInstalls 中

接下来我们看看如何绑定应用安装服务

### 一) 绑定应用安装服务
```
public class PackageManagerService extends IPackageManager.Stub
        implements PackageSender {
    
    // 应用安装服务 DefaultContainerService
    public static final ComponentName DEFAULT_CONTAINER_COMPONENT = new ComponentName(
            DEFAULT_CONTAINER_PACKAGE, "com.android.defcontainer.DefaultContainerService");
    
    // 描述与应用安装服务的连接
    final private DefaultContainerConnection mDefContainerConn =
            new DefaultContainerConnection();

    // 服务连接实现类
    class DefaultContainerConnection implements ServiceConnection {
        public void onServiceConnected(ComponentName name, IBinder service) {
            // 2. 获取与服务端交互的 Binder 代理对象
            final IMediaContainerService imcs = IMediaContainerService.Stub
                    .asInterface(Binder.allowBlocking(service));
            // 3. 发送消息表示绑定完毕
            mHandler.sendMessage(mHandler.obtainMessage(MCS_BOUND, imcs));
        }
        ......
    }
    
    class PackageHandler extends Handler {
        
        private boolean connectToService() {
            ......
            Intent service = new Intent().setComponent(DEFAULT_CONTAINER_COMPONENT);
            ......
            // 1. 绑定 DefaultContainerService
            if (mContext.bindServiceAsUser(service, mDefContainerConn,
                    Context.BIND_AUTO_CREATE, UserHandle.SYSTEM)) {
                ......
                mBound = true;
                return true;
            }
            ......
            return false;
        }
        
    }
    
}
```
PKMS 绑定的应用安装服务为 DefaultContainerService, 绑定完成之后会发送 MCS_BOUND 消息, 继续执行 mPendingInstalls 中的任务

接下来看看 MCS_BOUND 消息如何执行安装任务

### 二) 执行安装任务
```
public class PackageManagerService extends IPackageManager.Stub
        implements PackageSender {
    
    class PackageHandler extends Handler {
          
        void doHandleMessage(Message msg) {
            switch (msg.what) {
                ......
                case MCS_BOUND: {
                    if (msg.obj != null) {
                        // 获取与 DefaultContainerService 交互的 Binder 代理对象
                        mContainerService = (IMediaContainerService) msg.obj;
                    }
                    if (mContainerService == null) {
                        ......
                    } else if (mPendingInstalls.size() > 0) {
                        HandlerParams params = mPendingInstalls.get(0);
                        if (params != null) {
                            ......
                            // 调用 HandlerParams.startCopy 开启安装包拷贝操作
                            if (params.startCopy()) {
                                // 拷贝成功, 移除这个待安装项
                                if (mPendingInstalls.size() > 0) {
                                    mPendingInstalls.remove(0);
                                }
                                // 没有待安装任务了, 解绑 DefaultContainerService
                                if (mPendingInstalls.size() == 0) {
                                    if (mBound) {
                                        removeMessages(MCS_UNBIND);
                                        Message ubmsg = obtainMessage(MCS_UNBIND);
                                        sendMessageDelayed(ubmsg, 10000);
                                    }
                                } 
                                // 继续发送 MCS_BOUND, 安装 mPendingInstalls 中的任务
                                else {
                                    ......
                                    mHandler.sendEmptyMessage(MCS_BOUND);
                                }
                            }
                        }
                    }
                    ......
                    break;
                }
                ......
            }
        }
        
    }
}
```
可以看到 PKMS 对 MCS_BOUND 的处理, 最主要的还是调用了 HandlerParams.startCopy 开启了安装包的拷贝操作
```java
public class PackageManagerService extends IPackageManager.Stub
        implements PackageSender {
        
    private abstract class HandlerParams {
    
        final boolean startCopy() {
            boolean res;
            try {
                ......
                if (++mRetries > MAX_RETRIES) {
                    .......
                } else {
                    // 1. 拷贝安装包
                    handleStartCopy();
                    res = true;
                }
            } catch (RemoteException e) {
                ......
            }
            ......
            // 2. 安装应用程序
            handleReturnCode();
            return res;
        }
        
    }

}
```
HandlerParams 中的 startCopy
- 调用了 handleStartCopy 执行安装包的拷贝操作
- 执行了 handleReturnCode 进行应用程序的安装
 
这便是应用安装的终点操作了, HandlerParams 的实现类为 InstallParams, 我们分别看看它的实现

#### 1. 安装包的拷贝
```java
public class PackageManagerService extends IPackageManager.Stub
        implements PackageSender {

    class InstallParams extends HandlerParams {
    
        public void handleStartCopy() throws RemoteException {
            int ret = PackageManager.INSTALL_SUCCEEDED;
            ......
            final InstallArgs args = createInstallArgs(this);
            mArgs = args;

            if (ret == PackageManager.INSTALL_SUCCEEDED) {
                .......
                if (!origin.existing && requiredUid != -1
                        && isVerificationEnabled(
                                verifierUser.getIdentifier(), installFlags, installerUid)) {
                    ........
                } else {
                    // 调用 FileInstallArgs.copyApk 执行安装包拷贝
                    ret = args.copyApk(mContainerService, true);
                }
            }

            mRet = ret;
        }
        
    }
}
```
InstallParams 中的 handleStartCopy 它会调用 InstallArgs 的 copyApk 执行 apk 的拷贝操作, 我们看看它的实现

```java
public class PackageManagerService extends IPackageManager.Stub
        implements PackageSender {
        
    class FileInstallArgs extends InstallArgs {
        
        int copyApk(IMediaContainerService imcs, boolean temp) throws RemoteException {
            try {
                // 拷贝应用程序
                return doCopyApk(imcs, temp);
            } finally {
                ......
            }
        }

        private int doCopyApk(IMediaContainerService imcs, boolean temp) throws RemoteException {
            ......
            // 1. 若是通过 Session 拷贝过了, 则跳过拷贝的操作
            if (origin.staged) {
                if (DEBUG_INSTALL) Slog.d(TAG, origin.file + " already staged; skipping copy");
                codeFile = origin.file;
                resourceFile = origin.file;
                return PackageManager.INSTALL_SUCCEEDED;
            }
            // 2. 执行拷贝操作
            // 2.1 创建临时目录 "data/app/vmdl${sessionId}.tmp/"
            try {
                final boolean isEphemeral = (installFlags & PackageManager.INSTALL_INSTANT_APP) != 0;
                final File tempDir = mInstallerService.allocateStageDirLegacy(volumeUuid, isEphemeral);
                codeFile = tempDir;
            } catch (IOException e) {
                .....
            }
            // 2.2 实现一个获取文件描述符的工厂方法, 也是一个 Binder 对象
            final IParcelFileDescriptorFactory target = new IParcelFileDescriptorFactory.Stub() {
            
                @Override
                public ParcelFileDescriptor open(String name, int mode) throws RemoteException {
                    if (!FileUtils.isValidExtFilename(name)) {
                        throw new IllegalArgumentException("Invalid filename: " + name);
                    }
                    try {
                        // 用于打开文件描述符
                        final File file = new File(codeFile, name);
                        final FileDescriptor fd = Os.open(file.getAbsolutePath(),
                                O_RDWR | O_CREAT, 0644);
                        Os.chmod(file.getAbsolutePath(), 0644);
                        return new ParcelFileDescriptor(fd);
                    } catch (ErrnoException e) {
                        throw new RemoteException("Failed to open: " + e.getMessage());
                    }
                }
                
            };
            
            ......
            // 2.3 调用了 IMediaContainerService 代理对象的 copyPackage, 将 apk 拷贝到安装目录
            ret = imcs.copyPackage(origin.file.getAbsolutePath(), target);
            ......
            // 2.4 解压 Native 库文件
            final File libraryRoot = new File(codeFile, LIB_DIR_NAME);
            NativeLibraryHelper.Handle handle = null;
            try {
                handle = NativeLibraryHelper.Handle.create(codeFile);
                ret = NativeLibraryHelper.copyNativeBinariesWithOverride(handle, libraryRoot,
                        abiOverride);
            } catch (IOException e) {
                Slog.e(TAG, "Copying native libraries failed", e);
                ret = PackageManager.INSTALL_FAILED_INTERNAL_ERROR;
            } finally {
                IoUtils.closeQuietly(handle);
            }
            ......
            return ret;
        }
        
    }
    
}
```
InstallParams 中的 handleStartCopy 流程如下
- 若是已经通过 Session 拷贝过了, 跳过拷贝的过程
- 若未拷贝, 它会调用 InstallArgs 的 copyApk 执行 apk 的拷贝操作, 其主要步骤如下
  - 创建拷贝的目录 "data/app/vmdl${sessionId}.tmp/"
  - 实现一个获取文件描述符的工厂方法, 也是一个 Binder 对象
  - 调用了 IMediaContainerService 的 copyPackage, 在远程服务中执行 apk 拷贝操作
    - 拷贝到 "data/app/vmdl${sessionId}.tmp/base.apk" 中
  - 解压 Native 库文件

关于安装包的拷贝我们就看到这里, 接下来回到 PKMS 中看看执行应用的安装操作

### 二) 应用的安装
```
public class PackageManagerService extends IPackageManager.Stub
        implements PackageSender {
           
    class InstallParams extends HandlerParams {
    
        @Override
        void handleReturnCode() {
            if (mArgs != null) {
                // 执行安装操作
                processPendingInstall(mArgs, mRet);
            }
        }
        
    }
 
    private void processPendingInstall(final InstallArgs args, final int currentStatus) {
        // Queue up an async operation since the package installation may take a little while.
        mHandler.post(new Runnable() {
            public void run() {
                mHandler.removeCallbacks(this);
                // Result object to be returned
                PackageInstalledInfo res = new PackageInstalledInfo();
                res.setReturnCode(currentStatus);
                res.uid = -1;
                res.pkg = null;
                res.removedInfo = null;
                if (res.returnCode == PackageManager.INSTALL_SUCCEEDED) {
                    // 安装前准备
                    args.doPreInstall(res.returnCode);
                    synchronized (mInstallLock) {
                        // 执行安装操作        
                        installPackageTracedLI(args, res);
                    }
                    // 安装结束
                    args.doPostInstall(res.returnCode, res.uid);
                }
                ......
                // 记录正在安装的应用
                int token;
                if (mNextInstallToken < 0) mNextInstallToken = 1;
                token = mNextInstallToken++;
                PostInstallData data = new PostInstallData(args, res);
                mRunningInstalls.put(token, data);
            }
        });
    }
    
}
```
这里我们主要看看 installPackageTracedLI 是如何安装应用程序的

```
public class PackageManagerService extends IPackageManager.Stub
        implements PackageSender {
    
    private void installPackageTracedLI(InstallArgs args, PackageInstalledInfo res) {
        try {
            ......
            // 安装应用程序
            installPackageLI(args, res);
        } finally {
            ......
        }
    }
    
    private void installPackageLI(InstallArgs args, PackageInstalledInfo res) {
        // tmpPackageFile 即 base.apk
        final File tmpPackageFile = new File(args.getCodePath());
        final PackageParser.Package pkg;
        try {
            // 1. 解析安装包中的信息发布到 PKMS 中
            pkg = pp.parsePackage(tmpPackageFile, parseFlags);
            // 2. 解析 dex 文件
            DexMetadataHelper.validatePackageDexMetadata(pkg);
        } catch (PackageParserException e) {
           .......
        }
        
        // 3. 调用 InstallArgs 将 "data/app/vmdl${sessionId}.tmp/" 更换成正式名称
        if (!args.doRename(res.returnCode, pkg, oldCodePath)) {
            ......
        }
        
        // 4. 优化 dex 文件
        BackgroundDexOptService.notifyPackageChanged(pkg.packageName);
        ......
    }
    
}
```
installPackageLI 非常复杂, 这里进行了大量的删减, 其主要流程如下
- 调用了 PackageParser.Package 解析 base.apk 安装包文件
- 解析 apk 的 dex 文件
- 调用 InstallArgs.doRename 更换目录名称
  - 更名前: "data/app/vmdl${sessionId}.tmp/"
  - 更名后: "data/app/${PackageName}${Base64 随机码}/"
- 优化 dex 文件

重点的流程在于 PackageParser.Package,  这个方法我们在上一篇文章中已经分析过了, 不同的是上一篇文章扫描的是安装好的文件夹, 这里扫描的是 base.apk 中的信息, 最终都会将 apk 内部的 AndroidManifest.xml 中的信息发布到 PKMS 中, 这里就不再赘述了

我们之前一直使用的是 **vmdl${sessionId}.tmp** 这个目录, 发布成功之后, 将它更名为 **${PackageName}${Base64 随机码}**

这里我们旨在分析应用安装的流程, 关于 dex 文件解析和优化后面有机会单独找一篇文章来分析, 这里就不展开讨论了

## 总结
安装好的应用程序目录如下

![安装好的应用目录](https://i.loli.net/2019/11/15/Nn2OJy3ZaH6XLMo.png)

我们 Android 端, 整个应用程序的安装主要有两种方式:
- **通过 Session 发起(adb, 手动点击安装)**
  - 提前将安装包拷贝到 "data/app/vmdl${sessionId}.tmp/" 目录下 
  - 再通过 Session commit, 通知 PKMS 的 PKMS.installStage 进行安装
- **通过系统的软件商店自动发起**
  - 直接通过 PKMS 的 PKMS.installStage 进行安装

### 通过 Session 拷贝安装包
- 获取 SessionId 描述一个安装任务
  - 分配 SessionId
  - 创建临时目录 stageDir
    - "data/app/vmdl{sessionId}.tmp"
  - 创建服务进程的 PackageInstallerSession 对象
- 将安装包拷贝到 "data/app/vmdl${sessionId}.tmp/PackageInstaller" 位置下暂存
- 提交安装任务
  - 调用 sealAndValidateLocked 对拷贝过来的安装包进行验证
    - **将 "data/app/vmdl${sessionId}.tmp/PackageInstaller" 重命名为 "data/app/vmdl${sessionId}.tmp/base.apk"**
  - 解压 Native 库
  - 调用 PKMS.installStage 发起这次应用安装的请求

### PKMS 安装应用程序
- **连接远程应用安装服务 DefaultContainerService**
  - 获取 IMediaContainerService 的 Binder 代理对象
- 安装应用
  - 拷贝安装包
    - 若是已经通过 Session 拷贝过了, 跳过拷贝的过程
    - 若未拷贝, 它会调用 InstallArgs 的 copyApk 执行 apk 的拷贝操作, 其主要步骤如下
      - 创建拷贝的目录 "data/app/vmdl${sessionId}.tmp/"
      - 实现一个获取文件描述符的工厂方法, 也是一个 Binder 对象
      - 调用了 IMediaContainerService 的 copyPackage, 在远程服务中执行 apk 拷贝操作
        - 拷贝到 **"data/app/vmdl${sessionId}.tmp/base.apk"** 中
      - 解压 Native 库文件
  - 安装应用程序
    - 调用了 PackageParser.Package 解析 base.apk 安装包文件
      - 即将 apk 中的信息发布到 PKMS 
    - 解析 apk 的 dex 文件
    - 调用 InstallArgs.doRename 更换目录名称
      - 更名前: "data/app/vmdl${sessionId}.tmp/"
      - 更名后: "data/app/${PackageName}${Base64 随机码}/"
    - 优化 dex 文件

## 参考文献
- https://blog.csdn.net/c_z_w/article/details/79785108