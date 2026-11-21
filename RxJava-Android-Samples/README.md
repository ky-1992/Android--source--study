# RxJava-Android-Samples 源码解析

## 一句话定位

RxJava-Android-Samples 是 kaushikgopal 维护的 RxJava 2 示例工程：一个 `app` 模块里放二十来个 Fragment，每个 Fragment 演示一种操作符组合——去抖搜索、轮询、网络请求与组合、表单校验、并发调度等。它不提供库，价值在于"一条链一个示例"的可读组织，可以作为 RxJava 操作符与 Android 生命周期、线程调度结合方式的学习样本。

## 源码快照

- 上游：<https://github.com/kaushikgopal/RxJava-Android-Samples>
- commit：`b84865252c203498c1f0a2fa0f751f3c9127a781`（浅克隆，`origin` 指向该地址）
- 版本：`app/build.gradle` 中 `versionName "1.2"`、`versionCode 2`

目录结构概览：

| 路径 | 内容 |
| --- | --- |
| `app/src/main/java/com/morihacky/android/rxjava/` | `MainActivity`、`MyApp`（LeakCanary/Volley/Timber 初始化） |
| `…/rxjava/fragments/` | 20 个示例 Fragment 与 `BaseFragment`，全部挂在 `MainFragment` 下 |
| `…/rxjava/retrofit/` | `GithubService`、`GithubApi` 与 `Contributor`/`User` 模型 |
| `…/rxjava/rxbus/` | `RxBus`：基于 RxRelay 的事件总线 |
| `…/rxjava/pagination/` | 分页示例：`PaginationFragment`、`PaginationAutoFragment` 与两个适配器 |
| `…/rxjava/volley/`、`wiring/` | Volley 示例与日志适配器 `LogAdapter` |

## 核心类

| 类 | 职责 | 关键方法（行号取自快照） |
| --- | --- | --- |
| `MyApp` | 应用入口：装 LeakCanary、初始化 Volley、种 Timber | `onCreate()` L23 |
| `MainActivity` | 宿主；懒加载 `RxBus` 单例 | `onCreate()` L22、`getRxBusSingleton()` L34 |
| `RxBus` | `PublishRelay` 事件总线，对外只暴露 `Flowable` | `send()` L14、`asFlowable()` L18、`hasObservers()` L22 |
| `BaseFragment` | 销毁时把自身交给 LeakCanary 观测 | `onDestroy()` L10 |
| `MainFragment` | 示例入口，21 个按钮各 new 一个示例 Fragment | `clickedOn()` L130 |
| `DebounceSearchEmitterFragment` | 编辑框文本流去抖后搜索 | `onDestroy()` L50、`onActivityCreated()` L71、`_getSearchObserver()` L87 |
| `GithubService` | Retrofit 工厂：RxJava2 adapter + Gson + token 拦截器 | `createGithubService()` L18 |
| `GithubApi` | GitHub REST 的 `Observable` 声明 | `contributors()` L13、`user()` L21 |
| `RetrofitFragment` | 网络请求与 `flatMap`/`zip` 组合 | `onListContributorsClicked()` L90、`onListContributorsWithFullUserInfoClicked()` L128 |
| `PollingFragment` | `interval` 与 `repeatWhen` 两种轮询 | `onStartSimplePollingClicked()` L74、`onStartIncreasinglyDelayedPolling()` L100 |
| `NetworkDetectorFragment` | 广播转 `PublishProcessor`，判断联网状态 | `listenToNetworkConnectivity()` L92、`getConnectivityStatus()` L106 |

类图见 [class-diagram.html](./class-diagram.html)，图中保留宿主、总线、网络与四个代表性示例 Fragment，其余示例只在阅读路径里点名。

## 关键流程

去抖搜索（对应流程图）：`onActivityCreated()`（L71）里拼出 `RxTextView.textChangeEvents(_inputSearchText)`（L77）→ `debounce(400, TimeUnit.MILLISECONDS)`（L78）→ `filter(changes -> isNotNullOrEmpty(...))`（L79）→ `observeOn(AndroidSchedulers.mainThread())`（L80）→ `subscribeWith(_getSearchObserver())`（L81）。`_getSearchObserver()`（L87）返回 `DisposableObserver<TextViewTextChangeEvent>`，`onNext`（L101）把搜索词拼进日志列表，`onError`（L95）打日志，`onDestroy()`（L50）里 `_disposable.dispose()` 并 `unbinder.unbind()`。`debounce` 默认跑在 `Schedulers.computation()`，所以链尾必须切回主线程才能碰 UI。

