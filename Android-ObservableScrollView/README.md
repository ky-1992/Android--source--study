# Android-ObservableScrollView 源码解析

## 一句话定位

Android-ObservableScrollView 是 ksoichiro 的滚动联动基础库：给 `ScrollView`、`ListView`、`GridView`、`RecyclerView`、`WebView` 套一层可观察壳，把连续滚动位移换算成 `scrollY`，通过 `ObservableScrollViewCallbacks` 交给界面做工具栏收起、视差图、FAB 位移等效果。它只负责上报位置，动画与业务判断全部留给示例层。

## 源码快照

- 上游：`https://github.com/ksoichiro/Android-ObservableScrollView`
- 研读 commit：`47a5fb2db5e93d923a8c6772cde48bbb7d932345`（2016-02-14，默认分支 `master` 的 HEAD）
- 模块：`library` 是全部核心代码（11 个 Java 文件、约 3300 行），`samples` 是 40 多个效果示例，`docs`/`website` 为站点文档
- 包名 `com.github.ksoichiro.android.observablescrollview`，`gradle.properties` 里版本为 `1.7.0-SNAPSHOT`，该仓库此后未再更新（以该 commit 为准）

## 核心类

| 类 | 职责 | 关键方法与行号 |
|---|---|---|
| `Scrollable` | 可观察滚动控件统一协议 | `addScrollViewCallbacks()` 41、`getCurrentScrollY()` 72、`setTouchInterceptionViewGroup()` 81 |
| `ObservableScrollViewCallbacks` | 三段滚动回调接口 | `onScrollChanged()` 33、`onDownMotionEvent()` 38、`onUpOrCancelMotionEvent()` 45 |
| `ObservableScrollView` | `ScrollView` 可观察版本 | `onScrollChanged()` 80、`onTouchEvent()` 127、`dispatchOnScrollChanged()` 253 |
| `ObservableListView` | `ListView` 高度累加换算 | `onScrollChanged()` 274（`mScrollY` 计算在 330）、`init()` 269 |
| `ObservableRecyclerView` | `RecyclerView` 可观察版本 | `onScrollChanged()` 101、`scrollVerticallyToPosition()` 333 |
| `ScrollState` | 滚动方向枚举 | `STOP`/`UP`/`DOWN` 定义于 22–38 |
| `TouchInterceptionFrameLayout` | 触顶事件二次分发容器 | `onInterceptTouchEvent()` 107、`onTouchEvent()` 141、`duplicateTouchEventForChildren()` 253 |
| `TouchInterceptionListener` | 拦截判定与事件回调 | `shouldInterceptTouchEvent()` 51、`onMoveMotionEvent()` 67 |
| `ScrollUtils` | 颜色/夹取/布局工具 | `getFloat()` 41、`getColorWithAlpha()` 53、`addOnGlobalLayoutListener()` 67 |
| `CacheFragmentStatePagerAdapter` | ViewPager 页面缓存适配 | 150 行，配合 `ViewPagerTab` 示例使用 |

以上类型都在 `library` 模块内，除页面缓存适配器外都直接参与滚动事件的接收、换算与派发。

## 关键流程

1. **按下即通知**：`onInterceptTouchEvent()` 在 `ACTION_DOWN` 时先把 `mFirstScroll`/`mDragging` 置真并派发 `onDownMotionEvent()`（111–123），保证子 View 消费了 DOWN 也能复位动画。
2. **滚动换算**：`ScrollView` 直接用 `onScrollChanged(l,t,oldl,oldt)` 的 `t` 与 `mPrevScrollY` 比较判定 `UP`/`DOWN`（80–104）；`ListView`/`RecyclerView` 没有连续位移，改为以第一个可见子项为锚点，累加已滚过子项高度、减去 `getTop()` 再补 `divider` 与 `paddingTop`（`ObservableListView` 330）。
3. **统一派发**：`dispatchOnScrollChanged()`/`dispatchOnUpOrCancelMotionEvent()` 同时照顾旧的单监听 `mCallbacks` 与新的监听列表 `mCallbackCollection`（253–275）。
4. **触顶转发**：滑到顶部仍继续下拉时，`onTouchEvent()` 把 `ACTION_MOVE` 重新合成 `ACTION_DOWN` 交给 `setTouchInterceptionViewGroup()` 指定的祖先，让整页上滑/`TouchInterceptionFrameLayout` 接棒（138–193）。
5. **界面联动**：示例在 `onScrollChanged()` 里用 `ScrollUtils.getFloat()` 夹取位移，做工具栏平移、视差图缩放、FAB 位移（`FlexibleSpaceWithImageScrollViewActivity` 95–115）。

## 设计要点

- **接口最小化**：`Scrollable` 只保留监听增删、`scrollVerticallyTo`、`getCurrentScrollY` 与触顶转发，五个控件各自实现，调用方无需分支判断类型。
- **老 API 兼容**：监听既有单个 `setScrollViewCallbacks()`（已 `@Deprecated`），也有 `add/remove/clear` 集合版本，派发时两套都遍历。
- **五种控件同构**：`ScrollView`/`ListView`/`GridView`/`RecyclerView`/`WebView` 各自复制同一套触摸与派发代码，换来零依赖与可读性，代价是改一处要同步五个文件。
- **状态可保存**：各控件都自定义 `SavedState` 保存 `mScrollY` 等字段，旋转屏幕后滚动位置不丢。
- **列表类靠估算**：`SparseIntArray` 缓存子项高度，跳变时用首个子项高度近似，这是 `ListView`/`RecyclerView` 上 `scrollY` 抖动与小误差的根源。
- **触摸是显式契约**：`TouchInterceptionFrameLayout` 用手工 `dispatchTouchEvent`（`duplicateTouchEventForChildren`）伪造事件序列，注释明确警告调用 `dispatchTouchEvent()` 会栈溢出。

## 阅读路径

1. `library/src/main/java/com/github/ksoichiro/android/observablescrollview/Scrollable.java`（协议全貌）
2. `library/src/main/java/com/github/ksoichiro/android/observablescrollview/ObservableScrollViewCallbacks.java`（三个回调）
3. `library/src/main/java/com/github/ksoichiro/android/observablescrollview/ScrollState.java`
4. `library/src/main/java/com/github/ksoichiro/android/observablescrollview/ObservableScrollView.java`（80 行 onScrollChanged → 127 行 onTouchEvent）
5. `library/src/main/java/com/github/ksoichiro/android/observablescrollview/ObservableListView.java`（274 行高度累加）
6. `library/src/main/java/com/github/ksoichiro/android/observablescrollview/TouchInterceptionFrameLayout.java`（107 行拦截 → 253 行手工分发）
7. `library/src/main/java/com/github/ksoichiro/android/observablescrollview/ScrollUtils.java`
8. `samples/src/main/java/com/github/ksoichiro/android/observablescrollview/samples/FlexibleSpaceWithImageScrollViewActivity.java`（真实用法）

## 版本注记

以 commit `47a5fb2` 为准，仓库自 2016 年停更，`gradle.properties` 标注 `1.7.0-SNAPSHOT`、`SYNCED_VERSION_NAME=1.5.2`；代码基于旧版 support 库与 `ViewHelper`（NineOldAndroids 风格），迁到 AndroidX 时需替换 `ViewHelper` 与 `ListView`/`GridView` 相关 API。示例目录里的效果都建立在同一套回调之上，先读库再看 `samples` 最顺畅。
