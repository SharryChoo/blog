---
title: MVVM 架构演进 —— 初探
tags: Framework
---

## 一. 什么是 MVVM ?
MVVM 是前端一款数据驱动的架构
- Model: 网络接口数据, 本地缓存数据等
- View: Activity, Fragment, DialogFragment.....
- ViewModel: 提供数据源, 处理业务逻辑

## 二. MVVM 与 MVP 的区别是什么?
MVVM 与 MVP 十分的相似, MVVM 是从 MVP 架构演进而来的
- MVP 通过 Presenter 做到了数据和视图的解耦, 但是 Presenter 层的任务就变得十分之重, View 与 Presenter 交互要写很多的接口, 接口粒度的控制也不容易把控
- ViewModel 是为了解决 MVP 中 Presenter 过重这个问题而生的, ViewModel 和 View 之间的通信并不需要通过接口来回进行数据导入导出, 而是通过注册观察者, 当数据变更时会自动的反应到与之绑定的 View 上
 
<!--more-->
 
可以看到 MVVM 解决了 MVP 中最影响开发效率的一个环节, 它可以使用更加整洁的代码, 让开发变的更加的高效

## 三. 如何使用 MVVM
### 一) Model
```
// 全局的数据仓库
interface DataSource : LocalDataSource, RemoteDataSource {
    
    companion object {
        val INSTANCE: DataSource = DataRepository()
    }

}

// 唯一实现类
class DataRepository : DataSource
```
Model 层的设计非常简单, 即一个全局的数据仓库

### 二) View
#### 1. Activity
```
class MainActivity : AppCompatActivity() {

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.main_activity)
        if (savedInstanceState == null) {
            supportFragmentManager.beginTransaction()
                .replace(R.id.container, MainFragment.newInstance())
                .commitNow()
        }
    }

}
```
Activity 为应用壳, 布局也十分简单, 直接填充了一个 Fragment

#### 2. Fragment
```
class MainFragment : Fragment() {

    companion object {
        fun newInstance() = MainFragment()
    }

    override fun onCreateView(
        inflater: LayoutInflater, container: ViewGroup?,
        savedInstanceState: Bundle?
    ): View {
        // 这里 View 的创建交由 DataBinding 完成
        val dataBinding: MainFragmentBinding = MainFragmentBinding.inflate(inflater, container, false)
        // DataBinding 中持有 View 和 ViewModel 的引用, 他们直接的交互由 DataBinding 生成类 MainFragmentBinding 来控制
        dataBinding.view = this
        dataBinding.viewmodel = ViewModelProviders.of(this).get(MainViewModel::class.java)
        return dataBinding.root
    }

}
```
Fragment 中 View 的实例化非常的有趣, 它是通过 DataBinding 创建的, 并且通过 DataBinding 让 View 层与 ViewModel 建立联系, 这相当于将 MVP 中 View 与 Presenter 的交互交由了系统控制

#### 3. xml
```
<layout xmlns:android="http://schemas.android.com/apk/res/android"
        xmlns:app="http://schemas.android.com/apk/res-auto" xmlns:tool="http://schemas.android.com/tools">

    <data>

        <import type="android.view.View"/>

        <variable
                name="view"
                type="com.sharry.demo.mvvm.ui.main.MainFragment"/>

        <variable
                name="viewmodel"
                type="com.sharry.demo.mvvm.ui.main.MainViewModel"/>
    </data>

    <androidx.constraintlayout.widget.ConstraintLayout
            android:id="@+id/main"
            android:layout_width="match_parent"
            android:layout_height="match_parent">

        <Button
                android:id="@+id/btnCenter"
                android:layout_width="wrap_content"
                android:layout_height="wrap_content"
                android:textAllCaps="false"
                android:text="@{viewmodel.messageText}"
                android:onClick="@{() -> viewmodel.messageClicked()}"
                app:layout_constraintBottom_toBottomOf="parent"
                app:layout_constraintEnd_toEndOf="parent"
                app:layout_constraintStart_toStartOf="parent"
                app:layout_constraintTop_toTopOf="parent"
                tool:text="I'm a fragment."
        />

    </androidx.constraintlayout.widget.ConstraintLayout>
</layout>
```
xml 中指定了 View 与 ViewModel, 并且让控件的属性直接与 ViewModel 中的数据绑定, 当 ViewModel 中数据变更后便会直接反应到控件上去


### 三) ViewModel
```
class MainViewModel : ViewModel() {

    val messageText = ObservableField<String>()
    
    private val dataSource = DataSource.INSTANCE
     
    init {
        messageText.set("I'm a fragment.")
    }

    fun messageClicked() = messageText.set("Text changed by DataBinding.")

}
```
ViewModel 中的代码也非常的简单, 直接构建了一些需要用的 ObservableXXX 类型的数据, 让其变更时, DataBinding 会直接通知 View 更新

### UML 图
![MVVM 架构设计](https://i.loli.net/2019/05/23/5ce6004b11de920036.png)

## 四. 总结
从 MVVM 的示例实现上可知, DataBinding 在 MVVM 的架构中起到了不可或缺的作用, 它将原本 MVP 中 View 与 Presenter 的交互转移给了 DataBinding, 让系统帮我们去维护复杂的交互流程

### 优势
- 开发者无需关心 ViewModel 与 View 的交互, 这一操作交由了 DataBinding 完成, 因此开发者可以更高效的进行业务代码的开发
- ViewModel 与 View 之间没有无依赖, 因此 ViewModel 的可移植性要大大的强于 Presenter 

### 劣势
- 需要在 xml 中添加额外的代码, 增加了额外的操作
- View 数据的注入操作隐藏在 DataBinding 的生成类中, 增加了排查问题的难度