网络与组合：`RetrofitFragment.onCreate()`（L54）从资源里取 token 交给 `GithubService.createGithubService()`（L57），同时建 `CompositeDisposable`（L59）。第一次点击 `onListContributorsClicked()`（L90）直接 `contributors(...)` 后 `subscribeOn(io)`+`observeOn(mainThread)`；第二次点击 `onListContributorsWithFullUserInfoClicked()`（L128）先用 `flatMap(Observable::fromIterable)` 展平列表，再对每个 contributor `flatMap` 到 `user(login)` 并用 `Observable.zip` 与原 contributor 合成 `Pair`（L134–L143），最后仍切回主线程。两次请求都 `subscribeWith(new DisposableObserver<...>)` 并 `_disposables.add(...)`。

轮询：简单版用 `Flowable.interval(INITIAL_DELAY, POLLING_INTERVAL, MILLISECONDS)` + `map` + `take(pollCount)`（L79–L81）；递增延迟版用 `Flowable.just(1L).repeatWhen(new RepeatWithDelay(...))`（L111–L112），`RepeatWithDelay` 实现 `Function<Flowable<Object>, Publisher<Long>>`（L207），在 `apply` 里把每次重订阅的间隔递增。

联网检测与总线：`NetworkDetectorFragment.listenToNetworkConnectivity()`（L92）注册 `BroadcastReceiver`，`onReceive` 里 `publishProcessor.onNext(getConnectivityStatus(context))`（L98），`getConnectivityStatus()`（L106）用 `ConnectivityManager` 判网络；`RxBus` 用 `PublishRelay.create().toSerialized()`（L12）做线程安全总线，`send()` 走 `accept()`，`asFlowable()` 用 `BackpressureStrategy.LATEST` 转成 `Flowable`。

流程图见 [flow-diagram.html](./flow-diagram.html)，覆盖页面创建 → 输入绑定 → 去抖过滤 → 切线程 → 消费与释放。

## 设计要点

- **示例即文档**：每个 Fragment 只演示一条链，`MainFragment` 的 `clickedOn()`（L130）用 `addToBackStack(tag)` 后 `replace(android.R.id.content, ...)`，新增示例只要加一个按钮和一个类。
- **线程纪律统一**：网络示例一律 `subscribeOn(Schedulers.io())`/`newThread()` 加 `observeOn(AndroidSchedulers.mainThread())`；`debounce` 这类本身在 computation 上跑的操作符，链尾也必须显式切回主线程。
- **释放纪律**：`RetrofitFragment` 用 `CompositeDisposable` 聚合，`DebounceSearchEmitterFragment` 用单个 `Disposable`，都在 `onDestroy` 里 `dispose()` 并 `unbind()`；`BaseFragment.onDestroy()`（L10）再把 Fragment 交给 `RefWatcher.watch()`，把"销毁后是否泄漏"显式暴露出来。
- **总线用 Relay 不用 Subject**：`PublishRelay` 不完成也不报错，`toSerialized()` 解决多线程 `accept` 竞争，`asFlowable()` 让调用方拿到可背压的 `Flowable`，配合 `LATEST` 策略不会阻塞发送端。
- **Retrofit 与 RxJava 的桥接**：`GithubService.createGithubService()`（L18）用 `RxJava2CallAdapterFactory.create()`（L21）让接口方法直接返回 `Observable`，token 由 OkHttp 拦截器写进 `Authorization` 头（L29–L39）。
- **同一接口留两套签名**：`GithubApi` 里 `contributors`/`getContributors`、`user`/`getUser` 各有一份 `Observable` 版与同步版，方便对照阻塞调用与响应式调用的差异。

## 阅读路径

1. `app/src/main/java/com/morihacky/android/rxjava/fragments/DebounceSearchEmitterFragment.java` —— L70–L105，最短的完整链路。
2. `…/MainActivity.java`、`…/MyApp.java` —— 宿主与初始化，理解示例如何被装配。
3. `…/rxbus/RxBus.java` —— L10–L25，Relay 版总线。
4. `…/fragments/BaseFragment.java` —— L7–L15，LeakCanary 观测点。
5. `…/retrofit/GithubService.java`、`…/retrofit/GithubApi.java` —— Retrofit 与 RxJava2 的接口契约。
6. `…/fragments/RetrofitFragment.java` —— L90–L150，`flatMap` 展平与 `zip` 组合。
7. `…/fragments/PollingFragment.java` —— L74–L130 与 L200–L220，`interval` 与 `repeatWhen` 两种轮询。
8. `…/fragments/NetworkDetectorFragment.java` —— L92–L110，传统回调转 `PublishProcessor`。

## 版本注记

`app/build.gradle` 声明 `versionName "1.2"`、`versionCode 2`，AGP 2.3.3、compileSdk 24、minSdk 15，仍用 `android.support.*`（support 库 25.3.1）与 ButterKnife 8.5.1；Rx 侧是 RxJava 2.0.7、RxAndroid 2.0.1、RxRelay 2.0.0、RxBinding 2.0.0，Retrofit 2.0.0 需要 `retrofit2-rxjava2-adapter` 才能返回 `Observable`。这是 RxJava 2 早期生态的快照，仓库没有发布 tag，具体版本以该 commit 为准。
