---
title: MVVM 架构演进 —— DataBinding 实现原理
tags: Framework
---

## 前言
我们知道 DataBinding 在 MVVM 架构中起到了不可或缺的作用, 它是削弱 View 层与 ViewModel 之间耦合的重中之重, 那么它是**如何做到当数据变更时便能够直接推送到 View 中的呢?**

我们带着这个问题去探索一下 DataBinding 的工作流程.

<!--more-->

## 一. ViewDataBinding 的创建
```
class MainFragment : Fragment() {

    ......
    override fun onCreateView(
        inflater: LayoutInflater, container: ViewGroup?,
        savedInstanceState: Bundle?
    ): View {
        // 调用了 MainFragmentBinding.inflate 获取到了 MainFragmentBinding 对象
        val dataBinding: MainFragmentBinding = MainFragmentBinding.inflate(inflater, container, false)
        dataBinding.view = this
        dataBinding.viewmodel = ViewModelProviders.of(this).get(MainViewModel::class.java)
        return dataBinding.root
    }

}
```
ViewBinding 的实例化是通过生成类 MainFragmentBinding.inflate 完成的, 我们继续追踪
```
public abstract class MainFragmentBinding extends ViewDataBinding {
  
  @NonNull
  public static MainFragmentBinding inflate(@NonNull LayoutInflater inflater,
      @Nullable ViewGroup root, boolean attachToRoot) {
    // 1. 调用 DataBindingUtil.getDefaultComponent 获取了默认的 DataBindingComponent 对象, 为空
    // 调用了重载方法
    return inflate(inflater, root, attachToRoot, DataBindingUtil.getDefaultComponent());
  }

  @NonNull
  public static MainFragmentBinding inflate(@NonNull LayoutInflater inflater,
      @Nullable ViewGroup root, boolean attachToRoot, @Nullable DataBindingComponent component) {
    // 将任务分发给 DataBindingUtil.inflate
    return DataBindingUtil.<MainFragmentBinding>inflate(
             inflater,
             com.sharry.demo.mvvm.R.layout.main_fragment, 
             root, 
             attachToRoot, 
             component
           );
  }
    
}
```
好的, 可以看到 MainFragmentBinding.inflate 会转交给 DataBindUtil 去执行 inflate 操作, 这里已经可以看到这个 ViewDataBinding 对应的 xml 布局文件了

接下来我们看看这个 DataBindUtil.inflate 做了什么操作
```
public class DataBindingUtil {
    
    public static <T extends ViewDataBinding> T inflate(
            @NonNull LayoutInflater inflater, int layoutId, @Nullable ViewGroup parent,
            boolean attachToParent, @Nullable DataBindingComponent bindingComponent) {
        // 判断是否作为子孩子填充到父容器
        final boolean useChildren = parent != null && attachToParent;
        // 判断插入起始位置
        final int startChildren = useChildren ? parent.getChildCount() : 0;
        // 1. 构造视图
        final View view = inflater.inflate(layoutId, parent, attachToParent);
        // 2. 执行视图的绑定
        if (useChildren) {
            // 与新插入的的视图进行绑定
            return bindToAddedViews(bindingComponent, parent, startChildren, layoutId);
        } else {
            // 与新创建的 View 进行绑定
            return bind(bindingComponent, view, layoutId);
        }
    }
    
    private static <T extends ViewDataBinding> T bindToAddedViews(DataBindingComponent component,
            ViewGroup parent, int startChildren, int layoutId) {
        // 记录当前父容器子 View 的数量
        final int endChildren = parent.getChildCount();
        // 计算新插入的数量(一般为 1, 但新插入的视图为 <merge></merge> 标签就不好说了)
        final int childrenAdded = endChildren - startChildren;
        if (childrenAdded == 1) {
            // 2.1 与单 view 进行绑定
            final View childView = parent.getChildAt(endChildren - 1);
            return bind(component, childView, layoutId);
        } else {
            // 2.2 与多 view 进行绑定
            final View[] children = new View[childrenAdded];
            for (int i = 0; i < childrenAdded; i++) {
                children[i] = parent.getChildAt(i + startChildren);
            }
            return bind(component, children, layoutId);
        }
    }
    
    private static DataBinderMapper sMapper = new DataBinderMapperImpl();
    
    // 执行多 view 的绑定
    static <T extends ViewDataBinding> T bind(DataBindingComponent bindingComponent, View[] roots,
            int layoutId) {
        return (T) sMapper.getDataBinder(bindingComponent, roots, layoutId);
    }

    // 执行单 View 的绑定
    static <T extends ViewDataBinding> T bind(DataBindingComponent bindingComponent, View root,
            int layoutId) {
        return (T) sMapper.getDataBinder(bindingComponent, root, layoutId);
    }
    
}
```
好的, 可以看到 DataBindingUtil.inflate 的主要操作有两个
- 实例化 xml 布局为 View
- 对新的 view 构建执行绑定, 获取其对应的 ViewDataBinding 对象

