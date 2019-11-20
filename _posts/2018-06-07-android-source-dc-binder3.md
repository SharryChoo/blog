---
layout: article
title: "Android 系统架构 —— 数据通信篇 之 IBinder 对象的实例化"
key: "Android 系统架构 —— 数据通信篇 之 IBinder 对象的实例化" 
tags: AndroidFramework
aside:
  toc: true
---

## 前言
在 Binder 通信实例中我们了解到, Binder 驱动通过 IBinder 提供服务, 服务端为 Binder, 客户端未 BinderProxy

这里我们分别了解一下他们的实例化过程

<!--more-->

## 一. Binder 本地对象的实例化
使用过 Service 的都知道, 在 onBind 中会 new 一个我们实现好的 Binder 对象
```
public class SampleService extends Service {

    public IBinder onBind(Intent intent) {
        // 实例化了一个 IBinder 对象
        // 这个 SampleBinder 继承了 Binder 并且实现了我们定义好的 IInterface 接口
        return new SampleBinder();
    }
    
}
```
这个样子我们就创建了一个 Binder 本地对象, 我们看看他的构造方法

```
public class Binder implements IBinder {
    /**
     * Binder.constructor
     */
    public Binder() {
        // 调用了 getNativeBBinderHolder 方法, 获取一个 Native 层的 BBinderHolder 对象句柄值
        mObject = getNativeBBinderHolder();
        ......
    }
    
    private static native long getNativeBBinderHolder();
    
}
```
可以看见 Binder 中主要调用了 native 方法, 获取了一个句柄值, 当然这个 mObject 句柄值也非常重要, 因为 Binder 类内部很多方法都是由它代理实现的

### 一) Native 层实例化
```
// frameworks/base/core/jni/android_util_Binder.cpp
static jlong android_os_Binder_getNativeBBinderHolder(JNIEnv* env, jobject clazz)
{
    // new 了一个 JavaBBinderHolder 这个对象
    JavaBBinderHolder* jbh = new JavaBBinderHolder();
    return (jlong) jbh;
}
```
非常简单的实现 new 了一个 JavaBBinderHolder 这个对象, 然后将它的句柄返回给 Java 中 Binder 对象的 mObject 属性

下面看看 JavaBBinderHolder 的实例化流程

```
// frameworks/base/core/jni/android_util_Binder.cpp
class JavaBBinderHolder
{
public:
    // 获取 JavaBBinder 对象
    sp<JavaBBinder> get(JNIEnv* env, jobject obj)
    {
        AutoMutex _l(mLock);
        // 尝试将 mBinder 这个弱指针升级为强指针
        sp<JavaBBinder> b = mBinder.promote();
        // 升级失败则重新创建 JavaBBinder 对象
        if (b == NULL) {
            b = new JavaBBinder(env, obj);
            mBinder = b;
        }
        // 返回一个 JavaBinder
        return b;
    }

private:
    Mutex           mLock;   // 互斥锁
    wp<JavaBBinder> mBinder; // 一个 JavaBBinder 的弱指针
};
```
可以看到 JavaBBinderHolder 顾名思义果真持有了一个 JavaBBinder 的对象, 并且可以通过 get 方法获取到, 接下来看看这个 JavaBinder 

