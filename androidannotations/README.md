# AndroidAnnotations 源码解析

> 排行榜 #18 · 类别：注解（编译期代码生成）

## 一句话定位

AndroidAnnotations 是"注解驱动开发"的代表框架：给 Activity/Fragment/Bean 写上 `@EActivity`、`@ViewById`、`@Click` 等注解，`AndroidAnnotationProcessor` 在**编译期**生成 `Foo_` 子类，把 findViewById、事件注册、Intent 注入、状态恢复等样板代码全部生成出来；运行时只用 `Foo_.intent(context)` 启动，几乎没有反射开销。

## 源码快照

| 项 | 值 |
|---|---|
| 上游仓库 | https://github.com/androidannotations/androidannotations |
| 快照 commit | `0d24f1b64dd6b4c27d9a77d3b771a129a59b5b4c` |
| 克隆方式 | `--depth 1` |
| 状态 | 已停止维护 |

模块：`AndroidAnnotations/androidannotations-core/`（处理器核心）、`androidannotations-api/`（运行时 API 与注解）、`androidannotations-bootstrap/`（Eclipse 时代入口）、`examples/`。

## 核心类

| 类 | 职责 | 关键位置 |
|---|---|---|
| `AndroidAnnotationProcessor` | `AbstractProcessor` 实现，注解处理总入口 | `AndroidAnnotationProcessor.java` L63、L76、L116 |
| `AndroidAnnotationsEnvironment` | 处理器运行环境契约 | `AndroidAnnotationsEnvironment.java` L39 |
| `InternalAndroidAnnotationsEnvironment` | 环境实现：rClass、manifest、插件、ProcessHolder | `InternalAndroidAnnotationsEnvironment.java` L41 |
| `ModelExtractor` | 从 `RoundEnvironment` 提取被注解元素 | `ModelExtractor.java` L33、L39、L94 |
| `AnnotationElementsHolder` | 按注解名分组的元素集合 | `AnnotationElementsHolder.java` L28 |
| `AnnotationHandlers` | Handler 注册表：按注解类型分发 | `AnnotationHandlers.java` L29、L39 |
| `AnnotationHandler` | 单个注解处理的契约接口 | `AnnotationHandler.java` L1 |
| `BaseAnnotationHandler` | Handler 基类（持有环境、模型、校验） | `BaseAnnotationHandler.java` L37 |
| `ElementValidation` | 元素合法性校验 | `ElementValidation.java` |

## 关键流程

### 1. init：一次性初始化

`AndroidAnnotationProcessor.init(processingEnv)` 解析处理器选项（`-Aandroidannotations...`），构建 `InternalAndroidAnnotationsEnvironment`，加载插件（`AndroidAnnotationsPlugin`），初始化 rClass 与 `AndroidManifest` 解析器。

### 2. process：每轮提取模型

`process()` 每轮注解处理时调用 `ModelExtractor.extract(...)`：从 `RoundEnvironment` 里收集所有被支持的注解（含继承来的注解），按注解名塞进 `AnnotationElementsHolder`；再遍历所有 Handler 的 `validate` 做元素校验。

### 3. 分发与生成

`AnnotationHandlers` 持有全部 `AnnotationHandler`（生成型 `GeneratingAnnotationHandler` + 装饰型）；对每个宿主类，各 Handler 读取自己关心的注解元素，生成/修改 `Foo_` 子类：`onCreate` 里注入 View、注册监听、绑定 Intent extra。`CodeModelGenerator` 负责把生成模型写成 Java 源文件，交给下一轮编译。

### 4. 运行时

业务代码写 `@EActivity(R.layout.main) class MainActivity extends Activity`，编译后存在 `MainActivity_`；启动用 `MainActivity_.intent(this).start()`，`setContentView`、`findViewById`、`@Click` 的 `OnClickListener` 都已生成在子类里。

## 设计要点

1. **编译期生成而非运行时反射**：类名后缀 `_` 是生成类约定，运行时只 new 一次，性能接近手写。
2. **Handler 插件化**：每种注解对应一个 Handler，新增能力只需注册 Handler，核心处理器不需要改。
3. **模型-校验-生成三段式**：`ModelExtractor` 只负责收集，`ElementValidation` 只负责校验，Handler 只负责生成，职责清晰。
4. **环境抽象**：`AndroidAnnotationsEnvironment` 让同一套处理器能跑在 javac 注解处理与服务端独立运行两种模式。
5. **历史价值**：它是"注解处理器能做什么"的教科书，后来 Kotlin KAPT/KSP、Dagger、Hilt 的注解处理思路都与之同源。

## 阅读路径

1. `AndroidAnnotations/androidannotations-core/androidannotations/src/main/java/org/androidannotations/internal/AndroidAnnotationProcessor.java`
2. `.../internal/model/ModelExtractor.java`
3. `.../internal/model/AnnotationElementsHolder.java`
4. `.../internal/AnnotationHandlers.java`
5. `.../handler/BaseAnnotationHandler.java` 与一个具体 Handler（如 `ViewByIdHandler`）
6. `.../internal/generation/CodeModelGenerator.java`
7. `.../AndroidAnnotationsEnvironment.java`

## 版本注记

- 该 commit 是仓库停止维护前的最后状态（4.8 线），仍基于 `javax.annotation.processing` 与 Android 旧支持库。
- 排行榜成文时（2017）3.x/4.x 正流行；现在这类能力更常用 View Binding、Hilt、KSP 实现，本目录保留它用于理解编译期代码生成的完整链路。
