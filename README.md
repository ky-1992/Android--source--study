# Android--source--study

Android 百大框架源码研读仓库。每个框架一个目录（目录名即框架名），目录内包含三件套：

- `README.md`：源码解析 —— 一句话定位、源码快照（上游地址 + 固定 commit + 模块结构）、核心类表（带真实行号）、关键流程、设计要点、阅读路径、版本注记
- `class-diagram.html`：类图，浏览器直接打开，可交互（支持暗色/亮色、动效与导出）
- `flow-diagram.html`：流程图，同一套交互，描述该框架最值得看的一条运行时链路

所有图表都由 [Archify](https://github.com/tt-a1i/archify) 生成并通过四道校验（validate / deliver / check / browser-check），`sources` 里引用的文件与行号经 `--repo-root` 逐条核对；图表内嵌该框架上游仓库的 40 位 commit，可随时复核。

框架顺序沿用 [Android_100_TOP-Projects](https://github.com/ShaunSheep/Android_100_TOP-Projects) 的 2017 年百大排行榜，按能力域分组呈现。

## 目录结构

```
<FrameworkName>/
├── README.md                     # 源码解析
├── class-diagram.html            # 类图
├── class-diagram.archify.json    # 类图源描述（可再生成/二次编辑）
├── flow-diagram.html             # 流程图
└── flow-diagram.archify.json     # 流程图源描述
```

## 内容导航

- **网络、异步与事件总线**（7）：[okhttp](#okhttp)、[Retrofit](#retrofit)、[Android-async-http](#android-async-http)、[RxAndroid](#rxandroid)、[RxBinding](#rxbinding)、[EventBus](#eventbus)、[otto](#otto)
- **图片与多媒体**（11）：[glide](#glide)、[Android-Universal-Image-Loader](#android-universal-image-loader)、[picasso](#picasso)、[fresco](#fresco)、[lottie-android](#lottie-android)、[Luban](#luban)、[PhotoView](#photoview)、[CircleImageView](#circleimageview)、[uCrop](#ucrop)、[Calligraphy](#calligraphy)、[zxing](#zxing)
- **列表与布局**（8）：[BaseRecyclerViewAdapterHelper](#baserecyclerviewadapterhelper)、[UItimateRecyclerView](#uitimaterecyclerview)、[ListViewAnimations](#listviewanimations)、[Android-PullToRefresh](#android-pulltorefresh)、[AndroidSwipeLayout](#androidswipelayout)、[flexbox-layout](#flexbox-layout)、[AndroidAutoLayout](#androidautolayout)、[Android-ObservableScrollView](#android-observablescrollview)
- **导航、菜单与浮层**（9）：[MaterialDrawer](#materialdrawer)、[SlidingMenu](#slidingmenu)、[BottomBar](#bottombar)、[ViewPagerIndicator](#viewpagerindicator)、[FlycoTabLayout](#flycotablayout)、[AppIntro](#appintro)、[AndroidSlidingUpPanel](#androidslidinguppanel)、[material-dialogs](#material-dialogs)、[Android-Bootstrap](#android-bootstrap)
- **动画与图表**（4）：[MPAndroidChart](#mpandroidchart)、[Material-Animations](#material-animations)、[AndroidViewAnimations](#androidviewanimations)、[recyclerview-animators](#recyclerview-animators)
- **架构、依赖注入与代码生成**（7）：[Android-CleanArchitecture](#android-cleanarchitecture)、[EffectiveAndroidUI](#effectiveandroidui)、[dagger](#dagger)、[Butter-Knife](#butter-knife)、[androidannotations](#androidannotations)、[agera](#agera)、[RxJava-Android-Samples](#rxjava-android-samples)
- **数据与序列化**（3）：[realm-java](#realm-java)、[greenDAO](#greendao)、[fastjson](#fastjson)
- **性能、调试与测试**（4）：[leakcanary](#leakcanary)、[stetho](#stetho)、[logger](#logger)、[android-testing](#android-testing)
- **插件化与热修复**（2）：[tinker](#tinker)、[DroidPlugin](#droidplugin)

---

## 网络、异步与事件总线

> 把请求、线程调度与消息分发抽象成可组合的管道。

### Retrofit

> 类别：网络

Retrofit 把「HTTP API 接口」变成一个 Java 接口：你在接口方法上写注解，它在运行时用 **JDK 动态代理** 拦截调用，用 **ServiceMethod（注解解析模型）+ CallAdapter + Converter 双工厂** 把一次方法调用翻译成 OkHttp 请求，并把响应反序列化成声明式返回类型。

**核心类**：`Retrofit`、`Retrofit.Builder`、`ServiceMethod<T>`、`HttpServiceMethod`、`CallAdapter<R,T>`、`Converter<F,T>`

**图表**：类图 9 个类型 + 3 张说明卡；流程图 10 个节点 / 6 条泳道（Retrofit 请求执行流程）。

[源码解析](Retrofit/README.md) · [类图](Retrofit/class-diagram.html) · [流程图](Retrofit/flow-diagram.html)

### okhttp

> 类别：网络

OkHttp 是 Square 出品的 JVM / Android HTTP 客户端：用一条「可插拔的拦截器责任链」把重试、缓存、连接复用、HTTP/2 多路复用等能力串起来，再用 `ConnectionPool` 把 TCP/TLS 连接的生命周期延长到多次请求之间。它同时是 Retrofit、Glide、Coil 等一大批库的底层网络实现，读懂它就等于读懂了 Android 网络栈的地基。

**核心类**：`OkHttpClient`、`RealCall`、`Dispatcher`、`Interceptor / Interceptor.Chain`、`RealInterceptorChain`、`内置拦截器`

**图表**：类图 11 个类型 + 3 张说明卡；流程图 9 个节点 / 5 条泳道（OkHttp 一次请求的执行主流程）。

[源码解析](okhttp/README.md) · [类图](okhttp/class-diagram.html) · [流程图](okhttp/flow-diagram.html)

### EventBus

> 类别：事件消息

EventBus 是 greenrobot 出品的 Android / Java 发布-订阅事件总线：用 `@Subscribe` 注解声明回调方法，`post` 时按事件类型查询订阅表，再依据 `ThreadMode` 把回调切到目标线程执行。核心实现是纯 Java（不依赖 Android），总代码量约 2000 行，适合作为"注解 + 反射 + 线程调度"的入门读本。

**核心类**：`EventBus`、`EventBusBuilder`、`SubscriberMethodFinder`、`SubscriberMethod`、`Subscription`、`ThreadMode`

**图表**：类图 10 个类型 + 2 张说明卡；流程图 10 个节点 / 3 条泳道（EventBus 注册与事件投递主流程）。

[源码解析](EventBus/README.md) · [类图](EventBus/class-diagram.html) · [流程图](EventBus/flow-diagram.html)

### RxAndroid

> 类别：异步

RxAndroid 是 RxJava 3 的 Android 绑定层，只做一件事：提供一个运行在 Android 主线程（或任意 `Looper`）上的 `Scheduler`，并让 `Disposable` 的释放回调也回到主线程。主模块只有 4 个类，是学习 RxJava 调度器扩展点与 `Handler`/`Looper` 协作的最小样本。

**核心类**：`AndroidSchedulers`、`MainHolder`、`HandlerScheduler`、`HandlerWorker`、`ScheduledRunnable`、`MainThreadDisposable`

**图表**：类图 10 个类型 + 2 张说明卡；流程图 9 个节点 / 4 条泳道（RxAndroid 主线程调度流程）。

[源码解析](RxAndroid/README.md) · [类图](RxAndroid/class-diagram.html) · [流程图](RxAndroid/flow-diagram.html)

### Android-async-http

> 类别：网络

android-async-http 是 loopj 出品的 Android 异步 HTTP 客户端：用 `AsyncHttpClient` 统一封装 GET/POST/PUT 等方法，请求体经 `RequestParams` 变成 `HttpEntity`，任务丢进 `ExecutorService`，响应再由 `AsyncHttpResponseHandler` 通过 `Looper` 切回主线程回调。它建立在 HttpClient 4.x 的 Android 移植版 `cz.msebera.android.httpclient` 之上，是 2013 年前后 Android 网络层的主流写法。

**核心类**：`AsyncHttpClient`、`AsyncHttpRequest`、`RequestParams`、`RetryHandler`、`RequestHandle`、`ResponseHandlerInterface`

**图表**：类图 10 个类型 + 2 张说明卡；流程图 9 个节点 / 5 条泳道（android-async-http 请求执行主流程）。

[源码解析](Android-async-http/README.md) · [类图](Android-async-http/class-diagram.html) · [流程图](Android-async-http/flow-diagram.html)

### RxBinding

> 类别：响应式

RxBinding 把 Android 控件的事件包装成 RxJava 3 的 `Observable`：每个绑定都是一个 Kotlin 扩展函数（如 `View.clicks()`、`TextView.textChanges()`），函数内部返回一个 `private` 实现类，订阅时先校验主线程、再注册系统监听器，解绑时反注册。它解决的问题是把"匿名内部类 + 手动注销"换成可组合的操作符链，同时统一主线程与泄漏语义。

**核心类**：`InitialValueObservable`、`ViewClickObservable`、`checkMainThread`、`TextViewTextChangesObservable`、`TextViewTextChangeEventObservable`、`AdapterDataChangeObservable`

**图表**：类图 9 个类型 + 2 张说明卡；流程图 9 个节点 / 3 条泳道（RxBinding 点击绑定订阅流程）。

[源码解析](RxBinding/README.md) · [类图](RxBinding/class-diagram.html) · [流程图](RxBinding/flow-diagram.html)

### otto

> 类别：响应式

Otto 是 Square 基于 Guava EventBus 改造的进程内事件总线：方法用 `@Subscribe` / `@Produce` 注解标记，`Bus` 负责注册、按事件类型匹配并反射调用。

**核心类**：`Bus`、`HandlerFinder`、`AnnotatedHandlerFinder`、`EventHandler`、`EventProducer`、`ThreadEnforcer`

**图表**：类图 9 个类型 + 2 张说明卡；流程图 9 个节点 / 4 条泳道（Otto 注册与事件分发主流程）。

[源码解析](otto/README.md) · [类图](otto/class-diagram.html) · [流程图](otto/flow-diagram.html)

---

## 图片与多媒体

> 加载、解码、缓存、变换，以及图像识别。

### glide

> 类别：图片

Glide 是 Android 上的图片加载框架：以 `Glide` 单例为入口，用 `RequestManager` 把加载请求绑定到 Activity/Fragment 生命周期，`RequestBuilder` 负责链式配置并组装请求，`Engine` + `DecodeJob` 完成「活动资源 → 内存缓存 → 磁盘缓存 → 数据源」的逐级取数与解码。

**核心类**：`Glide`、`RequestManagerRetriever`、`RequestManager`、`RequestBuilder`、`SingleRequest`、`Engine`

**图表**：类图 11 个类型 + 3 张说明卡；流程图 10 个节点 / 5 条泳道（Glide 加载主流程：从 into 到 DecodeJob 取数）。

[源码解析](glide/README.md) · [类图](glide/class-diagram.html) · [流程图](glide/flow-diagram.html)

### Android-Universal-Image-Loader

> 类别：图片

UIL 是 2011–2015 年流行的 Android 图片加载库：以 `ImageLoader` 单例为入口，`displayImage()` 先查内存缓存，未命中就把 `LoadAndDisplayImageTask` 投入线程池；任务内按磁盘缓存 → 网络下载 → 解码的顺序取图，最后用 `DisplayBitmapTask` 回主线程写入 `ImageAware`。全部为 Java，主库约 50 个类。

**核心类**：`ImageLoader`、`ImageLoaderConfiguration`、`ImageLoaderEngine`、`LoadAndDisplayImageTask`、`DisplayBitmapTask`、`ImageAware`

**图表**：类图 10 个类型 + 2 张说明卡；流程图 9 个节点 / 4 条泳道（Universal Image Loader 一次加载的调度流程）。

[源码解析](Android-Universal-Image-Loader/README.md) · [类图](Android-Universal-Image-Loader/class-diagram.html) · [流程图](Android-Universal-Image-Loader/flow-diagram.html)

### zxing

> 类别：条码扫描

ZXing（"Zebra Crossing"）是 Java 实现的条码/二维码编解码库：解码侧把图片转成**亮度源 → 二值化位矩阵 → 定位/采样 → 纠错解码 → Result**；编码侧用 `Writer` 把内容渲染成 `BitMatrix`。Android 上常见的扫码 App 都是它的封装。

**核心类**：`MultiFormatReader`、`QRCodeReader`、`Detector`、`Decoder`、`BitMatrix`、`BinaryBitmap`

**图表**：类图 11 个类型 + 3 张说明卡；流程图 10 个节点 / 5 条泳道（ZXing 单张条码图解码主流程）。

[源码解析](zxing/README.md) · [类图](zxing/class-diagram.html) · [流程图](zxing/flow-diagram.html)

### picasso

> 类别：图片

Picasso 是 Square 出品的图片加载库：`Picasso.with(context).load(url).into(imageView)` 一行完成加载。核心是一条**单线程 Dispatcher 消息循环 + 多线程 BitmapHunter 执行 + 内存 LRU 缓存**的流水线，Action 负责把结果回写到 Target，并自动处理 View 复用（tag 机制取消旧请求）。

**核心类**：`Picasso`、`RequestCreator`、`Request`、`Dispatcher`、`BaseDispatcher`、`HandlerDispatcher`

**图表**：类图 12 个类型 + 3 张说明卡；流程图 9 个节点 / 5 条泳道（Picasso 一次图片加载的调度流程）。

[源码解析](picasso/README.md) · [类图](picasso/class-diagram.html) · [流程图](picasso/flow-diagram.html)

### lottie-android

> 类别：动画

Lottie 把 Adobe After Effects 导出的 **Bodymovin JSON** 解析成一棵图层树（`LottieComposition`），再由 `LottieDrawable` 在 Android Canvas 上逐帧绘制；`LottieValueAnimator` 只负责推进时间轴，动画属性由各 `BaseLayer` 按关键帧插值——不需要视频、不需要预渲染帧序列。

**核心类**：`LottieCompositionFactory`、`LottieComposition`、`LottieCompositionMoshiParser`、`LottieResult`、`LottieAnimationView`、`LottieDrawable`

**图表**：类图 11 个类型 + 3 张说明卡；流程图 8 个节点 / 4 条泳道（Lottie 动画加载与渲染流程）。

[源码解析](lottie-android/README.md) · [类图](lottie-android/class-diagram.html) · [流程图](lottie-android/flow-diagram.html)

### fresco

> 类别：图片

Fresco 是 Facebook 开源的 Android 图片库，核心是**生产者-消费者（Producer/Consumer）流水线**：一次图片请求被 `ProducerSequenceFactory` 编排成一条 Producer 链（缓存查询 → 磁盘 → 网络 → 解码），数据以 `CloseableImage` 沿链流动，`Drawee` 视图层负责占位图、渐进式展示与释放。

**核心类**：`ImagePipeline`、`ProducerSequenceFactory`、`Producer`、`Consumer`、`ProducerContext`、`BitmapMemoryCacheGetProducer`

**图表**：类图 11 个类型 + 2 张说明卡；流程图 9 个节点 / 5 条泳道（Fresco 解码取图主流程）。

[源码解析](fresco/README.md) · [类图](fresco/class-diagram.html) · [流程图](fresco/flow-diagram.html)

### PhotoView

> 类别：图片

PhotoView 是建立在 `AppCompatImageView` 之上的可缩放图片控件：对外只暴露 `PhotoView`，真正的手势识别、矩阵计算与边界回弹全部收拢在 `PhotoViewAttacher` 里，二者通过一个字段解耦视图层与交互逻辑。

**核心类**：`PhotoView`、`PhotoViewAttacher`、`CustomGestureDetector`、`OnGestureListener`、`Util`、`OnMatrixChangedListener / OnScaleChangedListener / OnViewDragListener`

**图表**：类图 8 个类型 + 2 张说明卡；流程图 11 个节点 / 5 条泳道（PhotoView 图片加载与手势缩放流程）。

[源码解析](PhotoView/README.md) · [类图](PhotoView/class-diagram.html) · [流程图](PhotoView/flow-diagram.html)

### CircleImageView

> 类别：图片

CircleImageView 是只有一个类的 Android 圆形头像控件：继承 `ImageView`，把当前 `Drawable` 转成 `Bitmap`，再用 `BitmapShader` 把这张位图铺进 `drawCircle` 的圆形区域，另配一支 `STROKE` 画笔描边框。因为不用 `clipPath`、不复制位图、不叠加两次绘制，它是学习"自定义 View 绘制管线"最省事的读本。

**核心类**：`CircleImageView`、`CircleImageView.OutlineProvider`、`R.styleable.CircleImageView`

**图表**：类图 9 个类型 + 2 张说明卡；流程图 9 个节点 / 4 条泳道（CircleImageView 圆形绘制主流程）。

[源码解析](CircleImageView/README.md) · [类图](CircleImageView/class-diagram.html) · [流程图](CircleImageView/flow-diagram.html)

### Calligraphy

> 类别：字体

Calligraphy 是一个「零侵入换字体」的 Android 库：它不要求你改用自定义 TextView，而是替换 `LayoutInflater`，在每次 View 创建完成的回调里，把 assets 中的字体设置到所有 `TextView` 上。

**核心类**：`CalligraphyConfig`、`CalligraphyConfig.Builder`、`CalligraphyContextWrapper`、`CalligraphyLayoutInflater`、`CalligraphyFactory`、`CalligraphyUtils`

**图表**：类图 10 个类型 + 2 张说明卡；流程图 10 个节点 / 5 条泳道（Calligraphy 字体注入主流程）。

[源码解析](Calligraphy/README.md) · [类图](Calligraphy/class-diagram.html) · [流程图](Calligraphy/flow-diagram.html)

### uCrop

> 类别：UI

uCrop 是一个「调用方只拼参数、编辑页负责手势与裁剪」的图片裁剪库：`UCrop` 把源图、目标图和各种开关写进 `Intent`，`UCropActivity` 用 `UCropView` 展示图片与裁剪框，最后交给 `BitmapCropTask` 在后台完成原生裁剪。

**核心类**：`UCrop`、`UCrop.Options`、`UCropActivity`、`UCropView`、`TransformImageView`、`CropImageView`

**图表**：类图 10 个类型 + 3 张说明卡；流程图 9 个节点 / 4 条泳道（uCrop 裁剪主流程）。

[源码解析](uCrop/README.md) · [类图](uCrop/class-diagram.html) · [流程图](uCrop/flow-diagram.html)

### Luban

> 类别：图片

Luban（鲁班）是通过逆向微信朋友圈压缩效果推算出的 Android 图片压缩库。当前 master 是 Kotlin 重写版 2.0.3：协程负责并发与线程切换，`CompressionCalculator` 先算目标尺寸与预估体积，`ImageLoader` 按 `inSampleSize` 采样解码，`JpegCompressor` 调用 libturbojpeg 的 JNI 编码，长图再用质量二分逼近目标大小。库模块 Kotlin 代码约一千一百行（另有 85 行 C++ 封装），是"先预算、再解码、后编码"这类图片处理流水线的典型样本。

**核心类**：`Luban`、`CompressionCalculator`、`CompressionTarget`、`ImageLoader`、`Compressor / JpegCompressor`、`TurboJpegNative`

**图表**：类图 9 个类型 + 2 张说明卡；流程图 10 个节点 / 4 条泳道（Luban 单图压缩主流程）。

[源码解析](Luban/README.md) · [类图](Luban/class-diagram.html) · [流程图](Luban/flow-diagram.html)

---

## 列表与布局

> 滑动容器、适配器增强与测量算法。

### Android-PullToRefresh

> 类别：刷新

给 `ListView`、`GridView`、`WebView` 等可滚动控件套一层"下拉 / 上拉刷新"容器的经典库：外层 `LinearLayout` 接管触摸事件，用一套六态状态机把手指拖动距离翻译成加载区动画与刷新回调，业务方只需要注册一个 `OnRefreshListener` 并在数据加载完后调用 `onRefreshComplete`。

**核心类**：`PullToRefreshBase`、`PullToRefreshAdapterViewBase`、`PullToRefreshListView`、`LoadingLayout`、`RotateLoadingLayout`、`FlipLoadingLayout`

**图表**：类图 10 个类型 + 3 张说明卡；流程图 10 个节点 / 4 条泳道（PullToRefresh 下拉手势到刷新的主流程）。

[源码解析](Android-PullToRefresh/README.md) · [类图](Android-PullToRefresh/class-diagram.html) · [流程图](Android-PullToRefresh/flow-diagram.html)

### flexbox-layout

> 类别：UI

把 CSS Flexbox 布局模型搬进 Android：`FlexboxLayout` 是 `ViewGroup` 版容器，`FlexboxLayoutManager` 是 RecyclerView 版容器，两者共用同一套测量、换行与对齐算法。开发者用 `flexDirection`、`flexWrap`、`justifyContent`、`alignItems` 等属性描述意图，具体每行几个子项、如何伸缩，全部由库在测量阶段算出来。

**核心类**：`FlexboxLayout`、`FlexboxLayoutManager`、`FlexContainer`、`FlexItem`、`FlexboxHelper`、`FlexLine`

**图表**：类图 8 个类型 + 3 张说明卡；流程图 9 个节点 / 4 条泳道（FlexboxLayout 测量与换行主流程）。

[源码解析](flexbox-layout/README.md) · [类图](flexbox-layout/class-diagram.html) · [流程图](flexbox-layout/flow-diagram.html)

### AndroidSwipeLayout

> 类别：UI

AndroidSwipeLayout 是一个「可四向滑出底部视图」的 `FrameLayout` 容器：它把触摸事件交给 `ViewDragHelper`，按拖出距离决定展开还是回弹，并向业务方暴露开合回调；列表场景则由 `SwipeItemMangerImpl` 统一记录哪些条目处于打开状态。

**核心类**：`SwipeLayout`、`SwipeLayout.Status`、`SwipeLayout.DragEdge`、`SwipeLayout.ShowMode`、`SwipeLayout.SwipeListener`、`SwipeLayout.SwipeDetector`

**图表**：类图 10 个类型 + 2 张说明卡；流程图 9 个节点 / 4 条泳道（AndroidSwipeLayout 一次滑动手势的主流程）。

[源码解析](AndroidSwipeLayout/README.md) · [类图](AndroidSwipeLayout/class-diagram.html) · [流程图](AndroidSwipeLayout/flow-diagram.html)

### BaseRecyclerViewAdapterHelper

> 类别：UI

BRVAH（BaseRecyclerViewAdapterHelper）是 CymChad 维护的 RecyclerView Adapter 增强库：在 `RecyclerView.Adapter` 外包一层模板基类 `BaseQuickAdapter`，把数据更新（`AsyncListDiffer`）、多布局、空状态、入场动画、上下加载更多、拖拽侧滑拆成可组合模块。4.x 用 Kotlin 重写，核心模块 `library` 只有 30 个源文件，适合当作"RecyclerView 组件化封装"的样板阅读。

**核心类**：`BaseQuickAdapter`、`QuickViewHolder`、`QuickAdapterHelper`、`LoadStateAdapter`、`TrailingLoadStateAdapter`、`BaseMultiItemAdapter`

**图表**：类图 10 个类型 + 3 张说明卡；流程图 10 个节点 / 4 条泳道（BRVAH 列表渲染与加载更多主流程）。

[源码解析](BaseRecyclerViewAdapterHelper/README.md) · [类图](BaseRecyclerViewAdapterHelper/class-diagram.html) · [流程图](BaseRecyclerViewAdapterHelper/flow-diagram.html)

### Android-ObservableScrollView

> 类别：UI

Android-ObservableScrollView 是 ksoichiro 的滚动联动基础库：给 `ScrollView`、`ListView`、`GridView`、`RecyclerView`、`WebView` 套一层可观察壳，把连续滚动位移换算成 `scrollY`，通过 `ObservableScrollViewCallbacks` 交给界面做工具栏收起、视差图、FAB 位移等效果。它只负责上报位置，动画与业务判断全部留给示例层。

**核心类**：`Scrollable`、`ObservableScrollViewCallbacks`、`ObservableScrollView`、`ObservableListView`、`ObservableRecyclerView`、`ScrollState`

**图表**：类图 8 个类型 + 2 张说明卡；流程图 9 个节点 / 4 条泳道（ObservableScrollView 滚动事件处理流程）。

[源码解析](Android-ObservableScrollView/README.md) · [类图](Android-ObservableScrollView/class-diagram.html) · [流程图](Android-ObservableScrollView/flow-diagram.html)

### ListViewAnimations

> 类别：动画

ListViewAnimations 是 2014 年前后的 ListView 增强库：用装饰器模式逐层包装 `BaseAdapter`，在入场动画、侧滑删除、拖拽换位之间自由组合；`DynamicListView` 负责把一次触摸分给拖拽或侧滑处理，保证同一时刻只有一方接管。核心思想是“数据装饰 + 手势分层”。

**核心类**：`ListViewWrapper`、`BaseAdapterDecorator`、`AnimationAdapter`、`ViewAnimator`、`SwipeTouchListener`、`SwipeDismissTouchListener`

**图表**：类图 9 个类型 + 3 张说明卡；流程图 9 个节点 / 4 条泳道（ListViewAnimations 手势分发与条目操作流程）。

[源码解析](ListViewAnimations/README.md) · [类图](ListViewAnimations/class-diagram.html) · [流程图](ListViewAnimations/flow-diagram.html)

### UItimateRecyclerView

> 类别：图片

在 `FrameLayout` 里包一层 `RecyclerView` + 下拉刷新，用适配器的附加视图位补齐 Header、Footer、空视图、加载更多与侧滑删除，让列表开箱即用，业务侧只关心数据与条目布局。

**核心类**：`UltimateRecyclerView`、`UltimateViewAdapter`、`NormalUltimateViewAdapter`、`SwipeableUltimateViewAdapter`、`VerticalSwipeRefreshLayout`、`RecyclerViewPositionHelper`

**图表**：类图 10 个类型 + 2 张说明卡；流程图 9 个节点 / 4 条泳道（UltimateRecyclerView 下拉刷新与加载更多流程）。

[源码解析](UItimateRecyclerView/README.md) · [类图](UItimateRecyclerView/class-diagram.html) · [流程图](UItimateRecyclerView/flow-diagram.html)

### AndroidAutoLayout

> 类别：适配

AndroidAutoLayout 让开发者按设计稿的 px 直接写布局，运行期把每个尺寸、间距、字号按「屏幕尺寸 / 设计稿尺寸」等比缩放：`AutoLayoutActivity` 把系统三个布局类换成 Auto 版容器，容器在测量前统一改写子 View 的属性。

**核心类**：`AutoLayoutActivity`、`AutoFrameLayout / AutoLinearLayout / AutoRelativeLayout`、`AutoLayoutHelper`、`AutoLayoutInfo`、`AutoAttr`、`WidthAttr / TextSizeAttr`

**图表**：类图 10 个类型 + 2 张说明卡；流程图 9 个节点 / 4 条泳道（AndroidAutoLayout 自动缩放流程）。

[源码解析](AndroidAutoLayout/README.md) · [类图](AndroidAutoLayout/class-diagram.html) · [流程图](AndroidAutoLayout/flow-diagram.html)

---

## 导航、菜单与浮层

> 抽屉、底栏、引导页、面板与对话框。

### SlidingMenu

> 类别：菜单

SlidingMenu 是"上古神器"级的侧滑菜单库：`SlidingMenu` 本身是一个 `RelativeLayout`，内部由 **CustomViewAbove（内容层）+ CustomViewBehind（菜单层）** 两层视图组成，拖动时内容层平移、菜单层按比例视差跟随，通过 `attachToActivity` 挂到 Activity 上，并提供 `SLIDING_WINDOW` / `SLIDING_CONTENT` 两种滑动模式。

**核心类**：`SlidingMenu`、`CustomViewAbove`、`CustomViewBehind`、`SlidingActivity`、`SlidingActivityHelper`、`MenuInterface`

**图表**：类图 7 个类型 + 2 张说明卡；流程图 7 个节点 / 4 条泳道（SlidingMenu 绑定与拖拽流程）。

[源码解析](SlidingMenu/README.md) · [类图](SlidingMenu/class-diagram.html) · [流程图](SlidingMenu/flow-diagram.html)

### material-dialogs

> 类别：UI

Material Dialogs 是一套遵循 Material 规范的 Android 对话框库。本次读到的 `main` 分支是 **3.x 的 Kotlin 重写版**：不再有 Java 时代的 `MaterialDialog.Builder` / `DialogBase` / `DialogWrapper`，`MaterialDialog` 自身就是配置入口，布局、主题与窗口行为被拆到 `DialogBehavior` 与三段式 `DialogLayout` 中。

**核心类**：`MaterialDialog`、`DialogBehavior`、`ModalDialog`、`DialogLayout`、`DialogTitleLayout`、`DialogContentLayout`

**图表**：类图 10 个类型 + 2 张说明卡；流程图 11 个节点 / 5 条泳道（Material Dialogs 创建、显示与回调流程）。

[源码解析](material-dialogs/README.md) · [类图](material-dialogs/class-diagram.html) · [流程图](material-dialogs/flow-diagram.html)

### ViewPagerIndicator

> 类别：UI

ViewPagerIndicator 是 Jake Wharton 写的 ViewPager 页码指示器组件集：一个 `PageIndicator` 接口约定绑定方式，六种实现（圆点、线、下划线、标题、标签、图标）负责"跟着 ViewPager 滚"的视觉反馈。整个库不到 3000 行 Java，是 Android 自定义 View 与 Canvas 绘制、`onMeasure` / `onDraw` / 触摸分发的经典教材。

**核心类**：`PageIndicator`、`CirclePageIndicator`、`LinePageIndicator`、`UnderlinePageIndicator`、`TitlePageIndicator`、`TabPageIndicator`

**图表**：类图 10 个类型 + 2 张说明卡；流程图 9 个节点 / 4 条泳道（ViewPagerIndicator 绑定与滚动回绘流程）。

[源码解析](ViewPagerIndicator/README.md) · [类图](ViewPagerIndicator/class-diagram.html) · [流程图](ViewPagerIndicator/flow-diagram.html)

### MaterialDrawer

> 类别：菜单

MaterialDrawer 是 mikepenz 出品的 Android 侧滑抽屉库：把抽屉拆成「Builder 配置 → Drawer 运行时门面 → IDrawerItem 条目」三段，底层直接复用 `DrawerLayout` 与 FastAdapter/RecyclerView，不接管 Activity 的布局与窗口标志。调用方只需链式写配置，剩下的充气、适配器、点击与状态恢复都交给库内部完成。

**核心类**：`DrawerBuilder`、`Drawer`、`AccountHeaderBuilder`、`AccountHeader`、`DrawerUtils`、`IDrawerItem`

**图表**：类图 10 个类型 + 2 张说明卡；流程图 9 个节点 / 3 条泳道（MaterialDrawer 构建与点击主流程）。

[源码解析](MaterialDrawer/README.md) · [类图](MaterialDrawer/class-diagram.html) · [流程图](MaterialDrawer/flow-diagram.html)

### BottomBar

> 类别：菜单

BottomBar 是 roughike 的 Android 底部导航自定义 View：用一个 `LinearLayout` 承载若干 `BottomBarTab`，tab 由 XML 声明、`TabParser` 解析，支持 fixed / shifting / tablet 三种排布，以及选中动画、角标、滚动隐藏（shy）等衍生能力。库代码约 2900 行，是"自定义 ViewGroup + 属性动画"的典型读本。

**核心类**：`BottomBar`、`BottomBarTab`、`TabParser`、`BottomBarTab.Config`、`BottomBarTab.Type`、`TabSelectionInterceptor`

**图表**：类图 10 个类型 + 2 张说明卡；流程图 10 个节点 / 4 条泳道（BottomBar 装配与选中切换流程）。

[源码解析](BottomBar/README.md) · [类图](BottomBar/class-diagram.html) · [流程图](BottomBar/flow-diagram.html)

### AndroidSlidingUpPanel

> 类别：UI

AndroidSlidingUpPanel 提供一个「从底部（或顶部）拉出面板」的容器控件 `SlidingUpPanelLayout`：布局里放主内容和面板两个子 View，用户拖动面板时库自己完成手势接管、位置换算、状态吸附与阴影遮罩。

**核心类**：`SlidingUpPanelLayout`、`PanelState`、`PanelSlideListener`、`DragHelperCallback`、`LayoutParams`、`ViewDragHelper`

**图表**：类图 9 个类型 + 2 张说明卡；流程图 10 个节点 / 5 条泳道（AndroidSlidingUpPanel 手势拖拽主流程）。

[源码解析](AndroidSlidingUpPanel/README.md) · [类图](AndroidSlidingUpPanel/class-diagram.html) · [流程图](AndroidSlidingUpPanel/flow-diagram.html)

### AppIntro

> 类别：UI

AppIntro 是一个用 Activity + Fragment + ViewPager2 拼出来的「首次启动引导页」库：宿主继承 `AppIntro`（或 `AppIntro2`）并在 `onCreate` 里 `addSlide`，翻页、指示器、跳过/完成按钮、逐屏权限申请、「这一屏必须满足条件才允许往下走」的策略校验，全部由基类 `AppIntroBase` 统一接管。

**核心类**：`AppIntroBase`、`AppIntro`、`AppIntro2`、`AppIntroViewPagerController`、`AppIntroViewPagerListener`、`SlidePolicy`

**图表**：类图 9 个类型 + 3 张说明卡；流程图 9 个节点 / 4 条泳道（AppIntro 装配与逐屏推进流程）。

[源码解析](AppIntro/README.md) · [类图](AppIntro/class-diagram.html) · [流程图](AppIntro/flow-diagram.html)

### Android-Bootstrap

用运行时生成的 Drawable 把 Bootstrap 的按钮、输入框、标签样式搬到 Android 自定义 View 上，可配置的主轴只有「品牌色」与「尺寸倍数」两组，其余视觉细节由代码按状态推导。

**核心类**：`BootstrapButton`、`AwesomeTextView`、`BootstrapEditText`、`BootstrapText`、`BootstrapDrawableFactory`、`BootstrapBrand / DefaultBootstrapBrand`

**图表**：类图 9 个类型 + 2 张说明卡；流程图 10 个节点 / 4 条泳道（Android-Bootstrap 属性解析与背景绘制流程）。

[源码解析](Android-Bootstrap/README.md) · [类图](Android-Bootstrap/class-diagram.html) · [流程图](Android-Bootstrap/flow-diagram.html)

### FlycoTabLayout

> 类别：UI

一个只有 8 个 Java 文件的 Tab 控件库：`CommonTabLayout` 不依赖 ViewPager、`SlidingTabLayout` 跟随 ViewPager 滑动、`SegmentTabLayout` 是分段控件，三者共用同一套 `tl_*` 属性与"测量矩形 → 自绘指示器"的实现套路。

**核心类**：`CommonTabLayout`、`SlidingTabLayout`、`SegmentTabLayout`、`CustomTabEntity`、`OnTabSelectListener`、`FragmentChangeManager`

**图表**：类图 9 个类型 + 2 张说明卡；流程图 8 个节点 / 4 条泳道（FlycoTabLayout 初始化与指示器绘制主流程）。

[源码解析](FlycoTabLayout/README.md) · [类图](FlycoTabLayout/class-diagram.html) · [流程图](FlycoTabLayout/flow-diagram.html)

---

## 动画与图表

> 属性动画、转场、列表动画与数据可视化。

### MPAndroidChart

> 类别：图表

MPAndroidChart 是 Android 上使用最广的图表库，v4.0 用 Kotlin 重写后仍保持“数据对象 + 视图 + 渲染器”三层结构：`Chart` 持有数据与视口，`BarLineChartBase` 负责坐标轴和边距，`BarChartRenderer` 通过 `Transformer` 把数值坐标画成像素。

**核心类**：`Chart<T>`、`BarLineChartBase<T>`、`BarChart`、`ChartData<T>`、`IDataSet<T> / BaseDataSet<T>`、`DataRenderer / BarChartRenderer`

**图表**：类图 10 个类型 + 2 张说明卡；流程图 9 个节点 / 4 条泳道（MPAndroidChart 数据到绘制的渲染主流程）。

[源码解析](MPAndroidChart/README.md) · [类图](MPAndroidChart/class-diagram.html) · [流程图](MPAndroidChart/flow-diagram.html)

### Material-Animations

> 类别：动画

Material-Animations 是 Google 官方 Material Design 动画示例工程（教学型项目），用 3 个页面演示 **Activity 转场（Transition）**、**共享元素（Shared Element）** 与 **Fragment 转场** 的正确用法：`ActivityOptions.makeSceneTransitionAnimation` + `Window.setEnterTransition/setExitTransition` + `SharedElementCallback`。

**核心类**：`MainActivity`、`SamplesRecyclerAdapter`、`SamplesViewHolder`、`Sample`、`BaseDetailActivity`、`TransitionHelper`

**图表**：类图 12 个类型 + 2 张说明卡；流程图 10 个节点 / 5 条泳道（Material-Animations 共享元素跳转流程）。

[源码解析](Material-Animations/README.md) · [类图](Material-Animations/class-diagram.html) · [流程图](Material-Animations/flow-diagram.html)

### AndroidViewAnimations

> 类别：动画

AndroidViewAnimations 是 daimajia 的 View 动画预设库：把 63 种常见效果（抖动、弹入、翻转、滑动、缩放、铰链等）各写成一个 `BaseViewAnimator` 子类，再用 `YoYo.with(Techniques.xxx).duration(...).playOn(view)` 的链式 API 播放。核心库 66 个 Java 文件、约 1700 行有效代码（每文件另有 20 余行许可证头），是"如何用 ObjectAnimator + AnimatorSet 封装动画预设"的典型读本。

**核心类**：`YoYo`、`YoYo.AnimationComposer`、`YoYo.YoYoString`、`Techniques`、`BaseViewAnimator`、`BounceAnimator`

**图表**：类图 10 个类型 + 2 张说明卡；流程图 8 个节点 / 4 条泳道（AndroidViewAnimations 从链式调用到动画播放）。

[源码解析](AndroidViewAnimations/README.md) · [类图](AndroidViewAnimations/class-diagram.html) · [流程图](AndroidViewAnimations/flow-diagram.html)

### recyclerview-animators

> 类别：动画

recyclerview-animators 是 wasabeef 维护的一套 RecyclerView 增删动画集合：核心是继承 `SimpleItemAnimator` 的抽象类 `BaseItemAnimator`，把「入队 → 分批延迟调度 → 调子类实现属性动画 → 派发完成」这条流水线写死，21 个具体动画类（Fade / Slide / Scale / Flip / Overshoot / Landing 系列）只需要覆写三个钩子；另有一条 Adapter 侧的 `AnimationAdapter` 路线负责入场动画。

**核心类**：`BaseItemAnimator`、`FadeInAnimator`、`SlideInLeftAnimator`、`ScaleInAnimator`、`AnimateViewHolder`、`ViewHelper`

**图表**：类图 8 个类型 + 3 张说明卡；流程图 9 个节点 / 4 条泳道（recyclerview-animators 单次 item 动画流程）。

[源码解析](recyclerview-animators/README.md) · [类图](recyclerview-animators/class-diagram.html) · [流程图](recyclerview-animators/flow-diagram.html)

---

## 架构、依赖注入与代码生成

> 分层、注入、注解处理与响应式数据流。

### Butter-Knife

> 类别：代码模板

Butter Knife 用**编译期注解处理**替代 `findViewById` 和匿名点击监听：`@BindView` / `@OnClick` 等注解在编译时被 `ButterKnifeProcessor` 扫描，为每个 Activity/Fragment 生成一个 `Xxx_ViewBinding` 类，运行时 `ButterKnife.bind()` 通过反射实例化这个生成类完成绑定，`unbind()` 时统一解绑。

**核心类**：`ButterKnife`、`ButterKnifeProcessor`、`BindingSet`、`ViewBinding`、`FieldViewBinding / MethodViewBinding`、`MemberViewBinding`

**图表**：类图 11 个类型 + 3 张说明卡；流程图 9 个节点 / 5 条泳道（Butter Knife 编译期生成与运行期绑定流程）。

[源码解析](Butter-Knife/README.md) · [类图](Butter-Knife/class-diagram.html) · [流程图](Butter-Knife/flow-diagram.html)

### androidannotations

> 类别：注解

AndroidAnnotations 是"注解驱动开发"的代表框架：给 Activity/Fragment/Bean 写上 `@EActivity`、`@ViewById`、`@Click` 等注解，`AndroidAnnotationProcessor` 在**编译期**生成 `Foo_` 子类，把 findViewById、事件注册、Intent 注入、状态恢复等样板代码全部生成出来；运行时只用 `Foo_.intent(context)` 启动，几乎没有反射开销。

**核心类**：`AndroidAnnotationProcessor`、`AndroidAnnotationsEnvironment`、`InternalAndroidAnnotationsEnvironment`、`ModelExtractor`、`AnnotationElementsHolder`、`AnnotationHandlers`

**图表**：类图 8 个类型 + 2 张说明卡；流程图 7 个节点 / 4 条泳道（AndroidAnnotations 编译期处理流程）。

[源码解析](androidannotations/README.md) · [类图](androidannotations/class-diagram.html) · [流程图](androidannotations/flow-diagram.html)

### Android-CleanArchitecture

> 类别：架构

这是 Fernando Cejas 的 Clean Architecture 示范工程：把 Android 应用切成 `presentation`（MVP + 界面）、`domain`（纯 Java 业务规则）、`data`（网络/磁盘实现）三层，依赖方向全部指向 domain——presentation 依赖 domain 的 `UseCase` 与 `Repository` 抽象，data 实现这些抽象，用 RxJava 2 的 `Observable` 串起异步数据流。

**核心类**：`UseCase<T, Params>`、`GetUserList`、`UserRepository`、`UserDataRepository`、`UserDataStoreFactory`、`RestApiImpl`

**图表**：类图 10 个类型 + 2 张说明卡；流程图 9 个节点 / 4 条泳道（Android-CleanArchitecture 用户列表加载流程）。

[源码解析](Android-CleanArchitecture/README.md) · [类图](Android-CleanArchitecture/class-diagram.html) · [流程图](Android-CleanArchitecture/flow-diagram.html)

### agera

> 类别：异步

Agera 是 Google 的 Android 响应式编程库：把数据源抽象成 `Repository`（既能被订阅、又能被 `get()` 取值），用 `Updatable` 表达"数据变了"，并把"取数 → 变换 → 校验 → 写值"的流程写成可编译的链式声明。核心模块是纯 Java 加少量 `android.os.Handler` 依赖，37 个文件约 4900 行。

**核心类**：`Repository`、`Observable / Updatable`、`BaseObservable`、`WorkerHandler`、`CompiledRepository`、`RepositoryCompiler`

**图表**：类图 10 个类型 + 2 张说明卡；流程图 10 个节点 / 4 条泳道（Agera 编译仓库的激活与数据流执行）。

[源码解析](agera/README.md) · [类图](agera/class-diagram.html) · [流程图](agera/flow-diagram.html)

### dagger

> 类别：依赖注入

Dagger 是 Google 的编译期依赖注入框架：用 `@Component` 声明注入入口、`@Module` 声明提供者、`@Inject` 标注构造器与字段，注解处理器在编译期把三者解析成一张绑定依赖图，校验通过后生成 `DaggerXxx` 实现类，运行时不再靠反射查找依赖，只调用生成的工厂类。

**核心类**：`ComponentProcessor`、`KspComponentProcessor`、`DelegateComponentProcessor`、`ComponentProcessingStep`、`ComponentDescriptor`、`BindingGraphFactory`

**图表**：类图 10 个类型 + 2 张说明卡；流程图 9 个节点 / 4 条泳道（Dagger 注解处理到 DaggerXxx 生成）。

[源码解析](dagger/README.md) · [类图](dagger/class-diagram.html) · [流程图](dagger/flow-diagram.html)

### RxJava-Android-Samples

> 类别：用例

RxJava-Android-Samples 是 kaushikgopal 维护的 RxJava 2 示例工程：一个 `app` 模块里放二十来个 Fragment，每个 Fragment 演示一种操作符组合——去抖搜索、轮询、网络请求与组合、表单校验、并发调度等。它不提供库，价值在于"一条链一个示例"的可读组织，可以作为 RxJava 操作符与 Android 生命周期、线程调度结合方式的学习样本。

**核心类**：`MyApp`、`MainActivity`、`RxBus`、`BaseFragment`、`MainFragment`、`DebounceSearchEmitterFragment`

**图表**：类图 10 个类型 + 2 张说明卡；流程图 9 个节点 / 4 条泳道（Debounce 搜索示例的输入到日志流程）。

[源码解析](RxJava-Android-Samples/README.md) · [类图](RxJava-Android-Samples/class-diagram.html) · [流程图](RxJava-Android-Samples/flow-diagram.html)

### EffectiveAndroidUI

> 类别：性能

Pedro Vicente Gómez Sánchez 的示例工程，用一个"读取并展示美剧目录"的场景演示 Android 表现层的两套写法：目录页走 MVP（Presenter 持有内部 View 接口），详情页走 MVVM（ViewModel 用 Listener 推送状态、用 ActionCommand 封装点击）；领域层是 Interactor + Executor 的用例模型，依赖注入用 Dagger 1 的 ObjectGraph 按 Application / Activity 两级作用域装配。整个工程只有一个 app 模块，代码量小，适合当成"表现层分层"的对照读本。

**核心类**：`TvShowCatalogPresenter`、`TvShowPresenter`、`TvShowViewModel`、`TvShowCatalogFragment`、`TvShowFragment`、`GetTvShowsInteractor`

**图表**：类图 9 个类型 + 2 张说明卡；流程图 10 个节点 / 4 条泳道（EffectiveAndroidUI 目录页加载主流程）。

[源码解析](EffectiveAndroidUI/README.md) · [类图](EffectiveAndroidUI/class-diagram.html) · [流程图](EffectiveAndroidUI/flow-diagram.html)

---

## 数据与序列化

> 对象存储与 JSON 编解码的底层实现。

### fastjson

> 类别：json

fastjson 是阿里巴巴开源的 Java JSON 库，主打“快”：对外是 `JSON` 静态门面，内部把工作拆成词法层（`JSONLexer`）、语法层（`DefaultJSONParser`）和编解码注册表（`ParserConfig` / `SerializeConfig`）。解析时按目标 `Type` 取 `ObjectDeserializer` 填充 Bean，序列化时按运行期 `Class` 取 `ObjectSerializer` 写出文本，JVM 上还会用 ASM 生成专用序列化器。

**核心类**：`JSON`、`DefaultJSONParser`、`JSONLexer`、`JSONLexerBase / JSONScanner`、`ParserConfig`、`JavaBeanDeserializer`

**图表**：类图 10 个类型 + 2 张说明卡；流程图 8 个节点 / 5 条泳道（fastjson 解析与序列化主流程）。

[源码解析](fastjson/README.md) · [类图](fastjson/class-diagram.html) · [流程图](fastjson/flow-diagram.html)

### realm-java

> 类别：数据库

realm-java 是 Realm 数据库的 Android Java 绑定层：对外提供 `Realm`、`RealmObject`、`RealmQuery` 等面向对象 API，对内把每个对象模型映射成 native 的表、行与版本句柄，真正的存储与查询全部在 Realm Core（C++）中执行，Java 侧只负责生命周期、线程约束和句柄管理。

**核心类**：`Realm`、`BaseRealm`、`RealmCache`、`RealmObject`、`RealmQuery`、`RealmResults`

**图表**：类图 10 个类型 + 2 张说明卡；流程图 10 个节点 / 4 条泳道（realm-java 打开实例、写事务与查询主流程）。

[源码解析](realm-java/README.md) · [类图](realm-java/class-diagram.html) · [流程图](realm-java/flow-diagram.html)

### greenDAO

> 类别：数据

greenDAO 是面向 Android 的轻量 ORM：编译期用 `DaoGenerator` 把实体 schema 生成 `DaoMaster`/`DaoSession`/`XxxDao` 与 `Xxx` 实体类，运行期 `DaoCore` 只保留 `Abstract*` 基类，通过 `Property` 元数据 + 预编译语句完成对 SQLite 的增删改查，并用 `IdentityScope` 做会话级对象缓存。

**核心类**：`AbstractDaoMaster`、`AbstractDaoSession`、`AbstractDao<T,K>`、`DaoConfig`、`TableStatements`、`Property`

**图表**：类图 10 个类型 + 2 张说明卡；流程图 9 个节点 / 4 条泳道（greenDAO 一次 insert 的主流程）。

[源码解析](greenDAO/README.md) · [类图](greenDAO/class-diagram.html) · [流程图](greenDAO/flow-diagram.html)

---

## 性能、调试与测试

> 内存泄漏、调试桥、日志与测试示例。

### leakcanary

> 类别：内存

LeakCanary 是 Square 出品的内存泄漏检测库：用 `ObjectWatcher` 给对象挂 `KeyedWeakReference` + `ReferenceQueue`，延迟检查对象是否被回收；保留对象超过阈值就触发 heap dump，再用 Shark 在堆里做支配树分析（GcRoot → 引用链），最终在 App 内给出泄漏报告。

**核心类**：`AppWatcher`、`ObjectWatcher`、`RetainedObjectTracker`、`ReferenceQueueRetainedObjectTracker`、`KeyedWeakReference`、`InternalLeakCanary`

**图表**：类图 12 个类型 + 3 张说明卡；流程图 9 个节点 / 4 条泳道（LeakCanary 检测流程：watch → 延迟检查 → 转储 → Shark 分析）。

[源码解析](leakcanary/README.md) · [类图](leakcanary/class-diagram.html) · [流程图](leakcanary/flow-diagram.html)

### stetho

> 类别：调试

stetho 是运行在 Android 进程内的调试桥：它只绑一条以 `_devtools_remote` 结尾的本地套接字，把 Chrome DevTools 协议和 `dumpapp` 命令行都接到这条连接上，让宿主应用无需改动业务代码就能被 `chrome://inspect` 和命令行工具观察。它的价值不在数据存储，而在于把「进程内状态」翻译成 Chrome 能渲染的协议消息，并且尽量把开销推迟到真正有调试器连入之后。

**核心类**：`Stetho`、`Stetho.Initializer`、`DumperPluginsProvider`、`ServerManager`、`LocalSocketServer`、`LazySocketHandler`

**图表**：类图 10 个类型 + 2 张说明卡；流程图 10 个节点 / 4 条泳道（stetho 启动、接入与请求上报主流程）。

[源码解析](stetho/README.md) · [类图](stetho/class-diagram.html) · [流程图](stetho/flow-diagram.html)

### logger

> 类别：调试

logger 是 orhanobut 出品的 Android 日志门面：`Logger.d("hello %s", "world")` 这类静态调用统一进入 `LoggerPrinter`，由 `LogAdapter` 决定是否放行，再由可插拔的 `FormatStrategy` 把一条日志美化成带边框、线程名、调用栈、JSON/XML 缩进的文本，最后交给 `LogStrategy` 落到 Logcat 或文件。

**核心类**：`Logger`、`Printer / LoggerPrinter`、`LogAdapter / AndroidLogAdapter / DiskLogAdapter`、`FormatStrategy / PrettyFormatStrategy`、`PrettyFormatStrategy.Builder`、`CsvFormatStrategy`

**图表**：类图 10 个类型 + 2 张说明卡；流程图 9 个节点 / 4 条泳道（logger 一次日志的格式化流程）。

[源码解析](logger/README.md) · [类图](logger/class-diagram.html) · [流程图](logger/flow-diagram.html)

### android-testing

> 类别：测试

这不是一个测试框架，而是 Google 官方的"测试示例仓库"：把 Espresso、UI Automator、AndroidJUnitRunner、本地 JVM 单元测试四类做法拆成 23 个可独立编译的示例工程（`projects.conf` 精选 19 个纳入批量脚本），用同一个"输入文字→点按钮→看结果"的业务场景反复演示不同 API 的写法。

**核心类**：`ChangeTextBehaviorTest`、`MainActivity`、`LongListActivityTest`、`LongListActivity`、`HintMatchersTest`、`HintMatcher`

**图表**：类图 10 个类型 + 2 张说明卡；流程图 10 个节点 / 4 条泳道（Espresso 用例执行主流程）。

[源码解析](android-testing/README.md) · [类图](android-testing/class-diagram.html) · [流程图](android-testing/flow-diagram.html)

---

## 插件化与热修复

> 在运行时替换或加载代码。

### tinker

> 类别：热修复

Tinker 是腾讯开源的 Android 热修复框架：构建期用 dex 差分打出小体积补丁包，运行期在独立 `:patch` 进程里校验并合成补丁产物，下一次进程启动时由 `TinkerLoader` 把补丁 dex 注入 `ClassLoader`，从而在不重装 APK 的前提下替换类、so 与资源。

**核心类**：`TinkerInstaller`、`Tinker`、`TinkerPatchService`、`AbstractPatch / UpgradePatch`、`DexDiffPatchInternal`、`ShareTinkerInternals`

**图表**：类图 10 个类型 + 2 张说明卡；流程图 10 个节点 / 4 条泳道（Tinker 补丁生成与应用主流程）。

[源码解析](tinker/README.md) · [类图](tinker/class-diagram.html) · [流程图](tinker/flow-diagram.html)

### DroidPlugin

> 类别：插件化

DroidPlugin 是一个「免安装、免修改、免重打包」就能运行第三方 apk 的 Android 插件框架：它用反射加动态代理 Hook 系统服务，再用宿主清单里预注册的空壳组件（占坑）骗过系统，最后在插件进程里把组件还原成插件自己的实现。

**核心类**：`PluginHelper`、`PluginManager`、`IPluginManagerImpl`、`PluginPackageParser`、`PluginClassLoader`、`HookFactory`

**图表**：类图 10 个类型 + 2 张说明卡；流程图 10 个节点 / 4 条泳道（DroidPlugin 插件安装与启动主流程）。

[源码解析](DroidPlugin/README.md) · [类图](DroidPlugin/class-diagram.html) · [流程图](DroidPlugin/flow-diagram.html)

---

## 阅读建议

- **先类图后流程**：类图回答「有哪些角色、谁依赖谁」，流程图回答「一次调用走过哪些角色」。
- **按版本读**：每个目录的 README 都标了快照 commit，读源码时先 `git checkout` 到该 commit，行号才对得上。
- **注意代际**：排行榜成文于 2017 年，部分框架已归档或被官方方案替代（Butter Knife → View Binding、otto → EventBus、androidannotations → Hilt/KSP、SlidingMenu → DrawerLayout 等），README 的「版本注记」里写了各自的断代情况。

