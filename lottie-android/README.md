# Lottie 源码解析

> 排行榜 #11 · 类别：动画（排名第一的动画框架）

## 一句话定位

Lottie 把 Adobe After Effects 导出的 **Bodymovin JSON** 解析成一棵图层树（`LottieComposition`），再由 `LottieDrawable` 在 Android Canvas 上逐帧绘制；`LottieValueAnimator` 只负责推进时间轴，动画属性由各 `BaseLayer` 按关键帧插值——不需要视频、不需要预渲染帧序列。

## 源码快照

| 项 | 值 |
|---|---|
| 上游仓库 | https://github.com/airbnb/lottie-android |
| 快照 commit | `05ea92e90381eb8a8ae06855ea2b74f322bebbec` |
| 克隆方式 | `--depth 1` |

模块：`lottie/`（核心）、`lottie-compose/`（Compose 封装）、`snapshot-tests/`（渲染快照测试）、`sample/`。

## 核心类

| 类 | 职责 | 关键位置 |
|---|---|---|
| `LottieCompositionFactory` | 从 raw 资源 / json / 网络加载并缓存 composition | `LottieCompositionFactory.java` L61、L434、L512 |
| `LottieComposition` | 动画数据模型：帧率、起止帧、图层列表、字体/图片资源表 | `LottieComposition.java` L43、L78、L152 |
| `LottieCompositionMoshiParser` | JSON → Composition 的解析器（Moshi 流式解析） | `LottieCompositionMoshiParser.java` L25、L40 |
| `LottieResult` | 解析结果封装（成功值 / 异常），配合缓存 | `LottieResult.java` L11 |
| `LottieAnimationView` | 视图门面：`setAnimation()`、播放控制、进度监听 | `LottieAnimationView.java` L65、L660、L486 |
| `LottieDrawable` | 可绘制对象：持有 composition、绘制到 Canvas、管理 animator 与缓存位图 | `LottieDrawable.java` L73、L423、L731 |
| `LottieValueAnimator` | 时间轴：每帧回调进度（0~1），驱动重绘 | `LottieValueAnimator.java` L18、L87、L177 |
| `CompositionLayer` | 根图层：遍历子图层并按变换/遮罩绘制 | `CompositionLayer.java` L27、L114、L184 |
| `BaseLayer` | 图层基类：属性动画求值、绘制分发 | `BaseLayer.java` L45、L235、L643 |
| `KeyPath` / `KeyPathElement` | 按图层路径定位属性（动态改颜色、进度等） | `KeyPath.java` L46、L87；`KeyPathElement.java` L16 |

## 关键流程

### 1. 加载与解析

`LottieAnimationView.setAnimation(...)` 把 raw/json 交给 `LottieCompositionFactory`。工厂按 key 查缓存（`LottieResult`），未命中则用 `LottieCompositionMoshiParser` 流式解析：读取 `v/fr/ip/op/w/h/layers` 等字段，递归构建图层树，同时收集图片与字体资源。

### 2. 交给 Drawable

解析成功后 `LottieDrawable.setComposition(composition)`：设置 bounds、帧率、缓存策略，并把 composition 的图层树实例化（`CompositionLayer` 持有所有子图层）。`LottieAnimationView` 本质上是把 Drawable 设为背景并转发播放控制。

### 3. 时间轴驱动

`LottieValueAnimator` 是一个 `ValueAnimator` 子类：系统每帧回调进度（按帧率换算成 composition 的帧号），`LottieDrawable` 收到进度后 invalidate，下一帧 `draw()` 用当前帧号求值。

### 4. 逐层绘制

`CompositionLayer.draw()` 遍历图层；每个 `BaseLayer` 取自身属性的关键帧插值（位置、缩放、旋转、透明度、颜色、遮罩），用 `Canvas` 的 save/restore 管理变换栈，最后绘制形状或图片。`KeyPath` 允许业务层精确取到某个图层/属性做动态修改。

## 设计要点

1. **数据与渲染分离**：`LottieComposition` 是纯数据（可序列化、可缓存、可跨 View 共享），`LottieDrawable` 才持有渲染状态。
2. **属性动画通用化**：所有图层属性都抽象成 `BaseKeyframeAnimation`，关键帧插值、缓动曲线统一处理。
3. **Canvas 绘制而非视频**：任意分辨率清晰，支持动态改颜色/进度，文件体积小。
4. **缓存与资源管理**：composition 按 key 缓存；绘制过程中的 Bitmap 有 `LottieDrawable` 的 bitmap 缓存策略。
5. **可组合 API**：`LottieAnimationView`（View）、`LottieDrawable`（Drawable）、Compose 版本共用同一套解析与渲染核心。

## 阅读路径

1. `lottie/src/main/java/com/airbnb/lottie/LottieCompositionFactory.java`
2. `.../parser/LottieCompositionMoshiParser.java`
3. `.../LottieComposition.java`
4. `.../LottieDrawable.java`（`setComposition` / `draw` / `drawLayer`）
5. `.../utils/LottieValueAnimator.java`
6. `.../model/layer/CompositionLayer.java`、`BaseLayer.java`
7. `.../model/KeyPath.java`

## 版本注记

- 该 commit 为当前 master（6.x 线），核心仍是 Java（`LottieCompositionFactory.java` 等），另提供 `lottie-compose` 模块。
- 排行榜成文时（2017）Lottie 2.x 刚发布；类名（`LottieComposition`、`LottieDrawable`、`LottieAnimationView`）自 2.x 起保持稳定，本目录图表以该 commit 为准。