```
// frameworks/base/core/jni/android_util_Binder.cpp
class JavaBBinder : public BBinder
{
public:
    // 构造函数
    JavaBBinder(JNIEnv* env, jobject /* Java Binder */ object)
        : mVM(jnienv_to_javavm(env)), mObject(env->NewGlobalRef(object))
    {
        ......
    }

    // 获取我们 Java 中实例化的 Binder 对象
    jobject object() const
    {
        return mObject;
    }

protected:
    // Native 层 JavaBBinder 的 onTransact 函数
    virtual status_t onTransact(
        uint32_t code, const Parcel& data, Parcel* reply, uint32_t flags = 0)
    {   
        // 将 JavaVM convert2 JNIEnv
        JNIEnv* env = javavm_to_jnienv(mVM);
        // 回调 java 中 Binder 对象的 execTransact 方法
        jboolean res = env->CallBooleanMethod(mObject, gBinderOffsets.mExecTransact,
            code, reinterpret_cast<jlong>(&data), reinterpret_cast<jlong>(reply), flags);
        
        // ......
        if (code == SYSPROPS_TRANSACTION) {
            // 回调父类的 onTransact 函数
            BBinder::onTransact(code, data, reply, flags);
        }
        
        return res != JNI_FALSE ? NO_ERROR : UNKNOWN_TRANSACTION;
    }

private:
    JavaVM* const   mVM;      // JavaVM 指针对象, 用于获取 JNIEnv*
    jobject const   mObject;  // Java 中的 Binder 对象, 即我们在 Java 中创建的实例
};
```
从上述代码可以看到 JavaBBinder 继承了 BBinder, 它内部持有了 JavaVM 和 Java 中的 Binder 对象实例, **因此 JavaBBinder 是 Native 层 Binder 与 Java 层 Binder 连接的桥梁**
- **其 JavaBBinder 的 onTransact 用于将 Binder 驱动发送的 IPC 请求回溯到 Java 的 Binder 方法中了**

下面我们看看 Native 层 BBinder 的构建

### 二) BBinder 的构建
```
/frameworks/native/include/binder/Binder.h
class BBinder : public IBinder
{
public:
    
    // 作为 Client 端, 向 Server 发送数据
    virtual status_t    transact(   uint32_t code,
                                    const Parcel& data,
                                    Parcel* reply,
                                    uint32_t flags = 0);
    ......
    
protected:
    // 作为 Server 端, 处理 Client 发送的数据
    virtual status_t    onTransact( uint32_t code,
                                    const Parcel& data,
                                    Parcel* reply,
                                    uint32_t flags = 0);
}
```
好的, 可以看到 BBinder 继承自 IBinder, 之中有两个非常重要的函数
- transact: 作为 Client 端时, 向 Server 发送数据 
- onTransact: 作为 Server 端, 处理 Client 发送的数据

可以看到 BBinder 是继承自 IBinder 的, 这就类似于 Java 层的 IBinder 对象, 因此我们猜想 Java 层的 BinderProxy 对应在 Native 层实现类也是继承自 IBinder 的

接下来我们来看看 BinderProxy 代理对象的实例化过程

## 二. BinderProxy 代理对象的实例化
```
final class BinderProxy implements IBinder {
    /**
     * BinderProxy.getInstance 是私有化的
     */
    private static BinderProxy getInstance(long nativeData, long iBinder) {
        BinderProxy result;
        try {
            result = sProxyMap.get(iBinder);
            if (result != null) {
                return result;
            }
            result = new BinderProxy(nativeData);
        } catch (Throwable e) {
            // We're throwing an exception (probably OOME); don't drop nativeData.
            NativeAllocationRegistry.applyFreeFunction(NoImagePreloadHolder.sNativeFinalizer,
                    nativeData);
            throw e;
        }
        NoImagePreloadHolder.sRegistry.registerNativeAllocation(result, nativeData);
        // The registry now owns nativeData, even if registration threw an exception.
        sProxyMap.set(iBinder, result);
        return result;
    }

    /**
     * BinderProxy 的构造方法也是私有化的
     */
    private BinderProxy(long nativeData) {
        mNativeData = nativeData;
    }
    
}
```
可以看到 BinderProxy 这个 java 的 Class 两个获取实例的方式都是私有化的, 而且其构造方法的形参是一个 Native 指针的句柄值

那么只有一种可能性, BinderProxy 这个对象是在 Native 层使用 JNI 实例化再返回 java 层的, 接下来我们便验证一下这个猜想

### 一) BinderProxy 对象的获取方式
我们在开发过程中, 通过调用 Context.bindService(), 启动一个远程的 Service, 这时候需要需要传入一个 ServiceConnection 参数对象, **在其 onServiceConnected 方法中会收到一个 IBinder 对象的回调, 这个对象即为 BinderProxy**
```
   private val connect: ServiceConnection = object : ServiceConnection {

        override fun onServiceDisconnected(name: ComponentName?) {

        }

        override fun onServiceConnected(name: ComponentName?, service: IBinder?) {
            // 这个 service 即一个 BinderProxy 的实例对象
        }
        
    }
```
因为这里还会牵扯到 Service 的启动流程, 所以我们采取另一种方式进行探究

