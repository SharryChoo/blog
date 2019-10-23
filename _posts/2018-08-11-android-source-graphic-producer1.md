---
layout: article
title: "Android 系统架构 —— 图形架构篇之 Window 与 WindowManager 的关系"
key: "Android 系统架构 —— 图形架构篇之 Window 与 WindowManager 的关系" 
tags: AndroidFramework
aside:
  toc: true
---

## 前言
通过图像渲染的背景知识, 我们知道一个 Window 是与 WindowManager 绑定的, 那他们是如何建立联系的呢? 

<!--more-->

我们知道 Activity 的创建过程中会有 Window 的创建, 我们从 Activity 的创建流程中寻找答案
```
public final class ActivityThread extends ClientTransactionHandler {
    
    private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
        ActivityInfo aInfo = r.activityInfo;
        ......
        ContextImpl appContext = createBaseContextForActivity(r);
        Activity activity = null;
        try {
            java.lang.ClassLoader cl = appContext.getClassLoader();
            activity = mInstrumentation.newActivity(
                    cl, component.getClassName(), r.intent);
            ......
        } catch (Exception e) {
            ......
        }

        try {
            Application app = r.packageInfo.makeApplication(false, mInstrumentation);
            if (activity != null) {
                ......
                Window window = null;
                ......
                // 将 Activity 中的 mBase 与 上面创建的 ContextImpl 绑定
                appContext.setOuterContext(activity);
                // 给 Activity 绑定数据
                activity.attach(appContext, this, getInstrumentation(), r.token,
                        r.ident, app, r.intent, r.activityInfo, title, r.parent,
                        r.embeddedID, r.lastNonConfigurationInstances, config,
                        r.referrer, r.voiceInteractor, window, r.configCallback);
                ......
                if (r.isPersistable()) {
                    // 回调 OnCreate()
                    mInstrumentation.callActivityOnCreate(activity, r.state, r.persistentState);
                } else {
                    mInstrumentation.callActivityOnCreate(activity, r.state);
                }
                ......
            }
            ......
            mActivities.put(r.token, r);

        } catch (SuperNotCalledException e) {
            ......
        } catch (Exception e) {
            ......
        }
        return activity;
    }
    
}
```
可以看到, 在 Activity 创建之后, 回调 onCreate 之前, 调用了 attach 方法进行了数据绑定, 所以我们就从这个方法入手, 分析 Window 和 WindowManager 的关系

## 一. Window 的创建
```
public class Activity extends ContextThemeWrapper {

    final void attach(Context context, ActivityThread aThread,
            Instrumentation instr, IBinder token, int ident,
            Application application, Intent intent, ActivityInfo info,
            CharSequence title, Activity parent, String id,
            NonConfigurationInstances lastNonConfigurationInstances,
            Configuration config, String referrer, IVoiceInteractor voiceInteractor,
            Window window, ActivityConfigCallback activityConfigCallback) {
        ......
        // 1. 构建了 PhoneWindow 对象
        mWindow = new PhoneWindow(this, window, activityConfigCallback);
        ......
        // 2. 给这个 Window 设置了 WindowManager
        mWindow.setWindowManager(
                (WindowManager)context.getSystemService(Context.WINDOW_SERVICE),
                mToken, mComponent.flattenToString(),
                (info.flags & ActivityInfo.FLAG_HARDWARE_ACCELERATED) != 0);
        ......
        // 3. 将与这个 Window 绑定的 WindowManager 保存在 mWindowManager 成员变量中
        mWindowManager = mWindow.getWindowManager();
    }
}
```
可以看到在 Activity 的 attach 方法中, 关于 Window 它主要做了如下三件事情
- 构建了当前 Activity 的 Window 对象
- 给构建的 Window 绑定管理器的 WindowManager
- 将这个 WindowManager 保存到 Activity 的 mWindowManager 中

接下来我们看看这个 Window 是如何绑定 WindowManager 的
```
public class Window {
    
    public void setWindowManager(WindowManager wm, IBinder appToken, String appName,
            boolean hardwareAccelerated) {
        // 若 wm 为 null, 则当前 Context 在 SystemServiceRegistry 中注册的 WindowManagerImpl 对象
        if (wm == null) {
            wm = (WindowManager)mContext.getSystemService(Context.WINDOW_SERVICE);
        }
        // 通过 WindowManager 的实例创建了一个新的 WindowManager 提供给自己使用
        mWindowManager = ((WindowManagerImpl)wm).createLocalWindowManager(this);
    }
    
}
```
很有意思, 在 Window.setWindowManager 时, 若 wm 未传值, 则通过 Context.getSystemService 获取 WindowMangaer 对象, 接着会创建一个新的 WindowMangerIml 对象专门提供给这个 Window 使用, 有两个问题
- **为什么 Context.getSystemService(Context.WINDOW_SERVICE) 获取到的是 WindowManager 对象?**
  - 这似乎违背了我们对 Service 的理解, 命名成 Context.WINDOW_MANAGER 不会更好吗?
