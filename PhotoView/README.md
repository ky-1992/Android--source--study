# PhotoView 源码解析

## 一句话定位

PhotoView 是建立在 `AppCompatImageView` 之上的可缩放图片控件：对外只暴露 `PhotoView`，真正的手势识别、矩阵计算与边界回弹全部收拢在 `PhotoViewAttacher` 里，二者通过一个字段解耦视图层与交互逻辑。

## 源码快照

- 上游：https://github.com/chrisbanes/PhotoView
- commit：`565505d5cb84f5977771b5d2ccb7726338e77224`（2021-01-20，`Update README.md`），Maven 发布版本号 `2.3.0`（见 `photoview/build.gradle` 的 `version = '2.3.0'`）。
- 模块：`photoview/` 是库本体，源码在 `photoview/src/main/java/com/github/chrisbanes/photoview/`，共 13 个类；`sample/` 是示例工程。库只依赖 `androidx.appcompat:appcompat:1.1.0`，没有第三方手势库。

## 核心类

| 类 | 职责 | 关键方法（行号） |
|---|---|---|
| `PhotoView` | 控件门面，把 ImageView 的 setter 与矩阵 getter 转发给 attacher | `init` 51-61、`setImageDrawable` 104-110、`setFrame` 129-135、`setScale` 229-239 |
| `PhotoViewAttacher` | 持有三层 Matrix、两个 GestureDetector 与全部监听器；实现 `View.OnTouchListener`、`View.OnLayoutChangeListener` | `onTouch` 339-391、`update` 498-506、`updateBaseMatrix` 602-653、`checkMatrixBounds` 655-711 |
| `CustomGestureDetector` | 组合 `ScaleGestureDetector`，自己用 `VelocityTracker` 判断拖拽与惯性 | `onTouchEvent` 112-120、`processTouchEvent` 122-213 |
| `OnGestureListener` | 手势增量回调接口 | `onDrag`、`onFling`、两个 `onScale`（18-27） |
| `Util` | ScaleType 校验、缩放级别校验、取 pointer index | `isSupportedScaleType` 23-32、`checkZoomLevels` 8-17 |
| `OnMatrixChangedListener` / `OnScaleChangedListener` / `OnViewDragListener` | 矩阵变化、缩放、拖动三条对外回调 | 均为单方法接口（对应文件 9-17、7-16、6-15 行） |
| `FlingRunnable` / `AnimatedZoomRunnable` | `PhotoViewAttacher` 的私有内部类，分别驱动 `OverScroller` 惯性滑动与插值缩放动画 | 763-821、728-761 |

## 关键流程

### 1. 图片设置后重建基准矩阵

`PhotoView` 覆写了 `setImageDrawable`、`setImageResource`、`setImageURI` 与 `setFrame`，每次调用父类实现后都执行 `attacher.update()`（104-135 行）。`update()` 在可缩放时调用 `updateBaseMatrix`（498-506 行），后者取出 View 宽高与 Drawable 固有尺寸，按 `ScaleType` 分支算出 `mBaseMatrix`：`FIT_CENTER/START/END/XY` 用 `setRectToRect`，`CENTER` 只平移，`CENTER_CROP/INSIDE` 取最大或最小缩放（602-653 行），最后 `resetMatrix` 清空 Supp 矩阵并显示。

### 2. 触摸事件的分发顺序

`onTouch`（339-391 行）先处理 ACTION_DOWN：禁止父容器拦截并取消正在进行的 fling；ACTION_UP/CANCEL 时若当前 scale 越界，就 `post` 一个 `AnimatedZoomRunnable` 回弹。随后先把事件交给 `mScaleDragDetector`，再交给 `mGestureDetector`；根据两个 detector 的状态设置 `mBlockParentIntercept`，决定父容器能否接管这次滑动。

### 3. 手势到矩阵增量的翻译

`CustomGestureDetector.processTouchEvent`（122-213 行）在 ACTION_MOVE 里用 `Math.sqrt(dx*dx+dy*dy) >= mTouchSlop` 判断是否进入拖拽（145 行），拖拽后回调 `mListener.onDrag`；ACTION_UP 时用 `VelocityTracker.computeCurrentVelocity(1000)` 算速度，超过最小 fling 速度则回调 `onFling`（175-185 行）。`PhotoViewAttacher` 里的匿名 `OnGestureListener`（98-161 行）把增量直接写进 `mSuppMatrix`：拖动 `postTranslate`、缩放 `postScale` 并补一次 `postTranslate`，然后统一调用 `checkAndDisplayMatrix`。

### 4. 边界校验与刷新

`checkAndDisplayMatrix`（574-578 行）先 `checkMatrixBounds` 再 `setImageViewMatrix`。`checkMatrixBounds`（655-711 行）把 Drawable 映射成 `RectF`，图片小于视口时按 ScaleType 居中/贴边，大于视口时把空白边平移回边界，同时记录 `mHorizontalScrollEdge`、`mVerticalScrollEdge` 供父容器拦截判断。`setImageViewMatrix`（560-569 行）调用 `ImageView.setImageMatrix` 并触发 `OnMatrixChangedListener`。

## 设计要点

1. **组合优于继承**：缩放能力没有塞进 `PhotoView`，而是由 `PhotoViewAttacher` 持有；控件只保留转发方法，便于单独替换交互实现，也便于用户直接拿 `getAttacher()` 做高级操作。
2. **三层矩阵**：`mBaseMatrix` 表达“图片如何铺满 View”，`mSuppMatrix` 表达“用户叠加的变换”，`mDrawMatrix` 是二者 `postConcat` 的结果（524-528 行）。重算基准矩阵不会丢失用户缩放。
3. **两套手势检测器分工**：`ScaleGestureDetector` 负责缩放，自研逻辑负责拖拽与 fling，`GestureDetector` 负责单击、双击、长按；三者通过 `mBlockParentIntercept` 协调。
4. **回弹而非硬夹取**：越界时不在手势过程中生硬限制，而是在抬手后用 200ms 插值动画归位，手感更自然。
5. **回调接口细粒度**：缩放、拖动、矩阵、点击图片/空白/视图分别暴露接口，业务按需注册，Attacher 不做事件聚合。

## 阅读路径

1. `photoview/src/main/java/com/github/chrisbanes/photoview/PhotoView.java`
2. `.../PhotoViewAttacher.java`（先读 498-711 的矩阵部分，再读 339-391 的 onTouch）
3. `.../CustomGestureDetector.java`
4. `.../OnGestureListener.java`
5. `.../Util.java`
6. `.../AnimatedZoomRunnable.java` 所在段落（`PhotoViewAttacher.java` 728-821 行）

## 版本注记

该 commit 属于 AndroidX 之后的 2.x 版本线，`PhotoView` 继承的是 `AppCompatImageView`，发布坐标为 `com.github.chrisbanes:PhotoView:2.3.0`。仓库自 2021 年后基本停止更新，代码规模保持在 1500 行左右，适合作为“View 手势 + Matrix”主题的精读样本。以上行号与结构均以该 commit 为准，不同分支可能存在差异。
