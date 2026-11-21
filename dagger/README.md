# Dagger 源码解析

## 一句话定位

Dagger 是 Google 的编译期依赖注入框架：用 `@Component` 声明注入入口、`@Module` 声明提供者、`@Inject` 标注构造器与字段，注解处理器在编译期把三者解析成一张绑定依赖图，校验通过后生成 `DaggerXxx` 实现类，运行时不再靠反射查找依赖，只调用生成的工厂类。

## 源码快照

- 上游：<https://github.com/google/dagger>
- commit：`28d3950005e9e8b6bda852f574c4533bcbb8b0bd`（浅克隆，`origin` 指向该地址）
- 版本：根 `gradle.properties` 中 `PUBLISH_VERSION=LOCAL-SNAPSHOT`，属主干开发快照，精确版本以该 commit 为准

目录结构概览：

| 路径 | 内容 |
| --- | --- |
| `dagger-runtime/main/java/dagger/` | 运行期注解与接口：`Component`、`Module`、`Provides`、`Binds`、`Lazy`、`MembersInjector`；`internal/` 下另有 `Factory`、`DoubleCheck`、`DaggerCollections` 等轻量支撑类 |
| `dagger-compiler/…/codegen/` | 处理器入口 `ComponentProcessor`、`KspComponentProcessor` 与共享实现 `DelegateComponentProcessor` |
| `dagger-compiler/…/codegen/binding/` | 绑定模型的模型层：`BindingGraph`、`BindingGraphFactory`、`ComponentDescriptor`、`BindingNode` |
| `dagger-compiler/…/codegen/processingstep/` | 按注解分派的处理步骤，`ComponentProcessingStep` 负责 `@Component` |
| `dagger-compiler/…/codegen/componentgenerator/`、`writing/` | 从图生成 `DaggerXxx`，`ComponentGenerator` 与 `ComponentImplementation` 负责拼装 |
| `dagger-compiler/…/codegen/xprocessing/` | 抹平 Javac 与 KSP 差异的抽象层，`XTypeElement`、`XProcessingEnv` 等 |
| `dagger-spi/main/java/dagger/spi/` | 对外扩展点 `BindingGraphPlugin`、`DiagnosticReporter` |

## 核心类

| 类 | 职责 | 关键方法（行号取自快照） |
| --- | --- | --- |
| `ComponentProcessor` | Javac 侧处理器，继承 `JavacBasicAnnotationProcessor` | `initialize()` L99、`processingSteps()` L118、`postRound()` L128 |
| `KspComponentProcessor` | KSP 侧同构处理器 | `initialize()` L48、`processingSteps()` L57、`postRound()` L67 |
| `DelegateComponentProcessor` | Javac 与 KSP 共用的真实实现 | `initialize()` L85、`processingSteps()` L116、`postRound()` L125 |
| `ComponentProcessingStep` | 处理 `@Component`：解析、建图、校验、生成 | `process()` L85、`processRootComponent()` L97、`processSubcomponent()` L123、`generateComponent()` L140 |
| `ComponentDescriptor` | 组件声明的只读模型（模块、依赖、scope、入口方法） | `componentMethods()` L138、`rootComponentDescriptor()` L535 |
| `BindingGraphFactory` | 从请求出发递归解析绑定，构建依赖图 | `create()` L104 |
| `BindingGraph` | 单个组件内的绑定网络，含 `ComponentNode` | `componentNode()` L284、`topLevelBindingGraph()` L292 |
| `BindingGraphValidator` | 跑全量校验与 SPI 插件 | `shouldDoFullBindingGraphValidation()` L46、`isValid()` L56/L61 |
| `ComponentGenerator` | 图到实现类源码 | `topLevelTypes()` L50、`originatingElement()` L45 |
| `ComponentNames` | 生成类命名：`Dagger` + 组件名 | `getTopLevelClassName()` L60 |

类图见 [class-diagram.html](./class-diagram.html)，图中把 `ComponentDescriptor`、`BindingGraph` 与两个处理器画在了一起，省略了 `validation/`、`writing/` 下的细节类。

## 关键流程

扫描与分派：`@Component` 接口（`Component.java:246`）被处理器看到后，`ComponentProcessor` 并不自己干活，而是把 `XProcessingEnv` 和自己注入的插件集合交给 `DelegateComponentProcessor.initialize()`（L85），由它创建 `processingSteps()`（L116）里的步骤列表；`ComponentProcessingStep.process()`（L85）先判断注解是否属于根组件、子组件或 creator，分别转给 `processRootComponent()`、`processSubcomponent()`、`processCreator()`。