- **为什么获取到了这个 WindowManager 不用它直接管理这个 Window, 而是重新创建了一个呢?**

解决了这两个问题, 对 Window 和 WindowManager 之间的关系便可以理清了, 接下来我们看看如何获取 WindowManager

## 二. WindowManager 的获取
我们先看看它是如何通过 Context.getSystemService(Context.WINDOW_SERVICE) 获取 WindowMangerImpl 这个实例对象的
```
class ContextImpl extends Context {
    
     @Override
    public Object getSystemService(String name) {
        return SystemServiceRegistry.getSystemService(this, name);
    }
    
}

final class SystemServiceRegistry {
    
    // 服务获取器缓存(并非服务对象)
    private static final HashMap<String, ServiceFetcher<?>> SYSTEM_SERVICE_FETCHERS =
            new HashMap<String, ServiceFetcher<?>>();
    
    public static Object getSystemService(ContextImpl ctx, String name) {
        // 1. 通过 SYSTEM_SERVICE_FETCHERS 缓存的服务获取器
        ServiceFetcher<?> fetcher = SYSTEM_SERVICE_FETCHERS.get(name);
        // 2. 通过服务获取器 fetcher, 获取服务对象
        return fetcher != null ? fetcher.getService(ctx) : null;
    }
}
```
好的, 可以看到 SystemServiceRegistry 的 getSystemService 方法首先通过缓存获取到了服务获取器的实例, 然后调用获取器的 getService 方法获取了我们需要的 WindowManager 对象

接下来我们看看这个服务获取器在什么地方注册的

### 1. ServiceFetcher 的注册
```
final class SystemServiceRegistry {

    // 服务名的缓存
    private static final HashMap<Class<?>, String> SYSTEM_SERVICE_NAMES =
            new HashMap<Class<?>, String>();
    // 服务获取器缓存(并非服务对象)
    private static final HashMap<String, ServiceFetcher<?>> SYSTEM_SERVICE_FETCHERS =
            new HashMap<String, ServiceFetcher<?>>();
    
    static {
        ......
        // 注册 WINDOW_SERVICE
        registerService(
            Context.WINDOW_SERVICE,
            WindowManager.class,
            // 注册了一个实例获取器
            new CachedServiceFetcher<WindowManager>() {
                @Override
                public WindowManager createService(ContextImpl ctx) {
                    return new WindowManagerImpl(ctx);
                }
            }
        );
        ......
    }
    
    private static <T> void registerService(String serviceName, Class<T> serviceClass,
            ServiceFetcher<T> serviceFetcher) {
        // 添加到 服务名称 缓存
        SYSTEM_SERVICE_NAMES.put(serviceClass, serviceName);
        // 添加 服务获取器 的缓存
        SYSTEM_SERVICE_FETCHERS.put(serviceName, serviceFetcher);
    }
    
}
```
好的可以看到, 服务获取器的注册操作是在 SystemServiceRegistry 类的静态代码块中进行的, 由 JVM 的知识可知, 静态代码块的执行, 是在类加载过程中的初始化阶段进行的, 也就是说只要触发了 SystemServiceRegistry 类初始化, 便会先将所有的服务获取器注册完成
- 注册这个 WINDOW_SERVICE 它 new 了一个 CachedServiceFetcher 服务获取器, 并且**重写了其 createService 方法**

由上面 SystemServiceRegistry.getSystemService 中的代码可知, 它是使用 ServiceFetcher.getService 获取 WindowManager 对象的, 接下来我们再看看这个服务获取器, 是如何通过 ServiceFetcher.getService 获取我们需要的 WindowManager 对象的

