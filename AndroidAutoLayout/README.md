# AndroidAutoLayout 源码解析

## 一句话定位

AndroidAutoLayout 让开发者按设计稿的 px 直接写布局，运行期把每个尺寸、间距、字号按「屏幕尺寸 / 设计稿尺寸」等比缩放：`AutoLayoutActivity` 把系统三个布局类换成 Auto 版容器，容器在测量前统一改写子 View 的属性。

## 源码快照

- 上游：<https://github.com/hongyangAndroid/AndroidAutoLayout>
- commit：`d0045f33d2a944bebbc07ab3d54a1c19b9d2963b`（2018-11-26，提交信息 *Update README.md*）
- 仓库：`autolayout`（核心库，包名 `com.zhy.autolayout`）、`autolayout-widget`（`AutoTableLayout`、`AutoToolbar`、`AutoCardView` 等支持库控件扩展）、`sample` 与 `widgetsample` 示例
- 依赖 Android Support Library（`android.support.v7.app.AppCompatActivity`），不是 AndroidX

## 核心类

| 类 | 职责 | 关键方法（行号） |
|---|---|---|
| `AutoLayoutActivity` | 把系统容器替换成 Auto 容器 | `onCreateView` L21 |
| `AutoFrameLayout` / `AutoLinearLayout` / `AutoRelativeLayout` | 持 `AutoLayoutHelper`，测量前调 `adjustChildren` | `onMeasure` L58 / L39 / L59、`generateLayoutParams` L52 / L55 / L53、`LayoutParams` 构造 L78 / L66 / L78 |
| `AutoLayoutHelper` | 解析 XML 中的 px 属性、统一驱动子 View | 构造 L97、`adjustChildren` L115、`getAutoLayoutInfo` L137 |
| `AutoLayoutInfo` | 一个 View 的属性集合 | `addAttr` L32、`fillAttrs` L38、`getAttrFromView` L47 |
| `AutoAttr` | 单个 px 属性的缩放抽象 | `apply` L46、`getPercentWidthSize` L84、`execute`（抽象）L114 |
| `WidthAttr` / `TextSizeAttr` | 分别写回宽高与字号 | `execute` L29 / L31、`generate` L35 / L39 |
| `AutoUtils` | 百分比换算与手动 `auto(view)` | `auto` L22、`getPercentWidth1px` L90、`getPercentWidthSizeBigger` L113、`getPercentHeightSizeBigger` L129 |
| `AutoLayoutConifg` | 设计稿与屏幕尺寸单例 | `getInstance` L51、`init` L78、`getMetaData` L88、`checkParams` L35 |

## 关键流程

1. **替换容器**：`AutoLayoutActivity.onCreateView` 判断 `name` 是不是 `LinearLayout`、`FrameLayout`、`RelativeLayout`，命中就 `new AutoFrameLayout/AutoLinearLayout/AutoRelativeLayout`，其余原样交给 `super.onCreateView`。三个名字是写死的常量，`AutoRelativeLayout` 与另外两个容器一样，只在 `onMeasure` 里插入缩放步骤，`onLayout` 仅做一次转发。
2. **解析属性**：Auto 容器重写 `generateLayoutParams`，返回自己的 `LayoutParams`；构造函数里调用 `AutoLayoutHelper.getAutoLayoutInfo(c, attrs)`，先用 `R.styleable.AutoLayout_Layout` 取 `layout_auto_basewidth` / `layout_auto_baseheight`，再遍历 `LL` 数组里的 17 个系统属性，仅保留 `DimenUtils.isPxVal` 为真的 px 值，最后按属性下标 `new` 出对应的 `TextSizeAttr`、`PaddingAttr`、`WidthAttr`、`MarginAttr` 等对象存进 `AutoLayoutInfo`。`LL` 覆盖 `textSize`、`padding` 系列、`layout_width/height`、`layout_margin` 系列与 `max/min` 尺寸，`wrap_content` 这类非 px 值会被过滤掉。
3. **测量前统一填属性**：`onMeasure` 里先判断 `!isInEditMode()`，再 `mHelper.adjustChildren()`；`adjustChildren` 遍历子 View，若其 `LayoutParams` 实现了 `AutoLayoutHelper.AutoLayoutParams` 就取 `AutoLayoutInfo` 并 `fillAttrs(view)`。这一步发生在 `super.onMeasure` 之前，所以系统拿到的已经是换算后的尺寸。
4. **逐个换算**：`AutoLayoutInfo.fillAttrs` 依次调用 `autoAttr.apply(view)`；`AutoAttr.apply` 依据 `baseWidth` / `baseHeight` 位掩码判断用宽还是高基准，然后调 `AutoUtils.getPercentWidthSizeBigger` 或 `getPercentHeightSizeBigger`，并对结果做 `Math.max(val, 1)` 保护，最后执行子类 `execute` 写回。
5. **写回与测量**：`WidthAttr.execute` 直接改 `view.getLayoutParams().width`，`TextSizeAttr.execute` 先 `setIncludeFontPadding(false)` 再 `setTextSize(COMPLEX_UNIT_PX, val)`；属性改完才 `super.onMeasure`，系统按新尺寸布局。所有写回都发生在父容器的 `onMeasure` 里，子 View 随后的测量直接读新值，因此不需要额外 `requestLayout`。
6. **手动补偿**：不在 Auto 容器里的 View 可以在代码里调 `AutoUtils.auto(view)`，内部走 `AutoLayoutInfo.getAttrFromView` 从当前 `LayoutParams`、padding、margin、字号反推属性，再 `fillAttrs` 一次；`autoed(view)` 用 `R.id.id_tag_autolayout_size` 打标，避免同一 View 被重复处理。

