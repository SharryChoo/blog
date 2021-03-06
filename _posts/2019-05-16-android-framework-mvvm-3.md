---
title: MVVM 架构演进 —— 架构的搭建
tags: Framework
---

## 前言
学习了 MVVM 的 Demo, 翻阅了 DataBinding 的实现源码, 让我们对 MVVM 框架有了一个整体上的了解, 用一句话来概括就是, **MVVM 即通过 DataBinding 来解除 Presenter 与 View 依赖的 MVP 架构**, 这样的 Presenter 称之为 ViewModel

不过 Demo 中的示例, 真实放到项目中, 会发现很多应用场景使用起来非常的困难, 甚至会让我们觉得没有 MVP 好用, 这篇文章记录了笔者在 MVP 向 MVVM 演进过程中的所见所想, 希望对看到这篇文章的人有所帮助

## 一. 设计要点
MVVM 架构的搭建核心重点是为了解决 View 与 ViewModel 的通信问题

<!--more-->

### 结构设计
- View 层可以持有 ViewModel 的引用
  - 当数据不方便 xml 中通过 DataBinding 绑定时, 需要用代码指定 
- ViewModel 中不存在 View 的引用
  - ViewModel 中可以持有 Application 的引用, 以保证功能的易用性
- 考虑后续功能拓展性

### 功能的易用性
- 需要在 ViewModel 中快捷的通知 view 状态变更
  - 通知弹窗, 空数据, 网络异常, 加载框等 
- 考虑数据的双向绑定, 如 EditText
- 考虑内存泄漏出现的常见场景
  - LiveData 去维护生命周期

## 二. 技术选取
View + DataBinding(LiveData/ObservableField) + ViewModel 
- View 层定义 View 常用操作
- DataBinding 处理 View 与 ViewModel 的双向绑定
  - 使用InverseBindingAdapter 处理数据反向注入
- LiveData 解决数据推送时声明周期的问题
- ViewModel 处理数据逻辑

## 三. 框架搭建的实施
### 一) View 层
#### 1. BaseView 的创建
View 层的搭建, 与 MVP 中的 View 基本一致
```
public interface BaseView<T extends ViewDataBinding> {

    /**
     * Do init operation when data binding created.
     *
     * @param dataBinding the data binding that need init.
     */
    void initDataBinding(@NonNull T dataBinding);

}
```
与 MVP 不同的是这里的 BaseView 关联的泛型是 ViewDataBinding 类型,  **为什么不直接使用 ViewModel 呢?**
- 这是因为 ViewModel 是不允许持有 View 引用的, 所以 ViewModel 的可移植性远远高于 Presenter, 我们可以所以的在 XML 中声明多个 ViewModel, 因此这里并没有让 View 层直接关联 ViewModel 的泛型

这里的 BaseView 中只有一个方法 initDataBinding, 即在获取到 ViewDataBinding 的实例之后, 执行 ViewDataBinding 初始化的操作, 我们可以在这个方法中, 为 DataBinding 的生产类, 关联对应的 View 和 ViewModel

最基础的 BaseView 实现了, 不过这个功能似乎太过于简单了一些, 我们在 Activity, Fragment 等页面搭建的过程中 Toast、 Tips、 EmptyData 几乎是必用的功能, 因此我们这里再定义一个 BaseView 的增强版

```
/**
 * The View provider more function.
 *
 * @author Sharry <a href="SharryChooCHN@Gmail.com">Contact me.</a>
 * @version 1.0
 * @since 2018/8/28 22:21
 */
public interface SupportView<T extends ViewDataBinding> extends BaseView<T> {

    /**
     * Show simple tips.
     */
    void tip(@Nullable String msg);

    /**
     * Show toast.
     */
    void toast(@Nullable String msg);

    /**
     * Show snack bar.
     */
    void snackBar(@Nullable String msg);

    /* ============================== Progress Bar =======================================*/

    /**
     * Show progress view associated with current page
     * Use default attach view {@code R.android.id.cåontent}.
     */
    void progress(boolean isShow);

    /**
     * Show progress view associated with current page.
     */
    void progress(@NonNull View attached, boolean isShow);

    /* ============================== Empty data =======================================*/

    /**
     * Show empty data without msg associated with current page.
     * Use default attach view {@code R.android.id.content}.
     */
    void showEmptyData();

    /**
     * Show empty data without msg associated with current page.
     */
    void showEmptyData(@NonNull View attached);

    /* ============================== Network Error =======================================*/

    /**
     * Show network disconnected associated with current page.
     * Use default attach view {@code R.android.id.content}.
     */
    void showNetworkError(OnNetworkErrorListener listener);

    /**
     * Show network disconnected associated with current page.
     */
    void showNetworkError(@NonNull View attached, OnNetworkErrorListener listener);

    /**
     * Callback associated with disconnected view.
     */
    interface OnNetworkErrorListener {
        void onNetworkError();
    }

}
```
好的, 可以看到这个 SupportView 几乎涵盖了我们开发中最常用的 View 层的通用方法, 只需要让我们的 BaseActivity/BaseFragment 实现这个 SupportView 就可以了

