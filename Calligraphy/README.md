# Calligraphy 源码解析

## 一句话定位

Calligraphy 是一个「零侵入换字体」的 Android 库：它不要求你改用自定义 TextView，而是替换 `LayoutInflater`，在每次 View 创建完成的回调里，把 assets 中的字体设置到所有 `TextView` 上。

## 源码快照

- 上游：<https://github.com/chrisjenx/Calligraphy>
- commit：`085e441954d787bd4ef31f245afe5f2f2a311ea5`（2018-05-22，Merge PR #453）
- 版本：`gradle.properties` 中 `VERSION_NAME=2.3.0`、`VERSION_CODE=24`
- 模块：`calligraphy`（库，包 `uk.co.chrisjenx.calligraphy`，10 个 Java 文件约 1500 行）与 `CalligraphySample`（示例 App）
- 资源仅 `attrs.xml`（声明 `fontPath`）、`ids.xml`（`calligraphy_tag_id`）与 `public.xml`

`CalligraphyConfig` 的静态块预先注册了 8 个标准控件（`TextView`、`Button`、`EditText`、`AutoCompleteTextView`、`MultiAutoCompleteTextView`、`CheckBox`、`RadioButton`、`ToggleButton`）对应的 style 属性；运行时通过 `canAddV7AppCompatViews()` 探测类路径后，再追加 8 个 AppCompat 版本，因此「类 → style 属性」这张表是库能对未知子类做主题回退的根本。

## 核心类

| 类 | 职责 | 关键方法（行号） |
|---|---|---|
| `CalligraphyConfig` | 全局不可变配置，持有默认字体路径、attrId 与类到 style 的映射 | `initDefault`:72、`get`:80、构造:120 |
| `CalligraphyConfig.Builder` | 配置装配，默认 `attrId = R.attr.fontPath` | `setDefaultFontPath`:228、`addCustomViewWithSetTypeface`:309、`build`:315 |
| `CalligraphyContextWrapper` | 接入点：从 `getSystemService` 换出注入版 inflater | `wrap`:30、`onActivityCreateView`:58、`getSystemService`:107 |
| `CalligraphyLayoutInflater` | 注入主体：包装 Factory/Factory2 与私有 Factory | `setFactory2`:94、`onCreateView`:147、`createCustomViewInternal`:188 |
| `CalligraphyFactory` | 单个 View 的字体注入逻辑 | `onViewCreated`:108、`resolveFontPath`:181、`applyFontToToolbar`:203 |
| `CalligraphyUtils` | 字体路径逐层解析与 apply | `applyFontToTextView`:55、`pullFontPathFromView`:157、`pullFontPathFromTheme`:252 |
| `TypefaceUtils` | Typeface 与 span 的静态缓存 | `load`:31、`getSpan`:54、`isLoaded`:72 |
| `HasTypeface` | 自定义 View 的字体契约 | `setTypeface`:20 |
| `CalligraphyTypefaceSpan` | 局部文本换字体的 Span | `apply`:29 |

## 关键流程

**1. 接入：换掉 LayoutInflater。** Activity 在 `attachBaseContext` 里调用 `CalligraphyContextWrapper.wrap(newBase)`（示例 `MainActivity` 63-66 行）。`getSystemService`（107-116 行）对 `LAYOUT_INFLATER_SERVICE` 返回 `CalligraphyLayoutInflater`，此后该 Activity 的所有 `inflate` 都走注入版。

**2. 拦截：包装 Factory 链。** 构造时 `setUpLayoutFactories` 先试 `Factory2`（HC+）再试 `Factory`；重写的 `setFactory`/`setFactory2` 把外部 Factory 包成 `WrapperFactory`/`WrapperFactory2`，回调原实现后再应用字体。HC+ 还会反射注入 `PrivateWrapperFactory2` 到 `LayoutInflater.setPrivateFactory`，以捕获 Activity 自定义 View 的创建。

