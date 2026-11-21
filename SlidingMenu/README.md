# SlidingMenu 源码解析

> 排行榜 #14 · 类别：菜单（侧滑菜单）

## 一句话定位

SlidingMenu 是"上古神器"级的侧滑菜单库：`SlidingMenu` 本身是一个 `RelativeLayout`，内部由 **CustomViewAbove（内容层）+ CustomViewBehind（菜单层）** 两层视图组成，拖动时内容层平移、菜单层按比例视差跟随，通过 `attachToActivity` 挂到 Activity 上，并提供 `SLIDING_WINDOW` / `SLIDING_CONTENT` 两种滑动模式。

## 源码快照

| 项 | 值 |
|---|---|
| 上游仓库 | https://github.com/jfeinstein10/SlidingMenu |
| 快照 commit | `4254feca3ece9397cd501921ee733f19ea0fdad8` |
| 克隆方式 | `--depth 1` |
| 状态 | 多年未更新（排行榜成文时已 4 年无提交），Google 官方 NavigationDrawer 已取代它 |

模块：`library/`（核心）、`library-maps-support/`、`example/`（各类用法：左/右/双侧菜单、Fragment 菜单、自定义动画）。

## 核心类

| 类 | 职责 | 关键位置 |
|---|---|---|
| `SlidingMenu` | 门面：组合两层视图、模式/触摸模式、show/toggle 与四个状态监听 | `SlidingMenu.java` L32、L291、L483、L533 |
| `CustomViewAbove` | 内容容器 ViewGroup：拦截触摸、滚动内容、回调 `OnPageChangeListener` | `CustomViewAbove.java` L34、L197、L250 |
| `CustomViewBehind` | 菜单层 ViewGroup：布局菜单视图、滚动/绘制、应用 `CanvasTransformer` | `CustomViewBehind.java` L18、L44、L100 |
| `SlidingActivity` | 宿主 Activity 基类，持有 `SlidingActivityHelper` | `SlidingActivity.java` L11、L19 |
| `SlidingActivityHelper` | 真正干活：创建 SlidingMenu、attachToActivity、转发 show/toggle | `SlidingActivityHelper.java` L14、L161、L192 |
| `MenuInterface` | 菜单布局策略接口（滚动、边界、触摸是否允许） | `MenuInterface.java` L7 |
| `CanvasTransformer` | 菜单层的 Canvas 变换扩展点（缩放/旋转/自定义） | `SlidingMenu.java` L154 |
| `OnPageChangeListener` | 滑动进度回调（`onPageScrolled` / `onPageSelected`） | `CustomViewAbove.java` L107 |
| `SimpleOnPageChangeListener` | 空实现适配器 | `CustomViewAbove.java` L135 |

## 关键流程

### 1. 绑定到 Activity

`SlidingActivity.onCreate` 创建 `SlidingActivityHelper` 并调用其 `onCreate`；`onPostCreate` 里 `attachToActivity`：把 Activity 的内容视图包进 `SlidingMenu` 的 `CustomViewAbove`，把 `setBehindContentView` 指定的菜单视图放进 `CustomViewBehind`，最后把 `SlidingMenu` 设为 Activity 根布局。

### 2. 两层视图与滑动模式

- `SLIDING_WINDOW`：菜单是独立的一层窗口，内容层整体平移；
- `SLIDING_CONTENT`：菜单在内容下面，通过 `scrollScale` 做视差。

`CustomViewAbove` 在 `onInterceptTouchEvent/onTouchEvent` 里根据触摸模式（MARGIN / FULLSCREEN / NONE）决定是否开始拖动，拖动时计算 `percentOpen`，回调 `onPageScrolled`。

### 3. 菜单层的绘制

`CustomViewBehind` 根据 `percentOpen` 做 `scrollTo`，并在 `dispatchDraw` 里通过 `CanvasTransformer.transformCanvas(canvas, percentOpen)` 对菜单层做缩放/旋转等动画（`CustomAnimation`/`CustomZoomAnimation` 示例即来自这里）。

### 4. 状态回调

`showMenu(true)` / `toggle(true)` 触发 `setCurrentItem` 平滑滚动；滚动过程中根据进度触发 `OnOpen`/`OnOpened`/`OnClose`/`OnClosed` 四个监听，业务可据此做联动（如动画、埋点）。

## 设计要点

1. **两层 ViewGroup 叠放**：用自定义容器而不是 DrawerLayout，能实现视差、缩放、自定义 Canvas 变换，动画自由度更高。
2. **Helper 组合代替继承**：`SlidingActivity` 只是薄壳，真正的绑定逻辑在 `SlidingActivityHelper`，因此能同时支持 `SlidingFragmentActivity`、`SlidingListActivity` 等宿主。
3. **触摸模式可配**：`TOUCHMODE_MARGIN/FULLSCREEN/NONE` 加边缘宽度，避免与 ViewPager、SeekBar 等横向控件冲突。
4. **CanvasTransformer 扩展点**：把菜单层的绘制变换交给业务方，示例中的缩放/旋转动画都是它的实现。
5. **历史价值**：它是 DrawerLayout 之前最流行的侧滑方案，读它可以理解"自定义 ViewGroup + 触摸拦截 + 滚动动画"的完整套路。

## 阅读路径

1. `library/src/com/jeremyfeinstein/slidingmenu/lib/SlidingMenu.java`（`attachToActivity`、`showMenu`、`toggle`）
2. `library/.../CustomViewAbove.java`（触摸拦截、`setCurrentItem`、`OnPageChangeListener`）
3. `library/.../CustomViewBehind.java`（`scrollTo`、`dispatchDraw`、`transformCanvas`）
4. `library/.../app/SlidingActivityHelper.java`（绑定流程）
5. `library/.../app/SlidingActivity.java`
6. `library/.../MenuInterface.java`
7. `example/` 下的 `CustomAnimation` / `ViewPagerActivity` / `LeftAndRightActivity`

## 版本注记

- 该 commit 是仓库最后状态（1.3 线），仍在用 `android.support.v4` 包，现代工程需迁移到 AndroidX。
- 排行榜成文时（2017）它已停止维护；现在应优先用 `DrawerLayout` + `NavigationView`，本目录保留它用于学习自定义 ViewGroup 的经典实现。
