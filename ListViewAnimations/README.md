# ListViewAnimations 源码解析

## 一句话定位

ListViewAnimations 是 2014 年前后的 ListView 增强库：用装饰器模式逐层包装 `BaseAdapter`，在入场动画、侧滑删除、拖拽换位之间自由组合；`DynamicListView` 负责把一次触摸分给拖拽或侧滑处理，保证同一时刻只有一方接管。核心思想是“数据装饰 + 手势分层”。

## 源码快照

- 上游：<https://github.com/nhaarman/ListViewAnimations>
- commit：`23617960178fb5af3099c07996569cc305a40945`（浅克隆，`origin` 指向该地址）
- 版本：`gradle.properties` 中 `VERSION_NAME=3.1.0`、`GROUP=com.nhaarman.listviewanimations`

目录结构概览：

| 路径 | 内容 |
| --- | --- |
| `lib-core/src/main/java/com/nhaarman/listviewanimations/` | `BaseAdapterDecorator`、`ArrayAdapter` 及 `util/`（`ListViewWrapper`、`AbsListViewWrapper`、`Swappable`、`Insertable`） |
| `…/listviewanimations/appearance/` | 入场动画：`AnimationAdapter`、`ViewAnimator`、`SingleAnimationAdapter`、`ResourceAnimationAdapter`、`simple/` |
| `lib-core-slh/` | StickyListHeaders 适配：`StickyListHeadersListViewWrapper`、`StickyListHeadersAdapterDecorator` |
| `lib-manipulation/…/itemmanipulation/dragdrop/` | 拖拽：`DragAndDropHandler`、`HoverDrawable`、`GripView`、`DraggableManager` |
| `lib-manipulation/…/itemmanipulation/swipedismiss/` | 侧滑：`SwipeTouchListener`、`SwipeDismissTouchListener`、`SwipeDismissAdapter`、`undo/` |
| `lib-manipulation/…/itemmanipulation/animateaddition/` | 插入动画：`AnimateAdditionAdapter`、`InsertQueue` |
| `example/` | 演示 app，覆盖上述几种组合 |

## 核心类

| 类 | 职责 | 关键方法（行号取自快照） |
| --- | --- | --- |
| `ListViewWrapper` | 列表能力抽象，隔离 `ListView`/`AbsListView`/SLH | `getChildAt()` L31、`getFirstVisiblePosition()` L33、`smoothScrollBy()` L48 |
| `BaseAdapterDecorator` | 装饰器基类，所有方法原样转发 | `getRootAdapter()` L74、`setListViewWrapper()` L99、`getView()` L124、`notifyDataSetChanged()` L165 |
| `AnimationAdapter` | 入场动画装饰器，创建并持有 `ViewAnimator` | `setListViewWrapper()` L88、`getView()` L130、`getAnimators()` L190 |
| `ViewAnimator` | 决定是否播动画、算延迟、缓存已播位置 | `animateViewIfNecessary()` L226、`animateView()` L242、`calculateAnimationDelay()` L262、`onSaveInstanceState()` L289 |
| `SwipeTouchListener` | 侧滑手势状态机（抽象基类） | `onTouch()` L292、`handleDownEvent()` L326、`handleMoveEvent()` L423、`handleUpEvent()` L476、`fling()` L264 |
| `SwipeDismissTouchListener` | 侧滑后的飞出与高度折叠 | `afterViewFling()` L138、`performDismiss()` L147、`finalizeDismiss()` L163、`notifyCallback()` L178 |
| `DragAndDropHandler` | 拖拽：按下定位、移动换位、松手回调 | `startDragging()` L210、`handleMoveEvent()` L362、`switchIfNecessary()` L392、`handleUpEvent()` L448 |
| `DynamicListView` | 触摸派发与两个 handler 的竞争仲裁 | `dispatchTouchEvent()` L234、`onTouchEvent()` L277、`sendCancelEvent()` L296 |

类图见 [class-diagram.html](./class-diagram.html)，图中保留装饰链与手势两个家族，省略 `HoverDrawable`、`InsertQueue` 等辅助类。

## 关键流程

绑定与装饰：`DynamicListView.setAdapter()`（L206）会在根适配器实现 `Insertable` 时套一层 `AnimateAdditionAdapter`（L220–L223），再把适配器同步给 `DragAndDropHandler`（L228–L230）；侧滑监听则由 `enableSwipeToDismiss()`（L155）挂上。`BaseAdapterDecorator.setListViewWrapper()`（L99）把包装器缓存下来并**向下透传**给被装饰者，子类覆盖后调 `super` 即可。`AnimationAdapter.setListViewWrapper()`（L88）额外 `new ViewAnimator(listViewWrapper)`，`getView()`（L130）则在返回前调 `animateViewIfNecessary()`（L157）触发动画。`getRootAdapter()`（L74）用 while 循环逐层剥掉装饰器，`swapItems()` 之类需要真实数据源时用它。