好的, 接下来我们看看他是如何获取 ViewDataBinding 的
```
// 这个 DataBinderMapperImpl 是 AndroidX 依赖包中的类
public class DataBinderMapperImpl extends MergedDataBinderMapper {
  DataBinderMapperImpl() {
    // 这个 DataBinderMapperImpl 是编译时的生成类
    addMapper(new com.sharry.demo.mvvm.DataBinderMapperImpl());
  }
}

public class MergedDataBinderMapper extends DataBinderMapper {
    
    private List<DataBinderMapper> mMappers = new CopyOnWriteArrayList<>();

    @Override
    public ViewDataBinding getDataBinder(DataBindingComponent bindingComponent, View view,
            int layoutId) {
        // 在 DataBinderMapperImpl 中添加了一个 DataBinderMapperImpl 对象
        for(DataBinderMapper mapper : mMappers) {
            // 因此这里调用的是 DataBinderMapperImpl.getDataBinder
            ViewDataBinding result = mapper.getDataBinder(bindingComponent, view, layoutId);
            if (result != null) {
                return result;
            }
        }
        ......
        return null;
    }

}

public class DataBinderMapperImpl extends DataBinderMapper {

  private static final int LAYOUT_MAINFRAGMENT = 1;
  private static final SparseIntArray INTERNAL_LAYOUT_ID_LOOKUP = new SparseIntArray(1);

  static {
    // 静态代码块中, 添加了我们使用了 databinding 的布局
    INTERNAL_LAYOUT_ID_LOOKUP.put(com.sharry.demo.mvvm.R.layout.main_fragment, LAYOUT_MAINFRAGMENT);
  }

  @Override
  public ViewDataBinding getDataBinder(DataBindingComponent component, View view, int layoutId) {
    int localizedLayoutId = INTERNAL_LAYOUT_ID_LOOKUP.get(layoutId);
    if(localizedLayoutId > 0) {
      // 使用 DataBinding 的 xml 要求使用 <layout></layout> 标签
      // 因此 DataBinding 的 xml 解析器会为内部的 view 注入一个 TAG
      final Object tag = view.getTag();
      ......
      switch(localizedLayoutId) {
        case  LAYOUT_MAINFRAGMENT: {
          if ("layout/main_fragment_0".equals(tag)) {
            // 返回了一个 MainFragmentBindingImpl 对象, 这也是编译时生成的
            return new MainFragmentBindingImpl(component, view);
          }
          throw new IllegalArgumentException("The tag for main_fragment is invalid. Received: " + tag);
        }
      }
    }
    return null;
  }
  
}
```
好的, 可以看到最终我们获取到了 MainFragmentBindingImpl 这个对象, 这便是 ViewDataBinding 的实现类了, 它也是编译时生成的

