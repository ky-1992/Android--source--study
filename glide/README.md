# Glide 源码解析

## 一句话定位

Glide 是 Android 上的图片加载框架：以 `Glide` 单例为入口，用 `RequestManager` 把加载请求绑定到 Activity/Fragment 生命周期，`RequestBuilder` 负责链式配置并组装请求，`Engine` + `DecodeJob` 完成「活动资源 → 内存缓存 → 磁盘缓存 → 数据源」的逐级取数与解码。

## 源码快照

- 上游仓库：<https://github.com/bumptech/glide>
- 本目录基于 commit `bdfcadf91da5fadd1be5b5c186de9dea46415019`（`gradle.properties` 中 `VERSION_NAME=5.0.10`）。
- 工程是 Gradle 多模块，`settings.gradle.kts` 里核心为 `:library`（Java 主库，包 `com.bumptech.glide`），另有 `:annotation`、`:annotation:compiler`、`:annotation:ksp` 用于生成 `GlideApp` API，`glide/` 模块是生成 API 的运行期支撑；`integration/ktx`、`integration/compose` 等是 Kotlin 侧扩展（`*Flows.kt`、`GlideImage`）。也就是说**主库仍是 Java，Kotlin 主要集中在 integration 模块**。
- 主库分包清晰：`Glide`/`RequestManager`/`RequestBuilder` 在根包，`load/engine` 放引擎与缓存，`request` 放 `SingleRequest`/`Target`，`manager` 放生命周期与 `RequestManagerRetriever`。

## 核心类

| 类 | 职责 | 关键方法 + 行号 |
|---|---|---|
| `Glide` | 单例门面，持有 `Engine`、`MemoryCache`、`BitmapPool`，负责初始化与内存回收 | `get(Context)` `Glide.java:141`；`with(Context)` `:567`；`clearMemory()` `:463`；`trimMemory(int)` `:477` |
| `RequestManagerRetriever` | 按 Context/Activity/Fragment/View 取生命周期正确的 `RequestManager` | `get(Context)` `RequestManagerRetriever.java:88`；`get(View)` `:157`；`getApplicationManager` `:62` |
| `RequestManager` | 生命周期回调、Target 与 Request 跟踪、`load()` 入口 | `load(Object)` `RequestManager.java:556`；`track(Target, Request)` `:690`；`onStart/onStop/onDestroy` `:368/:379/:393` |
| `RequestBuilder` | 链式选项、缩略图/错误 fallback，构建并提交请求 | `into(Target,...)` `RequestBuilder.java:847`；`into(ImageView)` `:910`；`buildRequest` `:1176`；`obtainRequest` `:1375` |
| `SingleRequest` | 单个请求的状态机，等尺寸、调 Engine、回调 Target | `begin()` `SingleRequest.java:217`；`onSizeReady` `:451`（`:469` 调 `engine.load`）；`onResourceReady` `:544` |
| `Engine` | 活动资源与任务调度，相同 key 的并发加载复用同一个 `EngineJob` | `load(...)` `Engine.java:170`；`loadFromMemory` `:315`；`waitForExistingOrStartNewJob` `:244`；`jobs.get` `:267` |
| `EngineJob` | 一次加载的回调集合，按是否走缓存选 Executor 执行 `DecodeJob` | `start(DecodeJob)` `EngineJob.java:127`；`addCallback` `:134` |
| `DecodeJob` | 缓存降级与解码，实现 `Runnable`/`Poolable` | `run()` `DecodeJob.java:255`；`runWrapped` `:305`；`getNextStage` `:413`；`Stage` 枚举 `:878` |
| `Target` | 结果生命周期接口（started/failed/ready/cleared） | `Target.java:31`，`onResourceReady` `:83`，`onLoadCleared` `:95` |

## 关键流程

### 1. 取 RequestManager 并 load

`Glide.with(context)`（`Glide.java:567`）先 `getRetriever()`，再交给 `RequestManagerRetriever.get`：主线程且是 `FragmentActivity`/`Fragment` 时走生命周期检索，否则落到 `getApplicationManager`（`RequestManagerRetriever.java:62`）。`RequestManager.load(model)` 直接 `asDrawable().load(model)`（`:556`），返回可继续链式调用的 `RequestBuilder`。

### 2. into：建请求、清旧请求、入队

`RequestBuilder.into(target)` 最终进入私有重载（`:854`）：先 `buildRequest` 递归组装（含 thumbnail/error 的 Coordinator），再 `requestManager.clear(target)` 取消旧请求、`target.setRequest(request)`、`requestManager.track(target, request)`。`track`（`RequestManager.java:690`）同时登记 `TargetTracker` 与 `RequestTracker.runRequest`：前者按 Target 维度收集、后者把 Request 放进 `WeakHashMap` 弱引用集合（`RequestTracker.java:28`），只有未完成且被暂停的请求才额外用 `pendingRequests` 强引用（`:33`），避免请求被提前回收。若同一请求等价且未在运行，`into` 会直接 `previous.begin()` 复用旧请求而不重建（`RequestBuilder.java:867`）。

