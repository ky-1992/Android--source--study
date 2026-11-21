# EventBus 源码解析

## 一句话定位

EventBus 是 greenrobot 出品的 Android / Java 发布-订阅事件总线：用 `@Subscribe` 注解声明回调方法，`post` 时按事件类型查询订阅表，再依据 `ThreadMode` 把回调切到目标线程执行。核心实现是纯 Java（不依赖 Android），总代码量约 2000 行，适合作为"注解 + 反射 + 线程调度"的入门读本。

## 源码快照

- 上游：<https://github.com/greenrobot/EventBus>
- 分支：`master`（浅克隆，`origin` 指向该地址）
- commit：`0194926b3bcf70cc0d7bfd3c5da16708dd5ab876`（2022-07-11，CI 配置提交）
- 版本：根 `build.gradle` 声明 `version = "3.3.1"`；浅克隆无 tag，精确版本以该 commit 为准

目录结构概览：

| 路径 | 内容 |
| --- | --- |
| `EventBus/src/org/greenrobot/eventbus/` | 纯 Java 核心：`EventBus`、`EventBusBuilder`、订阅表、`Poster`、`SubscriberMethodFinder`、`meta/` 索引接口 |
| `eventbus-android/src/main/java/org/greenrobot/eventbus/` | Android 附加模块：`HandlerPoster`、`AndroidLogger`、`DefaultAndroidMainThreadSupport`、`AndroidComponentsImpl` |
| `EventBusAnnotationProcessor/` | 注解处理器，编译期生成 `SubscriberInfoIndex` 以替代运行时反射 |
| `EventBusTest*`、`EventBusPerformance/` | 测试与性能对照（含 index 版测试） |
| `EventBus/src/org/greenrobot/eventbus/util/` | `AsyncExecutor`、`ThrowableFailureEvent` 等可选用法封装 |

## 核心类

| 类名 | 职责 | 关键方法（行号取自快照） |
| --- | --- | --- |
| `EventBus` | 门面：注册/注销、`post`、sticky、按 ThreadMode 分发 | `getDefault()` L82、`register()` L142、`subscribe()` L159、`post()` L262、`postToSubscription()` L437、`invokeSubscriber()` L506 |
| `EventBusBuilder` | 构造配置：线程池、索引、日志开关、事件继承 | `build()` L189、`installDefaultEventBus()` L177、`addIndex()` L135、`getMainThreadSupport()` L161 |
| `SubscriberMethodFinder` | 反射扫描 `@Subscribe` 方法，带方法缓存 | `findSubscriberMethods()` L55、`findUsingInfo()` L75、`findUsingReflectionInSingleClass()` L150 |
| `SubscriberMethod` | 单个订阅方法的元数据（Method/ThreadMode/优先级） | `equals()` L39、`hashCode()` L65、`checkMethodString()` L53 |
| `Subscription` | 订阅者对象与方法的绑定，`active` 控制是否还投递 | 构造 L27、`equals()`/`hashCode()` L34、L45 |
| `ThreadMode` | 五种线程模式枚举 | `POSTING`、`MAIN`、`MAIN_ORDERED`、`BACKGROUND`、`ASYNC`（L31–L67） |
| `Poster` | 按订阅投递事件的接口 | `enqueue(Subscription, Object)` L31 |
| `HandlerPoster` | Android 主线程投递，基于 `Handler` | `enqueue()` L37、`handleMessage()` L51 |
| `BackgroundPoster` | 单线程串行后台投递，空闲 1s 退出 | `enqueue()` L37、`run()` L49 |
| `AsyncPoster` | 每个事件交给线程池（不复用顺序） | `enqueue()` L34、`run()` L41 |
| `PendingPost` | 池化的投递节点，兼作链表节点 | `obtainPendingPost()` L33、`releasePendingPost()` L47 |
| `PendingPostQueue` | `synchronized` 的 head/tail 链式队列 | `enqueue()` L23、`poll()` L38、`poll(int)` L49 |

配套的类图见 [class-diagram.html](./class-diagram.html)，图中省略了 `PendingPostQueue`、`AsyncPoster` 与 `meta/` 系列接口，只保留主链路。

## 关键流程

注册：`register()` 先做 Android 依赖自检（L143–147），再调用 `SubscriberMethodFinder.findSubscriberMethods()`（L150）；拿到方法列表后在 `synchronized (this)` 内逐个 `subscribe()`（L151–155）。`subscribe()` 把 `Subscription` 按 `priority` 从高到低插入 `CopyOnWriteArrayList`（L173–179），登记 `typesBySubscriber`，若方法标注 `sticky` 则补投最近一次 sticky 事件（L188–206）。

post：`post()` 通过 ThreadLocal 的 `PostingThreadState` 维护事件队列并做重入保护（L262–282），随后 `postSingleEvent()` 按 `eventInheritance` 展开父类/接口（L386–408，`lookupAllEventTypes` L473），`postSingleEventForEventType()` 遍历订阅副本（L410–435），最终进入 `postToSubscription()`（L437–470）按 ThreadMode 分支：`POSTING` 直接反射调用；`MAIN` 在主线程直调、否则交 `HandlerPoster`；`MAIN_ORDERED` 总是入队；`BACKGROUND` 在主线程时交 `BackgroundPoster`，否则直调；`ASYNC` 一律交 `AsyncPoster`。

