# MPAndroidChart 源码解析

## 一句话定位

MPAndroidChart 是 Android 上使用最广的图表库，v4.0 用 Kotlin 重写后仍保持“数据对象 + 视图 + 渲染器”三层结构：`Chart` 持有数据与视口，`BarLineChartBase` 负责坐标轴和边距，`BarChartRenderer` 通过 `Transformer` 把数值坐标画成像素。

## 源码快照

- 上游：<https://github.com/PhilJay/MPAndroidChart>
- 本目录对应 commit：`fae4ad798564ea9ad494eaf037dca0e745400723`（4.0.1 线）
- 核心模块：`MPChartLib`（图表库，`src/main/kotlin/com/github/mikephil/charting` 下约 180 个 Kotlin 文件）、`MPChartCompose`（Compose 封装）、`MPChartExample`（示例 App）
- 主要包：`charts`（各图种）、`data`（Data/DataSet/Entry）、`renderer`（绘制）、`utils`（`Transformer`、`ViewPortHandler`）、`components`（轴、图例、MarkerView）
- 注意：3.x 是 Java 实现，本 commit 已是 Kotlin 重写版，类名保留但文件扩展名变成 `.kt`。

## 核心类

| 类 | 职责 | 关键方法（行号） |
|---|---|---|
| `Chart<T>` | 所有图表的抽象基类，持有 `data`、`renderer`、`viewPortHandler`，处理高亮、动画、空状态 | `data` setter `Chart.kt:79`，`onDraw` `Chart.kt:380`，`getHighlightByTouchPoint` `Chart.kt:520` |
| `BarLineChartBase<T>` | 直角坐标系基类：左右 Y 轴、X 轴、边距、手势与缩放 | `notifyDataSetChanged` `BarLineChartBase.kt:279`，`calculateOffsets` `:390`，`onDraw` `:175` |
| `BarChart` | 柱状图实现，`init` 里组装 `BarChartRenderer` 与 `BarHighlighter` | `init` `BarChart.kt:40`，`calcMinMax` `:51`，`groupBars` `:128` |
| `ChartData<T>` | 数据集容器，缓存各轴极值，提供增删数据集与 Entry 的接口 | `notifyDataChanged` `ChartData.kt:69`，`calcMinMax` `:88`，`addDataSet` `:202` |
| `IDataSet<T>` / `BaseDataSet<T>` | 单条数据集的只读契约与通用实现，渲染器只依赖接口 | `calcMinMax` `IDataSet.kt:47`，`notifyDataSetChanged` `BaseDataSet.kt:80` |
| `DataRenderer` / `BarChartRenderer` | 绘制抽象与柱形实现：先 `drawData`，再高亮、数值、额外装饰 | `drawData` `BarChartRenderer.kt:147`，`drawDataSet` `:171` |
| `Transformer` | 两级矩阵把值域坐标映射到内容区像素，并支持触摸点反查 | `prepareMatrixValuePx` `Transformer.kt:33`，`prepareMatrixOffset` `:50`，`getValuesByTouchPoint` `:170` |
| `ViewPortHandler` | 内容区尺寸、触摸矩阵、缩放/位移钳制 | `setChartDimens` `ViewPortHandler.kt:75`，`refresh` `:344`，`limitTransAndScale` `:359` |

## 关键流程

1. **设置数据**：`Chart.data` setter 先结束数值动画、给缺省 formatter 的数据集补 `DefaultValueFormatter`，再调用 `notifyDataSetChanged`。
2. **重算极值与轴范围**：`BarLineChartBase.notifyDataSetChanged` 依次执行 `data.notifyDataChanged()`（内部 `calcMinMax`）、`renderer.initBuffers`、`calcMinMax`、`computeAxes`、`calculateOffsets`，最后 `invalidate`。
3. **计算边距**：`calculateOffsets` 用图例、左右轴标签宽度和 X 轴标签高度累加四边偏移，再交给 `ViewPortHandler.restrainViewPort` 更新 `contentRect`；视口尺寸变化则由 `Chart.onSizeChanged`（`Chart.kt:1027` 附近）调用 `setChartDimens` 完成，随后重算偏移矩阵与值域矩阵。
4. **绘制**：`onDraw` 先画网格与轴线，然后 `renderer.drawData`（遍历可见数据集、`getTransformer(axisDependency)` 映射柱体）、`drawHighlighted`、`drawValues`，最后图例、描述与 MarkerView。
5. **触摸高亮**：`getHighlightByTouchPoint` 用 `Transformer.getValuesByTouchPoint` 把像素反解成值域坐标，`BarHighlighter` 据此找到最近柱子。

## 设计要点

- **数据与视图解耦**：`ChartData` 只管数据集和极值缓存，`Chart` 只管视图与生命周期，二者靠 `notifyDataSetChanged` 显式同步。
- **渲染器可替换**：`DataRenderer` 定义了 `drawData/drawValues/drawHighlighted/drawExtras` 四段式，图种只需换 Renderer 与 Highlighter，公共坐标逻辑留在基类。
- **坐标变换集中**：`Transformer` 把“值→像素”拆成 `matrixValueToPx` 与 `matrixOffset` 两个矩阵，轴、柱体、触摸反查共用同一套变换，保证绘制与交互一致。
- **视口统一钳制**：缩放与拖动的结果都经过 `ViewPortHandler.limitTransAndScale`，用最小/最大缩放和内容矩形限制矩阵，避免越界出现空白。
- **接口隔离**：`IDataSet` 只暴露渲染与高亮需要的只读方法，`BaseDataSet` 实现颜色、标签等通用能力，具体 DataSet 只补业务语义。
- **Java 兼容**：重写版大量使用 `@JvmOverloads` 保持旧的调用方式，迁移成本主要落在坐标与命名变化上。

## 阅读路径

1. `MPChartLib/src/main/kotlin/com/github/mikephil/charting/charts/Chart.kt`——先看 `data` setter 与生命周期。
2. `.../charts/BarLineChartBase.kt`——`notifyDataSetChanged`、`calculateOffsets`、`onDraw` 三个入口。
3. `.../charts/BarChart.kt`——最短的图种实现，看 `init` 如何装配 Renderer。
4. `.../data/ChartData.kt` 与 `.../data/BaseDataSet.kt`——数据层极值计算。
5. `.../interfaces/datasets/IDataSet.kt`——渲染器依赖的只读契约。
6. `.../utils/Transformer.kt` 与 `.../utils/ViewPortHandler.kt`——坐标与视口。
7. `.../renderer/BarChartRenderer.kt`——`drawData`/`drawDataSet`/`drawValues` 的落点。
8. `.../renderer/DataRenderer.kt`——最后回到抽象层，理解四段式绘制顺序。

## 版本注记

本目录依据上述 commit 的 4.0.1 源码；4.0 是 Kotlin 重写版，3.x 的 Java 版本方法名与坐标体系不同，阅读旧资料时注意区分。动画、Compose 封装与 MarkerView 仍在同一 commit 中，但本解析只覆盖核心 `MPChartLib`。
