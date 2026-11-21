# recyclerview-animators 源码解析

## 一句话定位

recyclerview-animators 是 wasabeef 维护的一套 RecyclerView 增删动画集合：核心是继承 `SimpleItemAnimator` 的抽象类 `BaseItemAnimator`，把「入队 → 分批延迟调度 → 调子类实现属性动画 → 派发完成」这条流水线写死，21 个具体动画类（Fade / Slide / Scale / Flip / Overshoot / Landing 系列）只需要覆写三个钩子；另有一条 Adapter 侧的 `AnimationAdapter` 路线负责入场动画。

## 源码快照

- 上游：<https://github.com/wasabeef/recyclerview-animators>
- commit：`6c5c7748e86f43d3d03f7b3122f473e1fd807681`（2021-02-14，`--depth 1` 浅克隆，该提交只改了 README）
- 版本线：`gradle.properties` 里 `VERSION_NAME=4.0.1`、`MIN_SDK_VERSION=21`、`COMPILE_SDK_VERSION=30`；`build.gradle` 用 Kotlin 1.3.72 与 AGP 4.2.0-beta04，依赖 `androidx.recyclerview:recyclerview:1.1.0`
- 模块：`animators/` 库本体（纯 Kotlin，约 1000 行）、`example/` 示例 App

| 包 | 内容 |
| --- | --- |
| `jp.wasabeef.recyclerview.animators` | `BaseItemAnimator` 与 21 个具体 animator |
| `jp.wasabeef.recyclerview.animators.holder` | `AnimateViewHolder` 扩展接口 |
| `jp.wasabeef.recyclerview.adapters` | `AnimationAdapter` 与 5 个包装 adapter |
| `jp.wasabeef.recyclerview.internal` | `ViewHelper`（属性复位） |

## 核心类

| 类名 | 职责 | 关键成员（行号取自快照） |
| --- | --- | --- |
| `BaseItemAnimator` | 排队、延迟、执行、取消的骨架 | `animateAdd()` L238、`animateRemove()` L227、`runPendingAnimations()` L91、`endAnimation()` L444、`isRunning()` L509、`endAnimations()` L527 |
| `FadeInAnimator` | alpha 淡入淡出 | `animateRemoveImpl()` L27、`preAnimateAddImpl()` L37、`animateAddImpl()` L41 |
| `SlideInLeftAnimator` | 从左侧滑入滑出 | `animateRemoveImpl()` L27、`preAnimateAddImpl()` L37 |
| `ScaleInAnimator` | 缩放出现 | `preAnimateAddImpl()` L37、`animateAddImpl()` L41 |
| `AnimateViewHolder` | 单个 holder 自定义增删动画 | 四个方法 L7–L13 |
| `ViewHelper` | 把 alpha/scale/translation/rotation 一次清干净 | `clear()` L22 |
| `AnimationAdapter` | 包装原 Adapter，在 bind 时播入场动画 | `onBindViewHolder()` L77、`getAnimators()` L112、`setFirstOnly()` L113 |
| `AlphaInAnimationAdapter` | 只给 alpha 的包装 | `getAnimators()` L28 |

## 关键流程

入队：RecyclerView 在 `notifyItemInserted` 之后调用 `animateAdd(holder)`。`BaseItemAnimator` 先 `endAnimation(holder)` 打断该 holder 上可能还在跑的动画，再 `preAnimateAdd`——内部先 `ViewHelper.clear(itemView)` 把残留属性归零，再调子类的 `preAnimateAddImpl` 摆好初始值（比如 Fade 把 alpha 置 0，Slide 把 translationX 置 `-rootView.width`），最后把 holder 丢进 `pendingAdditions` 并返回 true（L238–L243）。注意 `clear` 一定发生在子类钩子之前，所以 `preAnimateAddImpl` 只需关心动画起点，不必自己清残留。

调度：RecyclerView 稍后调 `runPendingAnimations()`（L91）。顺序是 remove 先跑、move 延迟 `removeDuration`、change 与 move 并行、add 延迟 `remove + max(move, change)`；每批先拷进 `movesList/additionsList/changesList` 再执行，`endAnimation` 时才能整批取消（L444–L507）。

执行：`doAnimateAdd(holder)` 判断 holder 是否实现 `AnimateViewHolder`，是则交给 holder 自己的 `animateAddImpl(holder, listener)`，否则走子类 `BaseItemAnimator.animateAddImpl`。两者最终都用 `ViewPropertyAnimator` 把属性归一，并挂上 `DefaultAddAnimatorListener` / `DefaultRemoveAnimatorListener`：`onAnimationEnd` 里 `clear` → `dispatchAddFinished` → 从 `addAnimations` 移除 → `dispatchFinishedWhenDone`，`isRunning()` 全空时通知 RecyclerView 一轮动画结束。

