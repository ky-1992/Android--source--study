# BottomBar 源码解析

## 一句话定位

BottomBar 是 roughike 的 Android 底部导航自定义 View：用一个 `LinearLayout` 承载若干 `BottomBarTab`，tab 由 XML 声明、`TabParser` 解析，支持 fixed / shifting / tablet 三种排布，以及选中动画、角标、滚动隐藏（shy）等衍生能力。库代码约 2900 行，是"自定义 ViewGroup + 属性动画"的典型读本。

## 源码快照

- 上游：<https://github.com/roughike/BottomBar>
- commit：`885d70e62359d15ed17dfd77a28809702d6ae513`（2020-09-02）
- 版本：`bottom-bar/build.gradle` 中 `libraryVersion = '2.3.1'`，`CHANGELOG.md` 首条亦为 2.3.1
- 工程结构：`bottom-bar/` 库模块；`app/` 示例（含 ThreeTabs、FiveColorChangingTabs、Badge 等 Activity）；`graphics/` 宣传素材
- 构建基线：`compileSdkVersion 25`、`minSdkVersion 11`、`supportLibraryVersion 25.3.0`

## 核心类

| 类 | 职责 | 关键成员 / 行号 |
| --- | --- | --- |
| `BottomBar` | 容器：读属性、装配 tab、处理点击与动画、存选中态 | `init` L136-150、`populateAttributes` L185-218、`setItems` L300-311、`updateItems` L327-381、`resizeTabsToCorrectSizes` L383-420、`handleClick` L913-927 |
| `BottomBarTab` | 单个 tab 的自绘与动画（icon + title） | `prepareLayout` L92-115、`select` L384-404、`deselect` L406-431、`updateWidth` L469-508 |
| `TabParser` | 用 `XmlResourceParser` 把 tab XML 解析成 `BottomBarTab` | `parseTabs` L77-96、`parseNewTab` L99-149、`getColorValue` L167-180 |
| `BottomBarTab.Config` | tab 外观默认值（alpha、颜色、字体） | 构造 L654-664、`Builder` L666-725 |
| `BottomBarTab.Type` | 排布枚举：`FIXED` / `SHIFTING` / `TABLET` | L639-641 |
| `TabSelectionInterceptor` | 切换前拦截，返回 true 可吞掉这次选中 | `shouldInterceptTabSelection` L32 |
| `BottomBarBadge` | 数字角标：显隐、定位、包一层 `BadgeContainer` | `setCount` L45、`show` L62、`hide` L75、`attachToTab` L94 |
| `ShySettings` | shy 模式下 show/hide 的公开入口 | `showBar` L21、`hideBar` L28 |
| `BottomNavigationBehavior` | `CoordinatorLayout.Behavior`：滚动隐藏、Snackbar 避让 | `onDirectionNestedPreScroll` L67、`setHidden` L104 |
| `BatchTabPropertyApplier` | 把某个属性批量下发给所有 tab | `applyToAllTabs` L16-24 |

类图见 [class-diagram.html](./class-diagram.html)，图中省略了 `BadgeContainer`、`BadgeCircle`、`VerticalScrollingBehavior` 之外的辅助类与监听接口。

## 关键流程

**解析与装配**：`init` 先 `populateAttributes` 读 `bb_tabXmlResource` 等自定义属性（L185-218），再 `initializeViews` 建外框与 `tabContainer`；`setItems` 用 `getTabConfig()` 组装默认 `Config`，交给 `TabParser.parseTabs()`（L300-311）。解析时按 `type` 选布局资源（fixed/shifting/tablet，`BottomBarTab` L118-135），并按属性名分发 id、icon、title、各种颜色、`iconOnly` 等（`TabParser` L99-149）；XML 里没写的属性由 `Config` 的默认值兜底，tab 只需声明自己用到的属性。

**宽度计算**：`updateItems` 把每个 tab `prepareLayout()`、按当前选中态 `select(false)` / `deselect(false)`，随后 `resizeTabsToCorrectSizes` 以屏宽均分得到 `proposedItemWidth`，再算出 `inActiveShiftingItemWidth = 90%`、`activeShiftingItemWidth = 100% + 其余项各 10%` 的预算（L383-420）；非 tablet 模式下宽度写回 `LayoutParams`。