### 2. CachedServiceFetcher.getService
```
final class SystemServiceRegistry {
    
    // 记录所有注册的缓存服务获取器的数量
    private static int sServiceCacheSize;
    
    static abstract class CachedServiceFetcher<T> implements ServiceFetcher<T> {
    
        // 构建的时候便会为这个 ServiceFetcher 创建一个 CacheIndex 代表其索引值
        private final int mCacheIndex;

        CachedServiceFetcher() {
            mCacheIndex = sServiceCacheSize++;
        }

        @Override
        @SuppressWarnings("unchecked")
        public final T getService(ContextImpl ctx) {
            // 从 ctx 中获取其中服务缓存数组(大小为 sServiceCacheSize)
            final Object[] cache = ctx.mServiceCache;
            // 从 ctx 中获取缓存的服务状态数组(大小为 sServiceCacheSize)
            final int[] gates = ctx.mServiceInitializationStateArray;

            for (;;) {
                boolean doInitialize = false;
                synchronized (cache) {
                    // 1. 尝试从 Context 的缓存中获取这个服务, 
                    // 我们知道这个 Context 是 Activity 创建时构建的一个 ContextImpl 对象
                    // 若是第一次调用 getService, ContextImpl 不存在这个缓存
                    T service = (T) cache[mCacheIndex];
                    if (service != null || gates[mCacheIndex] == ContextImpl.STATE_NOT_FOUND) {
                        // 1.1 若存在, 则直接将这个对象返回给调用方
                        return service;
                    }

                    // 1.2 如果 Context 缓存池中该服务的状态是 STATE_READY, 则说明之前创建过一次, 但实例被恶意清除了
                    if (gates[mCacheIndex] == ContextImpl.STATE_READY) {
                        // 将其置为 UNINITIALIZED 状态
                        gates[mCacheIndex] = ContextImpl.STATE_UNINITIALIZED;
                    }
                    // 1.3 将 doInitialize 置为 true, 状态变更为 INITIALIZING
                    if (gates[mCacheIndex] == ContextImpl.STATE_UNINITIALIZED) {
                        doInitialize = true;
                        gates[mCacheIndex] = ContextImpl.STATE_INITIALIZING;
                    }
                }
                // 2. 创建服务并且缓存到 Context 对象的缓存池中
                if (doInitialize) {
                    T service = null;
                    // 2.1 将这个服务的新状态初始化为 STATE_NOT_FOUND
                    int newState = ContextImpl.STATE_NOT_FOUND;
                    try {
                        // 2.2 这里调用了注册时重写的 createService, 便可拿到这个实例对象了
                        service = createService(ctx);
                        // 2.3 将这个服务的状态初始化为 STATE_READY
                        newState = ContextImpl.STATE_READY;
                    } catch (ServiceNotFoundException e) {
                        ......
                    } finally {
                        // 2.4 在 Context 的服务缓存数组中保存这个服务的缓存
                        synchronized (cache) {
                            cache[mCacheIndex] = service;
                            gates[mCacheIndex] = newState;
                            cache.notifyAll();
                        }
                    }
                    return service;
                }
                ......
            }
        }

        public abstract T createService(ContextImpl ctx) throws ServiceNotFoundException;
    }
}
```
好的, 可以看到非常有趣, 这个 CachedServiceFetcher 的 Cached 的含义, 并非是缓存在 SystemServiceRegistry 中, 而是**缓存 Context 对象的 mServiceCache 中**
- 也就是说这个 **WindowManger 并非是进程间唯一**的, 而**可能是 Context 对象中唯一**的(不恶意修改的情况下)

好的, 至此我们可以得出如下的结论
- 通过通过 context.getSystemService(Context.WINDOW_SERVICE) 获取到的对象时 WindowManagerImpl
  - 它并非系统服务进程提供的 WindowMangerService 代理对象
- 这个 WindowManagerImpl 并非进程间唯一的, **不同的 Context 通过 getSystemService 获取到的 WindowManagerImpl 实例对象都是不同的**

现在回过头来看看上面提出的问题: **为什么 Context.getSystemService(Context.WINDOW_SERVICE) 获取到的是 WindowManager 对象? 命名成 Context.WINDOW_MANAGER 不会更好吗?**
- 我认为在 Android 系统中, 各个应用进程虽然都依赖于系统服务进程提供的服务(AMS, WMS, PMS), 但这些都是 Android Framework 层的实现, 对于开发者来说是不可见的, 而且操作复杂, 因此 Android 在各个应用进程中抽象出了与系统服务进程交互的类, 与 WMS 交互的为 WindowManager(与 AMS 交互的为 ActivityManager......), **这些类屏蔽了与系统服务进行交互的细节, 对上层开发者提供服务, 因此可以理解为应用进程对上层开发者提供的服务**, 之所有会疑问, 我想可能是与系统服务进行的概念陷先入为主了