接下来我们以 Activity 为例, 看看 BaseView 的实现

#### 2. BaseView 的实现
我们先定义一个模板 Activity, 然后再此基础上进行 MVVM 的实现类拓展
```
public abstract class BaseActivity extends AppCompatActivity {

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        // 1. Parse intent from other activity.
        Intent data = getIntent();
        if (null != data) {
            parseIntent(data);
        }
        // 2. Inject layout resource to content view.
        createView(getLayoutResId());
        // 3. Initialize view
        initViews();
        // 4. Initialize data after view display on screen.
        new Handler(Looper.getMainLooper()).post(new Runnable() {
            @Override
            public void run() {
                initData();
            }
        });
    }

    /**
     * U can parse intent that transfer from other activity.
     *
     * @param intent data that from request Activity.
     */
    protected void parseIntent(@NonNull Intent intent) {
    }

    /**
     * Get layout resource associated with this activity.
     *
     * @return layout id.
     */
    protected abstract int getLayoutResId();

    /**
     * Create view by u custom.
     */
    protected void createView(int layoutResId) {
        setContentView(layoutResId);
    }
    ......
}
```
可以看到这里简单的定义了一些模板方法, 用户可以按照需求自己去重写实现, 接下来我们看看 BaseMvvmActivity 的实现
```
public abstract class BaseMvvmActivity<DataBinding extends ViewDataBinding> extends BaseActivity
        implements SupportView<DataBinding> {

    protected DataBinding dataBinding;

    @Override
    protected void createView(int layoutResId) {
        dataBinding = DataBindingUtil.setContentView(this, layoutResId);
        if (dataBinding == null) {
            throw new NullPointerException("Cannot find ViewDataBinding that layout id is: " + layoutResId);
        }
        initDataBinding(dataBinding);
    }

    @Override
    public void tip(@Nullable String msg) {
        // TODO: Custom u simple tip display.
    }

    @Override
    public void toast(@Nullable String msg) {
        Toast.makeText(this, msg, Toast.LENGTH_SHORT).show();
    }

    @Override
    public void snackBar(@Nullable String msg) {
        // TODO: Custom u snackbar display.
    }
    ......
}
```
可以看到支持 **MVVM 架构的 Activity 只需要重写 createView 这个方法就可以实现其功能了**
- SupportView 中定义的接口, 根据当前 App 的 UI 进行通用展示

好的, 从这里就可以看到多一个 BaseActivity 的好处了, 定义一个基础的模板, 我们可以在其基础上进行拓展, **在不改变使用方式的前提下实现对 MVP, MVVM 架构的支持, 遵守了开闭原则(对拓展开放, 对修改封闭), 也对日后新架构的拓展提供了可能**

#### 思考
在前面的文章我们了解到 View 层数据的变更是通过在 xml 中指定了 ViewModel 中数据源之后, 由数据源通知的, 如下所示
```
......
    <TextView
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:textAllCaps="false"
        // 这里绑定了 ViewModel 中的数据源
        android:text="@{viewmodel.messageText}"
    />
......
```
但我们在 BaseMvvmActivity 中实现的 SupportView 接口(如 toast), 很明显有些是无法在 xml 中与 ViewModel 中的数据源绑定的, 但 **ViewModel 中是 没有 View 引用的, 因它如何如同 MVP 一样定义一个 showMsg, 在 Presetner 中愉快的调用它让 View 弹出一个吐司, 那么 MVVM 应该它如何让 View 弹吐司呢?**