#### 1. 获取系统服务
```
class ContextImpl extends Context {

    @Override
    public void startActivityAsUser(Intent intent, Bundle options, UserHandle user) {
        try {
            ActivityManager.getService().startActivityAsUser(......);
        } catch (RemoteException e) {
            throw e.rethrowFromSystemServer();
        }
    }

}

public class ActivityManager {

    public static IActivityManager getService() {
        return IActivityManagerSingleton.get();
    }

    private static final Singleton<IActivityManager> IActivityManagerSingleton =
            new Singleton<IActivityManager>() {
                @Override
                protected IActivityManager create() {
                    final IBinder b = ServiceManager.getService(Context.ACTIVITY_SERVICE);
                    final IActivityManager am = IActivityManager.Stub.asInterface(b);
                    return am;
                }
            };
            
}
```
好的, 可以看到 AMS 的 Binder 代理对象是通过 ServiceManager 获取的, 我们继续往下剖析
```
public final class ServiceManager {

    /**
     * ServiceManager.getService 获取服务的 IBinder 对象
     */
    public static IBinder getService(String name) {
        try {
            // 尝试从缓存中获取
            IBinder service = sCache.get(name);
            if (service != null) {
                return service;
            } else {
                // 关注一下这个地方
                // 1. getIServiceManager() 获取 IServiceManager 服务提供对象
                // 2. 调用 IServiceManager.getService(name) 获取服务
                return Binder.allowBlocking(getIServiceManager().getService(name));
            }
        } catch (RemoteException e) {
            Log.e(TAG, "error in getService", e);
        }
        return null;
    }
    
    /**
     * ServiceManager.getIServiceManager
     */
    private static IServiceManager getIServiceManager() {
        if (sServiceManager != null) {
            return sServiceManager;
        }
        // 关注这里
        // 1. 这里通过 BinderInternal.getContextObject() 这个方法获取到了 BinderProxy 对象
        // 2. 通过 asInterface 将这个 BinderProxy 对象封装成了 ServiceManagerProxy 
        sServiceManager = ServiceManagerNative
                .asInterface(Binder.allowBlocking(BinderInternal.getContextObject()));
        return sServiceManager;
    }
    
}
```
可以看到 ServiceManager 其实是一个壳, 其方法的主要实现是通过 IServiceManager 实现的, 这个 IServiceManager 是 ServiceManager 在 Client 端的 Binder 代理对象, 其获取方式是通过该 BinderInternal.getContextObject()

**这可能是最直接的获取 BinderProxy 的方式了**, 下面我们就看看它是如何创建 ServiceManager 的 Binder 代理对象的

#### 2. 获取 ServiceManager 的 Binder 代理对象
```
public class BinderInternal {

    /**
     * BinderInternal.getContextObject 这就是获取 BinderProxy 对象的关键
     */
    public static final native IBinder getContextObject();
    
}
```
BinderInternal 很简单的调用了 Native 层的方式, 下面我们去 Native 层看看其构建流程

### 二) Native 构建 Binder 代理对象
```
// frameworks/base/core/jni/android_util_Binder.cpp
static jobject android_os_BinderInternal_getContextObject(JNIEnv* env, jobject clazz)
{
    // 1. 调用了 ProcessState::self() 获取 ProcessState 对象
    // 2. 调用了 ProcessState 的 getContextObject 一个 IBinder 对象
    sp<IBinder> b = ProcessState::self()->getContextObject(NULL);
    // 3. 调用 javaObjectForIBinder 返回一个 Java 中的 Binder 对象
    return javaObjectForIBinder(env, b);
}
```
可以看到 Native 层中的 getContextObject 中主要操作如下
- 获取 ProcessState 对象
- 通过 ProcessState->getContextObject(NULL) 获取了一个 IBinder 对象
- 通过 javaObjectForIBinder 这个方法获取 Java 的代理对象

