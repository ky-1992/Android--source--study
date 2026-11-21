# Butter Knife 源码解析

> 排行榜 #3 · 类别：代码模板

## 一句话定位

Butter Knife 用**编译期注解处理**替代 `findViewById` 和匿名点击监听：`@BindView` / `@OnClick` 等注解在编译时被 `ButterKnifeProcessor` 扫描，为每个 Activity/Fragment 生成一个 `Xxx_ViewBinding` 类，运行时 `ButterKnife.bind()` 通过反射实例化这个生成类完成绑定，`unbind()` 时统一解绑。

## 源码快照

| 项 | 值 |
|---|---|
| 上游仓库 | https://github.com/JakeWharton/butterknife |
| 快照 commit | `fcdebedf3276096db2f51bf6372b849b5a9c75ed` |
| 克隆方式 | `--depth 1` |
| 状态 | 已归档（作者宣布停止维护，推荐 View Binding） |

模块结构：`butterknife/`（运行时 API）、`butterknife-compiler/`（注解处理器）、`butterknife-annotations/`（注解定义）、`butterknife-gradle-plugin/`（R2 资源 id）、`sample/`。

## 核心类

| 类 | 职责 | 关键位置 |
|---|---|---|
| `ButterKnife` | 运行时门面：`bind()` 反射加载生成类并调用其构造器 | `ButterKnife.java` L74、L159 |
| `ButterKnifeProcessor` | 注解处理器：扫描 `@BindView`/`@OnClick` 等，生成 `_ViewBinding` 源码 | `ButterKnifeProcessor.java` L94、L209、L227 |
| `BindingSet` | 处理器内部模型：按目标类收集所有绑定，排序后输出源码 | `BindingSet.java` L42、L97 |
| `ViewBinding` | 单个 View 绑定的抽象基类 | `ViewBinding.java` L13 |
| `FieldViewBinding` / `MethodViewBinding` | 字段型（`@BindView`）与方法型（监听器）绑定 | `FieldViewBinding.java` L7；`MethodViewBinding.java` L7 |
| `MemberViewBinding` | 绑定成员接口（字段/方法统一抽象） | `MemberViewBinding.java` L3 |
| `BindView` | 字段注解定义 | `BindView.java` L17 |
| `ListenerClass` | 监听器注解元信息（目标方法名、参数） | `ListenerClass.java` L9 |
| `Unbinder` | 解绑接口，`unbind()` 置空引用、恢复监听 | `Unbinder.java` L6 |
| `DebouncingOnClickListener` | 防抖点击基类，避免快速重复点击 | `DebouncingOnClickListener.java` L11 |

## 关键流程

### 1. 编译期：处理器扫描注解

`ButterKnifeProcessor.process()` 在每一轮注解处理中收集 `@BindView`、`@OnClick`、`@OnLongClick` 等注解元素，按宿主类（Activity/Fragment/ViewHolder）分组，形成 `BindingSet`。

### 2. 归并生成源码

`BindingSet` 把同一宿主类的所有绑定按 id/顺序归并（多个字段可以绑定同一个 View，需要缓存 `findViewById` 结果），然后生成 `Xxx_ViewBinding.java`：构造器接收目标对象与 root view，内部逐个 `findViewById`、转型、赋给字段；监听器则 new 一个匿名实现并 `setOnClickListener`。

### 3. 运行期：bind 与反射

`ButterKnife.bind(activity)` 用 `Class.forName(target.getClass().getName() + "_ViewBinding")` 找到生成类，反射取构造器 `(Target, View)` 并实例化，返回 `Unbinder`。找不到生成类时会给出明确的错误提示（提示检查注解处理器是否生效）。

### 4. 解绑

`unbind()` 调用生成类的 `unbind()`：把字段置 null、把监听器置 null，避免持有 View 导致泄漏。列表 Item 复用时必须在 `onBindViewHolder`/回收时解绑再重新绑定。

## 设计要点

1. **零运行时反射扫描**：注解解析全部在编译期完成，运行时只有一次 `Class.forName` + 构造器反射，比运行时扫描注解快得多。
2. **生成类是普通 Java**：可读、可调试，`_ViewBinding` 能直接作为学习“代码生成”的范本。
3. **R2 资源 id**：gradle 插件把 `R.id` 拷贝成 `R2.id`，让 library 模块中的注解也能拿到最终 id（非 final 常量问题）。
4. **可组合**：`ButterKnife.bind(this, view)` 支持 Fragment/ViewHolder 的 root view；`Unbinder` 让生命周期管理显式化。
5. **历史意义**：它是 Android 注解处理器时代的代表作，后来被 Kotlin Synthetics 与 View Binding 取代，但“注解 → 编译期生成 → 运行时绑定”的思路仍是主流。

## 阅读路径

1. `butterknife-annotations/.../BindView.java`、`OnClick.java`
2. `butterknife-compiler/.../ButterKnifeProcessor.java`（`process` / `findAndParseTargets` / 生成逻辑）
3. `butterknife-compiler/.../BindingSet.java`（`brewJava` 源码生成）
4. `butterknife-compiler/.../ViewBinding.java`、`FieldViewBinding.java`、`MethodViewBinding.java`
5. `butterknife/.../ButterKnife.java`（`bind` / `findBindingConstructorForClass`）
6. `butterknife/.../Unbinder.java`
7. `sample/` 下的使用示例

## 版本注记

- 该 commit 是项目归档前最后状态（10.x 线），依赖 `androidx.annotation`。
- 排行榜成文时（2017）Butter Knife 8.x 正流行；如今官方推荐 View Binding，本目录保留它用于理解编译期代码生成与注解处理器。