#### 流程回顾
![ViewDataBinding 实例化流程](https://i.loli.net/2019/05/23/5ce600e05d2c321221.png)

这里我们还看没有看到 View 与 ViewModel 中的数据关联的过程, 接下来我们顺着 MainFragmentBindingImpl 的实例化继续探索

## 二. ViewDataBinding 的初始化
```
public class MainFragmentBindingImpl extends MainFragmentBinding implements com.sharry.demo.mvvm.generated.callback.OnClickListener.Listener {
    
    public MainFragmentBindingImpl(@Nullable androidx.databinding.DataBindingComponent bindingComponent, @NonNull View root) {
        this(bindingComponent, root, mapBindings(bindingComponent, root, 2, sIncludes, sViewsWithIds));
    }
    
    private MainFragmentBindingImpl(androidx.databinding.DataBindingComponent bindingComponent, View root, Object[] bindings) {
        // 将布局内的视图添加到父类中暂存
        super(bindingComponent, root, 1
            , (android.widget.Button) bindings[1]
            , (androidx.constraintlayout.widget.ConstraintLayout) bindings[0]
            );
        // 清空 tag
        this.btnCenter.setTag(null);
        this.main.setTag(null);
        // 将根视图标记为 databinding 的视图
        setRootTag(root);
        // 重置数据
        invalidateAll();
    }
    
}
```
好的, 可以看到 MainFragmentBindingImpl 中保存了我们 xml 的视图, 并且根据我们的设置创建了监听器

感觉距离 View 绑定 Observable 又近了一步, 定位到 invalidateAll 这个方法, 我们看看它做了什么
```
public class MainFragmentBindingImpl extends MainFragmentBinding implements com.sharry.demo.mvvm.generated.callback.OnClickListener.Listener {
    
    @Override
    public void invalidateAll() {
        synchronized(this) {
                mDirtyFlags = 0x8L;
        }
        requestRebind();
    } 
    
    private static final boolean USE_CHOREOGRAPHER = SDK_INT >= 16;
    
    protected void requestRebind() {
        // 初始化状态为 null
        if (mContainingBinding != null) {
            ......
        } else {
            ......
            if (USE_CHOREOGRAPHER) {
                // 投递到下一帧执行 callback, 最终还是会调用到 mRebindRunnable
                mChoreographer.postFrameCallback(mFrameCallback);
            } else {
                // 通过 Handler 直接 post 到消息队列
                mUIThreadHandler.post(mRebindRunnable);
            }
        }
    }
}
```
好的, 可以看到, invalidateAll 操作, 最终会回调 mRebindRunnable, 接下来我们它的实现
```
public abstract class ViewDataBinding extends BaseObservable {
    
    private final Runnable mRebindRunnable = new Runnable() {
        @Override
        public void run() {
            ......
            executePendingBindings();
        }
    };
   
    public void executePendingBindings() {
        // 判断当前 ViewDataBinding 是否持有其他的引用, 初始时是没有的
        if (mContainingBinding == null) {
            // 调用了 executeBindingsInternal 进行后续操作
            executeBindingsInternal();
        } else {
            // ......
        }
    }

    private void executeBindingsInternal() {
        // 若当前存在正在执行的任务, 则将本次操作投递到下一帧执行
        if (mIsExecutingPendingBindings) {
            requestRebind();
            return;
        }
        ......
        mIsExecutingPendingBindings = true;
        mRebindHalted = false;
        ......// 通知外界, 执行了 rebind 操作
        if (!mRebindHalted) {
            // 执行 Binding 操作
            executeBindings();
        }
        mIsExecutingPendingBindings = false;
    }
 
}
```
好的, 可以看到最终看到了它调用了 ViewDataBinding.executeBindings 执行 Binding 操作

## 三. View 与 ObservableFiled 的绑定
这个方法是交由子类重写的, 我们探究一下 MainFragmentBindingImpl 中是如何实现的
```
public class MainFragmentBindingImpl extends MainFragmentBinding implements com.sharry.demo.mvvm.generated.callback.OnClickListener.Listener {
    
    
    @Override
    protected void executeBindings() {
        ......
        // 获取 ViewModel, 因为这个方法是 post 到 MessageQueue 中执行的
        // 此时 mViewModel 我们已经通过 setViewModel 赋值了
        com.sharry.demo.mvvm.ui.main.MainViewModel viewmodel = mViewmodel;
        // 这个便是 ViewModel.中的 messageText 的引用
        androidx.databinding.ObservableField<java.lang.String> viewmodelMessageText = null;
        java.lang.String viewmodelMessageTextGet = null;

        if ((dirtyFlags & 0xdL) != 0) {
                if (viewmodel != null) {
                    // 1. 获取我们定义的 ObservableField 实例对象 
                    viewmodelMessageText = viewmodel.getMessageText();
                }
                // 2. 让 本地属性为 0 的控制与这个 viewmodelMessageText 相互注册
                updateRegistration(0, viewmodelMessageText);
                // 3. 获取 ObservableField 中的值
                if (viewmodelMessageText != null) {
                    viewmodelMessageTextGet = viewmodelMessageText.get();
                }
        }
        ......
    }
    
}
```
通过这段代码可以确定 View 与 Observable 的绑定操作是通过 updateRegistration 进行绑定的, 接下来我们看看具体的实现
```
public abstract class ViewDataBinding extends BaseObservable {
    
    private static final CreateWeakListener CREATE_PROPERTY_LISTENER = new CreateWeakListener() {
        @Override
        public WeakListener create(ViewDataBinding viewDataBinding, int localFieldId) {
            return new WeakPropertyListener(viewDataBinding, localFieldId).getListener();
        }
    };
    
    protected boolean updateRegistration(int localFieldId, Observable observable) {
        // 更新注册器
        return updateRegistration(localFieldId, observable, CREATE_PROPERTY_LISTENER);
    }
 
    private WeakListener[] mLocalFieldObservers;
 
    private boolean updateRegistration(int localFieldId, Object observable,
            CreateWeakListener listenerCreator) {
        ......
        // 从控件观察者表中获取 localFieldId 对应的观察者
        WeakListener listener = mLocalFieldObservers[localFieldId];
        // 首次进行注册操作, 故为 null
        if (listener == null) {
            //  调用 registerTo 进行后续操作
            registerTo(localFieldId, observable, listenerCreator);
            return true;
        }
        ......
        return true;
    }
    
    protected void registerTo(int localFieldId, Object observable,
            CreateWeakListener listenerCreator) {
        ......
        
        WeakListener listener = mLocalFieldObservers[localFieldId];
        if (listener == null) {
            // 1. 创建该控件对应的观察者, 即 WeakPropertyListener.getListener() 
            listener = listenerCreator.create(this, localFieldId);
            // 缓存到观察者表中
            mLocalFieldObservers[localFieldId] = listener;
            ......
        }
        // 2. 为 这个观察者 绑定要观察的对象, 即 ViewModel 中我们定义的 ObservableFiled 属性
        listener.setTarget(observable);
    }
    
}
```
好的, 可以看到这是一个典型的观察者设计模式
- ViewDataBinding 为 View 创建了一个观察者对象, 即 WeakListener, 用于监听被观察者
- 调用 WeakListener.setTarget 绑定要观察的对象, 即设置被观察者

接下来我们看看这个 WeakListener 的创建过程, 以及它与 ObservableFiled 的绑定

#### 一) 创建观察者
```
public abstract class ViewDataBinding extends BaseObservable {
    
    private static class WeakPropertyListener extends Observable.OnPropertyChangedCallback
            implements ObservableReference<Observable> {
    
        final WeakListener<Observable> mListener;

        public WeakPropertyListener(ViewDataBinding binder, int localFieldId) {
            // 创建了 WeakListener
            mListener = new WeakListener<Observable>(binder, localFieldId, this);
        }
                
        @Override
        public WeakListener<Observable> getListener() {
            return mListener;
        }
        
        @Override
        public void addListener(Observable target) {
            // 调用了 Observable.addOnPropertyChangedCallback, 为被观察者订阅一个观察者
            target.addOnPropertyChangedCallback(this);
        }
    }
    
    private static class WeakListener<T> extends WeakReference<ViewDataBinding> {
        
        private final ObservableReference<T> mObservable;
        protected final int mLocalFieldId;
        private T mTarget;

        public WeakListener(ViewDataBinding binder, int localFieldId,
                ObservableReference<T> observable) {
            super(binder, sReferenceQueue);
            mLocalFieldId = localFieldId;
            // 即 WeakPropertyListener 对象
            mObservable = observable;
        }

        public void setTarget(T object) {
            ......
            // 获取要观察的对象
            mTarget = object;
            if (mTarget != null) {
                // 调用 WeakPropertyListener.addListener 
                mObservable.addListener(mTarget);
            }
        }
    }
}
```
好的, 可以观察者 WeakListener 是由 WeakPropertyListener 在构造函数中实例化的
- WeakListener.setTarget 方法会回调 WeakPropertyListener.addListener 为被观察者 Observable 订阅观察者

这里可以看到**虽然 ViewDataBinding 中维护的观察者数组 mLocalFieldObservers 是 WeakListener, 但真正的观察者为 WeakPropertyListener**, 接下来我们看看 Observable 订阅观察者的过程

#### 二) 观察者的订阅与通知
我们知道我们在 ViewModel 中定义的 Observable 为 ObservableFiled 类型, 其 addOnPropertyChangedCallback 实现为 BaseObservable
```
public class BaseObservable implements Observable {
    private transient PropertyChangeRegistry mCallbacks;

    public BaseObservable() {
    }

    @Override
    public void addOnPropertyChangedCallback(@NonNull OnPropertyChangedCallback callback) {
        synchronized (this) {
            if (mCallbacks == null) {
                mCallbacks = new PropertyChangeRegistry();
            }
        }
        // 很简单, mCallbacks 为订阅该被对象的观察者集合
        mCallbacks.add(callback);
    }
    
}
```
可以看到 addOnPropertyChangedCallback 中的操作非常简单, 其内部维护了一个 mCallbacks 对象, 维护了订阅了该对象的观察者

#### 回顾
至此 DataBinding 框架便为我们成功的将 View 与 ViewModel 中的数据绑定了, 他们的关系如下

![数据绑定的关系图](https://i.loli.net/2019/05/23/5ce6015ae3dac56169.png)


## 四. ObservableField 通知观察者数据更新
当被观察者 Observable 调用 set 方法更新数据时, 便会通知订阅其观察者们, 我们看看它的实现
```
public class ObservableField<T> extends BaseObservableField implements Serializable {

    private T mValue;

    public void set(T value) {
        if (value != mValue) {
            mValue = value;
            // 通知观察者数据变更了
            notifyChange();
        }
    }
    
}
public class BaseObservable implements Observable {
    
    public void notifyChange() {
        synchronized (this) {
            if (mCallbacks == null) {
                return;
            }
        }
        // 通知观察者数据变更
        mCallbacks.notifyCallbacks(this, 0, null);
    }
    
}
```
可以看到 notifyChange 将任务分发给了 PropertyChangeRegistry.notifyCallbacks 我们看看它的实现
```
public class CallbackRegistry<C, T, A> implements Cloneable {
 
    private List<C> mCallbacks = new ArrayList<C>();   
 
    public synchronized void notifyCallbacks(T sender, int arg, A arg2) {
        ......
        notifyRecurse(sender, arg, arg2);
        ......
    }
   
    private void notifyRecurse(T sender, int arg, A arg2) {
        ......
        notifyCallbacks(sender, arg, arg2, startCallbackIndex, callbackCount, 0);
    }
    
    private final NotifierCallback<C, T, A> mNotifier;
    
    private void notifyCallbacks(T sender, int arg, A arg2, final int startIndex,
            final int endIndex, final long bits) {
        long bitMask = 1;
        // 遍历订阅的观察者
        for (int i = startIndex; i < endIndex; i++) {
            if ((bits & bitMask) == 0) {
                // 通过 NotifierCallback 推送数据
                mNotifier.onNotifyCallback(mCallbacks.get(i), sender, arg, arg2);
            }
            bitMask <<= 1;
        }
    }

}
```
可看到推送数据的操作, 是由 NotifierCallback 完成的, 它是实例化是在 PropertyChangeRegistry 构造的时候进行的
```
public class PropertyChangeRegistry extends
        CallbackRegistry<Observable.OnPropertyChangedCallback, Observable, Void> {

    private static final CallbackRegistry.NotifierCallback<Observable.OnPropertyChangedCallback, Observable, Void> NOTIFIER_CALLBACK = new CallbackRegistry.NotifierCallback<Observable.OnPropertyChangedCallback, Observable, Void>() {
        @Override
        public void onNotifyCallback(Observable.OnPropertyChangedCallback callback, Observable sender,
                int arg, Void notUsed) {
            // 调用了 WeakPropertyListener.onPropertyChanged 推送数据
            callback.onPropertyChanged(sender, arg);
        }
    };

    public PropertyChangeRegistry() {
        super(NOTIFIER_CALLBACK);
    }
            
}
```
可以看到 NotifierCallback 的实现类为 PropertyChangeRegistry.NOTIFIER_CALLBACK, 其内部调用了 WeakPropertyListener.onPropertyChanged 推送数据
```
public abstract class ViewDataBinding extends BaseObservable {
    
    private static class WeakPropertyListener extends Observable.OnPropertyChangedCallback
            implements ObservableReference<Observable> {
        
        @Override
        public void onPropertyChanged(Observable sender, int propertyId) {
            ViewDataBinding binder = mListener.getBinder();
            ......
            // 这里调用了 ViewDataBinding.handleFieldChange 对 View 进行赋值操作
            binder.handleFieldChange(mListener.mLocalFieldId, sender, propertyId);
        }
                    
    }
    
    private void handleFieldChange(int mLocalFieldId, Object object, int fieldId) {
        ......
        // 1. 调用了子类的 onFieldChange
        boolean result = onFieldChange(mLocalFieldId, object, fieldId);
        // 2. 返回 true, 调用 requestRebind, 这会重新执行 MainFragmentBindingImpl.executeBindings
        if (result) {
            requestRebind();
        }
    }
    
}

public class MainFragmentBindingImpl extends MainFragmentBinding implements com.sharry.demo.mvvm.generated.callback.OnClickListener.Listener {

    @Override
    protected boolean onFieldChange(int localFieldId, Object object, int fieldId) {
        switch (localFieldId) {
            case 0 :
                return onChangeViewmodelMessageText((androidx.databinding.ObservableField<java.lang.String>) object, fieldId);
        }
        return false;
    }
    
    private boolean onChangeViewmodelMessageText(androidx.databinding.ObservableField<java.lang.String> ViewmodelMessageText, int fieldId) {
        if (fieldId == BR._all) {
            synchronized(this) {
                    mDirtyFlags |= 0x1L;
            }
            // 1.1 返回 true, 说明需要数据重置
            return true;
        }
        return false;
    }
    
    @Override
    protected void executeBindings() {
        .....
        // 真正的为 View 赋值
        if ((dirtyFlags & 0xdL) != 0) {
            androidx.databinding.adapters.TextViewBindingAdapter.setText(this.btnCenter, viewmodelMessageTextGet);
        }
        ......
    }
    
}

```
好的, 可以看到观察者收到数据推送后, 最终会重新调用 MainFragmentBindingImpl.executeBindings 为控件更新数据, 至此一次数据推送就完成了


## 总结
通过上面的分析可知 DataBinding 的工作原理主要有如下三个重要的步骤
- ViewDataBinding 的构建
  - 解析 xml 为 view 视图
  - 构建视图对应的 ViewDataBinding
- ViewDataBinidng 处理视图与 ViewModel 中 Observable 的绑定
  - 为 view 创建其对应的 WeakListener
  - WeakListener 的实现由 WeakPropertyListener 实现
  - 将 WeakPropertyListener 当做观察者, 将其添加到 Observable 的订阅集合中
- 当 Observable 的数据变更时, 便会推送给其订阅者集合
  - 调用  WeakPropertyListener.onPropertyChanged 处理数据变更
  - 通过 ID 找到 WeakPropertyListener 对应的 View
  - 通过 ViewDataBinding 的生成实现类的 executeBindings 更新指定 view 的数据

可以看到这是一个典型的观察者的应用场景, TextView 为观察者, Observable 为被观察者, 数据变更时便会推送给观察者更新数据, 当然这只是一个最基础的应用场景, 其他的使用场景可以类比推理

### 问题的思考
**为什么 <layout> 标签, 会自动给 view 增加 tag**
- DataBinding 定义了 xml 解析器, 会给 view 增加 tag

**为什么 要用 WeakListener 弱引用类型接口?**
- WeakListener 数组是被 ViewDataBinding 持有的, ViewDataBinding 在使用时会被 View 层持有
- 当 WeakListener 被添加到数据源 ObservableFiled 中的订阅集合中时, 会被 ViewModel 中的数据源属性持有 

我们知道 ViewModel 的移植性是非常强的, 因此若该 ViewModel 被另一个生命周期更长的 View 持有, 导致 ViewModel 的生命周期长于当前 View, 这会导致当前 View 无法释放而造成内存泄漏

WeakListener 使用弱引用, 当内存即将溢出时, 会无视弱引用将该对象回收, 相当于做了一层保护措施, 但这并不意味着我们在开发过程中可以肆意妄为, 代码的质量还是要保证的, 内存泄漏更是需要体验预判

**DataBinding 设计上的优劣**
- 帮我们实现了 View 与 ViewModel 之间的交互, 让使用者关心的更少, 可以更加专注于业务代码的开发
- 目前支持的属性难以覆盖常用应用场景, 编译时生成类在一定程度上降低了编译速度
