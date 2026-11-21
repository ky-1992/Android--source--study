# flexbox-layout 源码解析

## 一句话定位

把 CSS Flexbox 布局模型搬进 Android：`FlexboxLayout` 是 `ViewGroup` 版容器，`FlexboxLayoutManager` 是 RecyclerView 版容器，两者共用同一套测量、换行与对齐算法。开发者用 `flexDirection`、`flexWrap`、`justifyContent`、`alignItems` 等属性描述意图，具体每行几个子项、如何伸缩，全部由库在测量阶段算出来。

## 源码快照

- 上游：<https://github.com/google/flexbox-layout>
- commit：`366b461fd0425bfaa9872b1d9af9da3055d3bc74`（2021-05-20，`Update README`）
- 工程结构：`flexbox/src/main/java/com/google/android/flexbox/` 是库本体；`demo-playground/`、`demo-cat-gallery/` 为示例 App；`tool/` 是辅助脚本。包内除核心类外，还有 `FlexDirection`、`FlexWrap`、`JustifyContent`、`AlignItems`、`AlignSelf`、`AlignContent` 六个常量类。
- 类关系概览：`FlexContainer` 抽象出容器能力，`FlexItem` 抽象出子项属性，`FlexboxHelper` 只面向这两个接口编程；`FlexboxLayout` 与 `FlexboxLayoutManager` 各自实现容器接口，子项参数由各自的 `LayoutParams` 实现 `FlexItem`。

## 核心类

| 类 | 职责 | 关键方法（行号） |
|---|---|---|
| `FlexboxLayout` | ViewGroup 容器，测量与布局入口 | `onMeasure` L263、`measureHorizontal` L345、`onLayout` L564、`LayoutParams` L1571 |
| `FlexboxLayoutManager` | RecyclerView 容器，复用同一 helper | `updateFlexLines` L864、`LayoutParams` L2576 |
| `FlexContainer` | 容器抽象：子项数、方向、对齐、行列表 | L27 |
| `FlexItem` | 子项抽象属性 | L27、`getFlexGrow` L102、`getAlignSelf` L140 |
| `FlexboxHelper` | 测量、换行、伸缩、对齐、摆放算法 | `calculateFlexLines` L388、`isWrapRequired` L867、`determineMainSize` L962、`determineCrossSize` L1438、`stretchViews` L1601 |
| `FlexLine` | 一行的账本：主轴/交叉轴尺寸与首尾索引 | L29、`mMainSize` L43、`mCrossSize` L52、`mFirstIndex` L89 |
| `FlexboxItemDecoration` | 行间、列间分隔线绘制 | L45 |

## 关键流程

1. **入口分派**：`onMeasure` 先用 `mOrderCache` 判断 `order` 是否变化并重排索引，再按 `mFlexDirection` 调 `measureHorizontal` 或 `measureVertical`。重排后所有遍历都改用 `getReorderedFlexItemAt` 取子项，保证 `order` 真正生效。这一步只负责分派方向，真正的测量与折行全部交给 helper 完成。
2. **换行**：`calculateFlexLines` 遍历子项，按 `flexBasisPercent` 与 MeasureSpec 算主轴尺寸并 `child.measure`，再用 `isWrapRequired` 判定——`NOWRAP` 直接 false、`isWrapBefore` 直接 true、`UNSPECIFIED` false、超 `maxLine` 停止；否则比较 `maxSize < currentLength + childLength`。需换行时把旧行交给 `addFlexLine`，再新建 `FlexLine` 累加。每行同时累加 `mTotalFlexGrow` 与 `mTotalFlexShrink`，并把子项交叉轴最大值记进 `mCrossSize`。
3. **主轴伸缩**：`determineMainSize` 先定 `mainSize`（EXACTLY 用测量值，否则取 `largestMainSize`），行内偏小且有 `flexGrow` 就 `expandFlexItems`，偏大且有 `flexShrink` 就 `shrinkFlexItems`；伸缩后重新测量并调用 `checkSizeConstraints` 校正 min/max。
4. **交叉轴**：`determineCrossSize` 在 EXACTLY 时按 `alignContent` 分配（单行直接撑满，多行按 STRETCH 等分支均分或整体偏移），随后 `stretchViews` 把 `alignSelf=STRETCH` 的子项撑到行高。注意单行容器里 `alignContent` 不生效，这是 flex 语义里常被忽略的规则。
5. **布局**：`onLayout` 依据方向与 RTL / `WRAP_REVERSE` 算出 `isRtl`，`layoutHorizontal` 逐行调 `layoutSingleChildHorizontal`，结合 `alignItems`、margin 与 baseline 调 `view.layout`；垂直方向对称地走 `layoutVertical`。

## 设计要点

- **接口隔离**：`FlexboxHelper` 只依赖 `FlexContainer`，两种容器各自实现，算法零重复。
- **行对象优先**：先把 `FlexLine` 算完再布局，避免重复测量子项。
- **增量能力**：helper 支持 `fromIndex` / `toIndex` 与 `FlexLinesResult`，LayoutManager 侧只补算可见区域需要的行，滚动时不必整表重算。
- **属性走 LayoutParams**：`order`、`flexGrow`、`flexShrink`、`alignSelf`、`flexBasisPercent` 由 `LayoutParams` 承载，与 XML 属性一一对应。
- **边界兜底**：`checkSizeConstraints` 在首测后校正 min/max，`CompoundButton` 还有最小尺寸特判，避免系统控件把行撑破；测量缓存也用来避免同一 spec 的重复测量。
- **装饰可插拔**：`FlexboxItemDecoration` 用 `SHOW_DIVIDER_*` 位标记控制行间与列间分隔线，布局算法只通过回调读取装饰长度。

## 阅读路径

1. `flexbox/src/main/java/com/google/android/flexbox/FlexContainer.java`
2. `flexbox/src/main/java/com/google/android/flexbox/FlexItem.java`
3. `flexbox/src/main/java/com/google/android/flexbox/FlexLine.java`
4. `flexbox/src/main/java/com/google/android/flexbox/FlexboxLayout.java`
5. `flexbox/src/main/java/com/google/android/flexbox/FlexboxHelper.java`
6. `flexbox/src/main/java/com/google/android/flexbox/FlexboxLayoutManager.java`
7. `flexbox/src/main/java/com/google/android/flexbox/FlexboxItemDecoration.java`

## 版本注记

以该 commit 为准。工程已启用 AndroidX（`gradle.properties` 中 `android.useAndroidX=true`，`FlexboxLayoutManager` 继承 `androidx.recyclerview` 的 `LayoutManager`），`demo-cat-gallery` 含 Kotlin 代码，库本体仍以 Java 为主。阅读时注意 `FlexboxLayoutManager` 走的是 RecyclerView 的回收与预取模型，与 `FlexboxLayout` 的一次性全量测量并不完全相同，同一个 helper 在两种宿主下的调用时机差别很大。
