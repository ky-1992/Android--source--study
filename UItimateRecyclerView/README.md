# UltimateRecyclerView 源码解析

## 一句话定位
在 `FrameLayout` 里包一层 `RecyclerView` + 下拉刷新，用适配器的附加视图位补齐 Header、Footer、空视图、加载更多与侧滑删除，让列表开箱即用，业务侧只关心数据与条目布局。

## 源码快照
- 上游：<https://github.com/cymcsg/UltimateRecyclerView>
- commit：`b05147d16dd2bfd9f4de4ebe0fdfc4617fb4e857`
- 结构：`UltimateRecyclerView/ultimaterecyclerview` 模块，主包 `com.marshalchen.ultimaterecyclerview` 放容器与适配器；`ui/` 含刷新布局、空视图、悬浮按钮与侧滑；`itemTouchHelper/`、`stickyheadersrecyclerview/`、`layoutmanagers/`、`quickAdapter/`、`swipe/` 分别承载拖拽、粘性头、布局管理器与快速适配器；布局资源在 `res/layout/`，入口是 `ultimate_recycler_view_layout.xml`。依赖为 AndroidX 版的 `recyclerview` 与 `swiperefreshlayout`。本 commit 中适配器基类叫 `UltimateViewAdapter`，没有名为 `UltimateRecyclerViewAdapter` 的类。

## 核心类

| 类 | 职责 | 关键方法与行号 |
|---|---|---|
| `UltimateRecyclerView` | 容器门面：持有列表、刷新层、适配器与滚动监听 | `initViews` L182、`setAdapter` L667、`setDefaultOnRefreshListener` L724、`scroll_load_more_detection` L416 |
| `UltimateViewAdapter` | 抽象适配器基类，管理 Header/Footer/空视图策略 | `internalExecuteLoadingView` L125、`onCreateViewHolder` L169、`getItemViewType` L248 |
| `NormalUltimateViewAdapter` | 普通条目版适配器 | `getItemViewType` L96、`getItemCount` L118 |
| `SwipeableUltimateViewAdapter` | 侧滑菜单版适配器 | `withBindHolder` L37 |
| `VerticalSwipeRefreshLayout` | 只拦截下拉手势的刷新布局 | `onInterceptTouchEvent` L20 |
| `RecyclerViewPositionHelper` | 对任意 LayoutManager 取可见项位置 | `findFirstVisibleItemPosition` L49、`findLastVisibleItemPosition` L73 |
| `CustomUltimateRecyclerview` / `SwipeableUltimateRecyclerview` | 换成 PtrFrameLayout / SwipeListView | `initViews` L34 / L33 |
| `UltimateRecyclerviewViewHolder` | 带 `SwipeLayout` 的通用 ViewHolder | `onBindView` L42、`findViewByIdEfficient` L150 |
| `ItemTouchHelperAdapter` / `SimpleItemTouchHelperCallback` | 拖拽与滑动删除的适配层 | `ItemTouchHelperAdapter` L27、`SimpleItemTouchHelperCallback` L35 |
| `UltimateGridLayoutAdapter` | 网格条目适配器 | `UltimateGridLayoutAdapter` L18（`easyRegularAdapter` 子类） |

## 关键流程
1. **装配容器**：`initViews` inflate `ultimate_recycler_view_layout`，取出 `mRecyclerView` 与 `mSwipeRefreshLayout`，刷新层默认关闭，并注册默认滚动监听。
2. **绑定适配器**：`setAdapter` 交给 `RecyclerView` 后再走 `setAdapterInternal`，注册 `AdapterDataObserver`、下发空视图策略、把内置加载更多视图注入适配器并置为可加载；若有 header，还会调 `setCustomHeaderView` 交给适配器。
3. **滚动触底**：`onScrolled` 调 `scroll_load_more_detection`，按 Linear/Grid/StaggeredGrid 三种 LayoutManager 算出首末可见位置，`bottomEdgeHit` 满足时只触发一次 `loadMore`，并自动把 `previousTotal` 更新为当前总数；回调参数是列表条目总数与最后可见位置。
4. **空视图与收尾**：`setAdapterInternal` 为适配器注册 `AdapterDataObserver`，每次数据变更都调 `updateHelperDisplays` 关掉刷新动画，并按 `STARTWITH_OFFLINE_ITEMS` / `STARTWITH_ONLINE_ITEMS` 策略决定显示还是隐藏空视图。
5. **加载更多视图**：`internalExecuteLoadingView` 把延迟 Runnable 投递到 Handler，`enableLoadMore` 控制 `enabled_custom_load_more_view`，`getItemViewType` 把最后一个位置映射成 `VIEW_TYPES.FOOTER`。
6. **下拉刷新**：`setDefaultOnRefreshListener` 打开刷新层并设置配色，`VerticalSwipeRefreshLayout` 在横向位移超过 `touchSlop` 时放弃拦截，避免与横滑控件抢事件；刷新结束后由宿主调用 `setRefreshing(false)`，或由数据变更回调统一关闭动画。

## 设计要点
- 组合优于继承：容器只负责把列表、刷新层、空视图、悬浮按钮拼到一起，具体条目交给适配器体系。
- 视图位编号统一：`VIEW_TYPES` 固定 NORMAL/HEADER/FOOTER/NOVIEW/ADVIEW/CUSTOMVIEW/SECTION 七种类型，`getItemCount` 在业务数量上叠加占位。
- 三种容器变体复用同一套门面：`CustomUltimateRecyclerview` 换 PtrFrameLayout，`SwipeableUltimateRecyclerview` 换 SwipeListView，只重写 `initViews`。
- 加载态延迟生效：`enableLoadMore` 只保存 Runnable，等 `internalExecuteLoadingView` 再 `post`，避免 notify 与布局竞争。
- 位置计算抽成 Helper：`RecyclerViewPositionHelper` 屏蔽不同 LayoutManager 的 API 差异。
- 空视图策略可组合：`EMPTY_CLEAR_ALL`、`EMPTY_SHOW_LOADMORE_ONLY`、`EMPTY_KEEP_HEADER`、`EMPTY_KEEP_HEADER_AND_LOARMORE` 四个常量控制通知数据时保留哪些占位。
- 刷新层默认关闭：`initViews` 里先 `setEnabled(false)`，必须显式调 `setDefaultOnRefreshListener` 或 `enableDefaultSwipeRefresh` 才会生效，避免误触。

## 阅读路径
1. `UltimateRecyclerView/ultimaterecyclerview/src/main/res/layout/ultimate_recycler_view_layout.xml`
2. `.../UltimateRecyclerView.java`
3. `.../UltimateViewAdapter.java`
4. `.../NormalUltimateViewAdapter.java`
5. `.../ui/VerticalSwipeRefreshLayout.java`
6. `.../uiUtils/RecyclerViewPositionHelper.java`
7. `.../SwipeableUltimateRecyclerview.java`
8. `.../itemTouchHelper/SimpleItemTouchHelperCallback.java`

## 版本注记
以 commit `b05147d` 为准。该版本已使用 `androidx` 包名，但部分命名仍保留早期风格（如类名 `UltimateRecyclerviewViewHolder`、`CustomUltimateRecyclerview`）；README 中的类名、方法与行号均取自该 commit。