### 3. begin → onSizeReady → Engine

`SingleRequest.begin`（`:217`）处理空 model、已完成请求复用，然后 `target.getSize(this)`；尺寸就绪后 `onSizeReady`（`:451`）调用 `engine.load(...)`（`:469`），把 model、签名、宽高、变换、缓存策略等一并传入。

### 4. Engine 的四级复用

`Engine.load`（`Engine.java:170`）先用 `EngineKeyFactory` 建复合 key，`loadFromMemory`（`:315`）先查 `ActiveResources`、再查 `MemoryCache`，命中就在锁外 `cb.onResourceReady(..., DataSource.MEMORY_CACHE, ...)` 返回。未命中才进入 `waitForExistingOrStartNewJob`（`:244`）：`jobs.get` 有同 key 任务就 `addCallback` 复用，否则新建 `EngineJob` 与 `DecodeJob`，`engineJob.start(decodeJob)` 交给线程池。

### 5. DecodeJob 逐级降级

`DecodeJob.run`（`:255`）→ `runWrapped`（`:305`）→ `getNextStage`（`:413`）：`diskCacheStrategy.decodeCachedResource()` 为真先走 `RESOURCE_CACHE`，`decodeCachedData()` 为真再走 `DATA_CACHE`，最后 `SOURCE`；只有 `onlyRetrieveFromCache` 时跳过源。取到数据后 `decodeFromRetrievedData`（`:482`）解码、`notifyEncodeAndRelease`（`:523`）先回调再写盘，内存缓存由 `EngineJob`/`Engine` 侧完成。

## 设计要点

1. **生命周期即请求边界**：`with(activity)` 拿到与宿主同生共死的 `RequestManager`，`onStop` 默认暂停、`onDestroy` 清空，无需业务手动取消。
2. **key 决定复用粒度**：`EngineKey`（`EngineKey.java:12`）把 model、签名、宽高、变换与 options 全量纳入 equals/hashCode，保证只有「完全同一份请求」才共享内存资源。
3. **锁外回调**：`Engine.load` 只在 `synchronized (this)` 内查缓存，命中后立刻出锁再回调（`:237` 注释），避免持锁调用外部代码造成死锁。
4. **磁盘策略是请求级开关**：`DiskCacheStrategy` 控制 `DecodeJob` 走哪几个 Stage，而不是「缓不缓存」一刀切，因而能按需只读缓存或只下原始数据。
5. **回调线程由 Executor 决定**：`EngineJob.addCallback(cb, callbackExecutor)` 把回调统一抛给 `Executors.mainThreadExecutor()`，`SingleRequest` 无需关心线程切换。
6. **对象池降低 GC 压力**：`EngineJob`、`DecodeJob`、`SingleRequest` 都实现 `Poolable`，一次加载结束后 `release()` 回池；Bitmap 另有 `BitmapPool` 复用，`Glide.clearMemory()` 明确先清 `memoryCache` 再清 `bitmapPool`，防止回收的 Bitmap 又被池重新持有。

## 阅读路径

1. `library/src/main/java/com/bumptech/glide/Glide.java`：先看 `get`/`with`/`clearMemory` 三个入口，建立单例与缓存的整体印象。
2. `library/src/main/java/com/bumptech/glide/manager/RequestManagerRetriever.java`：理解 `get` 重载如何把宿主映射到 `RequestManager`。
3. `library/src/main/java/com/bumptech/glide/RequestManager.java`：重点看构造时的 lifecycle/connectivity 注册，以及 `onStart`/`onStop`/`onDestroy`。
4. `library/src/main/java/com/bumptech/glide/RequestBuilder.java`：`into` 与 `buildRequestRecursive` 是请求组装的骨架。
5. `library/src/main/java/com/bumptech/glide/request/SingleRequest.java`：`begin`→`onSizeReady`→`onResourceReady` 是单个请求的完整状态迁移。
6. `library/src/main/java/com/bumptech/glide/load/engine/Engine.java`：`load`/`loadFromMemory`/`waitForExistingOrStartNewJob` 三段读懂了复用链。
7. `library/src/main/java/com/bumptech/glide/load/engine/DecodeJob.java`：`runWrapped`/`getNextStage`/`getNextGenerator` 决定降级顺序。
8. `library/src/main/java/com/bumptech/glide/load/engine/EngineJob.java`：`start` 与 `addCallback` 说明线程池与回调分发。

## 版本注记

本目录所有行号以 commit `bdfcadf91da5fadd1be5b5c186de9dea46415019`（5.0.10 开发线）为准。该版本 `RequestBuilder.into` 增加了 `experimentalIntoFront`（`:842`），`Engine` 增加了 `EnableActiveResourceReleaseRaceFix` 实验开关（`Engine.java:203`），`EngineJob` 则用 `SourceExecutor`/`AnimationExecutor` 区分源加载与动画池；这些实验性 API 在不同小版本间可能变动，引用时请以该 commit 源码为准。