移动与变更：`animateMove` 会先把 `fromX/fromY` 叠加上当前 `translationX/Y`（防止连续调用错位），差值都为零时直接 `dispatchMoveFinished` 并返回 false；`animateChange` 在 `oldHolder === newHolder` 时退化成 move，否则把新 holder 的 alpha 置 0、位移反着预置，再让两者同帧一段动画交叉。

Adapter 侧：`AnimationAdapter` 不碰 ItemAnimator，而是包一层原 Adapter，在 `onBindViewHolder` 里把 `adapter.onBindViewHolder` 调完后再取 `getAnimators(itemView)` 逐个 `setDuration(duration).start()`；`isFirstOnly` 与 `lastPosition` 保证「只给新出现的条目播动画」，回滚到旧位置的部分走 `clear` 静默复位（L77–L89）。`setStartPosition` 可以让第一屏不播、从指定位置起播。

## 设计要点

- **四条 pending 队列 + 三个批次列表**：`pending*` 收当帧请求，`*List` 保存已开跑的批次，取消时按目标回查，避免 `endAnimation` 之后仍收到回调。
- **按位置错峰**：`getRemoveDelay = |oldPosition * removeDuration / 4|`、`getAddDelay` 用 `adapterPosition`，越靠后越晚启动，滚动列表里呈现波纹感。
- **属性复位收口**：所有 clear 都走 `ViewHelper.clear`，连 `pivotX/pivotY` 都重置，翻页复用 View 时不会串味。
- **change 默认关闭**：构造时 `supportsChangeAnimations = false`（L48–L50），`animateChange` 遇到同一 holder 还会退化成 `animateMove`（L326–L330）。
- **两种扩展粒度**：整体换 animator 继承 `BaseItemAnimator`；想让同一列表每个 item 动画不同，就让 holder 实现 `AnimateViewHolder`。
- **默认插值器** `DecelerateInterpolator`，子类构造可换成任意 `Interpolator`；`AnimationAdapter` 默认则是 `LinearInterpolator` 配 300ms，两边默认值刻意不同。
- **代理要代理全**：`AnimationAdapter` 把 `registerAdapterDataObserver`、`onAttachedToRecyclerView`、`getItemViewType`、`getItemId`、`setHasStableIds` 等一一转发给被包裹的 Adapter，否则 RecyclerView 的动画与回收会拿到错误状态。
- **`isRunning()` 是一致性检查**：pending、运行中、批次列表任意一处非空都算忙，避免提前 `dispatchAnimationsFinished` 让 RecyclerView 重排。
- **change 关闭的代价**：`supportsChangeAnimations = false` 时 `notifyItemChanged` 不会产生交叉动画，视觉上只是内容直接替换；要交叉就得显式打开，并接受 `ChangeInfo` 里 old/new 两个 holder 的状态维护。

## 阅读路径

1. `animators/src/main/java/jp/wasabeef/recyclerview/animators/BaseItemAnimator.kt` —— 通读 L29–L181、L227–L350、L444–L525。
2. `animators/src/main/java/jp/wasabeef/recyclerview/animators/FadeInAnimator.kt` —— 最短的子类实现。
3. `animators/src/main/java/jp/wasabeef/recyclerview/animators/holder/AnimateViewHolder.kt` —— 扩展点。
4. `animators/src/main/java/jp/wasabeef/recyclerview/internal/ViewHelper.kt` —— 复位逻辑。
5. `animators/src/main/java/jp/wasabeef/recyclerview/animators/SlideInLeftAnimator.kt`、`ScaleInAnimator.kt` —— 对照三种属性的写法差异。
6. `animators/src/main/java/jp/wasabeef/recyclerview/adapters/AnimationAdapter.kt` —— Adapter 侧的另一条路线。
7. `animators/src/main/java/jp/wasabeef/recyclerview/adapters/AlphaInAnimationAdapter.kt` —— 包装 adapter 的最简实现。
8. `example/src/main/java/jp/wasabeef/example/recyclerview/AnimatorSampleActivity.kt` —— 如何挂到 RecyclerView 上。

配套图：[class-diagram.html](./class-diagram.html) 与 [flow-diagram.html](./flow-diagram.html)。

## 版本注记

构建依赖停留在 Kotlin 1.3.72 / AGP 4.2.0-beta04 / compileSdk 30，仓库自 2021 年后基本停更，4.0.1 是最后一条版本线。`CHANGELOG.md` 记录 4.0.0 的核心变化是「全部 Java 代码转 Kotlin」「minSdk 14→21、targetSdk 28→30」；本次读到的 `BaseItemAnimator` 已是没有 `m` 前缀字段名的 Kotlin 版本，与网上大量旧版 Java 源码解析文章在命名上不一致，阅读时需注意。具体行为以 commit `6c5c7748e86f43d3d03f7b3122f473e1fd807681` 为准。
