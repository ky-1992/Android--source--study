# RxBinding 源码解析

## 一句话定位

RxBinding 把 Android 控件的事件包装成 RxJava 3 的 `Observable`：每个绑定都是一个 Kotlin 扩展函数（如 `View.clicks()`、`TextView.textChanges()`），函数内部返回一个 `private` 实现类，订阅时先校验主线程、再注册系统监听器，解绑时反注册。它解决的问题是把"匿名内部类 + 手动注销"换成可组合的操作符链，同时统一主线程与泄漏语义。

## 源码快照

- 上游：<https://github.com/JakeWharton/RxBinding>
- commit：`78f7ebdeb9dcf373abb9fc43bde838cfe2fef62a`（浅克隆，`origin` 指向该地址）
- 版本：`gradle.properties` 中 `GROUP=com.jakewharton.rxbinding4`、`VERSION_NAME=4.0.1-SNAPSHOT`

目录结构概览：

| 路径 | 内容 |
| --- | --- |
| `rxbinding/src/main/java/com/jakewharton/rxbinding4/` | 核心模块：`InitialValueObservable.kt` 与 `internal/mainThread.kt` |
| `…/rxbinding4/view/` | `View` 系绑定，23 个文件：`clicks`、`attachEvents`、`layoutChanges`、`longClicks`、`drags`、`touches` 等 |
| `…/rxbinding4/widget/` | `TextView`、`AdapterView`、`AbsListView` 等控件绑定，32 个文件 |
| `rxbinding-core/` | 仅 `NestedScrollViewScrollChangeEventObservable.kt` 一个绑定 |
| `rxbinding-appcompat/`、`-recyclerview/`、`-drawerlayout/`、`-leanback/`、`-material/`、`-slidingpanelayout/`、`-swiperefreshlayout/`、`-viewpager/`、`-viewpager2/` | 按控件族拆分的模块，各自依赖对应的 androidx 库 |
| `testing-utils/` | 测试辅助，用于断言 Observable 的事件序列 |

## 核心类

| 类 | 职责 | 关键方法（行号取自快照） |
| --- | --- | --- |
| `InitialValueObservable` | 抽象基类：订阅时先挂监听器再补发首值 | `subscribeActual()` L9、`subscribeListener()` L14、`skipInitialValue()` L16 |
| `ViewClickObservable` | `View.clicks()` 的私有实现 | `subscribeActual()` L34、`Listener.onClick()` L48、`Listener.onDispose()` L54 |
| `checkMainThread` | 主线程断言，失败走 `onError` 而非抛异常 | `internal/mainThread.kt:25` |
| `TextViewTextChangesObservable` | `textChanges()` 实现，发 `CharSequence` | `subscribeListener()` L38、`initialValue` L44、`Listener.onDispose()` L63 |
| `TextViewTextChangeEventObservable` | `textChangeEvents()` 实现，发含位置信息的 data class | `TextViewTextChangeEvent` L40、`subscribeListener()` L55、`Listener.onTextChanged()` L69 |
| `AdapterDataChangeObservable` | `Adapter.dataChanges()` 实现 | `subscribeListener()` L29、`ObserverDisposable.onChanged()` L45、`onDispose()` L52 |
| `ViewAttachEventObservable` | `attachEvents()` 实现，发附着/分离事件 | `subscribeActual()` L30、`onViewAttachedToWindow()` L44、`onDispose()` L56 |

类图见 [class-diagram.html](./class-diagram.html)，图中保留 `InitialValueObservable` 与三个子类、点击流的内部 `Listener`，省略 `widget/` 下其余绑定。

## 关键流程

创建：`View.clicks()`（`ViewClickObservable.kt:26`）只做一件事——`ViewClickObservable(this)`；文件头用 `@file:JvmName("RxView")` 让 Kotlin 扩展函数在 Java 侧汇总成 `RxView` 静态类。

订阅：`subscribeActual()`（L34）先调 `checkMainThread(observer)`，非主线程时函数直接 `return`，而 `checkMainThread`（`mainThread.kt:25`）内部会先 `observer.onSubscribe(Disposable.empty())` 再 `observer.onError(IllegalStateException(...))`，符合 Rx 协议；主线程上才 `new Listener(view, observer)`、`observer.onSubscribe(listener)`，最后 `view.setOnClickListener(listener)`。