## 设计要点

- **换容器而不改 View 树**：用 `onCreateView` 这一个入口完成替换，业务布局文件几乎不用改，成本集中在三个 Auto 容器与三个 `LayoutParams`。
- **属性挂在 LayoutParams 上**：`AutoLayoutParams` 接口让 helper 能统一拿到每个子 View 的 `AutoLayoutInfo`，测量阶段批量处理，避免了每个 View 各自 hook。
- **一类属性一个 AutoAttr 子类**：宽高基准用 `layout_auto_basewidth` / `layout_auto_baseheight` 指定，默认为「宽属性按宽、高属性按高」，`TextSizeAttr` 则默认按高。
- **取整向上偏保守**：`getPercentWidthSizeBigger` 用整数除法，余数不为 0 时 `+1`；再叠加 `Math.max(val, 1)`，防止细分割线和小间距被压成 0。
- **设计稿尺寸来自 manifest**：`AutoLayoutConifg` 读取 meta-data 的 `design_width` / `design_height`，`checkParams` 缺失时直接抛 `RuntimeException`，把配置错误提前暴露。
- **手动补偿口子**：`AutoUtils.auto(view)` 等静态方法走 `AutoLayoutInfo.getAttrFromView`，可从现有 `LayoutParams` / padding / margin 反推属性，用于不在 Auto 容器里的 View；`autoed(view)` 用 `R.id.id_tag_autolayout_size` 打标，避免重复处理。需要注意 `getAttrFromView` 的 `PADDING_LEFT` 等分支复用了 `MarginLeftAttr` 一类实现，这是源码里的已知瑕疵，读时留意。
- **三个容器同一套结构**：`AutoFrameLayout`、`AutoLinearLayout`、`AutoRelativeLayout` 都是「一个 helper + 自己的 LayoutParams + onMeasure 里先 adjustChildren」，读通一个就能推出另外两个；同模块 `widget/MetroLayout` 也持有 `AutoLayoutHelper`（源码注释标了 do not use），可见这套结构是可复用的。
- **异常前置**：设计稿尺寸没配时 `checkParams` 直接抛异常，而不是静默按 1:1 处理；`isInEditMode` 则跳过缩放，保证 Android Studio 预览正常。
- **屏幕尺寸可切换口径**：`AutoLayoutConifg.init` 调用 `ScreenUtils.getScreenSize(context, useDeviceSize)`，`useDeviceSize` 决定是否包含状态栏与导航栏，方便需要「整屏铺满」的页面单独配置。

## 阅读路径

1. `autolayout/src/main/java/com/zhy/autolayout/AutoLayoutActivity.java`
2. `autolayout/src/main/java/com/zhy/autolayout/AutoFrameLayout.java`
3. `autolayout/src/main/java/com/zhy/autolayout/AutoLinearLayout.java`
4. `autolayout/src/main/java/com/zhy/autolayout/utils/AutoLayoutHelper.java`
5. `autolayout/src/main/java/com/zhy/autolayout/AutoLayoutInfo.java`
6. `autolayout/src/main/java/com/zhy/autolayout/attr/AutoAttr.java`
7. `autolayout/src/main/java/com/zhy/autolayout/utils/AutoUtils.java`
8. `autolayout/src/main/java/com/zhy/autolayout/config/AutoLayoutConifg.java`（类名拼写即 `Conifg`，源码如此）

## 版本注记

该 commit 停留在 2018 年，仍基于 Android Support Library；迁移到 AndroidX 需要替换 `AppCompatActivity` 与支持包 import。核心缩放逻辑集中在 `AutoAttr` 与 `AutoUtils`；`AutoTableLayout`、`AutoToolbar`、`AutoCardView` 等扩展控件在 `autolayout-widget` 模块。行号与结构以该 commit 为准。
