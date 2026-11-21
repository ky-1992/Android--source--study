# Material Dialogs 源码解析

## 一句话定位

Material Dialogs 是一套遵循 Material 规范的 Android 对话框库。本次读到的 `main` 分支是 **3.x 的 Kotlin 重写版**：不再有 Java 时代的 `MaterialDialog.Builder` / `DialogBase` / `DialogWrapper`，`MaterialDialog` 自身就是配置入口，布局、主题与窗口行为被拆到 `DialogBehavior` 与三段式 `DialogLayout` 中。

## 源码快照

- 上游：https://github.com/afollestad/material-dialogs
- commit：`709cfee9b45257af7ab4c428e4850bee38a075e7`（2021-02-19，`Enable android.defaultConfig.vectorDrawables.useSupportLibrary in the sample project`）。
- 仓库 `README.md` 明确写着：自 2.x 起只支持 Kotlin，最后的 Java 版本是 `0.9.6.0` 且已不维护；模块化拆分为 `core`、`input`、`datetime`、`color`、`files`、`bottomsheets`、`lifecycle`、`sample`，其中 `core` 是全部能力的基础。
- 包结构：`com.afollestad.materialdialogs` 下是入口与行为策略；`internal/main`、`internal/message`、`internal/button`、`internal/list` 分别放布局骨架、内容区、按钮区与列表适配器；`actions`、`list`、`callbacks` 等包用 Kotlin 扩展函数挂到 `MaterialDialog` 上。

## 核心类

| 类 | 职责 | 关键方法（行号） |
|---|---|---|
| `MaterialDialog` | 继承 `Dialog`，保存配置、监听器列表与根布局，链式 API 的操作对象 | `init` 104-121、`show` 353-359、`dismiss` 397-401、`onActionButtonClicked` 403-416 |
| `DialogBehavior` | 接口，抽象主题、根视图创建、窗口尺寸、背景、显示前后钩子 | 方法声明 37-79 |
| `ModalDialog` | `object` 单例，默认行为实现 | `createView` 92-103、`setWindowConstraints` 109-140、`onPostShow` 161-171 |
| `DialogLayout` | 根 `FrameLayout`，持有 title/content/buttons 三段子布局并统一测量 | `onFinishInflate` 94-99、`attachDialog` 101-104、`onMeasure` 129-176 |
| `DialogTitleLayout` | 图标 + 标题的水平排布与分隔线绘制 | `onMeasure` 66-96、`onDraw` 152-164 |
| `DialogContentLayout` | 消息、RecyclerView 列表、自定义 View 三种内容形态 | `setMessage` 75-97、`addRecyclerView` 99-112、`addCustomView` 114-150 |
| `DialogActionButtonLayout` | 三个动作按钮与 checkbox prompt 的排布，负责堆叠降级 | `onFinishInflate` 73-86、`onMeasure` 88-156 |
| `DialogActionButton` | 单个按钮的着色、selector、大小写与对齐 | `update` 59-100、`setEnabled` 108-111 |
| `DialogAdapter` / `WhichButton` | 列表对话框适配器契约；按钮枚举与索引映射 | `DialogAdapter.kt` 18-43、`WhichButton.kt` 22-34 |

## 关键流程

### 1. 构造即装配：init 块

`MaterialDialog` 的主构造器接收 `windowContext` 与 `dialogBehavior`，并把 `inferTheme` 作为 `Dialog` 的 theme 参数（56-59 行）。`init` 块里先 `dialogBehavior.createView(...)` 生成根 `ViewGroup`，`setContentView` 之后再 `getDialogLayout` 强转成 `DialogLayout` 并 `attachDialog(this)` 把 dialog 引用回填给子布局（104-121 行）。

### 2. 三段式布局的测量

`DialogLayout.onFinishInflate` 从 XML 找到 `titleLayout`、`contentLayout`、`buttonsLayout`（94-99 行）。`onMeasure` 先量标题，再在按钮可见时量按钮，把剩余高度交给内容区，最后按 `layoutMode` 决定 `WRAP_CONTENT` 还是全屏高度（129-176 行）。`DialogActionButtonLayout.onMeasure` 会先按不堆叠排布计算总宽，超过父宽时把 `stackButtons` 置 true 并重新测量（129-148 行），这是小屏上按钮纵向堆叠的实现。

### 3. 链式配置

`title`、`message`、`positiveButton` 等方法都返回 `this`。以 `message` 为例（167-180 行），它把工作委托给 `contentLayout.setMessage`；后者会先准备 `DialogScrollView`，再 inflate 消息 `TextView`，并支持 `DialogMessageSettings` 回调做二次定制（`DialogContentLayout.kt` 75-97 行）。`positiveButton` 把点击回调追加到 `positiveListeners`，再用 `getActionButton(POSITIVE)` 拿到按钮设置文案（189-212 行）。

### 4. 显示与回调

`show` 依次设置窗口约束、触发 `preShow` 监听器、调用 `dialogBehavior.onPreShow`、`super.show()`、再执行 `onPostShow`（353-359 行）；`ModalDialog.onPostShow` 会把焦点交给可见的负向或正向按钮（161-171 行）。按钮点击统一在 `DialogActionButtonLayout.onFinishInflate` 里注册为 `dialog.onActionButtonClicked(which)`（73-86 行），`onActionButtonClicked` 用 `invokeAll` 执行对应监听器，通知列表适配器 `positiveButtonClicked`，最后按 `autoDismissEnabled` 决定是否 `dismiss`（403-416 行）。

## 设计要点

1. **行为策略外置**：主题、窗口尺寸、背景圆角这些平台相关逻辑放在 `DialogBehavior`，默认实现是 `ModalDialog` 单例；`bottomsheets` 等模块可以给出另一套行为而不改核心。
2. **布局拆成独立自定义 View**：title/content/buttons 各自负责 `onMeasure` 与 `onDraw`，`DialogLayout` 只做纵向编排，避免一个巨型布局类。
3. **Kotlin 扩展替代继承式 Builder**：`getActionButton`、`listItems`、`onShow` 等都定义在扩展文件中，核心类保持精简，模块按需引入。
4. **监听器用列表而非单回调**：同一动作可注册多个 `DialogCallback`，`invokeAll` 顺序执行，便于组合。
5. **列表点击可延迟到正向按钮**：`listItems` 的 `waitForPositiveButton` 参数与 `DialogAdapter.positiveButtonClicked` 配合，实现单选/多选先收集再确认。

## 阅读路径

1. `core/src/main/java/com/afollestad/materialdialogs/MaterialDialog.kt`
2. `.../DialogBehavior.kt`
3. `.../internal/main/DialogLayout.kt`
4. `.../internal/main/DialogTitleLayout.kt`
5. `.../internal/message/DialogContentLayout.kt`
6. `.../internal/button/DialogActionButtonLayout.kt`
7. `.../internal/button/DialogActionButton.kt`
8. `.../actions/DialogActionExt.kt` 与 `.../callbacks/DialogCallbackExt.kt`

## 版本注记

该 commit 处于 3.x 的 Kotlin 版本线（README 徽章标注 `core:3.3.0`），与网上大量基于 0.8.x/0.9.x 的 Java 文章差异很大：Builder、`DialogBase`、`DialogWrapper` 等类在本 commit 中并不存在。仓库在 2021 年后基本停更。以上类名、行号均以该 commit 为准。
