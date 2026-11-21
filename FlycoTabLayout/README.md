# FlycoTabLayout 源码解析

## 一句话定位

一个只有 8 个 Java 文件的 Tab 控件库：`CommonTabLayout` 不依赖 ViewPager、`SlidingTabLayout` 跟随 ViewPager 滑动、`SegmentTabLayout` 是分段控件，三者共用同一套 `tl_*` 属性与"测量矩形 → 自绘指示器"的实现套路。

## 源码快照

- 上游：https://github.com/H07000223/FlycoTabLayout
- commit：`528fcffe88d54eeafbc85ffd1ce375df043b3a28`
- 模块：`FlycoTabLayout_Lib`（library，`com.flyco.tablayout`，含 `listener` / `utils` / `widget` 三个子包）与 `app`（示例工程 `com.flyco.tablayoutsamples`）
- 构建参数（`dependencies.gradle`）：`minSdkVersion = 16`、`compileSdkVersion = 29`、`versionName = '1.0.0'`；依赖 appcompat 1.2.0、material 1.2.1

## 核心类

| 类 | 职责 | 关键方法（行号） |
|---|---|---|
| `CommonTabLayout` | 继承 `FrameLayout`，用实体列表驱动 Tab，可顺带切换 Fragment | `setTabData` :194、`notifyDataSetChanged` :212、`setCurrentTab` :487、`onDraw` :392、`showMsg` :801 |
| `SlidingTabLayout` | 继承 `HorizontalScrollView` 并实现 `ViewPager.OnPageChangeListener` | `setViewPager` :173、`onPageScrolled` :318、`scrollToCurrentTab` :339、`calcIndicatorRect` :382、`onDraw` :440 |
| `SegmentTabLayout` | 分段控件，整块圆角背景 + 圆角滑块 | `setTabData` :164、`calcIndicatorRect` :285、`onDraw` :349、`getMsgView` :689 |
| `CustomTabEntity` | 接口，只暴露 title / 选中图标 / 未选中图标 | `getTabTitle` :6、`getTabSelectedIcon` :9 |
| `OnTabSelectListener` | 选中与重复点击回调 | `onTabSelect` :4、`onTabReselect` :5 |
| `FragmentChangeManager` | 用 `show`/`hide` 切换 Fragment | `initFragments` :25、`setFragments` :34 |
| `MsgView` | 角标 `TextView`，自绘圆角背景 | `onMeasure` :52、`onLayout` :64 |
| `UnreadMsgUtils` | 按位数决定红点尺寸与 "99+" 文案 | `show` :17、`setSize` :49 |
| `TabEntity` | 示例中的实体实现 | 见 `app/.../entity/TabEntity.java` |

## 关键流程

1. **装配数据**：`setTabData` 校验空列表后整体替换 `mTabEntitys` 并调用 `notifyDataSetChanged`；后者清空 `mTabsContainer`，按 `mIconGravity` 选择 `layout_tab_left/right/bottom/top` 模板 inflate，再逐个 `addTab`。
2. **建 Tab 与绑定点击**：`addTab` 从实体取标题、写入 `tv_tab_icon` 与 `tv_tab_title`，注册 `OnClickListener`；点击时若位置变化走 `setCurrentTab` + `onTabSelect`，否则只回调 `onTabReselect`。
3. **样式刷新**：`updateTabStyles` 统一设置 padding、文字大小、选中/未选中颜色、粗体与图标尺寸，图标边距按 gravity 分配到对应方向。
4. **选中动画**：`setCurrentTab` 记录 `mLastTab`，先 `updateTabSelection`、再让 `FragmentChangeManager` 切页；开了动画则 `calcOffset` 把新旧 Tab 的 left/right 组成 `IndicatorPoint`，交给 `ValueAnimator` + `PointEvaluator` 过渡，`onAnimationUpdate` 里改写 `mIndicatorRect` 并 `invalidate`。
5. **自绘**：`onDraw` 依次画 divider、underline，再按 `mIndicatorStyle` 分流——三角形用 `Path`、色块用整块 `GradientDrawable`、常规用底部横条；`indicatorWidth < 0` 时宽度取整个 Tab，否则居中取固定宽度。

## 设计要点

- **自绘而非子 View**：构造函数里 `setWillNotDraw(false)`、`setClipChildren(false)`，指示器就是一个 `Rect`，省掉一层 View 与布局开销。
- **配置先行**：所有属性在 `obtainAttributes` 一次性读入 `tl_*` 属性，并给出随 `indicatorStyle` 变化的默认值（三角形高 4dp、色块高 -1 表示铺满）。
- **三种控件同构**：都持有 `LinearLayout mTabsContainer` + `Rect mIndicatorRect` + `GradientDrawable mIndicatorDrawable`，差异只在容器类型与偏移来源（动画器 / ViewPager 偏移 / 分段滑块）。
- **Fragment 用 show/hide**：`FragmentChangeManager` 初始化时把所有 Fragment `add` 后 `hide`，切换只做 `show`/`hide`，保留各自视图状态。
- **可省可用的耦合**：只有调用 `setTabData(entitys, fa, containerId, fragments)` 这类重载时才创建 `FragmentChangeManager`，不传就完全不用 Fragment。
- **角标策略内置**：`UnreadMsgUtils.show` 用密度换算 5dp 圆点、18dp 圆形、两位数圆角矩形、三位以上 "99+"。
- **滚动居中**：`SlidingTabLayout.scrollToCurrentTab` 用"当前 Tab 的 left + 偏移量 − 半屏宽 + 半 Tab 宽"算出目标 `scrollX`，与本帧比较避免重复 `scrollTo`；`onPageScrolled` 每次先滚再 `invalidate`，因此手指拖动时指示器与标题同步移动。
- **状态保存**：`SlidingTabLayout` 在 `onSaveInstanceState` 里存 `mCurrentTab`，恢复时若子 View 已就绪则补一次 `updateTabSelection` 与 `scrollToCurrentTab`，避免旋转后 Tab 停在首位。

## 阅读路径

1. `FlycoTabLayout_Lib/src/main/java/com/flyco/tablayout/listener/CustomTabEntity.java`（数据契约）
2. `.../CommonTabLayout.java` :39-330（字段、属性解析、建 Tab、样式）
3. `.../CommonTabLayout.java` :326-499（偏移计算、动画、onDraw、setCurrentTab）
4. `.../SlidingTabLayout.java` :173-340（ViewPager 绑定与滚动居中）
5. `.../SlidingTabLayout.java` :382-527（偏移插值与绘制）、:851-886（`InnerPagerAdapter`）
6. `.../SegmentTabLayout.java` :164-300、:349-404（分段数据与滑块绘制）
7. `.../utils/FragmentChangeManager.java`（页面切换）
8. `app/src/main/java/com/flyco/tablayoutsamples/ui/CommonTabActivity.java`（8 个 Tab 配置差异的集中演示）

## 版本注记

该 commit 的 library 声明 `versionName = '1.0.0'`、`compileSdkVersion = 29`、`buildToolsVersion = 29.0.2`，但 `CHNAGELOG.md` 已记录到 `Version 3.0.0 (2021.09.30)`——正是"AndroidX & publish on Maven Central"那一次；源码里 `import androidx.fragment.app.*`、`androidx.viewpager.widget.ViewPager` 也印证了这次迁移。属性名仍是 2015 年就定下的 `tl_*` 前缀，没有跟随版本重命名；示例工程全部为 Java。阅读时以该 commit 为准。
