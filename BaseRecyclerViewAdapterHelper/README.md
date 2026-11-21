# BaseRecyclerViewAdapterHelper 4.x 源码解析

## 一句话定位

BRVAH（BaseRecyclerViewAdapterHelper）是 CymChad 维护的 RecyclerView Adapter 增强库：在 `RecyclerView.Adapter` 外包一层模板基类 `BaseQuickAdapter`，把数据更新（`AsyncListDiffer`）、多布局、空状态、入场动画、上下加载更多、拖拽侧滑拆成可组合模块。4.x 用 Kotlin 重写，核心模块 `library` 只有 30 个源文件，适合当作"RecyclerView 组件化封装"的样板阅读。

## 源码快照

- 上游：<https://github.com/CymChad/BaseRecyclerViewAdapterHelper>
- 分支：`master`（浅克隆，`origin` 指向该地址）
- commit：`40997b531652019b10ed2fdd2712da5709dd90d3`（2026-10-23，README 版本号更新提交）
- 版本：`library/build.gradle.kts` 声明 `versionName = "4.4.2"`，`compileSdk = 35`、`minSdk = 19`；Maven 坐标 `io.github.cymchad:BaseRecyclerViewAdapterHelper4:4.4.2`

| 路径 | 内容 |
| --- | --- |
| `library/src/main/java/com/chad/library/adapter4/` | 核心包：`BaseQuickAdapter`、`BaseMultiItemAdapter`、`BaseSingleItemAdapter`、`BaseNodeAdapter`、`QuickAdapterHelper` |
| `adapter4/animation/` | `ItemAnimator` 接口与 `AlphaInAnimation`、`ScaleInAnimation`、`SlideIn*Animation` 五个预设 |
| `adapter4/loadState/` | `LoadState` 状态机原语、`LoadStateAdapter` 基类，以及 `trailing/`、`leading/` 两套加载更多实现 |
| `adapter4/dragswipe/` | `QuickDragAndSwipe`（`ItemTouchHelper.Callback`）与拖拽/侧滑监听接口 |
| `adapter4/viewholder/` | `QuickViewHolder`、`StateLayoutVH`、`DataBindingHolder` |
| `app/`、`library/src/main/res/` | 61 个文件的 demo 工程，以及空状态/加载状态布局资源 |

## 核心类

| 类名 | 职责 | 关键方法（行号取自快照） |
| --- | --- | --- |
| `BaseQuickAdapter` | 模板基类：final 覆写 Adapter 入口，子类只实现 `protected abstract` 版本 | `getItemViewType()` L299、`onCreateViewHolder()` L304、`onBindViewHolder()` L316、`runAnimator()` L560、`submitList()` L621 |
| `QuickViewHolder` | 用 `SparseArray` 缓存 `findViewById` 结果，配成套 setter | `getView()` L36、`getViewOrNull()` L43、`setText()` L54 |
| `QuickAdapterHelper` | 用 `ConcatAdapter` 把内容、首尾加载 adapter 拼成单个 adapter | `adapter` L77、`Builder.build()` L427、`Builder.attachTo()` L442 |
| `LoadStateAdapter` | 加载状态变化自动转换成 `notifyItemInserted/Removed/Changed` | `loadState` setter L23、`getItemCount()` L76、`displayLoadStateAsItem()` L131 |
| `TrailingLoadStateAdapter` | 尾部加载更多：滚动预加载与自动加载判定 | `onViewAttachedToWindow()` L63、`checkPreload()` L102、`invokeLoadMore()` L110 |
| `BaseMultiItemAdapter` | `addItemType` 把 viewType 映射到监听，实现多布局 | `onCreateViewHolder()` L37、`onBindViewHolder()` L48、`addItemType()` L69 |
| `QuickDragAndSwipe` | 实现 `ItemTouchHelper.Callback`，把位置变化转交数据回调 | `onMove()` L171、`onMoved()` L179、`onSwiped()` L200 |
| `BaseSingleItemAdapter` | 固定一条数据的精简适配器 | `setItem()` L47、`getItemCount()` L32 |
| `BaseNodeAdapter` | 树形节点的展开/收起，维护 opened/closed 集合 | `getChildNodeList()` L89、`open()` L295、`closeAll()` L388 |
| `ItemAnimator` / `AlphaInAnimation` | 单条入场动画接口与默认 300ms 淡入实现 | `animator()` L10 / L20 |

完整关系见 [class-diagram.html](./class-diagram.html)；图中省略了 `LoadState`、`StateLayoutVH`、`LeadingLoadStateAdapter` 等分支实现，只保留协作主干。

## 关键流程

diff 更新：构造时传入 `DiffUtil.ItemCallback` 就会创建 `AsyncListDiffer`（L61–62），`submitList()` 把新列表交给异步 diff（L621 起），并把 `mLastPosition` 复位为 -1，使新数据重新播放入场动画；未配置 diffCallback 时退回直接替换 `_items`。

