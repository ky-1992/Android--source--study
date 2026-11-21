# Android-Bootstrap 源码解析

## 一句话定位
用运行时生成的 Drawable 把 Bootstrap 的按钮、输入框、标签样式搬到 Android 自定义 View 上，可配置的主轴只有「品牌色」与「尺寸倍数」两组，其余视觉细节由代码按状态推导。

## 源码快照
- 上游：<https://github.com/Bearded-Hen/Android-Bootstrap>
- commit：`b3d62cc1847e26d420c53c92665a4fe1e6ee7ecf`
- 结构：`AndroidBootstrap` 模块的 `com.beardedhen.androidbootstrap` 包内直接放 `BootstrapButton`、`BootstrapEditText`、`BootstrapText` 等控件；`api/attributes` 与 `api/defaults` 放品牌、尺寸、按钮模式与组内位置枚举；`api/view` 是各控件共用的能力接口；`font` 包放 FontAwesome、Typicon、MaterialIcons 三套图标的 `IconSet` 与 `AwesomeTypefaceSpan`；`utils` 提供颜色、尺寸、背景辅助；`res/values/attrs.xml` 声明全部自定义属性，`colors.xml` 存放七种品牌的基础色。

## 核心类

| 类 | 职责 | 关键方法与行号 |
|---|---|---|
| `BootstrapButton` | 圆角/描边/成组按钮，支持 Toggle、Checkbox、Radio | `initialise` L90、`updateBootstrapState` L163、`onTouchEvent` L197 |
| `AwesomeTextView` | 按钮与图标文本的共同基类 | `initialise` L77、`setIcon` L212、`updateBootstrapState` L250 |
| `BootstrapEditText` | 带品牌描边的输入框，默认文字垂直居中 | `initialise` L58、`updateBootstrapState` L116 |
| `BootstrapText` | 可含图标 Span 的 SpannableString | `Builder.addIcon` L105、`Builder.build` L114 |
| `BootstrapDrawableFactory` | 运行时拼装控件背景 | `bootstrapButton` L45、`setupStateDrawable` L229、`setupDrawableCorners` L269 |
| `BootstrapBrand` / `DefaultBootstrapBrand` | 品牌取色协议与七种默认枚举 | `fromAttributeValue` L44、`defaultFill` L65 |
| `BootstrapSizeView` / `DefaultBootstrapSize` | 缩放协议与 0.70~1.60 系数 | `scaleFactor` L32 |
| `BootstrapButtonGroup` | 成组下发样式、处理单选互斥 | `updateBootstrapGroup` L140、`onRadioToggle` L209 |
| `BootstrapDropDown` | 继承 `AwesomeTextView` 的下拉控件 | `setExpandDirection` L376、箭头绘制调用 L259 |
| `BootstrapBadge` | 用 `ImageView` 承载的角标 | `getBadgeDrawable` L81、`setBadgeText` L89 |

## 关键流程
1. **属性解析**：构造函数把 `AttributeSet` 交给 `initialise`，用 `obtainStyledAttributes` 读 `bootstrapBrand`、`bootstrapSize`、`roundedCorners`、`showOutline` 等序号，再经 `fromAttributeValue` 与 `scaleFactor` 换成枚举和 float；序号缺失时分别回落到 `MD` 与 `REGULAR`。
2. **状态计算**：`updateBootstrapState` 先取基线字号、内边距、描边、圆角资源，统一乘以尺寸倍数，再算出文字颜色，并调工厂生成背景。
3. **背景生成**：`bootstrapButton` 用三个 `GradientDrawable` 分别表示默认/激活/禁用态；`setupDrawableCorners` 按 `ViewGroupPosition` 决定保留哪几个圆角；`setupStateDrawable` 用负 inset 隐藏相邻描边，再按 pressed、focused、disabled 挂成 `StateListDrawable`。
4. **应用与变更**：工厂结果经 `ViewUtils.setBackgroundDrawable` 写回；`setShowOutline`、`setBootstrapBrand`、`setBootstrapSize` 都会重走 `updateBootstrapState`，旋转后由 `onSaveInstanceState` 恢复这些属性。
5. **成组联动**：`BootstrapButtonGroup` 在子 View 增删时重算每个按钮的 `ViewGroupPosition`，并统一下发品牌、尺寸、按钮模式。
6. **徽标与下拉**：`BootstrapButton` 的 `badgeText` 属性会创建 `BootstrapBadge`，`displayBadgeDrawable` 再用 `setCompoundDrawablesWithIntrinsicBounds` 把角标挂到右侧；`BootstrapDropDown` 按 `ExpandDirection` 决定箭头朝向与弹窗展开方向。

## 设计要点
- 不写 XML 背景：Drawable 全部由代码生成，七种品牌 × 五档尺寸不需要几百份 selector。
- 能力按接口拆分：`BootstrapBrandView`、`BootstrapSizeView`、`RoundableView`、`OutlineableView` 让控件只实现自己需要的能力，按钮组则通过接口批量设置子按钮。
- 视觉状态由颜色运算得到：`DefaultBootstrapBrand` 用 `decreaseRgbChannels`、`increaseOpacity` 从单色推出描边色、激活色与禁用色。
- 组内位置驱动绘制：`ViewGroupPosition` 让中间按钮只画一条边，避免相邻描边叠加变粗。
- 图标文本用 Span：`BootstrapText.Builder` 记录每个图标插入的位置，`build` 时在对应字符上挂 `AwesomeTypefaceSpan`。
- 状态自行保存：各控件用 `Bundle` 加接口里的 `KEY` 常量保存圆角、描边、尺寸、品牌，旋转重建后属性不丢。
- 工厂方法粒度细：按钮、输入框、标签、缩略图、警告条、角标各有独立入口，同一条状态机规则复用。

## 阅读路径
1. `AndroidBootstrap/src/main/res/values/attrs.xml`
2. `AndroidBootstrap/src/main/java/com/beardedhen/androidbootstrap/BootstrapButton.java`
3. `AndroidBootstrap/src/main/java/com/beardedhen/androidbootstrap/BootstrapEditText.java`
4. `AndroidBootstrap/src/main/java/com/beardedhen/androidbootstrap/BootstrapText.java`
5. `AndroidBootstrap/src/main/java/com/beardedhen/androidbootstrap/BootstrapDrawableFactory.java`
6. `.../api/defaults/DefaultBootstrapBrand.java` 与 `DefaultBootstrapSize.java`
7. `AndroidBootstrap/src/main/java/com/beardedhen/androidbootstrap/BootstrapButtonGroup.java`

## 版本注记
以 commit `b3d62cc` 为准。该版本仍使用 `android.support.annotation` 与旧式 Gradle 多模块结构，未迁移到 AndroidX；README 中的类名、方法与行号均取自该 commit。
