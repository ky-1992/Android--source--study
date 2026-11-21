# Fresco 源码解析

> 排行榜 #12 · 类别：图片

## 一句话定位

Fresco 是 Facebook 开源的 Android 图片库，核心是**生产者-消费者（Producer/Consumer）流水线**：一次图片请求被 `ProducerSequenceFactory` 编排成一条 Producer 链（缓存查询 → 磁盘 → 网络 → 解码），数据以 `CloseableImage` 沿链流动，`Drawee` 视图层负责占位图、渐进式展示与释放。

## 源码快照

| 项 | 值 |
|---|---|
| 上游仓库 | https://github.com/facebook/fresco |
| 快照 commit | `c35022415ffd9bebbc0f68e0f0c6281c44e26c11` |
| 克隆方式 | `--depth 1` |

关键目录：

```
imagepipeline/       # 核心：ImagePipeline、Producer/Consumer、缓存、解码
imagepipeline-base/  # 基础类型：CloseableImage、EncodedImage 等
drawee/              # 视图层：DraweeView、DraweeController、DraweeHierarchy
fresco/              # 门面 Fresco.initialize
```

## 核心类

| 类 | 职责 | 关键位置 |
|---|---|---|
| `ImagePipeline` | 管道门面：submitFetchRequest 取图、缓存查询、预热 | `ImagePipeline.kt` L54、L358、L1191 |
| `ProducerSequenceFactory` | 按请求类型编排 Producer 链（内存/磁盘/网络/解码/后处理） | `ProducerSequenceFactory.kt` L34、L185、L221 |
| `Producer` | 链式节点：`produceResults(Consumer, ProducerContext)` | `Producer.kt` L21 |
| `Consumer` | 链式消费：`onNewResult` / `onFailure` / `onProgressUpdate` | `Consumer.kt` L33 |
| `ProducerContext` | 单次请求上下文：监听器、优先级、取消状态 | `ProducerContext.kt` L26 |
| `BitmapMemoryCacheGetProducer` | 只读位图缓存命中路径（命中即结束） | `BitmapMemoryCacheGetProducer.kt` L17 |
| `BitmapMemoryCacheProducer` | 位图缓存读写（未命中则向后传递，命中回写） | `BitmapMemoryCacheProducer.java` L25、L47 |
| `DecodeProducer` | 把 `EncodedImage` 增量解码成 `CloseableImage`，支持渐进式 | `DecodeProducer.kt` L50、L74 |
| `NetworkFetchProducer` | 网络拉取原始字节流，向 Consumer 推送进度 | `NetworkFetchProducer.java` L41、L68 |
| `ImageRequest` | 不可变请求描述（URI、尺寸、缩放、缓存策略） | `ImageRequest.java` L52、L156 |
| `AbstractDraweeController` | 视图层控制器：提交请求、成功提交到 DraweeHierarchy | `AbstractDraweeController.java` L58、L503 |

## 关键流程

### 1. 构造请求

调用方通过 `ImageRequestBuilder` 构造 `ImageRequest`（URI、目标尺寸、`ResizeOptions`、缓存开关），然后 `ImagePipeline.submitFetchRequest` 或 `DraweeController` 发起取图。

### 2. 编排 Producer 链

`ProducerSequenceFactory` 根据请求特征选择序列，典型解码链：

```
BitmapMemoryCacheGetProducer → BitmapMemoryCacheProducer → DiskCacheReadProducer
  → NetworkFetchProducer（或本地文件）→ DecodeProducer → 后处理器
```

链上每一级都实现 `Producer`，通过 `Consumer` 把结果往下传；`ProducerContext` 贯穿全链，携带监听器与取消信号，所以任何一级都能提前终止或旁路。

### 3. 缓存与网络

位图缓存命中时，`BitmapMemoryCacheGetProducer` 直接回调结果，不继续请求；未命中则走磁盘缓存（`DiskCacheReadProducer`），再走 `NetworkFetchProducer` 拉流。网络层按块读取并回调 `onProgressUpdate`，支持渐进式 JPEG。

### 4. 解码与显示

`DecodeProducer` 用 `PlatformDecoder` 把 `EncodedImage` 解码成 `CloseableImage`，解码线程与 UI 线程分离。`AbstractDraweeController` 收到成功结果后提交给 `DraweeHierarchy`，由 `DraweeView` 绘制；失败/占位/重试由 Hierarchy 的分层 drawable 处理。

## 设计要点

1. **生产者-消费者链**是 Fresco 与 Glide/Picasso 最大的结构差异：每个能力（缓存、网络、解码、后处理）都是可插拔节点，链路由工厂按需拼装。
2. **CloseableImage 显式生命周期**：图片持有 native/ByteBuffer 内存，必须 close，避免 OOM；引用计数由 `CloseableReference` 管理。
3. **增量解码**：网络流未结束时就能开始解码，因此支持渐进式加载与低分辨率占位。
4. **缓存分层**：encoded memory cache（原始字节）、bitmap memory cache（解码后）、disk cache 三层，命中路径各自独立。
5. **Drawee 视图层分离**：图片状态（占位、进度、失败、重试）由 Hierarchy 统一管理，业务代码不直接操作 Bitmap。

## 阅读路径

1. `imagepipeline/src/main/java/com/facebook/imagepipeline/core/ImagePipeline.kt`
2. `imagepipeline/.../ProducerSequenceFactory.kt`
3. `imagepipeline-base/.../Producer.kt`、`Consumer.kt`、`ProducerContext.kt`
4. `imagepipeline/.../producers/BitmapMemoryCacheProducer.java`
5. `imagepipeline/.../producers/NetworkFetchProducer.java`
6. `imagepipeline/.../producers/DecodeProducer.kt`
7. `drawee/.../controller/AbstractDraweeController.java`

## 版本注记

- 该 commit 为当前 master，部分核心类已从 Java 迁到 Kotlin（`ImagePipeline.kt`、`ProducerSequenceFactory.kt`、`DecodeProducer.kt`），Java 时代的类名与职责仍然一一对应。
- 排行榜成文于 2017 年，当时主线是 1.x；本目录图表以该 commit 为准。
