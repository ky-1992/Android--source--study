# Android-PullToRefresh 源码解析

## 一句话定位

给 `ListView`、`GridView`、`WebView` 等可滚动控件套一层"下拉 / 上拉刷新"容器的经典库：外层 `LinearLayout` 接管触摸事件，用一套六态状态机把手指拖动距离翻译成加载区动画与刷新回调，业务方只需要注册一个 `OnRefreshListener` 并在数据加载完后调用 `onRefreshComplete`。

## 源码快照

- 上游：<https://github.com/chrisbanes/Android-PullToRefresh>
- commit：`3bd8ef6869c3297bfe874d2f15c2ee53c3456e99`（2013-02-02，`Merge branch 'dev'`）
- 工程结构：`library/` 是库本体，源码位于 `library/src/com/handmark/pulltorefresh/library/`，其中 `internal/` 子包放加载视图体系；`extras/` 提供 ListFragment、ViewPager 扩展；`sample/` 是演示 App。仓库 README 首行已注明项目不再维护。
- 类关系概览：`IPullToRefresh` 定义对外契约，`PullToRefreshBase` 实现它；具体控件继承 `PullToRefreshAdapterViewBase`，再把 `LoadingLayout` 的头尾两个实例挂进自己的布局。`ILoadingLayout` 负责文案与图标接口，由 `LoadingLayoutProxy` 代理到多个加载区。

## 核心类

| 类 | 职责 | 关键方法（行号） |
|---|---|---|
| `PullToRefreshBase` | 抽象容器：手势、状态机、回弹 | `setState` L539、`onTouchEvent` L310、`pullEvent` L1164 |
| `PullToRefreshAdapterViewBase` | AbsListView 系控件公共适配 | `onScrollStateChanged` L129、`setOnLastItemVisibleListener` L211 |
| `PullToRefreshListView` | 把加载视图塞进 ListView 头尾 | `onRefreshing` L66、`handleStyledAttributes` L235 |
| `LoadingLayout` | 加载区三态视图基类 | `onPull` L220、`refreshing` L235、`reset` L261 |
| `RotateLoadingLayout` | 箭头按下拉比例旋转 | `onPullImpl` L65、`refreshingImpl` L78 |
| `FlipLoadingLayout` | 翻转箭头并切换进度圈 | `refreshingImpl` L99、`resetImpl` L111 |
| `LoadingLayoutProxy` | 一次调用转发到多个加载区 | `addLayout` L26 |
| `State` | 六态枚举 | L1524 |
| `Mode` | 方向配置 | L1335 |

## 关键流程

1. **按下与拖动**：`onInterceptTouchEvent` 先判断当前是否已经滚到边界，决定要不要把事件从子控件手里抢过来；接管后 `onTouchEvent` 记录 `mInitialMotionY`，只有位移超过 `mTouchSlop` 才置 `mIsBeingDragged`，避免轻点也被当成下拉。`Mode.BOTH` 时按方向写 `mCurrentMode`，后续所有回调都靠它区分头尾。
2. **距离折算**：`pullEvent` 把位移除以 `FRICTION=2.0f` 得到头尾偏移并 `setHeaderScroll`，同时用 `scale=|scroll|/itemDimension` 调 `onPull` 驱动子类动画。偏移为零或正在刷新时不再推进状态，防止抖动。
3. **状态推进**：`itemDimension >= |scroll|` 进入 `PULL_TO_REFRESH`，越过这个临界值则进入 `RELEASE_TO_REFRESH`；`setState` 是唯一的状态入口，按新状态分发到 `onPullToRefresh` / `onReleaseToRefresh` / `onRefreshing` / `onReset`，最后统一回调 `OnPullEventListener`，方便业务加音效或埋点。
4. **松手刷新**：`ACTION_UP` 时若处于 `RELEASE_TO_REFRESH` 且注册过监听器，就 `setState(REFRESHING,true)`；`onRefreshing` 先让加载区进入刷新态，再平滑滚动，滚动结束才 `callRefreshListener`。ListView 覆写该方法后会把加载视图临时搬进列表头尾，避免刷新瞬间列表跳动。
5. **结束复位**：业务调 `onRefreshComplete`，仅当 `isRefreshing()` 为真时切回 `RESET`，`onReset` 复位两个加载区并滚回 0。手动 `setRefreshing` 触发的 `MANUAL_REFRESHING` 走的是同一套收尾逻辑。

## 设计要点

- **状态机集中**：所有 UI 变化都在 `setState` 的 `switch` 内完成，子类只覆写 `onXxx` 钩子，避免判断散落各处。
- **动画策略化**：`LoadingLayout` 定义 `onPullImpl` / `refreshingImpl` / `resetImpl` 抽象，`Rotate` 与 `Flip` 各自实现，也可走 `mUseIntrinsicAnimation` 的帧动画分支。
- **代理收敛 API**：`LoadingLayoutProxy` 持有 `HashSet<LoadingLayout>`，对外只暴露 `ILoadingLayout`，头尾文案一次设置。
- **与具体控件解耦**：基类只管手势与偏移，`AdapterViewBase` 补滚动状态与最后一项监听，`ListView` 才处理 `addHeaderView` 细节。
- **手感统一**：摩擦系数固定 2.0，滚动距离约为手指位移一半，所有控件表现一致。
- **方向可扩展**：`Orientation` 抽象让水平滚动控件复用同一套逻辑，子类只需实现 `getPullToRefreshScrollDirection`。

## 阅读路径

1. `library/src/com/handmark/pulltorefresh/library/IPullToRefresh.java`
2. `library/src/com/handmark/pulltorefresh/library/PullToRefreshBase.java`
3. `library/src/com/handmark/pulltorefresh/library/internal/LoadingLayout.java`
4. `library/src/com/handmark/pulltorefresh/library/internal/RotateLoadingLayout.java`
5. `library/src/com/handmark/pulltorefresh/library/internal/FlipLoadingLayout.java`
6. `library/src/com/handmark/pulltorefresh/library/PullToRefreshAdapterViewBase.java`
7. `library/src/com/handmark/pulltorefresh/library/PullToRefreshListView.java`

## 版本注记

以该 commit 为准。`pom.xml` 中版本为 `2.1.2-SNAPSHOT`，构建仍是 Maven + Eclipse 结构、目标 Android 4.1；README 顶部写明项目已停止维护，因此这份代码可以视为该库的最终形态。它没有依赖 AndroidX，也没有引入 Kotlin，阅读时不必担心版本漂移。
