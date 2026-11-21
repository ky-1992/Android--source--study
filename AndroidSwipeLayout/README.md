# AndroidSwipeLayout 源码解析

## 一句话定位

AndroidSwipeLayout 是一个「可四向滑出底部视图」的 `FrameLayout` 容器：它把触摸事件交给 `ViewDragHelper`，按拖出距离决定展开还是回弹，并向业务方暴露开合回调；列表场景则由 `SwipeItemMangerImpl` 统一记录哪些条目处于打开状态。

## 源码快照

- 上游：<https://github.com/daimajia/AndroidSwipeLayout>
- commit：`5f8678b04751fb8510a88586b22e07ccf64bca99`（2017-07-11，合并 PR #446）
- 版本：`gradle.properties` 中 `VERSION_NAME=1.2.0`，`minSdkVersion=9`，编译/目标 SDK 25
- 结构：`library/src/main/java/com/daimajia/swipe/` 下是全部核心代码，含 `SwipeLayout.java`（约 1660 行）、`adapters/`、`implments/`（目录名拼写如此）、`interfaces/`、`util/Attributes.java`；`demo/` 是示例 App

## 核心类

| 类 | 职责 | 关键成员（行号取自快照） |
| --- | --- | --- |
| `SwipeLayout` | 容器本体：接管触摸、创建 `ViewDragHelper`、维护拖拽边与开合状态 | 构造 L78、`mDragHelperCallback` L220、`onInterceptTouchEvent()` L903、`getOpenStatus()` L1303、`processHandRelease()` L1327、`open()` L1373、`close()` L1428 |
| `SwipeLayout.Status` | 三态枚举：`Middle` / `Open` / `Close` | L1291 |
| `SwipeLayout.DragEdge` | 可拖出的四条边：`Left` / `Top` / `Right` / `Bottom` | L64 |
| `SwipeLayout.ShowMode` | `LayDown`（底视图垫在下面）与 `PullOut`（底视图被拉出） | L71 |
| `SwipeLayout.SwipeListener` | 开合生命周期：`onStartOpen`、`onOpen`、`onStartClose`、`onClose`、`onUpdate`、`onHandRelease` | L115 |
| `SwipeLayout.SwipeDetector` | 手势识别：单击关闭与双击回调 | L1198 |
| `SwipeItemMangerImpl` | 适配器侧状态簿记：`bind()` 给条目挂监听，维护打开 position 集合 | `bind()` L50、`SwipeMemory` L176、`OnLayoutListener` L153 |
| `SwipeItemMangerInterface` | 按 position 开合/查询的契约 | L8 |
| `BaseSwipeAdapter` | `BaseAdapter` 基类，`getView()` 中调用 `mItemManger.bind()` 后交给 `fillValues()` | L15、`getView()` L51 |
| `SwipeAdapterInterface` | 适配器需实现的资源 id 与刷新接口 | `getSwipeLayoutResourceId()` L5、`notifyDatasetChanged()` L7 |

类图见 [class-diagram.html](./class-diagram.html)，只保留主链路，`SimpleSwipeListener`、`Attributes.Mode` 等辅助类型未画出。

## 关键流程

