---
title: Jetpack —— Navigation 导航组件
permalink: android-source-third-party/jetpack-navigation
key: android-source-third-party-jetpack-navigation
tags: AndroidJetpack
---

## 前言
最早接触这个组件是在 18 年 Google IO 大会上, 当时看到他们展示了可视化的导航图, 感觉挺有意思, 不过并没有弄明白它到底有何作用

直到最近接触到的新项目中使用了这个组件之后才明白它的妙用, **为了更好的使用和拓展 Navigation 的功能, 这里笔者分析一下它的实现原理**

## Navigation
关于 Navigation 的用法说明, 可以参考 [Android Developer 文档](https://developer.android.google.cn/guide/navigation/navigation-getting-started), Google 还提供一个学习的 [sample](https://github.com/android/sunflower/tree/master/app)

### 作用
体验下来之后, Navigation 可以帮助我们解决的问题如下
- 通过 id 进行 Activity/Fragment 的跳转
  - 减少了对类的耦合, 只需要关心 ID 即可
  - 也可以在此基础上进行拓展实现路由组件的功能, **注册过程即导航图的组合**
- 帮助我们管理 Fragment 跳转动画和回退栈, 让我们像使用 Activity 一样使用 Fragment
  - 为单 Activity 多 Fragment 的 app, 提供很好的技术支持

### 关于单 Activity 多 Fragment
很早之前就听过单 Activity 多 Fragment 的模式, 包括 Instagram 和 Twitter 等知名 app 都采用了这种方式? 到底有什么好处, 通过对 Android 系统架构深入了解之后, 对这个问题笔者也有了自己的看法
- Activity.attach 的时候会创建 Window, 同时会创建一个 ViewRootImpl, ViewRootImpl 中存在一个 Surface, 这个 Surface 初始化好后, 在 SurfaceFlinger 进程对应一个 Layer, 也就是说多 Activity 的应用会存在多个 Layer; 当 VSYNC 发出的时候, SF 会负责 Layer 的排序和合成, 当 Layer 较多的时候, 在一定程度上会增加 SF Compose 的负担, compose 超过 16ms 就无法达到 60 fps, 因此 Layer 较少时, 是可以在一定程度上增加我们 App 的流畅度的
- 并且每个 Activity 的启动都是需要和 SystemService 进行跨进程通信的, 而 Fragment 不需要, 理论上能够减少页面跳转的耗时

我们知道若想像使用 Activity 一样使用 Fragment 要面临的第一问题就是其回退栈的管理, 而 Navigation 组件就帮助我们解决了这个问题

这里我们就从一次 navigation 和 onBackPressed 返回来看看 Navigation 如何控制 Fragment 的启动和退出

## 一. 初始化
```
<fragment
    android:id="@+id/my_nav_host_fragment"
    android:name="androidx.navigation.fragment.NavHostFragment"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    app:defaultNavHost="true"
    app:navGraph="@navigation/mobile_navigation" />
```
想要使用 Navigation, 我们可以在主 Activity 的布局中, 添加这样的标签, 它指定了当前 Activity 内部有一个 NavHostFragment, 它的导航范围是 mobile_navigation, 下面我们看看 NavHostFragment 是如何初始化的

### 一. onCreate
```
public class NavHostFragment extends Fragment implements NavHost {

    private NavController mNavController;

    @NonNull
    @Override
    public NavController getNavController() {
        if (mNavController == null) {
            throw new IllegalStateException("NavController is not available before onCreate()");
        }
        return mNavController;
    }


    @Override
    public void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        final Context context = requireContext();
        // 1. 创建了一个导航控制器 NavController
        mNavController = new NavHostController(context);
        // 2. 注册生命周期回调
        mNavController.setLifecycleOwner(this);
        // 3. 注册返回事件分发器
        mNavController.setOnBackPressedDispatcher(requireActivity().getOnBackPressedDispatcher());
        // 4. 创建 Fragment 导航器, 将其添加到 NavController 中管理
        onCreateNavController(mNavController);
        ......
        if (navState != null) {
            // Navigation controller state overrides arguments
            mNavController.restoreState(navState);
        } else {
            final Bundle args = getArguments();
            // 5. 为 NavController 注入导航图的 resId
            final int graphId = args != null ? args.getInt(KEY_GRAPH_ID) : 0;
            if (graphId != 0) {
                mNavController.setGraph(graphId);
            } else {
                mNavController.setMetadataGraph();
            }
        }
    }

    protected void onCreateNavController(@NonNull NavController navController) {
        // 添加 DialogFragment 和  Fragment 的导航器
        navController.getNavigatorProvider().addNavigator(
                new DialogFragmentNavigator(requireContext(), getChildFragmentManager()));
        navController.getNavigatorProvider().addNavigator(createFragmentNavigator());
    }

}
```
从 NavHostFragment 的继承关系上可以看到, 它实现了 NavHost 接口, 也就是说它内部持有了 NavController, 它的 onCreate 方法主要工作是进行 NavController 的构建和初始化
- 创建了导航控制器 NavHostController, 它是一个 NavController 的包装类
- 注册生命周期回调
- 注册返回事件分发器
- 回调 onCreateNavController, 为 NavController 注入 Fragment/DialogFragment 导航器
- 为 NavController 注入导航图

这里我们先关注一下 NavController 的**实例化过程**, 关于**导航器的功能实现**和**返回事件的分发**, 我们到后面再分析

#### NavController 的实例化
```
public class NavController {

    // 导航地址的回退栈
    final Deque<NavDestination> mBackStack = new ArrayDeque<>();
    // 导航提供器
    private final SimpleNavigatorProvider mNavigatorProvider = new SimpleNavigatorProvider() {
        @Nullable
        @Override
        public Navigator<? extends NavDestination> addNavigator(@NonNull String name,
                @NonNull Navigator<? extends NavDestination> navigator) {
            Navigator<? extends NavDestination> previousNavigator =
                    super.addNavigator(name, navigator);
            if (previousNavigator != navigator) {
                // 清除之前 name 对应的 Navigator 的监听器
                if (previousNavigator != null) {
                    previousNavigator.removeOnNavigatorNavigatedListener(mOnNavigatedListener);
                }
                // 添加新 Navigator 的监听器
                navigator.addOnNavigatorNavigatedListener(mOnNavigatedListener);
            }
            return previousNavigator;
        }
    };

    public NavController(@NonNull Context context) {
        mContext = context;
        while (context instanceof ContextWrapper) {
            if (context instanceof Activity) {
                mActivity = (Activity) context;
                break;
            }
            context = ((ContextWrapper) context).getBaseContext();
        }
        // 添加一个 NavGraphNavigator
        mNavigatorProvider.addNavigator(new NavGraphNavigator(mContext));
        // 添加一个 ActivityNavigator
        mNavigatorProvider.addNavigator(new ActivityNavigator(mContext));
    }

}
```
NavController 的从命名上来看是导航控制器, 也确实如此, 其内部的持有一个 NavigatorProvider 用于缓存和提供导航器

在它的构造函数中可以看到, 它分别添加了 NavGraphNavigator 和 ActivityNavigator 两个 Navigator, 在加上之前看到的 FragmentNavigator 一共出现了三个导航器, 它们负责具体业务场景的导航实现, 我们在使用到时再具体分析

### 二) 回顾
![导航结构依赖](https://user-gold-cdn.xitu.io/2020/3/19/170f1741c8a4b5d0?w=682&h=507&f=png&s=36575)

到这里可以看到它的 Navigation 的结构设计就比较清晰了
- NavController: 为上层提供使用接口, 提供导航服务和栈的维护
- NavigationProvider: 维护导航器
- Navigator: 导航器, 负责具体的跳转和退出
  - ActivityNavigator
  - FragmentNavigator
  - GraphNavigator

了解设计结构之后, 接下来看看具体的 navigate 操作

## 二. 导航操作
```
public class NavController {

    public void navigate(@IdRes int resId, @Nullable Bundle args, @Nullable NavOptions navOptions,
            @Nullable Navigator.Extras navigatorExtras) {
        // 1. 获取当前栈顶元素, 不存在则使用导航图
        NavDestination currentNode = mBackStack.isEmpty()
                ? mGraph
                : mBackStack.getLast().getDestination();
        if (currentNode == null) {
            throw new IllegalStateException("no current navigation node");
        }
        // 2. 根据 resId 获取其对应的导航动作
        @IdRes int destId = resId;
        final NavAction navAction = currentNode.getAction(resId);
        Bundle combinedArgs = null;
        if (navAction != null) {
            // 2.1 从 NavAction 读取导航配置选项
            if (navOptions == null) {
                navOptions = navAction.getNavOptions();
            }
            // 2.2 获取目的地的 ID
            destId = navAction.getDestinationId();
            // 2.3 注入通用参数
            Bundle navActionArgs = navAction.getDefaultArguments();
            if (navActionArgs != null) {
                combinedArgs = new Bundle();
                combinedArgs.putAll(navActionArgs);
            }
        }

        if (args != null) {
            if (combinedArgs == null) {
                combinedArgs = new Bundle();
            }
            combinedArgs.putAll(args);
        }
        // 3. 若只存在 popUp 的 ID, 则执行出栈操作
        if (destId == 0 && navOptions != null && navOptions.getPopUpTo() != -1) {
            // 执行出栈逻辑
            popBackStack(navOptions.getPopUpTo(), navOptions.isPopUpToInclusive());
            return;
        }

        if (destId == 0) {
            throw new IllegalArgumentException("Destination id == 0 can only be used"
                    + " in conjunction with a valid navOptions.popUpTo");
        }

        // 4. 找寻目的地, 执行入栈逻辑
        NavDestination node = findDestination(destId);
        if (node == null) {
            final String dest = NavDestination.getDisplayName(mContext, destId);
            throw new IllegalArgumentException("navigation destination " + dest
                    + (navAction != null
                    ? " referenced from action " + NavDestination.getDisplayName(mContext, resId)
                    : "")
                    + " is unknown to this NavController");
        }
        // 执行入栈
        navigate(node, combinedArgs, navOptions, navigatorExtras);
    }

}
```
一次导航的流程具体事宜如下
- 获取栈顶的元素
- 获取 id 对应的 NavAction 对象, 描述本次导航动作
  - NavAction 就是我们在导航图的 xml 中定义的 <action/> 标签元素信息
- 若只存在 PopUpId, 则调用 popBackStack 执行出栈操作
- 存在 destId, 则调用重载方法 navigate, 执行入栈操作

接下来我们看看入栈的操作

### 一) 入栈
```
public class NavController {

    private void navigate(@NonNull NavDestination node, @Nullable Bundle args,
            @Nullable NavOptions navOptions, @Nullable Navigator.Extras navigatorExtras) {
        ......

        // 1. 根据 node 的 Name 获取对应的导航器
        Navigator<NavDestination> navigator = mNavigatorProvider.getNavigator(
                node.getNavigatorName());
        Bundle finalArgs = node.addInDefaultArgs(args);
        // 2. 导航到 node 对应的目的地去
        NavDestination newDest = navigator.navigate(node, finalArgs,
                navOptions, navigatorExtras);
        // 3. 导航成功, 更新栈元素
        if (newDest != null) {
            ......
            // 3.1 向回退栈中添加一个 mGraph 类型的元素, 保证它在底部
            if (mBackStack.isEmpty()) {
                mBackStack.add(new NavBackStackEntry(mGraph, finalArgs, mViewModel));
            }
            // 3.2 将新目的地的 parent 添加到栈内
            ArrayDeque<NavBackStackEntry> hierarchy = new ArrayDeque<>();
            NavDestination destination = newDest;
            while (destination != null && findDestination(destination.getId()) == null) {
                NavGraph parent = destination.getParent();
                if (parent != null) {
                    hierarchy.addFirst(new NavBackStackEntry(parent, finalArgs, mViewModel));
                }
                destination = parent;
            }
            mBackStack.addAll(hierarchy);
            // 3.3 将新的目的地添加到栈顶
            NavBackStackEntry newBackStackEntry = new NavBackStackEntry(newDest,
                    newDest.addInDefaultArgs(finalArgs), mViewModel);
            mBackStack.add(newBackStackEntry);
        }
        updateOnBackPressedCallbackEnabled();
        if (popped || newDest != null) {
            dispatchOnDestinationChanged();
        }
    }

}
```
入栈的操作还是比较清晰的, 主要有如下三个步骤
- 根据目的地的信息获取其对应的 Navigator
- 调用 Navigator.navigate 导航到对应的页面去
- 若存在 newDest 则说明是 Fragment 类型的导航操作, 添加到 mBackStack 栈中维护

ActivityNavigator 实现类导航完成之后返回 null, 因为 Activity 的栈由 AMS 维护, 的确不需要我们再手动维护一遍了, 因此这里我们选取 FragmentNavigator 看看它的实现

#### FragmentNavigator 入栈操作
```
public class FragmentNavigator extends Navigator<FragmentNavigator.Destination> {

    private final Context mContext;
    private final FragmentManager mFragmentManager;
    private final int mContainerId;
    private ArrayDeque<Integer> mBackStack = new ArrayDeque<>();

    public FragmentNavigator(@NonNull Context context, @NonNull FragmentManager manager,
            int containerId) {
        mContext = context;
        mFragmentManager = manager;
        // 容器的 id
        mContainerId = containerId;
    }

    @Override
    public void navigate(@NonNull Destination destination, @Nullable Bundle args,
                            @Nullable NavOptions navOptions) {
        // 创建 Fragment 实例对象
        final Fragment frag = destination.createFragment(args);
        // 构建 Transaction
        final FragmentTransaction ft = mFragmentManager.beginTransaction();
        // 获取 Fragment 的跳转动画
        int enterAnim = navOptions != null ? navOptions.getEnterAnim() : -1;
        int exitAnim = navOptions != null ? navOptions.getExitAnim() : -1;
        int popEnterAnim = navOptions != null ? navOptions.getPopEnterAnim() : -1;
        int popExitAnim = navOptions != null ? navOptions.getPopExitAnim() : -1;
        if (enterAnim != -1 || exitAnim != -1 || popEnterAnim != -1 || popExitAnim != -1) {
            enterAnim = enterAnim != -1 ? enterAnim : 0;
            exitAnim = exitAnim != -1 ? exitAnim : 0;
            popEnterAnim = popEnterAnim != -1 ? popEnterAnim : 0;
            popExitAnim = popExitAnim != -1 ? popExitAnim : 0;
            ft.setCustomAnimations(enterAnim, exitAnim, popEnterAnim, popExitAnim);
        }

        ft.replace(mContainerId, frag);

        final @IdRes int destId = destination.getId();
        final boolean initialNavigation = mBackStack.isEmpty();
        final boolean isClearTask = navOptions != null && navOptions.shouldClearTask();
        // 判断是否为 singleTop 的模式
        final boolean isSingleTopReplacement = navOptions != null && !initialNavigation
                && navOptions.shouldLaunchSingleTop()
                && mBackStack.peekLast() == destId;

        int backStackEffect;
        if (initialNavigation || isClearTask) {
            backStackEffect = BACK_STACK_DESTINATION_ADDED;
        } else if (isSingleTopReplacement) {
            // 处理 SingleTop
            if (mBackStack.size() > 1) {
                // If the Fragment to be replaced is on the FragmentManager's
                // back stack, a simple replace() isn't enough so we
                // remove it from the back stack and put our replacement
                // on the back stack in its place
                mFragmentManager.popBackStack();
                ft.addToBackStack(getBackStackName(destId));
                mPendingBackStackOperations++;
            }
            backStackEffect = BACK_STACK_UNCHANGED;
        } else {
            ft.addToBackStack(getBackStackName(destId));
            mPendingBackStackOperations++;
            backStackEffect = BACK_STACK_DESTINATION_ADDED;
        }
        ft.setReorderingAllowed(true);
        ft.commit();
        // The commit succeeded, update our view of the world
        if (backStackEffect == BACK_STACK_DESTINATION_ADDED) {
            mBackStack.add(destId);
        }
        dispatchOnNavigatorNavigated(destId, backStackEffect);
    }


}
```
FragmentNavigator 的构造我们在上面分析 NavController 启动的时候看到了, 它传入的是 NavHostFragment 的 childFragmentManager, 其 navigate 操作即对 FragmentTransaction 的 replace 操作

### 二) 出栈
```
public class NavController {

    public boolean popBackStack(@IdRes int destinationId, boolean inclusive) {
        boolean popped = popBackStackInternal(destinationId, inclusive);
        // Only return true if the pop succeeded and we've dispatched
        // the change to a new destination
        return popped && dispatchOnDestinationChanged();
    }

    boolean popBackStackInternal(@IdRes int destinationId, boolean inclusive) {
        if (mBackStack.isEmpty()) {
            // Nothing to pop if the back stack is empty
            return false;
        }
        ArrayList<Navigator> popOperations = new ArrayList<>();
        // 1. 遍历回退栈, 记录要出栈元素的 Navigator
        Iterator<NavBackStackEntry> iterator = mBackStack.descendingIterator();
        boolean foundDestination = false;
        while (iterator.hasNext()) {
            NavDestination destination = iterator.next().getDestination();
            // 获取要出栈元素对应的 Navigator.
            Navigator navigator = mNavigatorProvider.getNavigator(
                    destination.getNavigatorName());
            // 1.1 记录要出栈的元素的 Navigator.
            if (inclusive || destination.getId() != destinationId) {
                popOperations.add(navigator);
            }
            // 找到与目标 id 相同的元素时终止
            if (destination.getId() == destinationId) {
                foundDestination = true;
                break;
            }
        }
        // 没有找到, 处理异常情况
        if (!foundDestination) {
            ......
            return false;
        }
        // 2. 使用要出栈元素对应的 Navigator 执行其 popBackStack 方法
        boolean popped = false;
        for (Navigator navigator : popOperations) {
            // 2.1 调用 navigator 的 popBackStack
            if (navigator.popBackStack()) {
                // 从 mBackStack 中移除
                NavBackStackEntry entry = mBackStack.removeLast();
                ......
                popped = true;
            } else {
                // The pop did not complete successfully, so stop immediately
                break;
            }
        }
        updateOnBackPressedCallbackEnabled();
        return popped;
    }


}
```
可以看到 popUp 的操作有些类似于 Activity 的 singleTask, 若是之前存在, 则会将其栈顶所有的页面清除, 主要操作如下
- 遍历 NavController 的回退栈, 记录要出栈元素的 Navigator 到 popOperations 中
- 调用 Navigator.popBackStack 执行出栈操作&将 mBackStack 的最后一个元素移除

这里我们同样看看 FragmentNavigator 的实现

#### FragmentNavigator 出栈操作
```
public class FragmentNavigator extends Navigator<FragmentNavigator.Destination> {

   @Override
    public boolean popBackStack() {
        if (mBackStack.isEmpty()) {
            return false;
        }
        if (mFragmentManager.isStateSaved()) {
            Log.i(TAG, "Ignoring popBackStack() call: FragmentManager has already"
                    + " saved its state");
            return false;
        }
        mFragmentManager.popBackStack(
                generateBackStackName(mBackStack.size(), mBackStack.peekLast()),
                FragmentManager.POP_BACK_STACK_INCLUSIVE);
        mBackStack.removeLast();
        return true;
    }

}
```
FragmentNavigator 的出栈操作比入栈操作要简明的多, 直接调用了 mFragmentManager.popBackStack 将目标元素移除

### 三) 回顾
一次导航的流程具体事宜如下
- 获取栈顶的元素
- 获取 id 对应的 NavAction 对象, 描述本次导航动作
  - NavAction 就是我们在导航图的 xml 中定义的 <action/> 标签元素信息
- 若只存在 PopUpId, 则调用 popBackStack 执行**出栈操作**
  - 遍历 NavController 的回退栈, 记录要出栈元素的 Navigator 到 popOperations 中
  - 调用 Navigator.popBackStack 执行出栈操作&将 mBackStack 的最后一个元素移除
    - FragmentNavigator 直接调用了 mFragmentManager.popBackStack 将目标元素移除
- 存在 destId, 则调用重载方法 navigate, 执行**入栈操作**
  - 根据目的地的信息获取其对应的 Navigator
  - 调用 Navigator.navigate 导航到对应的页面去
    - FragmentNavigator 的 navigate 操作即对 FragmentTransaction 的 replace 操作
  - 若存在 newDest 则说明是 Fragment 类型的导航操作, 添加到 mBackStack 栈中维护

至此导航组件的入栈和出栈操作就介绍完了, 下面我们看看 Navigation 是如何分发返回事件的

## 三. 返回事件的分发
在初始化的时候我们看到 NavHostFragment 中注册了 onBackPressed 的监听器
```
public class NavHostFragment extends Fragment implements NavHost {

    @Override
    public void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        ......
        // 1. requireActivity().getOnBackPressedDispatcher() 获取分发器
        // 2. 交由 mNavController 注册回调
        mNavController.setOnBackPressedDispatcher(requireActivity().getOnBackPressedDispatcher());
        ......
    }

}

public class NavController {

    private final OnBackPressedCallback mOnBackPressedCallback =
            new OnBackPressedCallback(false) {
        @Override
        public void handleOnBackPressed() {
            popBackStack();
        }
    };

    void setOnBackPressedDispatcher(@NonNull OnBackPressedDispatcher dispatcher) {
        ......
        // 它为 Activity 的 mBackPressedCallback 注册了一个回调
        dispatcher.addCallback(mLifecycleOwner, mOnBackPressedCallback);
    }

}
```
NavController 在 NavHostFragment 中初始化的时候, 便会请求获取 Activity 的返回事件分发器, 然后 NavController 便会注册一个回调, 用于响应分发器

我们先看看 OnBackPressedDispatcher 添加回调的过程

### 一) 添加监听回调
```
public final class OnBackPressedDispatcher {

    @MainThread
    public void addCallback(@NonNull LifecycleOwner owner,
            @NonNull OnBackPressedCallback onBackPressedCallback) {
        Lifecycle lifecycle = owner.getLifecycle();
        if (lifecycle.getCurrentState() == Lifecycle.State.DESTROYED) {
            return;
        }
        // 为 onBackPressedCallback.addCancellable
        onBackPressedCallback.addCancellable(
                new LifecycleOnBackPressedCancellable(lifecycle, onBackPressedCallback));
    }

}
```
这里的实现比较有意思, 它并没有直接将我们传入的 onBackPressedCallback 添加到一个回调队列, 而是先创建了 LifecycleOnBackPressedCancellable, 然后调用了 onBackPressedCallback.addCancellable 方法

下面分别看看他们的实现

#### 1. LifecycleOnBackPressedCancellable 的创建
```
public final class OnBackPressedDispatcher {

    private class LifecycleOnBackPressedCancellable implements LifecycleEventObserver,
            Cancellable {
        private final Lifecycle mLifecycle;
        private final OnBackPressedCallback mOnBackPressedCallback;

        @Nullable
        private Cancellable mCurrentCancellable;

        LifecycleOnBackPressedCancellable(@NonNull Lifecycle lifecycle,
                @NonNull OnBackPressedCallback onBackPressedCallback) {
            mLifecycle = lifecycle;
            mOnBackPressedCallback = onBackPressedCallback;
            lifecycle.addObserver(this);
        }

        @Override
        public void onStateChanged(@NonNull LifecycleOwner source,
                @NonNull Lifecycle.Event event) {
            if (event == Lifecycle.Event.ON_START) {
                // 1. 当 onStart 回调之后, 调用外部类的 addCancellableCallback 将 OnBackPressedCallback 注入 mOnBackPressedCallbacks 队列
                mCurrentCancellable = addCancellableCallback(mOnBackPressedCallback);
            } else if (event == Lifecycle.Event.ON_STOP) {
                // Should always be non-null
                if (mCurrentCancellable != null) {
                    mCurrentCancellable.cancel();
                }
            } else if (event == Lifecycle.Event.ON_DESTROY) {
                // 2. onDestroy 时, 自动移除从 mOnBackPressedCallbacks 队列中移除
                cancel();
            }
        }


        @Override
        public void cancel() {
            mLifecycle.removeObserver(this);
            mOnBackPressedCallback.removeCancellable(this);
            if (mCurrentCancellable != null) {
                mCurrentCancellable.cancel();
                mCurrentCancellable = null;
            }
        }

    }

    final ArrayDeque<OnBackPressedCallback> mOnBackPressedCallbacks = new ArrayDeque<>();

    private class OnBackPressedCancellable implements Cancellable {

        private final OnBackPressedCallback mOnBackPressedCallback;

        OnBackPressedCancellable(OnBackPressedCallback onBackPressedCallback) {
            mOnBackPressedCallback = onBackPressedCallback;
        }

        @Override
        public void cancel() {
            // 移除这个 OnBackPressedCallback
            mOnBackPressedCallbacks.remove(mOnBackPressedCallback);
            // 移除 Cancelable
            mOnBackPressedCallback.removeCancellable(this);
        }
    }

    Cancellable addCancellableCallback(@NonNull OnBackPressedCallback onBackPressedCallback) {
        // 1.1 将 onBackPressedCallback 添加到 OnBackPressedDispatcher 的队列中
        mOnBackPressedCallbacks.add(onBackPressedCallback);
        // 1.2 创建了一个 OnBackPressedCancellable 添加到 onBackPressedCallback 中缓存
        // 因为 mOnBackPressedCallbacks 对 OnBackPressedCallback 是不可见的, 因此这个 Cancelable 的意义是帮助 OnBackPressedCancellable 主动从 mOnBackPressedCallbacks 中移除
        OnBackPressedCancellable cancellable = new OnBackPressedCancellable(onBackPressedCallback);
        onBackPressedCallback.addCancellable(cancellable);
        return cancellable;
    }

}
```
LifecycleOnBackPressedCancellable 是 OnBackPressedDispatcher 的内部类
- 当 onStart 回调时才会将 OnBackPressedCallback 投递到外部类的 mOnBackPressedCallbacks 中用于后续分发使用
- 当 onDestroy 回调时, 会将这个 OnBackPressedCallback 从 mOnBackPressedCallbacks 中移除

使用这种方式可以实现跟随生命周期自动化的管理 OnBackPressedCallback 回调的添加与释放

#### 2. 为 onBackPressedCallback 添加 Cancelable
```
public abstract class OnBackPressedCallback {

    private boolean mEnabled;
    private CopyOnWriteArrayList<Cancellable> mCancellables = new CopyOnWriteArrayList<>();

    void addCancellable(@NonNull Cancellable cancellable) {
        mCancellables.add(cancellable);
    }

}
```

下面我们就看看 Activity 是如何把返回事件分发到 NavController.mOnBackPressedCallback 的

![分发与回调的依赖图](https://user-gold-cdn.xitu.io/2020/3/19/170f173ea1e413ff?w=2048&h=1048&f=png&s=128043)

### 二) 返回事件分发流程
```
public class ComponentActivity extends androidx.core.app.ComponentActivity implements
        LifecycleOwner,
        ViewModelStoreOwner,
        SavedStateRegistryOwner,
        OnBackPressedDispatcherOwner {

    private final OnBackPressedDispatcher mOnBackPressedDispatcher =
            new OnBackPressedDispatcher(new Runnable() {
                @Override
                public void run() {
                    ComponentActivity.super.onBackPressed();
                }
            });

    @Override
    @MainThread
    public void onBackPressed() {
        mOnBackPressedDispatcher.onBackPressed();
    }

    @NonNull
    @Override
    public final OnBackPressedDispatcher getOnBackPressedDispatcher() {
        return mOnBackPressedDispatcher;
    }

}
```
返回事件交由 OnBackPressedDispatcher 分发在 ComponentActivity 提供了支持, 其 onBackPressed 中的实现代理给了 OnBackPressedDispatcher 的 onBackPressed 方法

```
public final class OnBackPressedDispatcher {

    @Nullable
    private final Runnable mFallbackOnBackPressed;

    @SuppressWarnings("WeakerAccess") /* synthetic access */
    final ArrayDeque<OnBackPressedCallback> mOnBackPressedCallbacks = new ArrayDeque<>();

    public OnBackPressedDispatcher() {
        this(null);
    }

    public OnBackPressedDispatcher(@Nullable Runnable fallbackOnBackPressed) {
        // 由 ComponentActivity 中的构造可知, 它就是 Activity 的 onBackPressed 的最终实现
        mFallbackOnBackPressed = fallbackOnBackPressed;
    }

    @MainThread
    public void onBackPressed() {
        // 1. 倒序遍历 mOnBackPressedCallbacks 队列
        Iterator<OnBackPressedCallback> iterator =
                mOnBackPressedCallbacks.descendingIterator();
        while (iterator.hasNext()) {
            OnBackPressedCallback callback = iterator.next();
            // 2. 将这个事件分发给第一个 enable 的 OnBackPressedCallback
            if (callback.isEnabled()) {
                callback.handleOnBackPressed();
                return;
            }
        }
        // 3. 若是不存在, 则交由 Activity 的 onBackPressed 处理
        if (mFallbackOnBackPressed != null) {
            mFallbackOnBackPressed.run();
        }
    }

    ......
}
```
分发流程比起监听器的添加流程要更为清晰, 主要步骤如下
- 倒序遍历监听回调队列 mOnBackPressedCallbacks
- 将这个事件分发给第一个 enable 的 OnBackPressedCallback
- 若是不存在, 则交由 Activity 的 onBackPressed 处理

OnBackPressedCallback 的 handleOnBackPressed 的实现, 我们在 NavController 中已经看到了, 即调用 popBackStack() 执行出栈操作这里就不再赘述了

### 三) 回顾
导航事件的分发, 由 ComponentActivity 中的 OnBackPressedDispatcher 提供支持
- 监听器回调的注册
  -  onBackPressedCallback 描述一个监听事件, 会将它注入到 OnBackPressedDispatcher.mOnBackPressedCallbacks 队列中
  -  Cancelable 用于帮助 onBackPressedCallback 将它从 OnBackPressedDispatcher.mOnBackPressedCallbacks 中移除
- 事件的分发
  - 倒序遍历监听回调队列 mOnBackPressedCallbacks
  - 将这个事件分发给第一个 enable 的 OnBackPressedCallback
  - 若是不存在, 则交由 Activity 的 onBackPressed 处理
- 事件的处理
  - 在 NavController 中实现, 即调用 popBackStack 执行出栈操作

## 总结
### 导航初始化
![导航结构依赖](https://user-gold-cdn.xitu.io/2020/3/19/170f1741c8a4b5d0?w=682&h=507&f=png&s=36575)

Navigation 各个组件的功能如下所示
- NavController: 为上层提供使用接口, 提供导航服务和栈的维护
- NavigationProvider: 维护导航器
- Navigator: 导航器, 负责具体的跳转和退出
  - ActivityNavigator
  - FragmentNavigator
  - GraphNavigator

### 导航的入栈与出栈
一次导航的流程具体事宜如下
- 获取栈顶的元素
- 获取 id 对应的 NavAction 对象, 描述本次导航动作
  - NavAction 就是我们在导航图的 xml 中定义的 <action/> 标签元素信息
- 若只存在 PopUpId, 则调用 popBackStack 执行**出栈操作**
  - 遍历 NavController 的回退栈, 记录要出栈元素的 Navigator 到 popOperations 中
  - 调用 Navigator.popBackStack 执行出栈操作&将 mBackStack 的最后一个元素移除
    - FragmentNavigator 直接调用了 mFragmentManager.popBackStack 将目标元素移除
- 存在 destId, 则调用重载方法 navigate, 执行**入栈操作**
  - 根据目的地的信息获取其对应的 Navigator
  - 调用 Navigator.navigate 导航到对应的页面去
    - FragmentNavigator 的 navigate 操作即对 FragmentTransaction 的 replace 操作
  - 若存在 newDest 则说明是 Fragment 类型的导航操作, 添加到 mBackStack 栈中维护

### 返回事件的分发
![分发与回调的依赖图](https://user-gold-cdn.xitu.io/2020/3/19/170f173ea1e413ff?w=2048&h=1048&f=png&s=128043)

导航事件的分发, 由 ComponentActivity 中的 OnBackPressedDispatcher 提供支持
- 监听器回调的注册
  -  onBackPressedCallback 描述一个监听事件, 会将它注入到 OnBackPressedDispatcher.mOnBackPressedCallbacks 队列中
  -  Cancelable 用于帮助 onBackPressedCallback 将它从 OnBackPressedDispatcher.mOnBackPressedCallbacks 中移除
- 事件的分发
  - 倒序遍历监听回调队列 mOnBackPressedCallbacks
  - 将这个事件分发给第一个 enable 的 OnBackPressedCallback
  - 若是不存在, 则交由 Activity 的 onBackPressed 处理
- 事件的处理
  - 在 NavController 中实现, 即调用 popBackStack 执行出栈操作

## 结语
Navigation 组件虽然接触的比较早很早, 但迟迟没有一个合适的契机让自己花功夫去看, 这次通读了它的源码也算是完成了自己的一个 P2 的任务吧

整体收获还是不小的, 至少在技术架构选取上又多了一个选择, 若是使用的话也更方便后续对 Navigation 的定制和拓展
