# ViewPagerIndicator 源码解析

## 一句话定位

ViewPagerIndicator 是 Jake Wharton 写的 ViewPager 页码指示器组件集：一个 `PageIndicator` 接口约定绑定方式，六种实现（圆点、线、下划线、标题、标签、图标）负责"跟着 ViewPager 滚"的视觉反馈。整个库不到 3000 行 Java，是 Android 自定义 View 与 Canvas 绘制、`onMeasure` / `onDraw` / 触摸分发的经典教材。

## 源码快照

- 上游：<https://github.com/JakeWharton/ViewPagerIndicator>
- commit：`8cd549f23f3d20ff920e19a2345c54983f65e26b`（2012-09-12，`[maven-release-plugin] prepare release 2.4.1`）
- 版本：根 `pom.xml` 声明 `<version>2.4.1</version>`，Maven 多模块（`library` + `sample`）
- 依赖：`android.support.v4`（`ViewPager`、`ViewPager.OnPageChangeListener`），无第三方依赖

目录结构仍是 Ant/Eclipse 时代的源码布局，没有 `src/main/java`：

| 路径 | 内容 |
| --- | --- |
| `library/src/com/viewpagerindicator/` | 库源码：1 个接口 + 6 个指示器 + `IcsLinearLayout` + `IconPagerAdapter` |
| `library/res/values/` | `vpi__attrs.xml` 自定义属性、`vpi__defaults.xml` 默认值、`vpi__styles.xml` 样式、`vpi__colors.xml` 颜色 |
| `sample/src/.../sample/` | 约 30 个示例 Activity，每种指示器配默认/styled/初始页等变体 |

## 核心类

| 类名 | 职责 | 关键方法（行号取自快照） |
| --- | --- | --- |
| `PageIndicator` | 统一契约，本身继承 `ViewPager.OnPageChangeListener` | `setViewPager()` L32/L40、`setCurrentItem()` L50、`setOnPageChangeListener()` L57、`notifyDataSetChanged()` L62 |
| `CirclePageIndicator` | 圆点：当前页填充、其余描边，支持横/竖排与吸附 | 构造 L75、`onDraw()` L198、`setViewPager()` L365、`onPageScrolled()` L411、`onPageSelected()` L422、`onMeasure()` L445 |
| `LinePageIndicator` | 线段式页码，选中段与未选中段分色 | `onDraw()` L157、`setViewPager()` L277 |
| `UnderlinePageIndicator` | 单条滑动下划线，可淡出 | `onDraw()` L156（L172–178 计算矩形）、`setViewPager()` L265 |
| `TitlePageIndicator` | 左中右三标题 + 底部三角/下划线指示，支持中央标题点击 | `onDraw()` L354、`calculateAllBounds()` L371、枚举 `IndicatorStyle` L79、`LinePosition` L98 |
| `TabPageIndicator` | 标签页式标题，`HorizontalScrollView` + 内部 `TabView` | `onMeasure()` L94、`addTab()` L152、`setViewPager()` L189、`notifyDataSetChanged()` L205 |
| `IconPageIndicator` | 图标页码，读 `IconPagerAdapter` 生成 `ImageView` | `setViewPager()` L111、`notifyDataSetChanged()` L127 |
| `IcsLinearLayout` | 自带分隔线绘制的 `LinearLayout`，供 Tab/Icon 装子 View | 类 L17 |
| `IconPagerAdapter` | 图标数据契约：`getIconResId(int)` + `getCount()` | L3–10 |

类图见 [class-diagram.html](./class-diagram.html)。

## 关键流程

**绑定**：`setViewPager()` 先判断是否同一个 Pager（相同直接返回），旧 Pager 存在则 `setOnPageChangeListener(null)` 解绑；然后校验 `view.getAdapter() != null`，不满足抛 `IllegalStateException`；最后 `view.setOnPageChangeListener(this)`——指示器把自己注册成监听器，一行完成双向连接（`CirclePageIndicator` L365–378）。

**滚动回调**：`onPageScrolled()` 只做两件事——缓存 `mCurrentPage` / `mPageOffset`，再 `invalidate()` 请求重绘；`onPageSelected()` 额外判断 `mSnap || mScrollState == SCROLL_STATE_IDLE` 才更新吸附页码；`onPageScrollStateChanged()` 记录状态。三个方法末尾都会把事件原样转发给通过 `setOnPageChangeListener()` 注入的外部监听（L402–437）。

