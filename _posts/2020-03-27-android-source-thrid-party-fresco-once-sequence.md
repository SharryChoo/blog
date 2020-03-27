---
title: Jetpack —— Navigation 导航组件
permalink: android-source-third-party/fresco-once-sequence
key: android-source-third-party-fresco-once-sequence
tags: AndroidSource
---
## 前言
之前一直使用的是 Glide 图片加载框架, 最近接触了 Fresco 这款出自 Facebook 的图片加载框架, 为了方便后期对其进行封装与拓展, 这里分析记录一下它的一次简单的加载流程

关于 Fresco 的简单使用可以参考 [Fresco 使用文档](https://www.fresco-cn.org/docs/drawee-branches.html), 与 Glide 不同, 它主要是通过 DraweeView 对上层提供便捷性的服务, 文章主体分为如下几个部分
- Fresco 初始化
- SimpleDraweeView 的创建
- 数据的加载

## 一. Fresco 初始化

```java
public class Fresco {

  private static volatile boolean sIsInitialized = false;

  public static void initialize(Context context) {
    initialize(context, null, null);
  }

  public static void initialize(
      Context context,
      @Nullable ImagePipelineConfig imagePipelineConfig,
      @Nullable DraweeConfig draweeConfig) {
    // 用于保证只初始化一次
    if (sIsInitialized) {
      ......
    } else {
      sIsInitialized = true;
    }
    // 1. 加载 so 库
    try {
      SoLoader.init(context, 0);
    } catch (IOException e) {
      ......
    }
    // we should always use the application context to avoid memory leaks
    context = context.getApplicationContext();
    // 2. 初始化 ImagePipeline 的构造工厂
    if (imagePipelineConfig == null) {
      ImagePipelineFactory.initialize(context);
    } else {
      ImagePipelineFactory.initialize(imagePipelineConfig);
    }
    // 3. 初始化 Drawee
    initializeDrawee(context, draweeConfig);
  }

}
```

可以看到 Fresco 的初始化过程做了如下事情
- 利用 SoLoader 加载 so 库
- 初始化 ImagePipeline 的构造工厂
- 初始化 Drawee

### 一) 初始化 ImagePipeline 工厂

```java
public class ImagePipelineFactory {

  private static ImagePipelineFactory sInstance = null;
  private final ThreadHandoffProducerQueue mThreadHandoffProducerQueue;

  public static void initialize(Context context) {
    initialize(ImagePipelineConfig.newBuilder(context).build());
  }

  public static void initialize(ImagePipelineConfig imagePipelineConfig) {
    sInstance = new ImagePipelineFactory(imagePipelineConfig);
  }

  private final ImagePipelineConfig mConfig;
  private final ThreadHandoffProducerQueue mThreadHandoffProducerQueue;

  public ImagePipelineFactory(ImagePipelineConfig config) {
    // 存储 Pipeline 的配置
    mConfig = Preconditions.checkNotNull(config);
    // 创建一个使用队列维护 Runnable 的 Execute, 方便控制 Runnable 交由 Executor 的执行时机
    mThreadHandoffProducerQueue = new ThreadHandoffProducerQueue(
        // 获取线程池提供者
        config.getExecutorSupplier().forLightweightBackgroundTasks()
    );
  }
}
```

ImpagePipelineFactory 可以理解为一个抽象工厂, 负责生产一些列与 ImagePipeline 相关的对象, 对后续数据编解码提供支持

### 二) 初始化 Drawee

```java
public class Fresco {

  private static PipelineDraweeControllerBuilderSupplier sDraweeControllerBuilderSupplier;

  /** Initializes Drawee with the specified config. */
  private static void initializeDrawee(
      Context context,
      @Nullable DraweeConfig draweeConfig) {
    // 创建 DraweeController 的 Builder 提供器
    sDraweeControllerBuilderSupplier =
        new PipelineDraweeControllerBuilderSupplier(context, draweeConfig);
    SimpleDraweeView.initialize(sDraweeControllerBuilderSupplier);
  }

}
```

#### 1. 创建 PipelineDraweeControllerBuilderSupplier

```java
public class PipelineDraweeControllerBuilderSupplier implements
    Supplier<PipelineDraweeControllerBuilder> {

  private final Context mContext;
  private final ImagePipeline mImagePipeline;
  private final PipelineDraweeControllerFactory mPipelineDraweeControllerFactory;
  private final Set<ControllerListener> mBoundControllerListeners;

  public PipelineDraweeControllerBuilderSupplier(
      Context context,
      ImagePipelineFactory imagePipelineFactory,
      Set<ControllerListener> boundControllerListeners,
      @Nullable DraweeConfig draweeConfig) {
    mContext = context;
    mImagePipeline = imagePipelineFactory.getImagePipeline();
    if (draweeConfig != null && draweeConfig.getPipelineDraweeControllerFactory() != null) {
      mPipelineDraweeControllerFactory = draweeConfig.getPipelineDraweeControllerFactory();
    } else {
      // 创建默认的工厂对象
      mPipelineDraweeControllerFactory = new PipelineDraweeControllerFactory();
    }
    // 初始化 mPipelineDraweeControllerFactory 工厂, 为 PipelineDraweeControllerBuilder 创建 PipelineDraweeController 提供支持
    mPipelineDraweeControllerFactory.init(
        context.getResources(),
        DeferredReleaser.getInstance(),
        imagePipelineFactory.getAnimatedDrawableFactory(context),
        UiThreadImmediateExecutorService.getInstance(),
        mImagePipeline.getBitmapMemoryCache(),
        draweeConfig != null
            ? draweeConfig.getCustomDrawableFactories()
            : null,
        draweeConfig != null
            ? draweeConfig.getDebugOverlayEnabledSupplier()
            : null);
    mBoundControllerListeners = boundControllerListeners;
  }

}
```
PipelineDraweeControllerBuilderSupplier 中持有了 ImagePipeline 和 PipelineDraweeControllerFactory 两个非常重要的对象, 为后续构建 PiplelineDraweeController 提供支持, 这个我们到后面在具体分析

#### 2. 初始化 SimpleDraweeView

关于 initializeDrawee 的过程, 它主要是将 PipelineDraweeControllerBuilderSupplier 注入 SimpleDraweeView

```java
public class SimpleDraweeView extends GenericDraweeView {

  private static Supplier<? extends AbstractDraweeControllerBuilder>
      sDraweecontrollerbuildersupplier;

  /** Initializes {@link SimpleDraweeView} with supplier of Drawee controller builders. */
  public static void initialize(
      Supplier<? extends AbstractDraweeControllerBuilder> draweeControllerBuilderSupplier) {
    sDraweecontrollerbuildersupplier = draweeControllerBuilderSupplier;
  }

}
```

### 三) 回顾
Fresco 的初始化流程主要如下
- 加载 Fresco 的 so 库
- 构建 ImagePipelineFactory 对后续对图像的编解码提供支持
- 初始化 Drawee
  - 创建 PipelineDraweeControllerBuilderSupplier 用于提供  PipelineDraweeControllerBuilder 对象
    - 持有 ImagePipeline 和 PipelineDraweeControllerFactory 两个非常重要的对象
  - 为 SimpleDraweeView 注入 Supplier, 方便 SimpleDraweeView 构建 DraweeControllerBuilder.

![Fresco 的初始化](https://i.loli.net/2020/03/27/fTDca1nVURvPABE.png)

## 二. SimpleDraweeView 的创建

```java
public class SimpleDraweeView extends GenericDraweeView {

  public SimpleDraweeView(Context context, AttributeSet attrs, int defStyle) {
    // 父类初始化
    super(context, attrs, defStyle);
    // 4. 解析一些提供的属性
    init(context, attrs);
  }

}
public class GenericDraweeView extends DraweeView<GenericDraweeHierarchy> {

  @TargetApi(Build.VERSION_CODES.LOLLIPOP)
  public GenericDraweeView(Context context, AttributeSet attrs, int defStyleAttr, int defStyleRes) {
    super(context, attrs, defStyleAttr, defStyleRes);
    inflateHierarchy(context, attrs);
  }

  protected void inflateHierarchy(Context context, @Nullable AttributeSet attrs) {
   // 获取层级构造器
    GenericDraweeHierarchyBuilder builder =
        GenericDraweeHierarchyInflater.inflateBuilder(context, attrs);
    // 2. 设置期望的比例
    setAspectRatio(builder.getDesiredAspectRatio());
    // 3. 创建 GenericDraweeHierarchy 并注入 DraweeView
    setHierarchy(builder.build());
  }

}

public class DraweeView<DH extends DraweeHierarchy> extends ImageView {

  @TargetApi(Build.VERSION_CODES.LOLLIPOP)
  public DraweeView(Context context, AttributeSet attrs, int defStyleAttr, int defStyleRes) {
    super(context,attrs,defStyleAttr,defStyleRes);
    init(context);
  }

  /** This method is idempotent so it only has effect the first time it's called */
  private void init(Context context) {
    if (mInitialised) {
      return;
    }
    mInitialised = true;
    // 1. 创建 DraweeHolder
    mDraweeHolder = DraweeHolder.create(null, context);
    ......
  }

}
```

SimpleDraweeView 中只做了一些简单的属性解析, 其具体的初始化流程依靠的是父类

- 创建 DraweeHolder
- 设置期望的比例
- 创建 GenericDraweeHierarchy 并注入 DraweeView
- 解析 View 的属性

这里我们主要 focus DraweeHolder 的创建和 GenericDraweeHierarchy 的创建过程

### 一) 创建 DraweeHolder

```java
public class DraweeHolder<DH extends DraweeHierarchy>
    implements VisibilityCallback {

  public static <DH extends DraweeHierarchy> DraweeHolder<DH> create(
      @Nullable DH hierarchy,
      Context context) {
    // 以 hierarchy 为入参, 创建 Holder 对象
    DraweeHolder<DH> holder = new DraweeHolder<DH>(hierarchy);
    // 处理与 context 相关的数据
    holder.registerWithContext(context);
    return holder;
  }

  /**
   * Creates a new instance of DraweeHolder.
   * @param hierarchy
   */
  public DraweeHolder(@Nullable DH hierarchy) {
    // 通过 DraweeView 的 init 方法可知, 这个 hierarchy 目前为 null
    if (hierarchy != null) {
      setHierarchy(hierarchy);
    }
  }

  private DH mHierarchy;

  public void setHierarchy(DH hierarchy) {
    final boolean isControllerValid = isControllerValid();
    ......
    // 存储到成员变量中
    mHierarchy = Preconditions.checkNotNull(hierarchy);
    // 获取最顶层的 Drawable
    Drawable drawable = mHierarchy.getTopLevelDrawable();

    onVisibilityChange(drawable == null || drawable.isVisible());

    setVisibilityCallback(this);

    // 为 Controller 绑定 Hierarchy
    if (isControllerValid) {
      mController.setHierarchy(hierarchy);
    }
  }

}
```

可以看到 DraweeHolder.create 实例化 DraweeHolder 的过程主要会尝试注入 DraweeHierarchy, 这意味着 DraweeHolder 所要 Hold 的对象即 DraweeHierarchy

接下来我们看看 GenericDraweeHierarchy 的构建

### 二) 创建 GenericDraweeHierarchy

GenericDraweeHierarchy 的构建是由 GenericDraweeHierarchyBuilder.build 完成

```java
public class GenericDraweeHierarchyBuilder {

  ......

  /**
   * Builds the hierarchy.
   */
  public GenericDraweeHierarchy build() {
    validate();
    return new GenericDraweeHierarchy(this);
  }

}
```

下面看看 GenericDraweeHierarchy 的构造函数中做了哪些处理

```java
public class GenericDraweeHierarchy implements SettableDraweeHierarchy {

  private static final int BACKGROUND_IMAGE_INDEX = 0;
  private static final int PLACEHOLDER_IMAGE_INDEX = 1;
  private static final int ACTUAL_IMAGE_INDEX = 2;
  private static final int PROGRESS_BAR_IMAGE_INDEX = 3;
  private static final int RETRY_IMAGE_INDEX = 4;
  private static final int FAILURE_IMAGE_INDEX = 5;
  private static final int OVERLAY_IMAGES_INDEX = 6;

  private final Drawable mEmptyActualImageDrawable = new ColorDrawable(Color.TRANSPARENT);

  private final Resources mResources;
  private @Nullable RoundingParams mRoundingParams;

  private final RootDrawable mTopLevelDrawable;
  private final FadeDrawable mFadeDrawable;
  private final ForwardingDrawable mActualImageWrapper;

  GenericDraweeHierarchy(GenericDraweeHierarchyBuilder builder) {
    // 资源信息
    mResources = builder.getResources();

    // 计算要叠加额外叠加的层级
    int numOverlays = (builder.getOverlays() != null) ? builder.getOverlays().size() : 1;
    numOverlays += (builder.getPressedStateOverlay() != null) ? 1 : 0;
    // 要叠加的总层级 = 通用层级 + 额外层级
    int numLayers = OVERLAY_IMAGES_INDEX + numOverlays;

    // 1. 创建一个 Drawable 数组, 填充通用层级特定的 Drawable 对象
    Drawable[] layers = new Drawable[numLayers];
    // 2. 初始化通用层级
    // 2.1 创建背景 Drawable
    layers[BACKGROUND_IMAGE_INDEX] = buildBranch(builder.getBackground(), null);
    // 2.2 占位 Drawable
    layers[PLACEHOLDER_IMAGE_INDEX] = buildBranch(
        builder.getPlaceholderImage(),
        builder.getPlaceholderImageScaleType());
    // 2.3 真实 Drawable
    mActualImageWrapper = new ForwardingDrawable(mEmptyActualImageDrawable);
    layers[ACTUAL_IMAGE_INDEX] = buildActualImageBranch(
        mActualImageWrapper,
        builder.getActualImageScaleType(),
        builder.getActualImageFocusPoint(),
        builder.getActualImageColorFilter());
    // 2.4 加载进度条的 Drawable
    layers[PROGRESS_BAR_IMAGE_INDEX] = buildBranch(
        builder.getProgressBarImage(),
        builder.getProgressBarImageScaleType());
    // 2.5 点击重试的 Drawable
    layers[RETRY_IMAGE_INDEX] = buildBranch(
        builder.getRetryImage(),
        builder.getRetryImageScaleType());
    // 2.6 加载失败的 Drawable
    layers[FAILURE_IMAGE_INDEX] = buildBranch(
        builder.getFailureImage(),
        builder.getFailureImageScaleType());

    // 3. 初始化额外叠加的层级
    if (numOverlays > 0) {
      int index = 0;
      // 3.1 实例化额外的层级
      if (builder.getOverlays() != null) {
        for (Drawable overlay : builder.getOverlays()) {
          layers[OVERLAY_IMAGES_INDEX + index++] = buildBranch(overlay, null);
        }
      } else {
        index = 1; // reserve space for one overlay
      }
      // 3.2 保证按压效果的 Drawable 在最顶层
      if (builder.getPressedStateOverlay() != null) {
        layers[OVERLAY_IMAGES_INDEX + index] = buildBranch(builder.getPressedStateOverlay(), null);
      }
    }

    // 4. 将所有的层级封装到 FadeDrawable 中
    mFadeDrawable = new FadeDrawable(layers);
    mFadeDrawable.setTransitionDuration(builder.getFadeDuration());

    // 5. 若需要圆角, 则对 FadeDrawable 进行包装
    // 圆角参数
    mRoundingParams = builder.getRoundingParams();
    Drawable maybeRoundedDrawable =
        WrappingUtils.maybeWrapWithRoundedOverlayColor(mFadeDrawable, mRoundingParams);

    // 6. 构建最终的 Drawable
    mTopLevelDrawable = new RootDrawable(maybeRoundedDrawable);
    mTopLevelDrawable.mutate();

    // 7. 重置 FadeDrawable 的状态
    resetFade();
  }

}
```

看到这里我们就清楚了 GenericDraweeHierarchy 的职责了, 它负责创建和组织需要上屏的 Drawable 集合, 层级如下

- RootDrawable
  - RoundedDrawable
    - FadeDrawable
  - FadeDrawable
    - 按压的 Drawable
    - Overlay 的 Drawables
    - 失败 Drawable: ScaleTypeDrawable
    - 重试 Drawable: ScaleTypeDrawable
    - 进度条 Drawable: maybe null
    - 真实目标的 Drawable: ScaleTypeDrawable(提供 ScaleType 缩放能力)
      - ForwardingDrawable (包装一下, 提供拓展空间)
        - Drawable(用户设置的)
    - 展位图 Drawable: ScaleTypeDrawable
    - 背景 Drawable

### 三) 回顾
![SimpleDraweeView 层级](https://i.loli.net/2020/03/27/W1n3ORJqeF6UIrt.png)

SimpleDraweeView 的创建流程如下

- 创建 DraweeHolder
  - 持有 DraweeHierarchy
- 创建 GenericDraweeHierarchy
  - RootDrawable
    - RoundedDrawable
      - FadeDrawable
    - FadeDrawable
      - 按压的 Drawable
      - Overlay 的 Drawables
      - 失败 Drawable: ScaleTypeDrawable
      - 重试 Drawable: ScaleTypeDrawable
      - 进度条 Drawable: maybe null
      - 真实目标的 Drawable: ScaleTypeDrawable(提供 ScaleType 缩放能力)
        - ForwardingDrawable (包装一下, 提供拓展空间)
          - Drawable(用户设置的)
      - 展位图 Drawable: ScaleTypeDrawable
      - 背景 Drawable
- 将 GenericDraweeHierarchy 注入 Holder 中暂存

## 三. 数据的加载

经过了上面两个步骤, Fresco 的资源就初始化好了, 我们只需要在应用层设置一个 Controller, 就可以触发图片的加载了

```java
val controller = Fresco.newDraweeControllerBuilder()
        // 构建 ImageRequest, 表示配置请求图片信息
        .setImageRequest(
                ImageRequestBuilder.newBuilderWithSource(Uri.parse("xxx"))
                        // 在 ImagePipeline 解码时改变内存中图片的大小
                        .setResizeOptions(ResizeOptions.forDimensions(100, 100))
                        // 根据图片信息自动旋转
                        .setRotationOptions(RotationOptions.autoRotate())
                        .build()
        )
        .build()
// 注入加载控制器
draweeView.controller = controller
```

DraweeController 中提供的参数非常丰富, 更多使用方式可以参考 [Fresco Controller](https://www.fresco-cn.org/docs/using-controllerbuilder.html)

这里的 Controller 为 PipelineDraweeController, 我们先看看它的实例化过程

### 一) PipelineDraweeController 的创建

```java
public class Fresco {
  /** Returns a new instance of Fresco Drawee controller builder. */
  public static PipelineDraweeControllerBuilder newDraweeControllerBuilder() {
    return sDraweeControllerBuilderSupplier.get();
  }
}
```

sDraweeControllerBuilderSupplier 的实例化, 我们在 Fresco 初始化的时候已经分析过了, 它是一个 PipelineDraweeControllerBuilderSupplier 对象, 这里看看它获取 PipelineDraweeControllerBuilder 的流程

#### 1. PipelineDraweeControllerBuilderSupplier 获取 PipelineDraweeControllerBuilder 对象

````java
public class PipelineDraweeControllerBuilderSupplier implements
    Supplier<PipelineDraweeControllerBuilder> {

  private final Context mContext;
  private final ImagePipeline mImagePipeline;
  private final PipelineDraweeControllerFactory mPipelineDraweeControllerFactory;
  private final Set<ControllerListener> mBoundControllerListeners;

  @Override
  public PipelineDraweeControllerBuilder get() {
    // 创建了 PipelineDraweeControllerBuilder
    return new PipelineDraweeControllerBuilder(
        mContext,
        mPipelineDraweeControllerFactory,
        mImagePipeline,
        mBoundControllerListeners);
  }
}
````

get 方法直接创建了一个 PipelineDraweeControllerBuilder 对象

```java
public class PipelineDraweeControllerBuilder extends AbstractDraweeControllerBuilder<
    PipelineDraweeControllerBuilder,
    ImageRequest,
    CloseableReference<CloseableImage>,
    ImageInfo> {

  private final ImagePipeline mImagePipeline;
  private final PipelineDraweeControllerFactory mPipelineDraweeControllerFactory;

  ......

  public PipelineDraweeControllerBuilder(
      Context context,
      PipelineDraweeControllerFactory pipelineDraweeControllerFactory,
      ImagePipeline imagePipeline,
      Set<ControllerListener> boundControllerListeners) {
    // 父类初始化
    super(context, boundControllerListeners);
    // 将 ImagePipleline 和 PipelineDraweeControllerFactory 注入成员变量
    mImagePipeline = imagePipeline;
    mPipelineDraweeControllerFactory = pipelineDraweeControllerFactory;
  }
}

public abstract class AbstractDraweeControllerBuilder <
    BUILDER extends AbstractDraweeControllerBuilder<BUILDER, REQUEST, IMAGE, INFO>,
    REQUEST,
    IMAGE,
    INFO>
    implements SimpleDraweeControllerBuilder {

  // components
  private final Context mContext;
  private final Set<ControllerListener> mBoundControllerListeners;
  ......

  protected AbstractDraweeControllerBuilder(
      Context context,
      Set<ControllerListener> boundControllerListeners) {
    // 将形参注入成员变量
    mContext = context;
    mBoundControllerListeners = boundControllerListeners;
    init();
  }

  /** Initializes this builder. */
  private void init() {
    // 重置成员变量
    ......
  }
}
```

PipelineDraweeControllerBuilder 实例化过程即将形参注入成员变量, 并没有做额外的逻辑操作

下面我们看看 PipelineDraweeControllerBuilder.build() 是如何创建 PipelineDraweeController 的

#### 2. PipelineDraweeControllerBuilder 构造 PipelineDraweeController 对象

```java
public abstract class AbstractDraweeControllerBuilder <
    BUILDER extends AbstractDraweeControllerBuilder<BUILDER, REQUEST, IMAGE, INFO>,
    REQUEST,
    IMAGE,
    INFO>
    implements SimpleDraweeControllerBuilder {

  /** Builds the specified controller. */
  @Override
  public AbstractDraweeController build() {
    ......
    // if only a low-res request is specified, treat it as a final request.
    if (mImageRequest == null && mMultiImageRequests == null && mLowResImageRequest != null) {
      mImageRequest = mLowResImageRequest;
      mLowResImageRequest = null;
    }
    // 1. 将构建工作转发给了 buildController
    return buildController();
  }

  /** Builds a regular controller. */
  protected AbstractDraweeController buildController() {
    // 2. 获取 Controller 对象
    AbstractDraweeController controller = obtainController();
    controller.setRetainImageOnFailure(getRetainImageOnFailure());
    controller.setContentDescription(getContentDescription());
    ......
    return controller;
  }

  @ReturnsOwnership protected abstract AbstractDraweeController obtainController();
}
```

这里会调用 obtainController 方法来获取一个 AbstractDraweeController 对象, 这是一个抽象方法, 它的具体实现如下

```java
public class PipelineDraweeControllerBuilder extends AbstractDraweeControllerBuilder<
    PipelineDraweeControllerBuilder,
    ImageRequest,
    CloseableReference<CloseableImage>,
    ImageInfo> {

  @Override
  protected PipelineDraweeController obtainController() {
    DraweeController oldController = getOldController();
    PipelineDraweeController controller;
    if (oldController instanceof PipelineDraweeController) {
      ....
    } else {
      // 交由工厂执行构造操作
      controller =
          mPipelineDraweeControllerFactory.newController(
              // 获取 Supplier<DataSource<CloseableReference<CloseableImage>>>
              obtainDataSourceSupplier(),
              generateUniqueControllerId(),
              getCacheKey(),
              getCallerContext(),
              mCustomDrawableFactories,
              mImageOriginListener);
    }
    return controller;
  }

}
```

Builder 最终会调用 Factory 去执行 PiplelineDraweeController 的创建, 在分析其创建之前, 我们需要关注 obtainDataSourceSupplier 这个方法, 它是用户想要加载数据源的提供器, 下面看看它的实现

##### 1) 构建 DataSource 提供器

```java
public abstract class AbstractDraweeControllerBuilder <
    BUILDER extends AbstractDraweeControllerBuilder<BUILDER, REQUEST, IMAGE, INFO>,
    REQUEST,
    IMAGE,
    INFO>
    implements SimpleDraweeControllerBuilder {

  /** Gets the top-level data source supplier to be used by a controller. */
  protected Supplier<DataSource<IMAGE>> obtainDataSourceSupplier() {
    if (mDataSourceSupplier != null) {
      return mDataSourceSupplier;
    }

    Supplier<DataSource<IMAGE>> supplier = null;

    // final image supplier;
    if (mImageRequest != null) {
      // 通过 Request 构建 DataSourceSupplier
      supplier = getDataSourceSupplierForRequest(mImageRequest);
    } else if (mMultiImageRequests != null) {
      supplier = getFirstAvailableDataSourceSupplier(mMultiImageRequests, mTryCacheOnlyFirst);
    }
    ......
    return supplier;
  }

  protected Supplier<DataSource<IMAGE>> getDataSourceSupplierForRequest(REQUEST imageRequest) {
    return getDataSourceSupplierForRequest(imageRequest, CacheLevel.FULL_FETCH);
  }

  protected Supplier<DataSource<IMAGE>> getDataSourceSupplierForRequest(
      final REQUEST imageRequest,
      final CacheLevel cacheLevel) {
    final Object callerContext = getCallerContext();
    // 构建了一个一个 DataSource 类型的 Supplier
    return new Supplier<DataSource<IMAGE>>() {
      @Override
      public DataSource<IMAGE> get() {
        // 获取 DataSource
        return getDataSourceForRequest(imageRequest, callerContext, cacheLevel);
      }
      @Override
      public String toString() {
        return Objects.toStringHelper(this)
            .add("request", imageRequest.toString())
            .toString();
      }
    };
  }

}

public class PipelineDraweeControllerBuilder extends AbstractDraweeControllerBuilder<
    PipelineDraweeControllerBuilder,
    ImageRequest,
    CloseableReference<CloseableImage>,
    ImageInfo> {
  @Override
  protected DataSource<CloseableReference<CloseableImage>> getDataSourceForRequest(
      ImageRequest imageRequest,
      Object callerContext,
      AbstractDraweeControllerBuilder.CacheLevel cacheLevel) {
    return mImagePipeline.fetchDecodedImage(
        imageRequest,
        callerContext,
        convertCacheLevelToRequestLevel(cacheLevel));
  }
}
```

可以看到 Supplier<DataSource> 的 get 方法最终返回的 DataSource 是由 mImagePipeline.fetchDecodedImage() 拿到的, 我们先看到这里, 关于具体获取数据源的过程我们到后面再分析

##### 2) 工厂实例化 PipelineDraweeController 对象

```java
public class PipelineDraweeControllerFactory {

  public PipelineDraweeController newController(
      Supplier<DataSource<CloseableReference<CloseableImage>>> dataSourceSupplier,
      String id,
      CacheKey cacheKey,
      Object callerContext,
      @Nullable ImmutableList<DrawableFactory> customDrawableFactories,
      @Nullable ImageOriginListener imageOriginListener) {
    // 实例化 PipelineDraweeController 对象
    PipelineDraweeController controller =
        internalCreateController(
            mResources,
            mDeferredReleaser,
            mAnimatedDrawableFactory,
            mUiThreadExecutor,
            mMemoryCache,
            mDrawableFactories,
            customDrawableFactories,
            dataSourceSupplier,
            id,
            cacheKey,
            callerContext);
    ......
    controller.setImageOriginListener(imageOriginListener);
    return controller;
  }


  protected PipelineDraweeController internalCreateController(
      Resources resources,
      DeferredReleaser deferredReleaser,
      DrawableFactory animatedDrawableFactory,
      Executor uiThreadExecutor,
      MemoryCache<CacheKey, CloseableImage> memoryCache,
      @Nullable ImmutableList<DrawableFactory> globalDrawableFactories,
      @Nullable ImmutableList<DrawableFactory> customDrawableFactories,
      Supplier<DataSource<CloseableReference<CloseableImage>>> dataSourceSupplier,
      String id,
      CacheKey cacheKey,
      Object callerContext) {
    // 直接 new 了一个对象
    PipelineDraweeController controller =
        new PipelineDraweeController(
            resources,
            deferredReleaser,
            animatedDrawableFactory,
            uiThreadExecutor,
            memoryCache,
            dataSourceSupplier,
            id,
            cacheKey,
            callerContext,
            globalDrawableFactories);
    controller.setCustomDrawableFactories(customDrawableFactories);
    return controller;
  }
}
```

下面我们看看这个 PipelineDraweeController 实例化的时候做了哪些操作

```java
public class PipelineDraweeController
    extends AbstractDraweeController<CloseableReference<CloseableImage>, ImageInfo> {

  private final Resources mResources;
  private final DrawableFactory mAnimatedDrawableFactory;
  private @Nullable MemoryCache<CacheKey, CloseableImage> mMemoryCache;
  private CacheKey mCacheKey;
  private final ImmutableList<DrawableFactory> mGlobalDrawableFactories;

  public PipelineDraweeController(
      Resources resources,
      DeferredReleaser deferredReleaser,
      DrawableFactory animatedDrawableFactory,
      Executor uiThreadExecutor,
      MemoryCache<CacheKey, CloseableImage> memoryCache,
      Supplier<DataSource<CloseableReference<CloseableImage>>> dataSourceSupplier,
      String id,
      CacheKey cacheKey,
      Object callerContext,
      @Nullable ImmutableList<DrawableFactory> globalDrawableFactories) {
    // 父类初始化
    super(deferredReleaser, uiThreadExecutor, id, callerContext);
    // 将参数注入成员变量
    mResources = resources;
    mAnimatedDrawableFactory = animatedDrawableFactory;
    mMemoryCache = memoryCache;
    mCacheKey = cacheKey;
    mGlobalDrawableFactories = globalDrawableFactories;
    init(dataSourceSupplier);
  }

  // Constant state (non-final because controllers can be reused)
  private Supplier<DataSource<CloseableReference<CloseableImage>>> mDataSourceSupplier;

  private void init(Supplier<DataSource<CloseableReference<CloseableImage>>> dataSourceSupplier) {
    mDataSourceSupplier = dataSourceSupplier;
  }

}
```

PipelineDraweeController 的构造函数并没有做什么特殊的事务, 它将形参中的数据暂存到了成员变量中

其中需要我们特别注意的是 mDataSourceSupplier, 它是数据源的提供者承担着数据加载的重任

下面我们看看它的父类是如何初始化的

```java
public abstract class AbstractDraweeController<T, INFO> implements
    DraweeController,
    DeferredReleaser.Releasable,
    GestureDetector.ClickListener {

  public AbstractDraweeController(
      DeferredReleaser deferredReleaser,
      Executor uiThreadImmediateExecutor,
      String id,
      Object callerContext) {
    mDeferredReleaser = deferredReleaser;
    mUiThreadImmediateExecutor = uiThreadImmediateExecutor;
    // 执行初始化操作
    init(id, callerContext, true);
  }

  private void init(String id, Object callerContext, boolean justConstructed) {
    mEventTracker.recordEvent(Event.ON_INIT_CONTROLLER);
    // cancel deferred release
    if (!justConstructed && mDeferredReleaser != null) {
      mDeferredReleaser.cancelDeferredRelease(this);
    }
    // 重置状态
    mIsAttached = false;
    mIsVisibleInViewportHint = false;
    releaseFetch();
    mRetainImageOnFailure = false;
    // 重新初始化 mRetryManager
    if (mRetryManager != null) {
      mRetryManager.init();
    }
    // 重置手势探测器
    if (mGestureDetector != null) {
      mGestureDetector.init();
      mGestureDetector.setClickListener(this);
    }
    // 重置 mControllerListener
    if (mControllerListener instanceof InternalForwardingListener) {
      ((InternalForwardingListener) mControllerListener).clearListeners();
    } else {
      mControllerListener = null;
    }
    mControllerViewportVisibilityListener = null;
    // 清空之前绑定的 DraweeHierarchy
    if (mSettableDraweeHierarchy != null) {
      mSettableDraweeHierarchy.reset();
      mSettableDraweeHierarchy.setControllerOverlay(null);
      mSettableDraweeHierarchy = null;
    }
    mControllerOverlay = null;
    // reinitialize constant state
    if (FLog.isLoggable(FLog.VERBOSE)) {
      FLog.v(TAG, "controller %x %s -> %s: initialize", System.identityHashCode(this), mId, id);
    }
    mId = id;
    mCallerContext = callerContext;
  }

}
```

可以看到 PipelineDraweeController 初始化的操作主要是为成员变量赋初值, 以及重置可变状态的数据

#### 3. 回顾
![PipelineDraweeController构建流程](https://i.loli.net/2020/03/27/UpRWd3wjaXC2hnb.png)

### 二) 为 DraweeView 注入 Controller

这里我们就看看 setController 的时候如何加载 uri 对应的图片, 这个方法直接定义在 DraweeView 中

```java
public class DraweeView<DH extends DraweeHierarchy> extends ImageView {

  /** Sets the controller. */
  public void setController(@Nullable DraweeController draweeController) {
    // 为 DraweeHolder 注入 Controller
    mDraweeHolder.setController(draweeController);
    // 获取顶层的 Hierarchy 的 Drawable, 让 ImageView 展示
    super.setImageDrawable(mDraweeHolder.getTopLevelDrawable());
  }

}
```

可以看到 DraweeView 将注入 Controller 的动作分发给了 DraweeHolder, 其实现如下

```java
public class DraweeHolder<DH extends DraweeHierarchy>
    implements VisibilityCallback {


  /**
   * Sets a new controller.
   */
  public void setController(@Nullable DraweeController draweeController) {
    // 1. 解绑之前的 Controller
    boolean wasAttached = mIsControllerAttached;
    if (wasAttached) {
      detachController();
    }

    // 2. 让之前 Controller 与 Hierarchy 解绑
    if (isControllerValid()) {
      mEventTracker.recordEvent(Event.ON_CLEAR_OLD_CONTROLLER);
      mController.setHierarchy(null);
    }
    // 3, 更新 mController 对象
    mController = draweeController;
    if (mController != null) {
      mEventTracker.recordEvent(Event.ON_SET_CONTROLLER);
      // 3.1 绑定 Hierarchy
      mController.setHierarchy(mHierarchy);
    } else {
      mEventTracker.recordEvent(Event.ON_CLEAR_CONTROLLER);
    }

    // 4 绑定 Controller
    if (wasAttached) {
      attachController();
    }
  }

}
```

可以看到这里主要的操作有两个步骤

- 解绑之前 Controller
  - 解绑 Hierarchy
  - 回调 Controller 的 onDetach
- 绑定新的 Controller
  - 绑定 Hierarchy
  - 回调 Controller 的 onAttach

这里我们主要 focus 一下新 Controller 的绑定过程

#### 1. 绑定 DraweeHierarchy

PipelineDraweeController 的

```java
public abstract class AbstractDraweeController<T, INFO> implements
    DraweeController,
    DeferredReleaser.Releasable,
    GestureDetector.ClickListener {

  /**
   * Sets the hierarchy.
   *
   * <p>The controller should be detached when this method is called.
   * @param hierarchy This must be an instance of {@link SettableDraweeHierarchy}
   */
  @Override
  public void setHierarchy(@Nullable DraweeHierarchy hierarchy) {
    ......
    // 清空之前回调的数据
    if (mIsRequestSubmitted) {
      mDeferredReleaser.cancelDeferredRelease(this);
      release();
    }
    // 清空之前的 DraweeHierarchy
    if (mSettableDraweeHierarchy != null) {
      mSettableDraweeHierarchy.setControllerOverlay(null);
      mSettableDraweeHierarchy = null;
    }
    // 更新到成员变量 mSettableDraweeHierarchy 中
    if (hierarchy != null) {
      Preconditions.checkArgument(hierarchy instanceof SettableDraweeHierarchy);
      mSettableDraweeHierarchy = (SettableDraweeHierarchy) hierarchy;
      ......
    }
  }

}
```

可以看到为 PipelineDraweeController 设置新的 Hierarchy 的过程主要是清空之前的数据之后将新的 Hierarchy 注入到成员变量

除此之之外没有做额外的工作, 接下来我们看看 onAttach 中做了些什么

#### 2. 回调 onAttach

```java
public abstract class AbstractDraweeController<T, INFO> implements
    DraweeController,
    DeferredReleaser.Releasable,
    GestureDetector.ClickListener {

  @Override
  public void onAttach() {
    ......
    mDeferredReleaser.cancelDeferredRelease(this);
    mIsAttached = true;
    if (!mIsRequestSubmitted) {
      // 提交这个请求
      submitRequest();
    }
  }
}
```

可以看到, 在绑定了 Hierarchy 之后, 会提交这个请求, 这便是图片加载的核心所在了, 下面我们看看它的实现

```java
public abstract class AbstractDraweeController<T, INFO> implements
    DraweeController,
    DeferredReleaser.Releasable,
    GestureDetector.ClickListener {

  protected void submitRequest() {
    // 1. 从缓存中直接获取当前 Controller 要加载的图片
    final T closeableImage = getCachedImage();
    if (closeableImage != null) {
      mDataSource = null;
      mIsRequestSubmitted = true;
      mHasFetchFailed = false;
      ......
      getControllerListener().onSubmit(mId, mCallerContext);
      onImageLoadedFromCacheImmediately(mId, closeableImage);
      // 回调图片加载成功
      onNewResultInternal(mId, mDataSource, closeableImage, 1.0f, true, true);
      return;
    }
    getControllerListener().onSubmit(mId, mCallerContext);
    // 重置加载进度
    mSettableDraweeHierarchy.setProgress(0, true);
    mIsRequestSubmitted = true;
    mHasFetchFailed = false;
    // 2. 获取数据源
    mDataSource = getDataSource();
    final String id = mId;
    final boolean wasImmediate = mDataSource.hasResult();
    // 3. 构建观察者
    final DataSubscriber<T> dataSubscriber =
        new BaseDataSubscriber<T>() {
          @Override
          public void onNewResultImpl(DataSource<T> dataSource) {
            // isFinished must be obtained before image, otherwise we might set intermediate result
            // as final image.
            boolean isFinished = dataSource.isFinished();
            float progress = dataSource.getProgress();
            T image = dataSource.getResult();
            if (image != null) {
              // 加载成功
              onNewResultInternal(id, dataSource, image, progress, isFinished, wasImmediate);
            } else if (isFinished) {
              // 加载失败
              onFailureInternal(id, dataSource, new NullPointerException(), /* isFinished */ true);
            }
          }
          @Override
          public void onFailureImpl(DataSource<T> dataSource) {
            onFailureInternal(id, dataSource, dataSource.getFailureCause(), /* isFinished */ true);
          }
          @Override
          public void onProgressUpdate(DataSource<T> dataSource) {
            // 跟新加载进度
            boolean isFinished = dataSource.isFinished();
            float progress = dataSource.getProgress();
            onProgressUpdateInternal(id, dataSource, progress, isFinished);
          }
        };
    // 4. 为数据源添加一个观察者
    mDataSource.subscribe(dataSubscriber, mUiThreadImmediateExecutor);
  }

}
```

这里我们就初步看到 Fresco 的图片加载动作了, 主要流程如下

- 尝试从缓存中获取数据, 若命中则直接返回
- 通过 getDataSource 获取数据源
- 构建这个数据源的观察者 DataSubscriber
- 为数据源订阅观察者, 数据源准备好之后便会通知观察者, 回调其中的方法

这里我们主要关注一下 getDataSource 是如何构建数据源的

```java
public class PipelineDraweeController
    extends AbstractDraweeController<CloseableReference<CloseableImage>, ImageInfo> {
  @Override
  protected DataSource<CloseableReference<CloseableImage>> getDataSource() {
    ......
    // 直接通过 mDataSourceSupplier 获取 DataSource
    return mDataSourceSupplier.get();
  }
}
```

通过上面分析可知, get 方法最终会调用到 ImagePipeline.fetchDecodedImage, 下面看看它的具体实现

```java
public class ImagePipeline {

  public DataSource<CloseableReference<CloseableImage>> fetchDecodedImage(
      ImageRequest imageRequest,
      Object callerContext,
      ImageRequest.RequestLevel lowestPermittedRequestLevelOnSubmit) {
    try {
      // 1. 创建数据生产者
      Producer<CloseableReference<CloseableImage>> producerSequence =
          mProducerSequenceFactory.getDecodedImageProducerSequence(imageRequest);
      // 提交这个获取请求
      return submitFetchRequest(
          producerSequence,
          imageRequest,
          lowestPermittedRequestLevelOnSubmit,
          callerContext);
    } catch (Exception exception) {
      ......// 处理失败的情况
    }
  }

  private <T> DataSource<CloseableReference<T>> submitFetchRequest(
      Producer<CloseableReference<T>> producerSequence,
      ImageRequest imageRequest,
      ImageRequest.RequestLevel lowestPermittedRequestLevelOnSubmit,
      Object callerContext) {
    final RequestListener requestListener = getRequestListenerForRequest(imageRequest);
    try {
      // 获取请求优先级
      ImageRequest.RequestLevel lowestPermittedRequestLevel =
          ImageRequest.RequestLevel.getMax(
              imageRequest.getLowestPermittedRequestLevel(),
              lowestPermittedRequestLevelOnSubmit);
      // 2. 构建生产者上下文数据
      SettableProducerContext settableProducerContext = new SettableProducerContext(
          imageRequest,
          generateUniqueFutureId(),
          requestListener,
          callerContext,
          lowestPermittedRequestLevel,
        /* isPrefetch */ false,
          imageRequest.getProgressiveRenderingEnabled() ||
              imageRequest.getMediaVariations() != null ||
              !UriUtil.isNetworkUri(imageRequest.getSourceUri()),
          imageRequest.getPriority());
      // 3. 调用 CloseableProducerToDataSourceAdapter 的 create 方法, 创建 DataSource
      return CloseableProducerToDataSourceAdapter.create(
          // 生产者序列
          producerSequence,
          // 生产者上下文
          settableProducerContext,
          // 请求监听
          requestListener);
    } catch (Exception exception) {
      ... // 处理失败情况
    }
  }
}
```

可以看到 ImagePipeline 的构建过程, 主要有如下几个步骤

- 构建 Producer<CloseableReference<CloseableImage>> producerSequence 数据生产者
- 构建 SettableProducerContext 生产者上下文
- 通过  CloseableProducerToDataSourceAdapter.create 将一个 Producer 通过 Adapter 的方式适配成 DataSource

这里我们主要关注生产者的构建和 DataSource 的创建

##### 1) 创建数据生产者

```java
public class ProducerSequenceFactory {

  public Producer<CloseableReference<CloseableImage>> getDecodedImageProducerSequence(
      ImageRequest imageRequest) {
    // 1. 获取最基础的图片解码生产者
    Producer<CloseableReference<CloseableImage>> pipelineSequence =
        getBasicDecodedImageSequence(imageRequest);
    // 2. 若 imageRequest 的 Postprocessor 非空, 则对基础的 Producer 进行装饰, 增强其功能
    if (imageRequest.getPostprocessor() != null) {
      pipelineSequence = getPostprocessorSequence(pipelineSequence);
    }
    // 3. 若设置了需要在 mUseBitmapPrepareToDraw 标记位, 则再使用一层装饰, 增强这个功能
    if (mUseBitmapPrepareToDraw) {
      pipelineSequence = getBitmapPrepareSequence(pipelineSequence);
    }
    return pipelineSequence;
  }
}
```

从这里可以看到 Fresco 是采用装饰器的方式对生产者的能力进行增强, 由于篇幅已经很长了, 生产者的具体实现就不再这里展开了, 感兴趣的同学可以先顺着这个思路深入探究

##### 2) 将生产者适配成 DataSource

下面我们主要看看 CloseableProducerToDataSourceAdapter.create 是如何构建 DataSource 的

```java
public class CloseableProducerToDataSourceAdapter<T>
    extends AbstractProducerToDataSourceAdapter<CloseableReference<T>> {

  public static <T> DataSource<CloseableReference<T>> create(
      Producer<CloseableReference<T>> producer,
      SettableProducerContext settableProducerContext,
      RequestListener listener) {
    // new 了这个对象
    return new CloseableProducerToDataSourceAdapter<T>(
        producer, settableProducerContext, listener);
  }

  private CloseableProducerToDataSourceAdapter(
      Producer<CloseableReference<T>> producer,
      SettableProducerContext settableProducerContext,
      RequestListener listener) {
    // 回调父类方法
    super(producer, settableProducerContext, listener);
  }

}

public abstract class AbstractProducerToDataSourceAdapter<T> extends AbstractDataSource<T>
    implements HasImageRequest {

  private final SettableProducerContext mSettableProducerContext;
  private final RequestListener mRequestListener;

  protected AbstractProducerToDataSourceAdapter(
      Producer<T> producer,
      SettableProducerContext settableProducerContext,
      RequestListener requestListener) {
    mSettableProducerContext = settableProducerContext;
    mRequestListener = requestListener;
    mRequestListener.onRequestStart(
        settableProducerContext.getImageRequest(),
        mSettableProducerContext.getCallerContext(),
        mSettableProducerContext.getId(),
        mSettableProducerContext.isPrefetch());
    // 1. 创建 Consumer 用于接收生产后的数据
    // 2. 调用 producer 的 produceResults 方法, 进行数据生产
    producer.produceResults(createConsumer(), settableProducerContext);
  }

  private Consumer<T> createConsumer() {
    return new BaseConsumer<T>() {
      ......
  }
}
```
从这里可以看到 Fresco 的数据生产与接收是一个生产者与消费者的模型, DataSource 是对外提供的接口, 其内部由 Producer 和 Consumer 组成

### 三) 回顾
![数据加载流程](https://i.loli.net/2020/03/27/MFai8LBDt3njHkc.png)

Drawee 的数据加载主要步骤如下
- PipelineDraweeController 创建
  - 通过 PipelineDraweeControllerBuilderSupplier 获取 PipelineDraweeControllerBuilder
  - 通过 PipelineDraweeControllerBuilder 构造 PipelineDraweeController 对象
    - 构建 DataSource 提供器
      - 最终由 ImagePipeline.fetchDecodedImage 获取到 ImageRequest 对应的 DataSource
    - 交由 PipelineDraweeControllerFactory 工厂创建 PiplelineDrawee 对象
      - 注入成员变量, 为可变参数赋初值
- 为 DraweeView 注入 Controller
  - 绑定 DraweeHierarchy
  - 回调 onAttach
    - 通过  ImagePipeline.fetchDecodedImage 获取 DataSource
      - 创建数据生产者 Producer, 使用装饰器对其功能进行增强
      - 为生产者注入 Consumer, 生产的结果通过 Consumer 对外抛出
    - 为 DataSource 注入观察者, 用于监听数据加载状态

## 总结
![整体结构图](https://i.loli.net/2020/03/27/z3yHU6PET5O7Iv2.png)

### 初始化
Fresco 的初始化流程主要如下
- 加载 Fresco 的 so 库
- 构建 ImagePipelineFactory 对后续对图像的编解码提供支持
- 初始化 Drawee
  - 创建 PipelineDraweeControllerBuilderSupplier 用于提供  PipelineDraweeControllerBuilder 对象
    - 持有 ImagePipeline 和 PipelineDraweeControllerFactory 两个非常重要的对象
  - 为 SimpleDraweeView 注入 Supplier, 方便 SimpleDraweeView 构建 DraweeControllerBuilder.

### SimpleDraweeView 的创建

SimpleDraweeView 的创建流程如下

- 创建 DraweeHolder
  - 持有 DraweeHierarchy
- 创建 GenericDraweeHierarchy
  - RootDrawable
    - RoundedDrawable
      - FadeDrawable
    - FadeDrawable
      - 按压的 Drawable
      - Overlay 的 Drawables
      - 失败 Drawable: ScaleTypeDrawable
      - 重试 Drawable: ScaleTypeDrawable
      - 进度条 Drawable: maybe null
      - 真实目标的 Drawable: ScaleTypeDrawable(提供 ScaleType 缩放能力)
        - ForwardingDrawable (包装一下, 提供拓展空间)
          - Drawable(用户设置的)
      - 展位图 Drawable: ScaleTypeDrawable
      - 背景 Drawable
- 将 GenericDraweeHierarchy 注入 Holder 中暂存

### 数据加载

Drawee 的数据加载主要步骤如下

- PipelineDraweeController 创建
  - 通过 PipelineDraweeControllerBuilderSupplier 获取 PipelineDraweeControllerBuilder
  - 通过 PipelineDraweeControllerBuilder 构造 PipelineDraweeController 对象
    - 构建 DataSource 提供器
      - 最终由 ImagePipeline.fetchDecodedImage 获取到 ImageRequest 对应的 DataSource
    - 交由 PipelineDraweeControllerFactory 工厂创建 PiplelineDrawee 对象
      - 注入成员变量, 为可变参数赋初值
- 为 DraweeView 注入 Controller
  - 绑定 DraweeHierarchy
  - 回调 onAttach
    - 通过  ImagePipeline.fetchDecodedImage 获取 DataSource
      - 创建数据生产者 Producer, 使用装饰器对其功能进行增强
      - 为生产者注入 Consumer, 生产的结果通过 Consumer 对外抛出
    - 为 DataSource 注入观察者, 用于监听数据加载状态

## 结语
这里梳理了一遍 Fresco 的加载流程, 能够感受到 Facebook 的工程师设计功底的深厚的, 这里简单的记录一下自己的阅读感受

从使用层面上来看
- Fresco
  - 优势: 提供了 DraweeView, 其中的 Hierarchy 提供了非常丰富的层级, 能够应对绝大多数场景, 使用方便
  - 劣势: 对控件进行二次包装存在一定的成本, 容易产生直接性的依赖
- Glide
  - 优势: 与 View 没有直接依赖, 所有的操作最终会生成一个 Drawable 交付给 ImageView 进行最后的渲染, 便于封装
  - 劣势: 由于对外提供的是链式调用的函数, 因此对于图片圆角等处理需要写额外的代码, 使用起来比 Fresco 稍有不便

从整体设计上来看
- Fresco
  - **数据加载模块使用了生产者消费者模型**, **生产者使用装饰器进行独立的增强**, 职责非常清晰, 是值得借鉴和学习的封装思路
- Glide
  - 数据模块使用大量回调, 阅读难度高, 职责有些不够清晰, 有提升空间

Fresco 图片框架中最核心的 Producer 的数据生产本篇中并没有深入探究, 后面可能会再开一个专题进行探究, 对比它与 Glide 的优劣