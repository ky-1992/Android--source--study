# Universal Image Loader 源码解析

## 一句话定位

UIL 是 2011–2015 年流行的 Android 图片加载库：以 `ImageLoader` 单例为入口，`displayImage()` 先查内存缓存，未命中就把 `LoadAndDisplayImageTask` 投入线程池；任务内按磁盘缓存 → 网络下载 → 解码的顺序取图，最后用 `DisplayBitmapTask` 回主线程写入 `ImageAware`。全部为 Java，主库约 50 个类。

## 源码快照

- 上游：<https://github.com/nostra13/Android-Universal-Image-Loader>
- commit：`ba33ec64d0daaa881d35852460e78c58d086bc18`（2024-05-17，仅 README 更新）
- 版本：`library/build.gradle` 中 `versionName "1.9.5"`，`minSdkVersion 16`
- 仓库由 `library/`（主库）与 `sample/`（示例 App）两个模块组成，包名 `com.nostra13.universalimageloader`

| 路径 | 内容 |
| --- | --- |
| `library/.../core/` | `ImageLoader`、`ImageLoaderConfiguration`、`ImageLoaderEngine`、`LoadAndDisplayImageTask`、`DisplayBitmapTask`、`ImageLoadingInfo`、`DefaultConfigurationFactory` |
| `core/imageaware/` | `ImageAware` 及 `ViewAware`、`ImageViewAware`、`NonViewAware` 实现 |
| `core/decode/`、`core/download/`、`core/display/` | 解码器、下载器、结果展示器三组可替换接口及默认实现 |
| `cache/memory/`、`cache/disc/` | 内存缓存与磁盘缓存接口、抽象基类和各种 LRU/FIFO 实现 |

## 核心类

| 类名 | 职责 | 关键方法（行号） |
| --- | --- | --- |
| `ImageLoader` | 单例门面，校验初始化、组装任务并对外暴露 `displayImage` 重载 | `getInstance()` L70、`init()` L92、`displayImage()` L234、`defineHandler()` L800 |
| `ImageLoaderConfiguration` | 不可变全局配置，`Builder.build()` 时才补默认线程池/缓存/解码器 | `Builder.build()` L558、`initEmptyFieldsWithDefaultValues()` L563 |
| `ImageLoaderEngine` | 持有线程池与 URI 锁，按图片是否已落盘选择执行器 | `submit()` L70、`prepareDisplayTaskFor()` L125、`cancelDisplayTaskFor()` L135 |
| `LoadAndDisplayImageTask` | 一次加载的完整流水线（锁、缓存、下载、解码、前后处理） | `run()` L119、`tryLoadBitmap()` L214、`decodeImage()` L261、`tryCacheImageOnDisk()` L269 |
| `DisplayBitmapTask` | 主线程写图前校验视图是否被回收/复用 | `run()` L61、`isViewWasReused()` L77 |
| `ImageAware` | 取图目标抽象：尺寸、缩放类型、GC 状态、写图 | `getWidth()` L42、`isCollected()` L74、`setImageBitmap()` L113 |
| `MemoryCache` / `BaseMemoryCache` | 内存缓存契约；基类用 `Reference<Bitmap>` 存软引用并留 `createReference()` 扩展点 | `put()` L35、`get()` L38、`createReference()` L70 |
| `DiskCache` / `BaseDiskCache` | 磁盘缓存契约；基类负责文件名生成、压缩格式与父目录创建 | `get()` L45、`save()` L59、`getFile()` L166 |
| `ImageDecoder` / `BaseImageDecoder` | 解码契约；默认实现按目标尺寸采样、再做 EXIF 旋转与精确缩放 | `decode()` L70、`defineImageSizeAndRotation()` L101、`prepareDecodingOptions()` L155 |
| `DefaultConfigurationFactory` | 统一创建线程池、缓存、下载器、解码器与展示器 | `createDiskCache()` L83、`createMemoryCache()` L114、`createImageDecoder()` L146 |