**绘制**：所有指示器都在 `onDraw()` 里按"总页数 × 页间距"布局。`CirclePageIndicator` 用 `threeRadius = mRadius * 3` 作步长，先循环画描边圆，再按 `(mSnap ? mSnapPage : mCurrentPage) * threeRadius + mPageOffset * threeRadius` 画填充圆（L230–278）；`UnderlinePageIndicator` 用 `pageWidth * (mCurrentPage + mPositionOffset)` 算出滑动矩形（L172–178）。

**触摸翻页**：`CirclePageIndicator.onTouchEvent()` 用 `mTouchSlop` 判定拖动，进入拖动态后调用 `mViewPager.beginFakeDrag()` / `fakeDragBy(deltaX)`，让指示器区域也能拖 Pager（L281–361）；`TitlePageIndicator` 另有中央标题点击回调 `OnCenterItemClickListener`（L607–608）。

**样式读取**：每个指示器构造时先从 `res` 读默认值，再用 `context.obtainStyledAttributes(attrs, R.styleable.XxxPageIndicator, defStyle, 0)` 覆盖，最后 `a.recycle()`；颜色、半径、线宽、淡出时长都能在 XML 里配。

流程图见 [flow-diagram.html](./flow-diagram.html)，覆盖初始化 → 绑定 → 注册监听 → 滑动回调 → 重绘。

## 设计要点

- **接口即监听器**：`PageIndicator extends ViewPager.OnPageChangeListener`，避免再定义一个监听接口，绑定语义天然成立；代价是每个实现都要自己实现三个回调并做状态缓存。
- **状态缓存 + invalidate**：回调不直接画，只更新 `mCurrentPage` / `mPageOffset` 并请求重绘，保证一帧只画一次，也让 `onDraw` 成为唯一渲染入口。
- **两类实现路线**：Circle/Line/Underline/Title 继承 `View` 用 Canvas 画；Tab/Icon 继承 `HorizontalScrollView` 用子 View 排布，所以它们额外实现了点击反推页码、`animateToTab()` 平滑滚动、`onMeasure` 里限制单 Tab 最大宽度（L94–118）。
- **触摸透传**：指示器区域滑动通过 `fakeDrag` 转交给 ViewPager，用户不用精确点中 Pager。
- **配置全部走自定义属性**：`vpi__attrs.xml` + `vpi__defaults.xml`，默认值可被 theme/style 覆盖，同样式可复用到多个指示器。
- **图标数据靠强转**：`IconPageIndicator.notifyDataSetChanged()` 直接把 `getAdapter()` 强转成 `IconPagerAdapter`，用 `getIconResId()` 建 ImageView，简单但要求 adapter 必须实现该接口。

## 阅读路径

1. `library/src/com/viewpagerindicator/PageIndicator.java` —— 先看契约（L26–62）。
2. `library/src/com/viewpagerindicator/CirclePageIndicator.java` —— 完整样本：构造读样式 L75、`onDraw()` L198、绑定 L365、回调 L402、触摸 L281。
3. `library/src/com/viewpagerindicator/UnderlinePageIndicator.java` —— 最简 `onDraw` 与淡出 Runnable（L60–71）。
4. `library/src/com/viewpagerindicator/TitlePageIndicator.java` —— 标题测量、裁剪与枚举样式（L79–115、L354–450）。
5. `library/src/com/viewpagerindicator/TabPageIndicator.java` —— 子 View 路线与 `animateToTab()`（L120–164、L189–260）。
6. `library/src/com/viewpagerindicator/IconPageIndicator.java` + `IconPagerAdapter.java` —— 图标强转契约。
7. `library/res/values/vpi__attrs.xml`、`vpi__defaults.xml` —— 属性名与默认值。
8. `sample/src/com/viewpagerindicator/sample/SampleCirclesDefault.java` —— 最小接入示例（L9–20）。

## 版本注记

该 commit 是 `2.4.1` 发布准备提交，属于 2.x 代码线：依赖 `android.support.v4` 的 `ViewPager.OnPageChangeListener`（AndroidX 时代该接口已废弃，迁移需改 `ViewPager2.OnPageChangeCallback`），源码布局为 Eclipse/Ant 风格，构建用 Maven + `android-maven-plugin`。后续 2.4.2 起逐步迁移到 Gradle，如需精确对应发布版本，以该 commit 为准。
