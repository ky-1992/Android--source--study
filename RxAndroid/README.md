# RxAndroid 源码解析

## 一句话定位

RxAndroid 是 RxJava 3 的 Android 绑定层，只做一件事：提供一个运行在 Android 主线程（或任意 `Looper`）上的 `Scheduler`，并让 `Disposable` 的释放回调也回到主线程。主模块只有 4 个类，是学习 RxJava 调度器扩展点与 `Handler`/`Looper` 协作的最小样本。

## 源码快照

- 上游：<https://github.com/ReactiveX/RxAndroid>
- commit：`311e44531ae7e795fcba14af64f6ad2280cf2205`（2026-11-02，依赖升级）
- 版本：`gradle.properties` 中 `VERSION_NAME=3.1.0-SNAPSHOT`；README 文档示例为 `rxandroid:3.0.2` + `rxjava:3.1.5`
- 仓库由 `rxandroid/`（主库）与 `sample-app/` 两个模块组成，包名 `io.reactivex.rxjava3.android`

| 路径 | 内容 |
| --- | --- |
| `rxandroid/.../android/` | `MainThreadDisposable` |
| `rxandroid/.../android/schedulers/` | `AndroidSchedulers`、`HandlerScheduler`（含内部类 `HandlerWorker`、`ScheduledRunnable`） |
| `rxandroid/.../android/plugins/` | `RxAndroidPlugins` 全局钩子 |
| `rxandroid/src/test/` | 四个类对应的 Robolectric 测试 |

## 核心类

| 类名 | 职责 | 关键方法（行号） |
| --- | --- | --- |
| `AndroidSchedulers` | 对外工厂：主线程调度器、按 Looper 构造调度器 | `mainThread()` L41、`from(looper)` L52、`internalFrom()` L69 |
| `MainHolder` | 静态内部类，懒加载默认调度器 | `DEFAULT` L28 |
| `HandlerScheduler` | `Scheduler` 实现：用 `Handler` 投递 `Runnable` | `scheduleDirect()` L35、`createWorker()` L50 |
| `HandlerWorker` | `Worker` 实现：带 token 的顺序执行单元 | `schedule()` L67、`dispose()` L98 |
| `ScheduledRunnable` | 消息体 + `Disposable` 句柄，异常统一上报 | `run()` L121、`dispose()` L130 |
| `MainThreadDisposable` | 保证 `onDispose()` 在主线程执行的 `Disposable` | `verifyMainThread()` L55、`dispose()` L70、`onDispose()` L80 |
| `RxAndroidPlugins` | 两个函数式钩子，可替换主线程调度器的初始化与查询 | `initMainThreadScheduler()` L33、`onMainThreadScheduler()` L48、`reset()` L78 |
| `Scheduler` / `Worker` / `Disposable` | RxJava 3 外部契约（不在本仓库） | `scheduleDirect`、`createWorker`、`dispose` |

配套类图见 [class-diagram.html](./class-diagram.html)。

## 关键流程

**取主线程调度器**：`AndroidSchedulers.mainThread()` 并不直接返回静态字段，而是走 `RxAndroidPlugins.onMainThreadScheduler(MAIN_THREAD)`；`MAIN_THREAD` 在类初始化时由 `RxAndroidPlugins.initMainThreadScheduler(() -> MainHolder.DEFAULT)` 得到，`MainHolder.DEFAULT` 再调用 `internalFrom(Looper.getMainLooper(), true)`，最终 `new HandlerScheduler(new Handler(looper), async)`（`AndroidSchedulers` L27–43、L63–86）。`internalFrom()` 里对 API < 16 强制 `async=false`，API 16–21 先探测 `Message.setAsynchronous()` 是否可用，失败自动降级。

**任务投递**：`HandlerScheduler.scheduleDirect()` 先用 `RxJavaPlugins.onSchedule()` 包装任务，再新建 `ScheduledRunnable`，用 `Message.obtain(handler, scheduled)` 取出消息；`async` 为 true 时 `setAsynchronous(true)`，随后 `sendMessageDelayed()` 交给 `Handler`（L35–47）。主线程 `Looper` 取出消息后执行 `ScheduledRunnable.run()`，内部 try/catch 把异常交给 `RxJavaPlugins.onError()`（L121–127）。

**Worker 与取消**：`createWorker()` 返回 `HandlerWorker`；它的 `schedule()` 在 `Message.obj` 上写入 worker 自身作为 token，再用 `sendMessageDelayed` 入队，并在入队后二次检查 `disposed`，若已取消则 `removeCallbacks`（L67–95）。`HandlerWorker.dispose()` 用 `removeCallbacksAndMessages(this)` 一次性清掉该 worker 的全部消息；`ScheduledRunnable.dispose()` 则只 `removeCallbacks(this)` 精确取消单条任务。

**线程切换位置**：`observeOn(AndroidSchedulers.mainThread())` 把下游回调切到主线程，`subscribeOn` 只影响订阅发生的线程；RxAndroid 仅提供被 `observeOn` 使用的调度器，不改变 RxJava 的上游执行模型。

**主线程释放**：`MainThreadDisposable.dispose()` 用 `AtomicBoolean.compareAndSet` 防重入；调用线程已是主线程就直接执行 `onDispose()`，否则 `AndroidSchedulers.mainThread().scheduleDirect(this::onDispose)` 把释放动作投回主线程（`MainThreadDisposable` L70–78）。

流程图见 [flow-diagram.html](./flow-diagram.html)，覆盖取调度器 → 创建 Worker → 消息入队 → Looper 分发 → 回调与释放。

## 设计要点

- **极薄绑定层**：主库只有 4 个类，`Scheduler`/`Worker`/`Disposable` 全部复用 RxJava 3，RxAndroid 只补 Android 的线程模型。
- **钩子可测试**：`RxAndroidPlugins` 用 `volatile Function` 保存两个钩子，测试时可把主线程调度器替换成同步实现，`mainThread()` 每次调用都经过钩子。
- **async 消息**：API 16+ 使用 `Message.setAsynchronous(true)`，绕开 VSYNC 同步屏障，避免主线程消息被延迟；低版本自动退回同步消息。
- **按 token 批量取消**：`HandlerWorker` 把自身塞进 `Message.obj`，`removeCallbacksAndMessages(token)` 一次取消整批任务，无需额外集合。
- **异常不吞**：`ScheduledRunnable.run()` 捕获 `Throwable` 后交给 `RxJavaPlugins.onError()`，与 RxJava 全局错误处理保持一致。

## 阅读路径

1. `rxandroid/src/main/java/io/reactivex/rxjava3/android/schedulers/AndroidSchedulers.java` —— 调度器工厂与 API 兼容处理
2. `schedulers/HandlerScheduler.java` —— `Handler` 投递、Worker、取消句柄
3. `android/MainThreadDisposable.java` —— 释放回调切主线程
4. `android/plugins/RxAndroidPlugins.java` —— 钩子机制与 `reset()`
5. `rxandroid/src/test/java/.../schedulers/HandlerSchedulerTest.java` —— 用测试固定行为边界

## 版本注记

3.x 绑定 RxJava 3，包名统一为 `io.reactivex.rxjava3.android`；`VERSION_NAME` 仍是 `3.1.0-SNAPSHOT`，README 中固化的发布版为 3.0.2。该 commit 只改 CI 依赖，源码以上述行号为准。