配套类图见 [class-diagram.html](./class-diagram.html)。

## 关键流程

**入口判定**：`displayImage()` 先用 `ImageSizeUtils.defineTargetSizeForView()` 算出目标尺寸，再以 URI + 尺寸生成内存缓存 key，并 `engine.prepareDisplayTaskFor()` 登记“这个 ImageAware 当前在加载哪个 key”。内存命中且未被回收时直接 `displayer.display()` 返回；若配置了后处理则改走 `ProcessAndDisplayImageTask`（L267–284）。

**投递任务**：未命中时构造 `ImageLoadingInfo` 与 `LoadAndDisplayImageTask`，交给 `engine.submit()`。`submit()` 先在一个分发线程里查磁盘是否已有该图，已落盘或本地 URI 走缓存专用线程池，否则走普通池（`ImageLoaderEngine` L70–85），避免网络任务挤占缓存命中任务。

**任务内取图**：`run()` 先 `waitIfPaused()`/`delayIfNeed()`，再在 URI 锁内二次查内存缓存；仍未命中才 `tryLoadBitmap()`：读磁盘缓存并解码，失败则 `tryCacheImageOnDisk()` 下载落盘，再 `decodeImage()` 交给 `ImageDecoder` 解码，最后按配置做前/后处理并写回内存缓存（L119–177、L214–266）。

**显示与取消**：结果封装成 `DisplayBitmapTask`，由 `Handler` 回主线程；`run()` 先判 `imageAware.isCollected()` 与缓存 key 是否已被复用，任一成立就回调 `onLoadingCancelled`，否则 `displayer.display()` 后回调 `onLoadingComplete`。

流程图见 [flow-diagram.html](./flow-diagram.html)，覆盖入口 → 投递 → 磁盘/网络/解码 → 显示。

## 设计要点

- **接口 + 工厂**：缓存、下载、解码、展示全部面向接口编程，`DefaultConfigurationFactory` 集中给出默认实现，用户可在 Builder 中逐项替换。
- **双线程池**：`taskExecutor` 与 `taskExecutorForCachedImages` 分离，磁盘命中的图不会被下载任务饿死；队列类型支持 FIFO/LIFO。
- **URI 锁去重**：`getLockForUri()` 用 `WeakHashMap` 缓存 `ReentrantLock`，同一 URL 的多个任务串行取图，后续任务直接吃第一份缓存。
- **视图复用防护**：`cacheKeysForImageAwares` 记录每个 ImageAware 的最新 key，旧任务显示前比对，避免列表滑动时图片错位。
- **配置不可变 + Builder**：`ImageLoaderConfiguration` 字段全部 package 可见且构建后不变，保证多线程读取安全；不填的项在 `build()` 里补默认值。

## 阅读路径

1. `library/src/main/java/com/nostra13/universalimageloader/core/ImageLoader.java` —— 入口与分支判定
2. `core/ImageLoaderConfiguration.java` —— 配置项与 Builder 默认值
3. `core/ImageLoaderEngine.java` —— 线程池选择与锁
4. `core/LoadAndDisplayImageTask.java` —— 完整加载流水线
5. `core/DisplayBitmapTask.java` —— 主线程显示与取消判定
6. `core/imageaware/ImageAware.java` + `ImageViewAware.java` —— 视图抽象
7. `core/decode/BaseImageDecoder.java` —— 采样、缩放与 EXIF
8. `cache/memory/BaseMemoryCache.java`、`cache/disc/impl/BaseDiskCache.java` —— 两级缓存骨架

## 版本注记

该仓库 2015 年 11 月后停止维护，快照即最后的 `master`。阅读时注意：它早于 Glide/Fresco 的“生命周期感知”设计，没有 Activity 级自动取消；`ImageAware.isCollected()` 与 key 比对是当年替代方案。版本号以 `library/build.gradle` 的 1.9.5 为准，该 commit 之后无功能变更。