#### 1. 获取 ProcessState
```
// frameworks/native/libs/binder/ProcessState.cpp
sp<ProcessState> ProcessState::self()
{
    Mutex::Autolock _l(gProcessMutex);
    if (gProcess != NULL) {
        return gProcess;
    }
    // 获取了 ProcessState 的单例
    gProcess = new ProcessState("/dev/binder");
    return gProcess;
}

#define BINDER_VM_SIZE ((1*1024*1024) - (4096 *2))

ProcessState::ProcessState(const char *driver)
    : mDriverName(String8(driver))
    // 1. 打开 Binder 驱动, 获取驱动设备的文件描述符
    , mDriverFD(open_driver(driver))
    , mVMStart(MAP_FAILED)
    , mThreadCountLock(PTHREAD_MUTEX_INITIALIZER)
    , mThreadCountDecrement(PTHREAD_COND_INITIALIZER)
    , mExecutingThreadsCount(0)
    // binder 线程池最大线程数
    , mMaxThreads(DEFAULT_MAX_BINDER_THREADS)
    , mStarvationStartTimeMs(0)
    , mManagesContexts(false)
    , mBinderContextCheckFunc(NULL)
    , mBinderContextUserData(NULL)
    , mThreadPoolStarted(false)
    , mSpawnThreadOnStart(true)
    , mThreadPoolSeq(1)
    , mMmapSize(mmap_size)
    ......
{
    if (mDriverFD >= 0) {
        // 2. 通过 mmap 系统调用, 映射一块缓冲区, 方便与内核驱动进行交互
        mVMStart = mmap(0, BINDER_VM_SIZE, PROT_READ, MAP_PRIVATE | MAP_NORESERVE, mDriverFD, 0);
        ......
    }
}
```
ProcessState 是负责连接 Android 运行时库和 Linux 内核中的 Binder 驱动的桥梁, 通过其 self 函数可知, 其实进程间单例的, ProcessState 实例化时做了两件事
- **打开 Binder 设备文件保存在 mDriverFD 中**
- **获取内核缓冲区的首地址 mVMStart**
  - **mmap 最的值为 1M - 8k**
  
也就是说只要 ProcessState 的 self 被调用了, Binder 驱动便会被初始化, 关于其相关实现, 我们到后面的 Binder 驱动中具体分析

下面我们看看 ProcessState 是如何获取 IBinder 对象的

#### 2. 获取 IBinder
```
sp<IBinder> ProcessState::getContextObject(const sp<IBinder>& /*caller*/)
{
    return getStrongProxyForHandle(0);
}

sp<IBinder> ProcessState::getStrongProxyForHandle(int32_t handle)
{
    sp<IBinder> result;
    AutoMutex _l(mLock);
    // 1. 从 ProcessState 的缓存中, 查找 handle 所对应的 IBinder 对象
    handle_entry* e = lookupHandleLocked(handle);
    if (e != NULL) {
        IBinder* b = e->binder;
        // 缓存不存在, 则创建这个 handle 对应的 BpBinder Binder 对象
        if (b == NULL || !e->refs->attemptIncWeak(this)) {
            if (handle == 0) {
                //... handle 为 0 意为 ServiceManager 的 Binder 代理对象, 做一些特殊处理
            }
            // 创建了 BpBinder
            b = BpBinder::create(handle);
            e->binder = b;
            if (b) e->refs = b->getWeakRefs();
            result = b;
        } 
        // 缓存存在, 增加强引用
        else {
            result.force_set(b);
            e->refs->decWeak(this);
        }
    }
    return result;
}
```
好的, 可以看到 ProcessState 的 getContextObject 会调用到 getStrongProxyForHandle 中, 最终会调用会调用 BpBinder 的 create 来创建一个 BpBinder 代理对象, 刚好与 BBinder 相对应