创建与绑定：`getItemCount()`/`getItemViewType()` 是 final 的（L283–302），先判断 `displayEmptyView()`，列表为空且启用 stateView 时返回 `EMPTY_VIEW`，由 `StateLayoutVH` 占住唯一 item；否则调用子类的 `getItemViewType(position, list)`。`onCreateViewHolder()` 在拿到子类 VH 后立刻 `bindViewClickListener(this, viewType)`（L311–313）；`onBindViewHolder()` 判断 payloads，空则整条绑定，非空走带 payload 的局部刷新。

入场动画：`runAnimator()`（L560–570）只在 `animationEnable` 为真且 `holder.layoutPosition` 超过 `mLastPosition` 时执行，动画对象默认 `AlphaInAnimation`，由 `setItemAnimation()` 按枚举切换。

加载更多：`QuickAdapterHelper.Builder` 用 `setTrailingLoadStateAdapter()` 注册尾部适配器，`build()/attachTo()` 后才把它们塞进 `ConcatAdapter`（L427–451）。`TrailingLoadStateAdapter` 在 footer 附着（`preloadSize = 0`）或 `checkPreload()` 判定距尾部不足 `preloadSize` 时进入 `loadAction()`（L73–100），通过 `invokeLoadMore()` 把状态设为 `LoadState.Loading` 并回调 `onTrailingListener.onLoad()`；业务方改 `trailingLoadState` 即可让 footer 自动增删。

拖拽侧滑：`QuickDragAndSwipe.attachToRecyclerView()` 挂上内部的 `ItemTouchHelper`；`getMovementFlags()` 用 `makeMovementFlags` 下发方向，`onMove()` 只允许同 viewType 互换，`onMoved()` 调用 `dataCallback.dataMove(from, to)`，`onSwiped()` 调用 `dataRemoveAt(position)`——库本身不改数据，避免和业务列表冲突。

流程图见 [flow-diagram.html](./flow-diagram.html)，覆盖数据提交 → diff → 类型判断 → 创建绑定 → 加载更多预加载。

## 设计要点

- **组合优于继承**：4.x 把加载更多、拖拽、多布局从基类拆成独立类型，`BaseQuickAdapter` 只管数据与绑定，功能靠 `QuickAdapterHelper` 与 `ItemTouchHelper` 组合进来。
- **final 收口 + 双签名**：Adapter 的对外方法一律 `final override`，再暴露 `protected abstract` 版本（L232–244），子类无法绕过空状态、点击绑定等公共逻辑。
- **状态机式加载**：`LoadState` 是 sealed class，`LoadStateAdapter.loadState` setter 在状态切换时计算是否需要显示，自动发 `notifyItemInserted/Removed/Changed`，调用方只改状态不碰 UI。
- **按需动画**：`mLastPosition` 水位线保证滚动复用时不会重复播放；动画策略由接口 `ItemAnimator` 抽象，便于替换。
- **空状态也是 item**：空布局不额外套一层容器，而是让 `StateLayoutVH` 作为第 0 条 item 参与布局，因此能自然继承 `StaggeredGrid` 全跨列与滚动行为，代价是 `getItemCount()` 需要 +1 的特殊分支。
- **弱引用防泄漏**：`addItemType()` 里若监听器实现了 `OnMultiItem` 接口，会用 `WeakReference` 持有 adapter（L72–74）。

## 阅读路径

1. `library/src/main/java/com/chad/library/adapter4/BaseQuickAdapter.kt`：先看类头与 L232–339 的四个 final 入口。
2. `library/src/main/java/com/chad/library/adapter4/viewholder/QuickViewHolder.kt`：理解 VH 的缓存 setter。
3. `library/src/main/java/com/chad/library/adapter4/animation/ItemAnimator.kt` 与 `AlphaInAnimation.kt`：入场动画最小实现。
4. `library/src/main/java/com/chad/library/adapter4/loadState/LoadStateAdapter.kt`：loadState setter 的状态驱动写法。
5. `library/src/main/java/com/chad/library/adapter4/loadState/trailing/TrailingLoadStateAdapter.kt`：预加载与自动加载。
6. `library/src/main/java/com/chad/library/adapter4/QuickAdapterHelper.kt`：ConcatAdapter 拼接与 Builder。
7. `library/src/main/java/com/chad/library/adapter4/BaseMultiItemAdapter.kt` 与 `dragswipe/QuickDragAndSwipe.kt`：两个可选模块。
8. `app/src/main/java/com/chad/baserecyclerviewadapterhelper/activity/`：demo 里各功能的最小用法。

## 版本注记

该 commit 属于 4.4.2 版本线，库已完全 Kotlin 化并改用 `ConcatAdapter` 拼装首尾适配器（2.x/3.x 的 `LoadMoreView`、`BaseItemDraggableAdapter` 在这条线里已不存在）；`compileSdk = 35`、`minSdk = 19`，依赖以 `api` 暴露 `recyclerview` 与 `annotation`，`DataBindingHolder` 仅在启用 databinding 时生效。仓库里保留 2.x、3.x 分支，行号与 API 均以本 commit 为准。
