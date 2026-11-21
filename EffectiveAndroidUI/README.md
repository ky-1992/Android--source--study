# EffectiveAndroidUI 源码解析

## 一句话定位

Pedro Vicente Gómez Sánchez 的示例工程，用一个"读取并展示美剧目录"的场景演示 Android 表现层的两套写法：目录页走 MVP（Presenter 持有内部 View 接口），详情页走 MVVM（ViewModel 用 Listener 推送状态、用 ActionCommand 封装点击）；领域层是 Interactor + Executor 的用例模型，依赖注入用 Dagger 1 的 ObjectGraph 按 Application / Activity 两级作用域装配。整个工程只有一个 app 模块，代码量小，适合当成"表现层分层"的对照读本。

## 源码快照

- 上游：<https://github.com/pedrovgs/EffectiveAndroidUI>
- commit：`b9e6fcc795de5d01c9f720dec7511cda0bab6c02`（2026-01-16，"Remove broken Travis CI badge"）
- 单模块 `app`，约 54 个 Java 文件，包 `com.github.pedrovgs.effectiveandroidui` 分为 `di`（RootModule/ActivityModule）、`domain`（用例、Catalog、TvShow、Chapter）、`executor`（Executor、Interactor、MainThread）、`ui`（activity、fragment、presenter、viewmodel、renderer）、`util`
- `app/build.gradle` 声明 `compileSdkVersion 23`、`minSdkVersion 8`、`targetSdkVersion 23`，依赖 `dagger:1.2.+`、`butterknife:5.1.+`、`renderers:1.1.+`、`picasso:2.3.+`、`draggablepanel:1.+`

## 核心类

| 类 | 职责 | 关键方法（行号取自快照） |
| --- | --- | --- |
| `TvShowCatalogPresenter` | 目录页 MVP 表现器，持有内部 `View` 接口并做空值校验 | `initialize()` L60、`loadTvShows()` L102、`View` L145 |
| `TvShowPresenter` | 详情页 MVP 表现器，可被拖拽面板复用 | `loadTvShow()` L78、`showTvShow()` L111 |
| `TvShowViewModel` | MVVM 视图模型，用 `Listener` 回调驱动视图 | `loadTvShow()` L56、`notifyTvShowLoaded()` L108、`Listener` L133 |
| `TvShowCatalogFragment` | 实现 `TvShowCatalogPresenter.View`，只做视图操作 | `onViewCreated()` L61、`renderVideos()` L106 |
| `TvShowFragment` | 实现 `TvShowViewModel.Listener`，演示 MVVM | `bindViewModel()` L147、`onChaptersLoaded()` L104 |
| `GetTvShowsInteractor` | 目录用例，随机失败加延时模拟网络 | `execute()` L56、`run()` L66、`mainThread.post` L103 |
| `GetTvShowByIdInteractor` | 按 id 查询单条用例 | `execute()` L58、`searchTvShow()` L92 |
| `Catalog` | 硬编码的内存数据集，查找未命中抛异常 | `getTvShows()` L385、`getTvShowById()` L399 |
| `ThreadExecutor` / `MainThreadImpl` | 线程边界：线程池执行、Handler 回主线程 | `run()` L56、`post()` L36 |
| `TvShowsApplication` / `BaseActivity` | 根图与 Activity 作用域注入 | `plus()` L77、`injectDependencies()` L67 |

两个关键接口都定义在使用方内部：`TvShowCatalogPresenter.View` 由 `TvShowCatalogFragment` 实现，`TvShowViewModel.Listener` 由 `TvShowFragment` 实现。这样表现层只声明"需要视图做什么"，不依赖任何 Activity/Fragment 类型。

## 关键流程