1. **拦截与方向判定**：`onInterceptTouchEvent()`（L903）先处理 `clickToClose` 与 `SwipeDenier`，`ACTION_DOWN` 时把事件喂给 `mDragHelper`；`ACTION_MOVE` 调 `checkCanDrag()`（L829）。若当前是 `Close`，按移动角度选边：水平 45° 内选 `Left`/`Right`，否则选 `Top`/`Bottom`；再结合 `mTouchSlop` 与角度阈值决定是否真的开始拖。
2. **拖拽跟随**：`mDragHelperCallback`（L220）只捕获 surface 与当前 bottom 视图，`clampViewPositionHorizontal/Vertical`（L223/L264）把位移限制在 `0~mDragDistance`；`onViewPositionChanged`（L348）在 `PullOut` 模式下同时偏移底视图，然后派发 reveal 与 swipe 事件。
3. **松手落位**：`onViewReleased`（L337）调用 `processHandRelease()`（L1327），先看速度是否超过 `getMinVelocity()`，否则用拖出百分比与 `mWillOpenPercentAfterClose`（默认 0.25）或 `mWillOpenPercentAfterOpen`（默认 0.75）比较，决定 `open()` 还是 `close()`；平滑开合走 `smoothSlideViewTo` + `computeScroll()`（L666）。
4. **状态与通知**：`getOpenStatus()`（L1303）直接比较 surface 的 `getLeft()/getTop()` 与 padding/`mDragDistance`，只有恰好到位才是 `Open`/`Close`，其余为 `Middle`；`dispatchSwipeEvent`（L517/L533）用 `mEventCounter` 保证一次拖动只发一次 `onStartOpen`/`onStartClose`，到达终态再发 `onOpen`/`onClose`。`safeBottomView()`（L573）在关闭时把底视图置 `INVISIBLE`，避免 LayDown 模式下底视图抢触摸。适配器侧的 `SwipeItemMangerImpl.openItem()`（L72）只记录 position 并通知刷新，真正的开合发生在 convertView 复用的 `onLayout` 回调（`OnLayoutListener.onLayout` L166）里，所以列表滚动不会触发额外动画。

流程图见 [flow-diagram.html](./flow-diagram.html)。

## 设计要点

- **拖拽完全委托系统 `ViewDragHelper`**，自己只实现 `clamp`/`tryCapture`/`onViewReleased` 与方向判定，避免手写 `Scroller` 与速度追踪。
- **两套布局模式共存**：`PullOut` 用 `computeSurfaceLayoutArea`/`layoutPullOut`，`LayDown` 用 `computeBottomLayDown`/`layoutLayDown`；`onLayout`（L785）在重新布局时从 `mViewBoundCache` 恢复子 View 边界，防止滚动列表回收时视图跳动。
- **适配器复用靠 tag**：`bind()`（L50）把 `ValueBox`（含 `SwipeMemory` 与 `OnLayoutListener`）存进 `SwipeLayout` 的 tag，复用的 convertView 只更新 position，不重复添加监听。
- **Single / Multiple 两种打开模式**：`SwipeMemory.onStartOpen` 在 `Single` 模式下 `closeAllExcept`，保证列表同时只有一条打开。
- **回调粒度可控**：`open(boolean smooth, boolean notify)` 允许在布局恢复时传 `notify=false`，避免 `onLayout` 触发业务回调造成递归。
- **触摸屏蔽**：关闭态底视图被置为 `INVISIBLE`，既拿不到点击也不会遮挡 surface；`SwipeDenier` 则让内嵌的横向滚动容器在需要时拒绝本次滑动。

## 阅读路径

1. `library/src/main/java/com/daimajia/swipe/SwipeLayout.java`（先看 L30–L220 字段与构造，再看 L220–L400 的回调）
2. `SwipeLayout.java` 的 `onInterceptTouchEvent` 与 `checkCanDrag`（L829–L1000）
3. `SwipeLayout.java` 的 `getOpenStatus`/`processHandRelease`/`open`/`close`（L1291–L1465）
4. `interfaces/SwipeItemMangerInterface.java` 与 `util/Attributes.java`
5. `implments/SwipeItemMangerImpl.java`（L20–L215）
6. `adapters/BaseSwipeAdapter.java`
7. `interfaces/SwipeAdapterInterface.java` 与 `SimpleSwipeListener.java`

## 版本注记

本次快照是 1.2.0 线（2017 年），代码里已找不到网上旧教程常提的 `SwipeLayoutManager` 与自定义 `DragHelper`，拖拽直接建在系统 `ViewDragHelper` 上；状态也只有 `Status` 的 `Middle/Open/Close` 三态，没有独立的 `Opening`/`Closing`，中间过程用 `Middle` 加位置比较表达（浅克隆只有一个 commit，旧版 API 需另取 tag 核对）。本文所有结论以该 commit 为准。