#####  BpBinder 的创建
```
BpBinder* BpBinder::create(int32_t handle) {
    ......
    return new BpBinder(handle, trackedUid);
}
BpBinder::BpBinder(int32_t handle, int32_t trackedUid)
    : mHandle(handle)
    , mAlive(1)
    , mObitsSent(0)
    , mObituaries(NULL)
    , mTrackedUid(trackedUid)
{
    ......
    // 增加一个 handle 的弱引用
    IPCThreadState::self()->incWeakHandle(handle, this);
}
```
好的, 可以看到 BpBinder 的构建比较简单, 它保留了上面传入的 handle 句柄值, Binder 驱动便是通过这个 handle 句柄值来定位 Server 进程的 Binder 本地对象的, 这个我们到后面的章节进行验证

接下来看看 Java 层 BinderProxy 的创建

#### 3. 创建 Java 层对象
```
// frameworks/base/core/jni/android_util_Binder.cpp
jobject javaObjectForIBinder(JNIEnv* env, const sp<IBinder>& val)
{
    if (val == NULL) return NULL;

    // 若为本地对象, 则返回 Java 中的 Binder 对象
    // 很显然这个 val 为 BpBinder, 我们关注下面的代码
    if (val->checkSubclass(&gBinderOffsets)) {
        jobject object = static_cast<JavaBBinder*>(val.get())->object();
        return object;
    }

    // 互斥锁
    AutoMutex _l(mProxyLock);

    // 1. 判断是否已经为当前驱动创建了 BinderProxy 对象了
    jobject object = (jobject)val->findObject(&gBinderProxyOffsets);
    if (object != NULL) {
        // 通过引用升级的方式, 判断这个 BinderProxy 对象是否有效
        jobject res = jniGetReferent(env, object);
        if (res != NULL) {
            // 若有效则返回到 java 层
            return res;
        }
        ......
        // 若无效则解除这个对象与 BpBinder 的关联
        val->detachObject(&gBinderProxyOffsets);
        ......
    }
    
    // 2. 这里通过 JNI 的方式调用了 BinderProxy 的构造函数
    object = env->NewObject(gBinderProxyOffsets.mClass, gBinderProxyOffsets.mConstructor);
    if (object != NULL) {
        // 3. 给 BinderProxy 中的 mObject 对象赋值, 值为 BpBinder 的句柄
        // 如此一来 Java 中的 BinderProxy 就可以通过它找到 BpBinder 了
        env->SetLongField(object, gBinderProxyOffsets.mObject, (jlong)val.get());
        val->incStrong((void*)javaObjectForIBinder);
        // 4. 将这个 BinderProxy 缓存到 BpBinder 中
        jobject refObject = env->NewGlobalRef(
                env->GetObjectField(object, gBinderProxyOffsets.mSelf));
        val->attachObject(&gBinderProxyOffsets, refObject,
                jnienv_to_javavm(env), proxy_cleanup);
        ......
    }

    return object;
}
```
可以看到这个方法中做了如下几件事情
- 若 val 为 BBinder, 则返回其关联的 Java 层的 Binder 对象
- 若 val 为 BpBinder
  - 从缓存中查找对应的 BinderProxy 对象, 若有效则返回给 Java 层
  - 若无效则通过 JNI 实例化一个 BinderProxy 对象
    - 给 BinderProxy.mObject 赋值, 指向 Native 层对应的 BpBinder 对象
    - 将这个 BinderProxy 对象加入 BpBinder 的缓存

**至此 Java 中的 BinderProxy 就在 通过 JNI 创建出来了**

## 总结
这篇文章主要分析了 Binder 对象在应用层和 Android 运行时库层的实例化过程, 他们最终都是通过 ProcessState 与 Binder 驱动进行交互
- Server 端
  - 应用层: 提供了 Binder 对象, 需要根据接口自行实现
  - Android 运行时库层:
     - 使用 JavaBBinder 建立 BBinder 与 Java 层 Binder 的桥梁
- Client 端
  - 应用层: 提供了 BinderProxy 对象
  - Android 运行时库层:
    - 使用 BpBinder 来描述

![IBinder 驱动模型依赖关系图](https://i.loli.net/2019/10/23/85LnWsDXI9HaE2e.png)