分发过程中若某订阅者调用 `cancelEventDelivery()`，只允许在 `POSTING` 模式下取消当前事件（L291–305），`postSingleEventForEventType()` 会据 `postingState.canceled` 跳出后续订阅；没有任何订阅者时，默认会再 post 一个 `NoSubscriberEvent`（L399–407）。

索引替代反射：`findSubscriberMethods()` 在配置了 `SubscriberInfoIndex` 时走 `findUsingInfo()`（L75–93），由注解处理器在编译期生成的索引直接给出方法元数据，只有索引缺失的类才回退到反射扫描。

注销：`unregister()` 用 `typesBySubscriber` 反查事件类型（L249–259），`unsubscribeByEventType()` 先把 `Subscription.active` 置 false 再移除（L232–246）；已入队的事件在 `invokeSubscriber(PendingPost)` 处检查 `active`（L506–513）后丢弃，避免注销后回调。

流程图见 [flow-diagram.html](./flow-diagram.html)，覆盖初始化 → 注册 → post → ThreadMode 分发 → 注销。

## 设计要点

- **PendingPost 池化 + 链表复用**：`obtainPendingPost()` 从静态池尾部取节点（L33–45），`releasePendingPost()` 清空字段后归池且上限 10000（L47–57）；`next` 字段让节点直接充当 `PendingPostQueue` 的链表单元，投递高峰期几乎零分配。
- **反射 + 方法缓存**：`METHOD_CACHE` 以订阅者 `Class` 为键缓存方法列表（L39、L55–73），`FindState` 另有 `POOL_SIZE = 4` 的对象池（L45–46、L95–120）；`SubscriberMethod.methodString` 惰性拼接后参与 `equals`（L53–62），绕开 `Method.equals` 的已知问题。
- **订阅表读多写少**：`subscriptionsByEventType` 是 `Map<Class, CopyOnWriteArrayList<Subscription>>`，投递遍历不加锁，写操作集中在 `synchronized (this)`。
- **Poster 线程模型**：`EventBus` 构造时统一创建三种 Poster（L113–132）；主线程 Poster 由 `MainThreadSupport` 注入，非 Android 环境为 null，此时 `MAIN`/`MAIN_ORDERED` 退回发布线程直调（L118–119、L442–456）；`BackgroundPoster` 与 `AsyncPoster` 共用 builder 的 cached 线程池（`EventBusBuilder` L31）。
- **Android 零硬依赖**：`AndroidDependenciesDetector`（L9–47）用反射探测 `android.os.Looper` 与 `AndroidComponentsImpl`，核心 jar 可直接跑在 JVM 上。
- **eventInheritance 与 sticky**：事件类型继承默认开启并缓存查找结果（L49、L473–488）；`postSticky()` 先入 sticky 表再 post（L311–317）。

## 阅读路径

1. `EventBus/src/org/greenrobot/eventbus/EventBus.java` —— 先读 L41–132 字段与构造，再读 L142–207、L262–282、L386–523。
2. `EventBus/src/org/greenrobot/eventbus/ThreadMode.java` —— 理解五种线程语义（L24–67）。
3. `EventBus/src/org/greenrobot/eventbus/Subscribe.java` —— 注解的 `threadMode`/`sticky`/`priority` 三个属性（L29–43）。
4. `EventBus/src/org/greenrobot/eventbus/SubscriberMethodFinder.java` —— 反射扫描、`METHOD_CACHE` 与 `FindState`（L29–195）。
5. `EventBus/src/org/greenrobot/eventbus/Subscription.java`、`PendingPost.java`、`PendingPostQueue.java` —— 绑定关系、池化与队列。
6. `eventbus-android/src/main/java/org/greenrobot/eventbus/HandlerPoster.java` —— 主线程投递与 10ms 让出策略（`DefaultAndroidMainThreadSupport` L18）。
7. `EventBus/src/org/greenrobot/eventbus/BackgroundPoster.java`、`AsyncPoster.java` —— 线程池侧投递。
8. `EventBus/src/org/greenrobot/eventbus/EventBusBuilder.java` 与 `meta/SubscriberInfoIndex.java` —— 配置项与编译期索引接口。

## 版本注记

该 commit 位于 `master` 分支的 EventBus 3.x 代码线上，根 `build.gradle` 声明 `version = "3.3.1"`；此时项目已拆分为纯 Java 的 `EventBus` 模块与 Android 附加的 `eventbus-android` 模块，并用 `AndroidDependenciesDetector` 反射适配 Android 依赖。本次为 `--depth 1` 浅克隆，未取到 tag，若需精确对应发布版本，以 commit `0194926b3bcf70cc0d7bfd3c5da16708dd5ab876` 为准。
