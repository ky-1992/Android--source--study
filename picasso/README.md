# Picasso 源码解析

> 排行榜 #10 · 类别：图片

## 一句话定位

Picasso 是 Square 出品的图片加载库：`Picasso.with(context).load(url).into(imageView)` 一行完成加载。核心是一条**单线程 Dispatcher 消息循环 + 多线程 BitmapHunter 执行 + 内存 LRU 缓存**的流水线，Action 负责把结果回写到 Target，并自动处理 View 复用（tag 机制取消旧请求）。

## 源码快照

| 项 | 值 |
|---|---|
| 上游仓库 | https://github.com/square/picasso |
| 快照 commit | `e94d6116e3fff99b3932103e0ab22ff5b44a1273` |
| 克隆方式 | `--depth 1` |

结构：`picasso/`（核心：Picasso、RequestCreator、Dispatcher、BitmapHunter、Action）、`picasso-pollexor`、`picasso-provider`、`sample/`。

## 核心类

| 类 | 职责 | 关键位置 |
|---|---|---|
| `Picasso` | 单例门面：持有缓存、Dispatcher、线程池；`load()`、`cancelRequest()` | `Picasso.kt` L61、L272、L416、L432 |
| `RequestCreator` | 链式请求构造器：占位图、resize、transform、priority | `RequestCreator.kt` L47、L583、L651 |
| `Request` | 不可变请求描述：URI、资源 id、尺寸、变换、缓存策略 | `Request.kt` L31、L561 |
| `Dispatcher` | 调度接口：`dispatchSubmit` / `dispatchComplete` / `performCancel` | `Dispatcher.kt` L18 |
| `BaseDispatcher` | 调度基类：维护 hunter 映射、按 key 合并、顺序与并发控制 | `BaseDispatcher.kt` L52、L89 |
| `HandlerDispatcher` | 主线程消息循环：把提交/完成/重试/取消转成 Handler 消息 | `HandlerDispatcher.kt` L29、L57 |
| `BitmapHunter` | 单次取图任务：缓存查找 → 网络/磁盘 → 解码 → 变换 | `BitmapHunter.kt` L39、L69 |
| `Action` | 请求与目标（Target/ImageView）的绑定，负责回调与取消 | `Action.kt` L20 |
| `ImageViewAction` | ImageView 专用 Action：处理占位、淡入、错误图 | `ImageViewAction.kt` L25、L36 |
| `RequestHandler` | 取图策略抽象（网络、资源、联系人、ContentProvider…） | `RequestHandler.kt` L38 |
| `PlatformLruCache` | 内存 LRU 缓存（按字节数计算） | `PlatformLruCache.kt` L24、L51 |
| `PicassoExecutorService` | BitmapHunter 使用的线程池（按网络状态调整并发） | `PicassoExecutorService.kt` L30 |

## 关键流程

### 1. 发起加载并绑定目标

`load(uri)` 返回 `RequestCreator`；`into(imageView)` 时构造 `Request`，再拿到 `ImageViewAction`。Action 会把自身 set 到 `ImageView` 的 tag 上，这样同一个 View 的新请求能取消旧请求（避免错图/闪烁）。

### 2. 查内存缓存

先查 `PlatformLruCache`，命中则直接把 Bitmap 交付；未命中才提交给 Dispatcher。

### 3. 消息化调度

`Dispatcher.dispatchSubmit` 把请求投递到主线程 Handler；`HandlerDispatcher` 处理 `REQUEST_SUBMIT` 消息，在 `BaseDispatcher` 里按 key 合并相同请求（同一 URL 只跑一个 hunter，其他等待者挂到 hunter 上）。

### 4. 多线程取图

`PicassoExecutorService` 执行 `BitmapHunter.hunt()`：按顺序尝试 `RequestHandler`（网络/磁盘/资源…），拿到底层数据后解码、应用 transformation、写入内存缓存。

### 5. 回调与展示

hunter 完成 → `dispatchComplete` 回到主线程 → Action 把结果交给 Target（`ImageViewAction` 负责占位图、淡入动画、错误占位）。Activity 销毁或 View 回收时通过 `cancelRequest` 清理。

## 设计要点

1. **Handler 消息循环做调度**：Dispatcher 单线程串行化请求管理，避免并发数据结构竞争；实际耗时操作在 hunter 线程池。
2. **请求合并**：同一 key 的重复请求挂在同一个 BitmapHunter 上，回调分发时统一通知，节省流量。
3. **View tag 取消机制**：Picasso 在 ImageView 上打 tag 记录当前 Action，新请求自动取消旧请求，这是列表复用的经典解法。
4. **内存缓存按字节 LRU**：`PlatformLruCache` 以 bitmap 字节数计价，比按条目数更贴近真实内存压力。
5. **变换链**：`Transformation` 在 hunter 内执行，结果可被缓存（带变换的 key）。

## 阅读路径

1. `picasso/src/main/java/com/squareup/picasso/Picasso.kt`（`load` / `enqueue` / `cancelRequest`）
2. `.../RequestCreator.kt`（`into` / `buildRequest`）
3. `.../Dispatcher.kt`、`BaseDispatcher.kt`、`HandlerDispatcher.kt`
4. `.../BitmapHunter.kt`（`hunt` / `run`）
5. `.../RequestHandler.kt` 与各实现
6. `.../Action.kt`、`ImageViewAction.kt`
7. `.../PlatformLruCache.kt`

## 版本注记

- 该 commit 已把核心从 Java 迁到 Kotlin（`Picasso.kt`、`BitmapHunter.kt` 等），类名与职责与 Java 时代一致。
- 排行榜成文时（2017）是 2.5.x；Picasso 之后进入低维护状态，但“Dispatcher + Hunter + Action”仍是学习图片库调度的经典样本。