**3. 分发：`onViewCreated`。** 所有创建出口统一进 `CalligraphyFactory.onViewCreated`（108 行）：先用 `R.id.calligraphy_tag_id` 打标记避免重复处理；`TextView` 走字体解析，`Toolbar` 走 `applyFontToToolbar`，`HasTypeface` 或注册过的自定义 View 走接口/反射 `setTypeface`。这里还有一条快速路径——若 `TypefaceUtils.isLoaded(view.getTypeface())` 说明字体已被处理过，直接返回，主要应对 Toolbar 里稍后才 inflate 的标题 TextView。

对于完全自定义的 View，`LayoutInflater` 不会回调 `onCreateView`，所以 `CalligraphyLayoutInflater.createCustomViewInternal`（188 行）用反射拿到父类私有的 `mConstructorArgs`，把第 0 位换成当前 `viewContext` 后再调 `createView(name, null, attrs)`，结束再还原，从而让自定义控件的构造参数与原生行为一致；`disableCustomViewInflation()` 可关掉这一步换性能。

**4. 解析：路径逐层回退。** `resolveFontPath`（181 行）依次尝试 xml 属性、style、`TextAppearance`；全空时 `getStyleForTextView` 先判 ActionBar 标题/副标题特例，再按 `getClassStyles()` 取类对应 style，用 `pullFontPathFromTheme` 读主题里的 `fontPath`；最后退回 `CalligraphyConfig.getFontPath()` 的默认路径。

**5. 应用：缓存 Typeface。** `TypefaceUtils.load`（31 行）以文件路径为 key 缓存，规避旧版本重复 `createFromAsset` 的泄漏，失败也写 null 防重复告警。`CalligraphyUtils.applyFontToTextView`（55 行）对需要延迟的 ActionBar 标题挂 `TextWatcher`，等系统真正写字后再 set。

## 设计要点

1. **不改继承体系**：相比自定义 TextView 方案，第三方控件与既有 XML 全都不用动。
2. **包装而非替换 Factory**：`WrapperFactory2` 保留宿主已有 Factory 行为，兼容 ButterKnife 之类同样接管 inflate 的库。
3. **反射只用在两处**：`setPrivateFactory` 与自定义 View 的构造上下文修补，其余全走公开 API，这是它能覆盖多版本的关键。
4. **配置不可变**：构造函数用 `unmodifiableMap`/`unmodifiableSet` 冻结映射，避免运行时被意外修改。
5. **tag 去重**：同一个 View 只处理一次，防止 Factory 链与私有 Factory 双重回调导致重复设置。
6. **保留手动后路**：`disablePrivateFactoryInjection()` 关掉反射注入后，宿主可在 `Activity.onCreateView` 里调 `CalligraphyContextWrapper.onActivityCreateView`，用公开回调补上同样的注入，库本身不强制反射。

## 阅读路径

1. `calligraphy/src/main/java/uk/co/chrisjenx/calligraphy/CalligraphyConfig.java`
2. `calligraphy/src/main/java/uk/co/chrisjenx/calligraphy/CalligraphyContextWrapper.java`
3. `calligraphy/src/main/java/uk/co/chrisjenx/calligraphy/CalligraphyLayoutInflater.java`
4. `calligraphy/src/main/java/uk/co/chrisjenx/calligraphy/CalligraphyFactory.java`
5. `calligraphy/src/main/java/uk/co/chrisjenx/calligraphy/CalligraphyUtils.java`
6. `calligraphy/src/main/java/uk/co/chrisjenx/calligraphy/TypefaceUtils.java`
7. `CalligraphySample/src/main/java/uk/co/chrisjenx/calligraphy/sample/MainActivity.java` 与同目录 `CalligraphyApplication.java`

## 版本注记

以该 commit 为准：2.3.0 是 1.x 大重构后的最终形态（2.0.0 起改为 Builder + Factory 注入模型），对 `support-v7` 的 `AppCompatTextView`、`Toolbar` 用 `Class.forName` 反射探测，因此不带 v7 依赖也能编译。整体仍是 AppCompat 时代实现，未覆盖 AndroidX 与新版本 TextView 特性；上游此后基本停更。
