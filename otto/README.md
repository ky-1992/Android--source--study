# Otto 源码解析

## 一句话定位

Otto 是 Square 基于 Guava EventBus 改造的进程内事件总线：方法用 `@Subscribe` / `@Produce` 注解标记，`Bus` 负责注册、按事件类型匹配并反射调用。

## 源码快照

- 上游：<https://github.com/square/otto>
- commit：`2a67589abd91879cf23a8c96542b59349485ec3d`（2018-04-24，仓库最后一次提交）
- 包结构：核心库在 `otto/src/main/java/com/squareup/otto/`，共 9 个主类；`otto-sample/` 是示例，`website/` 是文档。`pom.xml` 版本为 `1.3.9-SNAPSHOT`，`CHANGELOG.md` 里最新的正式版本是 1.3.8（2015-06-17）。

## 核心类

| 类 | 职责 | 关键方法（行号） |
|---|---|---|
| `Bus` | 注册表与分发门面 | `register`(Bus.java:184)、`post`(:313)、`flattenHierarchy`(:421) |
| `HandlerFinder` | 查找器接口，`ANNOTATED` 常量委托给反射实现 | HandlerFinder.java:23-40 |
| `AnnotatedHandlerFinder` | 反射解析注解方法并做两级缓存 | `loadAnnotatedMethods`(:62)、`findAllSubscribers`(:153) |
| `EventHandler` | 包装单个 `@Subscribe` 方法 | `handleEvent`(:84)、`invalidate`(:72) |
| `EventProducer` | 包装单个 `@Produce` 方法 | `produceEvent`(:79) |
| `ThreadEnforcer` | 线程约束策略，内置 `ANY` / `MAIN` | `enforce`(:33) |
| `DeadEvent` | 无人订阅事件的重投包装 | DeadEvent.java:27 |

## 关键流程

### 1. 注册

`register` 先调 `enforcer.enforce`，再用 `handlerFinder.findAllProducers` 找生产者并 `putIfAbsent` 到 `producersByType`（同类型重复注册直接抛 `IllegalArgumentException`）；若该类型已有订阅者，立即 `dispatchProducerResultToHandler` 补投。随后 `findAllSubscribers` 把处理器 `addAll` 进 `CopyOnWriteArraySet`，并让新处理器补收已有生产者的结果。

### 2. 发布与类型展开

`post` 先 `flattenHierarchy(event.getClass())`，结果缓存于 `flattenHierarchyCache`；对每个类型取出 `handlersByType` 里的集合，把事件与处理器包成 `EventWithHandler` 放进 `ThreadLocal` 队列。若一个处理器都没匹配到且事件本身不是 `DeadEvent`，就把它包成 `DeadEvent` 重新 `post`。

### 3. 出队与分发

`dispatchQueuedEvents` 用 `ThreadLocal<Boolean> isDispatching` 防重入：分发过程中再次 `post` 的事件只入队，等当前分发结束后按 FIFO 统一执行。`dispatch` 最终走到 `EventHandler.handleEvent` → `Method.invoke`，处理器抛出的异常被包装成 `RuntimeException` 抛出。生产者返回 `null` 时会直接跳过投递，因此生产者可以安全地表达「暂时没有可用值」。

## 设计要点

1. 单线程模型：默认 `ThreadEnforcer.MAIN` 用 `Looper.myLooper()` 校验，强制主线程访问总线，规避 Android 回调线程切换问题。
2. 线程本地队列：`eventsToDispatch` 与 `isDispatching` 都是 `ThreadLocal`，多线程天然隔离且保证同线程内事件顺序。
3. fail fast：参数个数不为 1、事件类型是接口、方法非 `public`，在 `AnnotatedHandlerFinder.loadAnnotatedMethods` 里都直接抛 `IllegalArgumentException`。
4. 生产者与订阅者双向补齐：先注册的一方不会丢事件，后注册的一方立刻收到补投。
5. 缓存友好：`PRODUCERS_CACHE` / `SUBSCRIBERS_CACHE` 按类缓存方法集合，避免每次注册都反射。
6. 去重与失效：`EventHandler.equals` 同时比较 `method` 与 `target` 引用，配合 `invalidate` 让注销后的处理器静默跳过已经入队的事件，不会出现「注销后还被回调」的意外。
7. 与 Javadoc 的偏差：`Bus.getClassesFor` 实际只沿 `getSuperclass` 上溯，注释声称会展开接口，本 commit 并未实现接口展开。

## 阅读路径

1. `otto/src/main/java/com/squareup/otto/Bus.java`
2. `otto/src/main/java/com/squareup/otto/HandlerFinder.java`
3. `otto/src/main/java/com/squareup/otto/AnnotatedHandlerFinder.java`
4. `otto/src/main/java/com/squareup/otto/EventHandler.java`
5. `otto/src/main/java/com/squareup/otto/EventProducer.java`
6. `otto/src/main/java/com/squareup/otto/ThreadEnforcer.java`
7. `otto/src/main/java/com/squareup/otto/DeadEvent.java`
8. `otto-sample/src/main/java/com/squareup/otto/sample/BusProvider.java`

## 版本注记

仓库 README 顶部已标 `Deprecated!`，官方说法是迁移到 RxJava / RxAndroid；这也说明它的定位是模板式总线，在 2018 年最后一次提交后不再演进。在实际工程里，Otto 已被 EventBus 这类注解式事件总线取代：EventBus 提供 `ThreadMode` 多线程投递、sticky 事件与注解索引，能力覆盖了 Otto 的 `@Subscribe` / `@Produce` 模型，因此新项目不再选 Otto。以上结构、行号与版本号均以本 commit 为准。
