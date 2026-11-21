# AndroidSlidingUpPanel 源码解析

## 一句话定位

AndroidSlidingUpPanel 提供一个「从底部（或顶部）拉出面板」的容器控件 `SlidingUpPanelLayout`：布局里放主内容和面板两个子 View，用户拖动面板时库自己完成手势接管、位置换算、状态吸附与阴影遮罩。

## 源码快照

- 上游：<https://github.com/umano/AndroidSlidingUpPanel>
- commit：`45a460435b07e764138a700328836cafc1ed5c42`（2017-09-17，Fix the group）
- 版本：`gradle.properties` 中 `VERSION_NAME=3.4.0`、`VERSION_CODE=17`
- 模块：`library`（核心只有 3 个 Java 文件：`SlidingUpPanelLayout` 1490 行、`ViewDragHelper` 1502 行、`ScrollableViewHelper` 63 行）与 `demo`（示例 `DemoActivity`）

## 核心类

| 类 | 职责 | 关键方法（行号） |
|---|---|---|
| `SlidingUpPanelLayout` | 唯一的 ViewGroup，管测量、布局、触摸仲裁与状态 | `onMeasure`:725、`onLayout`:814、`onInterceptTouchEvent`:884、`dispatchTouchEvent`:958 |
| `PanelState` | 面板停靠状态 `EXPANDED/COLLAPSED/ANCHORED/HIDDEN/DRAGGING` | 定义:169 |
| `PanelSlideListener` | 滑动与状态变化回调 | `onPanelSlide`:241、`onPanelStateChanged`:248 |
| `DragHelperCallback` | 拖拽策略：夹取范围、松手吸附、状态收敛 | `onViewReleased`:1394、`clampViewPositionVertical`:1435、`onViewDragStateChanged`:1362 |
| `LayoutParams` | 在 MarginLayoutParams 上补 `weight` | 构造:1478 |
| `ViewDragHelper` | support 库拖拽引擎的拷贝，处理 slop、捕获与平滑 | `create`:347、`shouldInterceptTouchEvent`:983、`processTouchEvent`:1093、`continueSettling`:745 |
| `ViewDragHelper.Callback` | 策略基类，`tryCaptureView` 决定可否拖 | `tryCaptureView`:291、`onViewReleased`:202 |
| `ScrollableViewHelper` | 判断内嵌 ListView/ScrollView/RecyclerView 当前滚动余量 | `getScrollableViewScrollPosition`:24 |

## 关键流程

**1. 布局约束。** `onMeasure`（725 行）先强制 `width/height` 为确定值，再要求子 View 恰好 2 个：第 0 个 `mMainView`、第 1 个 `mSlideableView`；随后算出 `mSlideRange = 面板高度 - mPanelHeight`（默认 68dp）。`onLayout` 首次布局时按当前 `PanelState` 反推 `mSlideOffset`，之后所有像素位置都由 `computePanelTopPosition`（1062 行）从 offset 换算，`mIsSlidingUp` 只影响正负号。

**2. 触摸仲裁。** 手势先在 `dispatchTouchEvent`（958 行）做嵌套滚动仲裁：横向滑动直接放行，纵向时用 `ScrollableViewHelper` 判断列表是否还有滚动余量；该列表处理就继续下发，该面板处理就把事件转给 `onTouchEvent`，必要时给子 View 补发 `CANCEL`、给自己伪造 `DOWN` 来正确交接事件流。之后 `onInterceptTouchEvent`（884 行）再做拖拽区命中与横纵向 slop 判定。

**3. 拖动。** 真正的位移由 `ViewDragHelper.processTouchEvent` 驱动；`tryCaptureView` 只允许捕获 `mSlideableView`。位置变化回调 `onViewPositionChanged` 触发 `onPanelDragged`（1158 行）：置 `DRAGGING`、用 `computeSlideOffset` 反算 offset、应用视差、派发 `onPanelSlide`，并在非 overlay 模式下临时改主内容高度以免露白。

**4. 松手吸附。** `onViewReleased`（1394 行）按速度方向与锚点比较决定目标是展开、锚定还是收起；零速度时用 `(1+anchor)/2`、`anchor/2` 两个阈值兜底。目标 top 交给 `settleCapturedViewAt`，由 `continueSettling` 在 `computeScroll`（1246 行）里逐帧推进。回到 `STATE_IDLE` 后，`onViewDragStateChanged` 依 offset 收敛为 `EXPANDED/COLLAPSED/ANCHORED/HIDDEN`，HIDDEN 时把面板设为 `INVISIBLE`。

**5. 绘制。** `drawChild`（1186 行）把主内容按面板 top 裁剪，并用 `mCoveredFadeColor` 的 alpha 随 offset 叠加遮罩；`draw`（1258 行）在面板边缘画 `above_shadow`/`below_shadow`。

## 设计要点

1. **单一偏移真值**：只维护 `mSlideOffset`（0~1），状态与像素都由它推导，避免 top 与 state 双写不一致。
2. **策略下沉**：布局容器本身不做位移计算，全部通过 `ViewDragHelper.Callback` 暴露的夹取/回调挂钩实现，与官方控件手感一致。
3. **锚点吸附**：`mAnchorPoint` 让面板有中间停留档位，松手判定把速度与距离阈值结合，是「半展开」体验的来源。
4. **提前仲裁嵌套滚动**：把冲突处理放在 `dispatchTouchEvent` 而非拦截阶段，才能主动补发 `CANCEL/DOWN`，在列表与面板之间无缝切换。
5. **状态可恢复**：重写 `onSaveInstanceState`，用 `SLIDING_STATE` 存面板状态。
6. **全部行为可配**：构造函数（273 行起）从 `R.styleable.SlidingUpPanelLayout` 读取 `umanoPanelHeight`、`umanoShadowHeight`、`umanoParallaxOffset`、`umanoFlingVelocity`、`umanoFadeColor`、`umanoDragView`、`umanoScrollableView`、`umanoOverlay`、`umanoClipPanel`、`umanoAnchorPoint`、`umanoInitialState`、`umanoScrollInterpolator`，未配置项再按 density 换算默认 dp 值。

## 阅读路径

1. `library/src/main/java/com/sothree/slidinguppanel/SlidingUpPanelLayout.java` 的 `onMeasure`/`onLayout`
2. 同文件 `dispatchTouchEvent` + `onInterceptTouchEvent`
3. 同文件 `onPanelDragged` + `computePanelTopPosition`
4. 同文件 `DragHelperCallback`（松手吸附）
5. `library/src/main/java/com/sothree/slidinguppanel/ViewDragHelper.java` 的 `shouldInterceptTouchEvent`/`processTouchEvent`/`continueSettling`
6. `library/src/main/java/com/sothree/slidinguppanel/ScrollableViewHelper.java`
7. `demo/src/main/java/com/sothree/slidinguppanel/demo/DemoActivity.java`

## 版本注记

以该 commit 为准：3.4.0 是该项目最后一个发布版本线，库内 `ViewDragHelper` 是 Android support 库同名类的整份拷贝（因此不依赖 DrawerLayout），`ScrollableViewHelper` 明确支持 `ListView`、`ScrollView`、`RecyclerView` 三类并允许覆写扩展。上游 README 已声明维护不活跃，仅收 PR。
