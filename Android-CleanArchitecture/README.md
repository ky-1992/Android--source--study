# Android-CleanArchitecture 源码解析

## 一句话定位

这是 Fernando Cejas 的 Clean Architecture 示范工程：把 Android 应用切成 `presentation`（MVP + 界面）、`domain`（纯 Java 业务规则）、`data`（网络/磁盘实现）三层，依赖方向全部指向 domain——presentation 依赖 domain 的 `UseCase` 与 `Repository` 抽象，data 实现这些抽象，用 RxJava 2 的 `Observable` 串起异步数据流。

## 源码快照

- 上游：<https://github.com/android10/Android-CleanArchitecture>
- commit：`8ed4222c537e40db05e9e685bbc253fafb6b8e1f`（2020-12-28，合并 PR #252 更新工程配置）
- 工具链：`compileSdkVersion 26`、`minSdkVersion 15`、Java 8；`buildsystem/dependencies.gradle` 声明 RxJava `2.0.2`、Dagger `2.8`、ButterKnife `7.0.1`、OkHttp `2.5.0`、Gson `2.3`
- 模块：

| 模块 | 内容 |
| --- | --- |
| `presentation/` | 唯一的 app 模块：Activity/Fragment/Adapter、`Presenter`、`UserModel`、Dagger 组件与模块 |
| `domain/` | 纯 Java：`User`、`UseCase`、`GetUserList`/`GetUserDetails`、`UserRepository`、线程抽象 |
| `data/` | Android library：`UserDataRepository`、`UserDataStore` 家族、`RestApi`、缓存、实体与 mapper |
| `buildsystem/` | 统一版本号与依赖表 |

## 核心类

| 类名 | 职责 | 关键方法（行号取自快照） |
| --- | --- | --- |
| `UseCase<T, Params>` | 执行单元模板：造 Observable、切线程、管理 Disposable | `buildUseCaseObservable()` L50、`execute()` L59、`dispose()` L70 |
| `GetUserList` | 取全部用户用例，构造注入 `UserRepository` | 类 L30、`buildUseCaseObservable()` L41 |
| `UserRepository` | 领域层数据契约（接口在 domain） | `users()` L29、`user()` L36 |
| `UserDataRepository` | 数据层实现：建 DataStore 并映射成 `User` | `users()` L50、`user()` L56 |
| `UserDataStoreFactory` | 按缓存状态选 Disk / Cloud 数据源 | `create()` L45、`createCloudDataStore()` L60 |
| `RestApiImpl` | OkHttp 拉 JSON 并交给 `UserEntityJsonMapper` 解析 | `userEntityList()` L50、`isThereInternetConnection()` L105 |
| `UserEntityDataMapper` | `UserEntity` → `User`（去网络/存储细节） | `transform()` L42、集合版 L61 |
| `Presenter` | 视图生命周期契约 | `resume()` L26、`pause()` L32、`destroy()` L38 |
| `UserListPresenter` | 列表页编排：调用例、转模型、驱动 View 状态 | `initialize()` L68、`getUserList()` L113、`UserListObserver` L117 |
| `UserModelDataMapper` | `User` → `UserModel`（UI 字段） | `transform()` L42、集合版 L62 |
| `UserListFragment` | View 实现：注入 presenter、渲染列表 | 注入 L63、`setView()` L78、`renderUserList()` L128 |

类图见 [class-diagram.html](./class-diagram.html)。

## 关键流程

**初始化**：`UserListFragment.onCreate()` 通过 `getComponent(UserComponent.class).inject(this)` 注入 `UserListPresenter` 与 adapter（L63–66）；`onViewCreated()` 里 `presenter.setView(this)` 并把自身交给 presenter，再调 `initialize()`（L76–82）。

**Presenter 编排**：`initialize()` → `loadUserList()` 先 `hideRetry()`、`showLoading()`，再 `getUserList()`（L75–79）；`getUserList()` 执行 `getUserListUseCase.execute(new UserListObserver(), null)`（L113–115）。`UserListObserver` 继承 `DefaultObserver<List<User>>`：`onNext` 把用户集合经 `UserModelDataMapper` 转成 `Collection<UserModel>` 后 `renderUserList()`（L107–111、L129–131）；`onError` 收起 loading、用 `DefaultErrorBundle` 加 `ErrorMessageFactory` 转文案并 `showRetry()`。

**UseCase 切线程**：`execute()` 对 `buildUseCaseObservable(params)` 依次 `.subscribeOn(Schedulers.from(threadExecutor))` 与 `.observeOn(postExecutionThread.getScheduler())`，再 `subscribeWith(observer)` 并加入 `CompositeDisposable`（L59–65）；`JobExecutor` 内部是 3 到 5 线程的 `ThreadPoolExecutor`（L31–41），`UIThread` 返回 `AndroidSchedulers.mainThread()`（L34–36）。`destroy()` 时 presenter 调 `useCase.dispose()` 释放订阅（L60–63）。