两次建图：`processRootComponent()`（L97）先用工厂解析出 `ComponentDescriptor`，再 memoize 一个 `create(descriptor, true)` 的全量图（L109）；`BindingGraphValidator.shouldDoFullBindingGraphValidation()` 为真时先校验全量图，这样 `BindingGraphPlugin.visitGraph()`（`BindingGraphPlugin.java:38`）能看到所有绑定并报错。校验通过后改用 `create(descriptor, false)` 只保留入口点可达的绑定（L116），再交给 `isValid(prunedGraph, supplier)`（L61）做二次校验，最终 `generateComponent()`（L140）把裁剪后的图送给 `ComponentGenerator`。

生成与收尾：`ComponentGenerator.topLevelTypes()`（L50）返回类型列表，由 `SourceFileGenerator` 统一用 `XFiler` 写出；命名规则在 `ComponentNames.getTopLevelClassName()`（L60）里拼 `Dagger` 前缀，所以 `@Component interface CoffeeShop` 得到 `DaggerCoffeeShop`。每轮结束时 `DelegateComponentProcessor.postRound()`（L125）若还没到最后一轮，就调 `injectBindingRegistry.generateSourcesForRequiredBindings()`（L127）为 `@Inject`、`@Provides` 补出 `Factory` 与 `MembersInjector`；到最后一轮才让插件 `endPlugins()` 并清缓存。

流程图见 [flow-diagram.html](./flow-diagram.html)，覆盖源码扫描 → 处理器分派 → 声明解析 → 建图校验 → 生成实现五段。

## 设计要点

- **双处理器共用一套实现**：`ComponentProcessor` 与 `KspComponentProcessor` 各自只有几十行，都持有一个 `DelegateComponentProcessor`（`ComponentProcessor.java:82` 直接 `new`），真正逻辑只写一遍。
- **自举**：`DelegateComponentProcessor` 的依赖用 `@Inject` 字段声明（如 `injectBindingRegistry`，L76），同一文件末尾定义 `@Component interface Injector`（L148），把 8 个 module 装配到自己身上——编译器用自己生成的代码构建自己。
- **`xprocessing` 抽象层**：`processingstep/`、`binding/` 中的类基本只面向 `XTypeElement`、`XProcessingEnv` 这类抽象类型，Javac 与 KSP 的差异收敛在 `xprocessing/` 与 `javac/` 两个目录，接入新前端时主要补这两处实现。
- **两遍建图是性能与可诊断性的折中**：全量图能让插件看到不可达绑定并给出警告，裁剪图让生成器只处理真正用到的节点，避免生成死代码。
- **校验分两层**：`validation/` 下的 validator 处理声明合法性（如 `AnyBindingMethodValidator`），`BindingGraphValidator` 处理图结构，SPI 插件再叠加自定义检查，三者都通过才生成。
- **`@Component` 的 `modules`、`dependencies` 默认空数组**（`Component.java:252`、`:258`），`includes` 与组件依赖的展开都在 `ComponentDescriptor` 构建期完成。

## 阅读路径

1. `dagger-runtime/main/java/dagger/Component.java` —— 先看 L246 起的注解定义与 `modules`/`dependencies` 两个属性。
2. `dagger-compiler/main/java/dagger/internal/codegen/ComponentProcessor.java` —— 入口只有 `initialize()`、`processingSteps()`、`postRound()` 三个方法（L99–L130）。
3. `dagger-compiler/…/codegen/DelegateComponentProcessor.java` —— 共享实现与自举用的 `Injector`（L70–L160）。
4. `dagger-compiler/…/codegen/processingstep/ComponentProcessingStep.java` —— 重点读 `processRootComponent()`（L97–L122）看两次建图。
5. `dagger-compiler/…/codegen/binding/BindingGraphFactory.java` 与 `BindingGraph.java` —— 绑定请求如何递归解析成节点。
6. `dagger-compiler/…/codegen/validation/BindingGraphValidator.java` —— 全量图与裁剪图的两种校验入口（L46–L75）。
7. `dagger-compiler/…/codegen/componentgenerator/ComponentGenerator.java`、`…/writing/ComponentImplementation.java` —— 图如何变成字段与方法。

## 版本注记

该 commit 位于 Dagger 主干，`PUBLISH_VERSION=LOCAL-SNAPSHOT`，不是某个已发布版本；代码里已有独立的 `KspComponentProcessor` 与 `xprocessing/` 包，说明 KSP 支持与"编译器前端抽象"已合入主干。`CHANGELOG.md` 只保留 Dagger 2 的发布页链接，不记录版本号，因此具体版本线以该 commit 为准。