1. **绑定**：`TvShowCatalogFragment.onViewCreated` 先建好 GridView 适配器，再 `setView(this)` 与 `initialize()`，后者校验视图非空并调用 `loadTvShows()`。
2. **执行**：`GetTvShowsInteractor.execute(callback)` 只保存回调，随即把自己交给 `Executor`；`ThreadExecutor` 用 core 3 / max 5、`LinkedBlockingQueue` 的线程池 `submit`。
3. **回调**：`run()` 先用 `Thread.sleep(1500)` 模拟网络，再由 `RandomUtils.percent(50)` 决定走成功还是 `onConnectionError`，两种结果都经 `MainThread.post` 切回主线程。
4. **渲染**：Presenter 每次写视图前检查 `view.isReady()`，成功时依次 `renderVideos`、`hideLoading`、`updateTitleWithCountOfTvShows`。
5. **恢复**：`onSaveInstanceState` 把 `TvShowCollection` 存进 Bundle，`onViewStateRestored` 用 `loadCatalog` 回填，旋转后不必重新请求。
6. **分栏**：`MainActivity` 在平板布局里同时挂 `TvShowFragment`（MVVM）与 `TvShowDraggableFragment`（MVP），两者都可见时禁用拖拽面板的实例状态保存，避免旋转后互相覆盖；点击缩略图时由 `TvShowCatalogPresenter.onTvShowThumbnailClicked` 交给 `Navigator.openTvShowDetails`：当前 Activity 若已挂详情 Fragment 就直接通知它，否则启动 `TvShowActivity`，由 Activity 把 id 传给 Fragment。

## 设计要点

1. **Presenter 不碰 Android 类型**：`View` 只是 Presenter 内部的接口，Fragment 去实现它，表现层因此可以在 JVM 上单测。
2. **Dagger 1 的 plus() 做作用域**：`TvShowsApplication` 用 `RootModule` 建根图并提供 `inject`/`plus`；`BaseActivity` 把 `getModules()` 加上 `ActivityModule` 后 `plus` 出 Activity 图，Activity 结束即丢弃。
3. **模块只暴露接口**：`TvShowsModule` 用 `@Provides GetTvShows` 把 `GetTvShowsInteractor` 实例以接口返回，`ExecutorModule` 同样把 `ThreadExecutor`、`MainThreadImpl` 分别以 `Executor`、`MainThread` 暴露，调用方看不到实现类。
4. **MVVM 侧用命令对象**：`ShowTvShowOnBrowserActionCommand` 的 URL 由 ViewModel 在数据到达时写入，Fragment 点击时只调 `getTvShowClickedCommand().execute()`，视图不关心"打开什么"。
5. **两道守卫防错**：`isReady()`（Fragment 是否仍 attached）与 `isAlreadyLoaded()`（适配器是否已有数据）让迟到的异步回调既不崩溃，也不会覆盖已有内容。
6. **为对比而保留重复**：工程同时有 `TvShowFragment`（MVVM）与 `TvShowDraggableFragment`（MVP），注释明确说明是刻意不做继承，方便读者并排比较。
7. **样板代码收进基类**：`BaseActivity` 与 `BaseFragment` 分别在创建时完成 Dagger 注入和 `ButterKnife.inject`，子类只声明字段；列表项交给 renderers 库的 `Renderer`/`RendererAdapter`，由 `TvShowUIModule` 提供各自的 Builder。

## 阅读路径

阅读时建议先只看 MVP 一条线（1~3 步），把 `onViewCreated` 到 `renderVideos` 的调用链走通，再回头看 MVVM 与 Dagger 的部分。

1. `app/src/main/java/com/github/pedrovgs/effectiveandroidui/ui/presenter/Presenter.java`
2. `app/src/main/java/com/github/pedrovgs/effectiveandroidui/ui/presenter/TvShowCatalogPresenter.java`
3. `app/src/main/java/com/github/pedrovgs/effectiveandroidui/ui/fragment/TvShowCatalogFragment.java`
4. `app/src/main/java/com/github/pedrovgs/effectiveandroidui/domain/GetTvShowsInteractor.java` 与 `domain/GetTvShows.java`
5. `app/src/main/java/com/github/pedrovgs/effectiveandroidui/executor/ThreadExecutor.java`、`executor/MainThreadImpl.java`
6. `app/src/main/java/com/github/pedrovgs/effectiveandroidui/TvShowsApplication.java`、`di/RootModule.java`、`domain/TvShowsModule.java`
7. `app/src/main/java/com/github/pedrovgs/effectiveandroidui/ui/viewmodel/TvShowViewModel.java` 与 `ui/fragment/TvShowFragment.java`

## 版本注记

所有结论以 commit `b9e6fcc` 为准。工程停在 Dagger 1 + ButterKnife 5.1 的写法：`@Module(includes=..., injects=...)`、`ObjectGraph.create/plus`、`ButterKnife.inject`、`ActionBarActivity`，与今天的 Dagger 2/Hilt、ViewBinding 不是同一代 API，横向对比时注意版本线。库依赖全部写 `.+`，实际解析到的补丁版本会随构建时间变化。