想清楚这个问题, MVVM 架构就已经完全难不倒你了, 接下来我们看看 ViewModel 层的定义

### 二) ViewModel 层
```
public abstract class SupportViewModel extends AndroidViewModel {

    /**
     * The viewStatusSource associated with the special view that using this ViewModel.
     */
    protected final SingleLiveData<SupportViewStatus> viewStatusSource = new SingleLiveData<>();

    /**
     * The tip message associated with the special view that using this ViewModel.
     */
    protected final SingleLiveData<String> tipMsgSource = new SingleLiveData<>();

    /**
     * The toast message associated with the special view that using this ViewModel.
     */
    protected final SingleLiveData<String> toastMsgSource = new SingleLiveData<>();

    public SupportViewModel(@NonNull Application application) {
        super(application);
    }

    /**
     * Set a observer for toastMsgSource.
     */
    public void setToastMsgSourceObserver(@NonNull LifecycleOwner owner,
                                          @NonNull ToastObserver toastObserver) {
        Preconditions.checkNotNull(owner);
        Preconditions.checkNotNull(toastObserver);
        toastMsgSource.observe(owner, toastObserver);
    }
    ......
}
```
是否有种恍然大悟的感觉, 笔者在 SupportViewModel 中定义了一组数据源, 可以看到一个 SingleLiveData 类型的 toastMsg, 这里的 SingleLiveData 可以先当做一个 LiveData, 那么 LiveData 的作用是什么呢?
- LiveData 与 Observable 类似, 也是一个可被观察的数据源, **不过它的优势在于, 它可以帮助我们管控生命周期**, 这正是它迷人的地方

除了定义数据源之外, 还为数据源提供了添加观察者的方法, 如 setToastMsgSourceObserver 等, 其他数据源的添加观察者的方式与之类似
- 只需要在 view 层调用这个方法, 将 view 层实现的观察者加入, 便可以实现数据推送了

接下来看看 Model 层的定义

### 三) Model 层
笔者项目中 Model 层的设计, 可能有些不同, 它异常的简单
```
/**
 * 定义网络数据源
 *
 * @author Sharry <a href="SharryChooCHN@Gmail.com">Contact me.</a>
 * @version 1.0
 * @since 2019-05-20 16:28
 */
public interface RemoteDataSource {

}


/**
 * 定义本地数据源(SP, 数据库...)
 *
 * @author Sharry <a href="SharryChooCHN@Gmail.com">Contact me.</a>
 * @version 1.0
 * @since 2019-05-20 16:28
 */
public interface LocalDataSource {

}

/**
 * @author Sharry <a href="SharryChooCHN@Gmail.com">Contact me.</a>
 * @version 1.0
 * @since 2019-05-20 16:28
 */
public interface DataSource extends LocalDataSource, RemoteDataSource {

    DataSource INSTANCE = new DataSourceRepository();

}


/**
 * 数据源实现类
 *
 * @author Sharry <a href="SharryChooCHN@Gmail.com">Contact me.</a>
 * @version 1.0
 * @since 2019-05-20 16:30
 */
class DataSourceRepository implements DataSource {

}
```
可以看到 Model 层的设计非常简单, 这是一个全局的数据源, 所有的 ViewModel 都可以通过 DataSource.INSTANCE 获取实现类, 从中获取数据

这个设计我第一次看到时, 也非常的震惊, 因为在之前的印象中 Model 与 Presenter 是一一对应的, 所以看到一个单一的数据源时有些难以接受, 不过用下来之后却发现异常的舒服
- 不用考虑一个 Presetner/ViewModel 对应多个 Model 的苦恼
- 通过单一数据源对上层提供, 能够取到所有的数据, 组件化落实时也可以减轻跨模块获取数据的困扰
- **最后, 这个设计师从 Goggle**

到这里 MVVM 架构的搭建基本上就结束了, 最后再看一个数据双向绑定的问题

### 四) 数据的双向绑定
因为 ViewModel 层与 View 完成隔离, 所以 ViewModel 层只能够通过提供数据源, 让 View 层观察的方式进行通信(DataBinding 的实现原理也是如此), 不过我们不能忽略的是, 有些数据是在 View 层主动产生的, 如 EditText 的主动输入, 这种场景下我们如何将数据反向注入到 ViewModel 中的数据源呢? 