### 3. 结构图
![image](https://i.loli.net/2019/10/23/YR5SCeKWym2q1N4.png)

## 三. 构建新的 WindowManager
搞清楚了 WindowManager 是如何构建的, 这里再回过头来看看 WindowManagerImpl.createLocalWindowManager 创建新的实例的用意何在
```
public class Window {
    
    public void setWindowManager(WindowManager wm, IBinder appToken, String appName,
            boolean hardwareAccelerated) {
        // 如何获取 WindowManger 这里已经分析过了
        if (wm == null) {
            wm = (WindowManager)mContext.getSystemService(Context.WINDOW_SERVICE);
        }
        // 通过 WindowManager 的实例创建了一个新的 WindowManager 提供给自己使用
        mWindowManager = ((WindowManagerImpl)wm).createLocalWindowManager(this);
    }
    
}

public final class WindowManagerImpl implements WindowManager {
    // 管理 Window 真正的实现
    private final WindowManagerGlobal mGlobal = WindowManagerGlobal.getInstance();
    // 上下文
    private final Context mContext;
    // 描述 WindowManager 要管理的 Window 
    private final Window mParentWindow;
    
    // 调用了该方法重新创建了一个 WindowManager
    public WindowManagerImpl createLocalWindowManager(Window parentWindow) {
        // 这里创建时的 Context 沿用了当前对象的成员变量
        return new WindowManagerImpl(mContext, parentWindow);
    }

    // Constructor 1
    public WindowManagerImpl(Context context) {
        this(context, null);
    }
    
    // Constructor 2
    private WindowManagerImpl(Context context, Window parentWindow) {
        mContext = context;
        mParentWindow = parentWindow;
    }

}
```
好的, 可以看到 WindowManagerImpl.createLocalWindowManager 对象重新创建了一个 WindowManagerImpl 对象, **重新创建这一举动的用意何在呢? 直接给当前 WindowManagerImpl 的 mParentWindow 赋值这种方式不可行吗? **

好的, 难道是因为 Activity Window 的特殊性? 我们再看看 Dialog 中 Window 与 WindowManager 的绑定方式
```
public class Dialog implements DialogInterface, Window.Callback,
        KeyEvent.Callback, OnCreateContextMenuListener, Window.OnWindowDismissedCallback {
    
    Dialog(@NonNull Context context, @StyleRes int themeResId, boolean createContextThemeWrapper) {
        ......
        // 通过 context 获取其内部缓存的 WindowManager
        mWindowManager = (WindowManager) context.getSystemService(Context.WINDOW_SERVICE);
        // 创建了 Window 对象
        final Window w = new PhoneWindow(mContext);
        mWindow = w;
        // 绑定 WindowManager
        w.setWindowManager(mWindowManager, null, null);
    }       
}

public class Window {
    
    public void setWindowManager(WindowManager wm, IBinder appToken, String appName) {
        setWindowManager(wm, appToken, appName, false);
    } 
     
    public void setWindowManager(WindowManager wm, IBinder appToken, String appName,
            boolean hardwareAccelerated) {
        .......
        // 可以看到与 Activity 的 Window 绑定 WindowManager 一致
        if (wm == null) {
            wm = (WindowManager)mContext.getSystemService(Context.WINDOW_SERVICE);
        }
        mWindowManager = ((WindowManagerImpl)wm).createLocalWindowManager(this);
    }
}
```
好的, 可以看到 Dialog 中 Window 与 WindowManager 的绑定与 Activity 中是一致的, 那**不能使用 Context 缓存的 WindowManager 直接与我们新创建的 Window 绑定的原因是什么呢?**

好的, 假设我们使用 Context 缓存的 WindowManager 与我们 Activity 中创建的 Window 绑定, 此时我在这个 Activity 中弹出一个 Dialog, 它也会创建一个 Window, 创建 Dialog 时传入的 Context 显然是我们 Activity, 如果它也使用 Context 缓存的 WindowManager 与其创建的 Window 绑定, 那么会出现如下的情况

![错误的依赖](https://i.loli.net/2019/10/23/NknBb9tKLJuPrD4.png)


可以看到 Activity 中的 Window 虽然指向了 WindowManager, 但 WindowManager 中的 Window 已经是 Dialog 中的 Window 了, Activity 中 Window 调用相应的方法时, 会发现变更的居然是 Dialog 的 Window, 这...**显然是不合理的**

**因此为了保证每一个 Window 都有一个 WindowManager 对其进行管理, WindowManagerImpl 中的将相关变量置为了 final 类型, 每有一个 Window 创建, 便会为其创建一个 WindowManger**

## 四. 总结
通过这里的复习, 我们先理清了 Context 是如何缓存 WindowManger 的, **确定了 Window 与 WindowManager 一一对应的关系**, 为后续源码的分析打下了基础