**数据链路**：`UserDataRepository.users()` 固定走云端——`userDataStoreFactory.createCloudDataStore()` 后 `userEntityList().map(userEntityDataMapper::transform)`（L50–54）；`user(id)` 才走 `create()` 判断缓存：未过期且已缓存用 `DiskUserDataStore`，否则云端（L45–55）。`CloudUserDataStore.userEntityDetails()` 用 `doOnNext(userCache::put)` 顺带写缓存（L47–49）。`RestApiImpl` 先查网络连通性，再用 `Observable.create` 包一次同步 OkHttp 调用，失败发 `NetworkConnectionException`（L50–69）。缓存的落盘由 `UserCacheImpl` 负责：实体序列化后写 `getCacheDir()` 下的文件，`isExpired()` 依据 `FileManager` 存在 SharedPreferences 里的最后更新时间判断（L32–144）。

**详情页复用同一套分层**：`UserDetailsFragment.onViewCreated()` 在 `savedInstanceState == null` 时调 `loadUserDetails()`（L68–74），后者执行 `userDetailsPresenter.initialize(currentUserId())`（L135–139），重试按钮也走同一入口；`UserDetailsPresenter.initialize(userId)`（L68）把 userId 装进 `GetUserDetails.Params.forUser()`，用例再调 `userRepository.user(params.userId)`；`GetUserDetails` 对 `Params` 做了 `Preconditions.checkNotNull`，数据来源则交给 `UserDataRepository.user()` 的缓存分支。

流程图见 [flow-diagram.html](./flow-diagram.html)，覆盖 View → Presenter → UseCase → Repository → RestApi → 映射 → 回调 → 渲染。

## 设计要点

- **依赖倒置**：`UserRepository` 放在 `domain`，`UserDataRepository` 放在 `data`；domain 只依赖 `javax.inject`、RxJava、Arrow 断言，没有任何 Android 类，可纯 JVM 测试（domain 单测即用 Mockito + AssertJ）。
- **UseCase 收口线程与生命周期**：业务用例只负责 `buildUseCaseObservable`，线程切换、订阅注册、批量 `dispose` 全在基类，避免每个用例重复写 `subscribeOn/observeOn`。
- **两次模型转换**：`UserEntity`（JSON 字段名）→`User`（领域）→`UserModel`（UI 字段），每层只认识自己的模型，换接口或换 UI 都不连锁改动。
- **DataStore 策略**：工厂按缓存状态选择磁盘或云端实现，Repository 不关心数据来自哪；`CloudUserDataStore` 写缓存、`DiskUserDataStore` 读缓存，职责单一。
- **MVP 用接口隔离 View**：`LoadDataView` 定义 loading/retry/error/context 通用能力，`UserListView` 只加业务方法；Fragment 实现接口，Presenter 持有接口引用，测试里可换 mock。
- **Dagger 分层作用域**：`ApplicationComponent` 暴露 `ThreadExecutor`、`PostExecutionThread`、`UserRepository` 等单例（`ApplicationModule` L48–65），`@PerActivity` 的 `UserComponent` 提供 presenter 与 mapper，Activity 销毁即释放。

## 阅读路径

1. `domain/.../interactor/UseCase.java` —— 先读基类模板与线程装配（L35–83）。
2. `domain/.../repository/UserRepository.java` —— 看清领域层只声明契约。
3. `domain/.../interactor/GetUserList.java` —— 最薄用例：构造注入 + 转发（L30–43）。
4. `data/.../repository/UserDataRepository.java` —— 实现类与 mapper 组合（L50–59）。
5. `data/.../repository/datasource/UserDataStoreFactory.java` —— 磁盘/云端选择逻辑（L45–65）。
6. `data/.../net/RestApiImpl.java` + `entity/mapper/UserEntityDataMapper.java` —— 网络与实体转换。
7. `presentation/.../presenter/UserListPresenter.java` —— MVP 编排与 `UserListObserver`（L38–132）。
8. `presentation/.../view/fragment/UserListFragment.java` + `internal/di/modules/ApplicationModule.java` —— View 实现与依赖注入装配。

## 版本注记

该 commit 是 2020 年底的工程配置更新，代码线已迁到 RxJava 2（`io.reactivex.rxjava2`）、Dagger 2.8 与 `annotationProcessor`，但仍用 `android.support.*`（未迁 AndroidX），OkHttp 停留在 2.5.0，`versionName` 为 `1.0`、`versionCode` 为 `1`，`compileSdkVersion` 26 而 `minSdkVersion` 只有 15。工程同时保留了 Robolectric（3.1.1）、Espresso 与 Mockito 的测试依赖，`presentation` 的 UI 测试放在 `androidTest`，`domain`/`data` 用纯 JUnit。本次为 `--depth 1` 浅克隆，没有更早历史，所有行为以该 commit 为准。