当然, 可以在 ViewModel 中定义一个方法, 当 View 层数据主动变更时, 通过调用 ViewModel 中的方法, 将数据注入, 似乎有些不太优雅, 这个时候 **@BindingAdapter/@InverseBindingAdapter** 就派上用场了

```
public class Sample1BindingAdapters {


    /**
     * 数据的正向推送
     * <p>
     * {@code app:text="@{viewmodel.xxx}"} viewmodel.xxx 发生变更时, 将数据推送给观察者
     */
    @BindingAdapter("text")
    public static void setEditTextContent(EditText editText, String newStr) {
        String oldStr = editText.getText().toString();
        // 解决正向推送与反向注入的死循环
        if (!oldStr.equals(newStr)) {
            editText.setText(newStr);
        }
    }

    /**
     * 获取反向注入的数据
     * <p>
     * {@code app:text="@={viewmodel.xxx}"} app:text 发生变更时, 将数据反向注入给被观察者
     */
    @InverseBindingAdapter(
            attribute = "text",
            event = "onEditTextChanged"
    )
    public static String getEditTextContent(EditText editText) {
        return editText.getText().toString();
    }

    /**
     * 反向注入发起
     */
    @BindingAdapter(value = "onEditTextChanged", requireAll = false)
    public static void onEditTextChanged(EditText editText, final InverseBindingListener textAttrChanged) {
        if (textAttrChanged != null) {
            editText.addTextChangedListener(new TextWatcher() {
                @Override
                public void beforeTextChanged(CharSequence s, int start, int count, int after) {

                }

                @Override
                public void onTextChanged(CharSequence s, int start, int before, int count) {

                }

                @Override
                public void afterTextChanged(Editable s) {
                    // 文本变更之后, 发起数据的反向注入
                    textAttrChanged.onChange();
                }
            });
        }
    }

}
```
上面的注释也非常的清晰, 这是 DataBinding 提供的拓展功能的实现, 其使用语法为 app:text = "@={viewmodel.xxx}", 描述的是 text 属性与 viewmodel.xxx 的双向绑定
- 当 ViewModel 中的数据源变更时, 会调用 setEditTextContent 方法做出相应的 UI 变更
- 当 View 层发生变更时, 会调用 getEditTextContent 获取数据注入到 ViewModel 的数据源中
  - 推送的时机在 onEditTextChanged 中自行定义

好的, 到这里我们 MVVM 的框架的搭建便进入尾声了, 接下来做个总结

## 总结
不知道大家是否有这样的感觉, MVVM 框架的搭建比起 MVP 要简单的多, 我认为这是因为系统帮我们做了最重要的事情, 那便是 DataBinding, 初始写 MVVM 架构的时候, 可能会因为 ViewModel 中没有 View 而手足无措, 这个时候只需要将思维转变, **让 View 主动订阅 ViewModel 中的数据源即可实现最终目标**

这是一个响应式的过程, 笔者把这里的内容整理成了 [Demo](https://github.com/SharryChoo/MVVMSample), 希望能够帮助大家进一步理解 MVVM 架构

### 展望
这样的 MVVM 架构, 已经能够满足日常开发需求了, 不过因为在 ViewModel 中含有对 ObservableField, LiveData 等 Android 依赖库, 让 ViewModel 层的单元测试变得比 MVP 中 Presenter 要困难的多, 有兴趣的小伙伴, 可以研究一下如何改进

面对复杂的逻辑关系控制, LiveData 和 ObservableField 可能难以胜任, 熟悉 RxJava 的开发者们可以在 ViewModel 中使用 RxJava 中的热信号作为数据源, 从而简化逻辑代码的实现, 当然这需要自己管控好生命周期

## 参考文献
- [https://tech.meituan.com/2016/11/11/android-mvvm.html](https://tech.meituan.com/2016/11/11/android-mvvm.html)
- [https://listenzz.github.io/android-lifecyle-works-perfectly-with-rxjava.html](https://listenzz.github.io/android-lifecyle-works-perfectly-with-rxjava.html)
- [https://developer.android.com/topic/libraries/architecture](https://developer.android.com/topic/libraries/architecture)