发射与解绑：`Listener` 同时实现 `MainThreadDisposable` 与 `OnClickListener`，`onClick()`（L48）在 `!isDisposed` 时 `observer.onNext(Unit)`——点击值本身无意义，只作通知；`onDispose()`（L54）执行 `view.setOnClickListener(null)`。因为监听器在 `dispose()` 时由 `MainThreadDisposable` 保证在主线程回调 `onDispose`，反注册是安全的。

带首值的流：`textChanges()` 走 `InitialValueObservable`，`subscribeActual()`（`InitialValueObservable.kt:9`）先 `subscribeListener(observer)` 再 `observer.onNext(initialValue)`，所以订阅瞬间就能拿到 `view.text`；`dataChanges()` 同理：先注册 `DataSetObserver`，再把 adapter 作为首值发出。不需要首值时调 `skipInitialValue()`（L16），它返回内部 `Skipped`，只挂监听器不发首值。

流程图见 [flow-diagram.html](./flow-diagram.html)，覆盖扩展函数 → 订阅校验 → 注册监听 → 点击发射 → 解除订阅五段。

## 设计要点

- **函数即 API**：所有实现类都是 `private`，公开面只有带 `@CheckResult` 的扩展函数，既能避免误用，也让 Java 侧通过 `@JvmName` 得到 `RxView`、`RxTextView` 这类入口。
- **主线程契约统一**：`view/`、`widget/` 下的绑定几乎都在 `subscribeActual` 或 `subscribeListener` 首行调 `checkMainThread`，把"必须在主线程订阅"写成运行时检查，而不是文档约定。
- **监听器继承 `MainThreadDisposable`**：`dispose()` 时 `onDispose()` 回到主线程执行，反注册不会抛 `CalledFromWrongThreadException`；同时 `!isDisposed` 判断挡掉了晚到的事件。
- **首值语义显式化**：`InitialValueObservable` 把"订阅即补发当前值"做成独立基类，`textChanges`、`dataChanges`、`textChangeEvents` 共用，`skipInitialValue()` 提供不补发的变体。
- **与系统 API 一一对应**：`clicks` 用 `setOnClickListener`，因此同一 View 只能挂一个绑定流，Kdoc 里明确写了这条限制；`attachEvents`、`layoutChanges` 改用 add/remove 形式的监听器，可多个流并存。

## 阅读路径

1. `rxbinding/src/main/java/com/jakewharton/rxbinding4/view/ViewClickObservable.kt` —— 最短的完整样例，L26–L57 把创建、订阅、发射、解绑全走一遍。
2. `…/rxbinding4/internal/mainThread.kt` —— `checkMainThread`（L25–L33）的失败路径。
3. `…/rxbinding4/InitialValueObservable.kt` —— 首值补发与 `skipInitialValue`（L6–L22）。
4. `…/rxbinding4/widget/TextViewTextChangesObservable.kt` —— `TextWatcher` 版本（L30–L66）。
5. `…/rxbinding4/widget/TextViewTextChangeEventObservable.kt` —— 带位置信息的事件对象（L40–L81）。
6. `…/rxbinding4/widget/AdapterDataChangeObservable.kt` —— 额外做了 `checkMainThread`，可对照 `DataSetObserver` 的注册反注册。
7. `…/rxbinding4/view/ViewAttachEventObservable.kt` —— 可多流并存的监听器形态。

## 版本注记

`CHANGELOG.md` 显示 4.0.0（2020-05-19）起切换到 RxJava 3，包名与 groupId 改为 `com.jakewharton.rxbinding4`，本快照的 4.0.1-SNAPSHOT 就建立在这条线上；代码里 `io.reactivex.rxjava3.*` 与 `androidx.*` 已是唯一依赖，构建配置为 Kotlin 1.3.72、compileSdk 29、minSdk 14。与 3.x 相比 API 形状基本不变，只是替换了 RxJava 版本号。