手势竞争：`DynamicListView.dispatchTouchEvent()`（L234）先判断 `mCurrentHandlingTouchEventHandler` 是否为空——为空时依次把事件给 `DragAndDropHandler.onTouchEvent()`（`DragAndDropHandler.java:266`）和 `SwipeTouchListener.onTouchEvent()`（`SwipeTouchListener.java:287`）；某一方 `isInteracting()` 为真就把自己登记为当前 handler，并用 `sendCancelEvent()`（L296）给另一方补发 `ACTION_CANCEL`，同时向 `super.onTouchEvent()` 补一个 cancel。之后同一次手势内的事件都走 `onTouchEvent()`（L277）只交给当前 handler，`ACTION_UP`/`ACTION_CANCEL` 时清空，下一次按下重新竞争。

拖拽链路：`handleDownEvent()`（L306）借 `GripView` 与 `DraggableManager` 判断是否可拖，`startDragging()`（L210）用 `HoverDrawable` 抬起当前 View；移动时 `handleMoveEvent()`（L362）计算 `getPositionForId()`，必要时 `switchIfNecessary()`（L392）→ `switchViews()`（L425）交换相邻 View 并同步数据；松手 `handleUpEvent()`（L448）落位后经 `OnItemMovedListener` 通知业务。

侧滑链路：`SwipeTouchListener.handleMoveEvent()`（L423）只做 `translationX` 与 alpha；`handleUpEvent()`（L476）判断位移或速度超阈值，超阈值调 `fling()`（L264）做飞出动画，动画结束进 `afterViewFling()`（L138）；`SwipeDismissTouchListener.performDismiss()`（L147）用 `ValueAnimator` 把高度从 view 高折到 1，全部动画结束由 `finalizeDismiss()`（L163）汇总位置，`notifyCallback()`（L178）按**倒序**回调 `OnDismissCallback`。

动画链路：`ViewAnimator.animateViewIfNecessary()`（L226）先看位置是否在 `mLastAnimatedPosition` 之后、是否已播过，`animateView()`（L242）把子装饰器返回的 animator 与自身组合成 `AnimatorSet`，`calculateAnimationDelay()`（L262）按位置算 `startDelay`；复用 convertView 前会 `cancelExistingAnimation()`。

流程图见 [flow-diagram.html](./flow-diagram.html)，覆盖触摸分发 → 拖拽/侧滑两条支线 → 落位与回收调。

## 设计要点

- **装饰器而不是继承**：`BaseAdapterDecorator` 实现 `SectionIndexer`、`Swappable`、`Insertable`、`ListViewWrapperSetter` 四类能力，子类只覆盖需要增强的一两个方法，其余全部 `super` 转发，因此动画、侧滑、插入可以任意叠放。
- **依赖倒置到 `ListViewWrapper`**：核心层只认 `getChildAt()`、`getFirstVisiblePosition()`、`smoothScrollBy()` 三个能力，`AbsListViewWrapper`（L26）包装普通 `AbsListView`，`lib-core-slh` 再包装 StickyListHeaders——换列表控件不用改动画代码。
- **单一赢家竞争**：`DynamicListView` 用 `mCurrentHandlingTouchEventHandler` 加 `ACTION_CANCEL` 把"拖拽优先、侧滑兜底"写进派发逻辑，避免两个手势同时改 `translationY` 与 `translationX`。
- **手势与视觉分层**：`SwipeTouchListener` 只管位移与阈值判定，飞出和高度折叠交给子类 `SwipeDismissTouchListener` 的 `afterViewFling()` 钩子，`undo/` 包可以在同一个钩子上换成回滚实现。
- **动画只播一次**：`ViewAnimator` 用 `mLastAnimatedPosition` 与 `SparseArray<Animator>` 记录状态，并实现 `onSaveInstanceState()`/`onRestoreInstanceState()`（L289、L304），旋转后不重播。
- **动画库兼容**：全程基于 nineoldandroids 的 `Animator`/`ViewHelper`，为 API 11 以下设备提供属性动画。

## 阅读路径

1. `lib-core/src/main/java/com/nhaarman/listviewanimations/BaseAdapterDecorator.java` —— 先读 L39–L130，理解装饰与 `getRootAdapter()` 的剥离逻辑。
2. `lib-core/…/util/ListViewWrapper.java` 与 `AbsListViewWrapper.java` —— 抽象只有四个方法，决定后面所有代码的依赖面。
3. `lib-core/…/appearance/AnimationAdapter.java` —— `setListViewWrapper()`（L88）与 `getView()`（L130）如何接入动画。
4. `lib-core/…/appearance/ViewAnimator.java` —— 重点 L226–L288，看"只播一次"与延迟计算。
5. `lib-manipulation/…/itemmanipulation/DynamicListView.java` —— `dispatchTouchEvent()`（L234–L274）的竞争逻辑是全库最关键的几十行。
6. `lib-manipulation/…/itemmanipulation/dragdrop/DragAndDropHandler.java` —— L306–L470 的按下、移动、换位、落位四段。
7. `lib-manipulation/…/itemmanipulation/swipedismiss/SwipeTouchListener.java`、`SwipeDismissTouchListener.java` —— 手势判定与收尾动画的分离。

## 版本注记

该 commit 对应库的 3.1.0 版本线（`VERSION_NAME=3.1.0`）。工程仍用 `com.android.tools.build:gradle:0.12.2`、`nineoldandroids` 与 `pom.xml`/Gradle 双构建，属 Android 4.x 时代；`lib-core-slh` 依赖第三方 StickyListHeaders 适配器，说明快照保留了可选的列表控件扩展。仓库无 tag，版本以该 commit 为准。