**点击切换**：`onClick` 转给 `handleClick`（L890-927），先问 `TabSelectionInterceptor` 是否拦截；否则旧 tab `deselect(true)`、新 tab `select(true)`，再 `shiftingMagic` 变宽、`handleBackgroundColorChange` 换背景、`updateSelectedTab` 回调监听并记下 `currentTabPosition`。

**动画细节**：`select`/`deselect` 用 150ms 的 `ValueAnimator` 做颜色渐变（`animateColors` L433-446），用 `ViewCompat.animate` 做标题缩放、图标 alpha 与顶部 padding（L538-570）；`shiftingMagic` 对旧/新 tab 分别 `updateWidth(...,true)`，`updateWidth` 用 `ValueAnimator` 逐帧改 `LayoutParams.width`，并用 `postDelayed` 规避 `onAnimationEnd` 的旧坑（L469-508）。背景色在 5.0+ 用 `ViewAnimationUtils.createCircularReveal`，低版本走 `backgroundCrossfadeAnimation`（BottomBar L991-1027）。

流程图见 [flow-diagram.html](./flow-diagram.html)，覆盖配置 → 解析 → 装配 → 点击 → 选中/变宽/背景动画。

## 设计要点

- **XML 单一数据源**：tab 的声明集中在 `bb_tabXmlResource` 指向的 XML，`TabParser` 只负责把属性翻译成 setter 调用，新增属性只要加一条 `case` 和一个 `@StringDef` 常量（L182-204）。
- **behavior 位掩码而非子类**：`BEHAVIOR_SHIFTING/SHY/DRAW_UNDER_NAV/ICONS_ONLY` 用 `hasBehavior` 按位判断（L242-244），`isShiftingMode` 还额外排除 tablet（L220-222），避免为每种组合派生子类。
- **没有独立的 ShiftingMode 类**：本 commit 里 shifting 由行为位 `BEHAVIOR_SHIFTING` 与 `BottomBarTab.Type.SHIFTING` 共同表达（L220-222、L338-344）；对照旧版本资料时按这个映射理解。
- **Type 决定布局资源**：`FIXED` 用固定 item、`SHIFTING` 用带 spacer 的 item、`TABLET` 用横排 item（`BottomBarTab` L118-135），因此同一个类在不同模式下长得完全不同。
- **动画常量集中**：`ANIMATION_DURATION = 150`（L41）统一了配色、宽度、标题、badge 的时长；`BottomBarBadge` 的 show/hide 也复用它。
- **shy 模式解耦**：滚动隐藏完全交给 `BottomNavigationBehavior`（继承 `VerticalScrollingBehavior`），BottomBar 只负责在布局完成后 `setBehavior` 并把高度传进去（L829-832），View 本身不写滚动逻辑。

## 阅读路径

1. `bottom-bar/src/main/java/com/roughike/bottombar/BottomBar.java` —— 先读属性与 `init`（L136-244），再读 `setItems`/`updateItems`（L292-420）。
2. `TabParser.java` —— XML 属性到 tab 的映射（L51-204）。
3. `BottomBarTab.java` —— `prepareLayout`（L92-115）与 `Type`/`Config`（L639-725）。
4. `BottomBarTab.java` 的 `select`/`deselect`/`updateWidth`（L384-508）—— 动画主战场。
5. `BottomBar.java` 的 `selectTabAtPosition`/`handleClick`/`shiftingMagic`（L552-567、L913-967）。
6. `BottomBarBadge.java` —— 角标与 `BadgeContainer` 包装（L32-170）。
7. `BottomNavigationBehavior.java`、`VerticalScrollingBehavior.java` —— shy 行为（L20-175、L21-148）。

## 版本注记

该 commit 是 2020-09-02 的 `master` 快照，库版本 2.3.1（`bottom-bar/build.gradle`），构建基线 Support Library 25.3.0、minSdk 11；浅克隆未取 tag，精确版本以 commit `885d70e62359d15ed17dfd77a28809702d6ae513` 为准。本 commit 已无 `ShiftingMode` 类，shifting 语义由 behavior 位掩码与 `BottomBarTab.Type` 共同表达。
