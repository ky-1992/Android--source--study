# Agera 源码解析

## 一句话定位

Agera 是 Google 的 Android 响应式编程库：把数据源抽象成 `Repository`（既能被订阅、又能被 `get()` 取值），用 `Updatable` 表达"数据变了"，并把"取数 → 变换 → 校验 → 写值"的流程写成可编译的链式声明。核心模块是纯 Java 加少量 `android.os.Handler` 依赖，37 个文件约 4900 行。

## 源码快照

- 上游：<https://github.com/google/agera>
- commit：`04045ccbf41ac5fc6806724127b33cbe0dda372c`（2018-02-27）
- 目录：`agera/` 核心库；`extensions/{content,database,net,rvadapter,rvdatabinding}` 可选扩展；`samples/BasicSample`、`samples/BasicSampleWithoutCallbacks`、`testapp/` 示例
- 核心包：`agera/src/main/java/com/google/android/agera/`

## 核心类

| 类 | 职责 | 关键成员 / 行号 |
| --- | --- | --- |
| `Repository` | 仓库接口：`Observable` + `Supplier`，分 direct / deferred 两态 | 声明 L49；两态说明 L30-47 |
| `Observable` / `Updatable` | 订阅契约：成对 add/remove，回调回到添加线程 | `addUpdatable` L43、`removeUpdatable` L50；`update()` L27 |
| `BaseObservable` | Observable 骨架：数组存订阅者、消息驱动派发 | `addUpdatable` L67-87、`dispatchUpdate` L106-113、`sendUpdate` L149-174 |
| `WorkerHandler` | 每线程单例 Handler，统一承载激活/派发/续跑消息 | `workerHandler()` L23-31、`handleMessage` L48-74 |
| `CompiledRepository` | 指令数组解释器：跑数据流、维护 runState | `observableActivated` L134-138、`maybeStartFlow` L156-173、`runFlowFrom` L285-346、`get` L611-620 |
| `RepositoryCompiler` | 编译期 fluent 状态机，校验调用顺序并压出指令数组 | `observe` L134-145、`compile` L416-420、重置 L432-447 |
| `RepositoryCompilerStates` | 链式 DSL 的分状态接口（REventSource/RFlow/RConfig） | 声明 L56、`goTo` L177、`goLazy` L187 |
| `Repositories` | 工厂：静态仓库、可变仓库、DSL 入口 | `repository` L36-38、`repositoryWithInitialValue` L44-46、`SimpleRepository` L56-82 |
| `RepositoryConfig` | 取消/重置策略位标记 | `CANCEL_FLOW` L53、`SEND_INTERRUPT` L72 |
| `Result` | 成功 / 失败 / 缺失三态值对象，供 attempt 指令使用 | 工厂 L76-121、`get` L163、`ifSucceededMap` L412 |

类图见 [class-diagram.html](./class-diagram.html)，其中省略了 `Observables`、`WorkerHandler` 与扩展模块。

## 关键流程

**声明与编译**：`Repositories.repositoryWithInitialValue(v)` 返回 DSL 首状态（L44-46），`observe(...)` 收集事件源、`getFrom/thenTransform` 追加指令，`compile()` 前由 `checkExpect` 状态机逐条校验顺序（`RepositoryCompiler` L118-120），最后 `compileRepositoryAndReset` 调 `compiledRepository(...)` 把指令列表转成 `Object[]` 并创建 `CompiledRepository`（L432-447、L43-57）。

**激活与启动**：首个 `Updatable` 加入后 `BaseObservable` 发 `MSG_FIRST_ADDED` 调 `observableActivated()`（L67-87）；`CompiledRepository` 在此把自己挂到事件源并 `maybeStartFlow()`（L134-138、L156-173），同时把 `intermediateValue` 置为 `currentValue`。

**执行与线程切换**：`runFlowFrom` 用 `switch` 解释 `GET_FROM/MERGE_IN/TRANSFORM/CHECK/GO_TO/SEND_TO/BIND/FILTER_*` 等指令（L267-277、L285-346）；`goTo` 把自身作为 `Runnable` 交给 `Executor`，`run()` 从暂停索引续跑并把当前线程登记为可打断（L415-424、L571-602）；`goLazy` 让剩余指令等到 `get()` 时才在调用线程执行（L604-620）。

**写值与通知**：`END` 或 `check/terminate` 走到 `setNewValueAndEndFlow`，先用 `notifyChecker.merge(old,new)` 判重，再 `dispatchUpdate()`（L536-563）；`BaseObservable.sendUpdate()` 按 `shortestUpdateWindowMillis` 节流后经 `WorkerHandler.update` 投递 `Updatable.update()`（L149-174、`WorkerHandler` L42-46）。

流程图见 [flow-diagram.html](./flow-diagram.html)，覆盖订阅 → 激活 → 执行 → 回写，以及取消重启与惰性两条支线。

## 设计要点

- **延迟启动**：deferred 仓库在无订阅者时不跑流程、可释放数据（`Repository` L30-47）；`BaseObservable` 只在 0→1 个订阅者时回调激活，1→0 时回调失活，避免空转。
- **编译期而不是运行期做校验**：`RepositoryCompiler` 用 `Expect` 状态机（L75-119）拦截非法的链式顺序；产物只是 `Object[]` 指令数组，解释器用 table-switch 而非多态子类，兼顾体积与分发开销（L310-312 注释）。
- **回调线程即添加线程**：`WorkerHandler` 按 `ThreadLocal` 复用（L18-31），所有派发都过消息队列，因此 `Observable` 契约里"回调回到添加 Updatable 的线程"（`Observable` L33）能成立。
- **取消协议**：`CANCEL_FLOW` / `SEND_INTERRUPT` 让长任务在指令边界甚至线程中断处停下，`acknowledgeCancel` 统一定格回 IDLE 并回收中间值（L230-249），重启通过 `restartNeeded` 排队。
- **通知去重与节流**：默认 `objectsUnequal` 比较新旧值，只有真的变了才派发；`perMillisecondObservable` 一类组合函数把最短更新间隔下沉到 `shortestUpdateWindowMillis`（`Observables` L101-104、`BaseObservable` L154-163）。
- **函数式接口而非事件类**：`Supplier`、`Function`、`Merger`、`Receiver`、`Predicate` 都是单方法接口，DSL 可以组合任意用户代码，`Result` 则把"失败"当作一等值参与 `attempt*` 指令。

## 阅读路径

1. `agera/src/main/java/com/google/android/agera/Repository.java` —— 先理解 direct/deferred 两态。
2. `Observable.java`、`Updatable.java`、`MutableRepository.java` —— 订阅契约与可写仓库。
3. `BaseObservable.java` —— 订阅者数组、消息派发与节流（L40-188）。
4. `WorkerHandler.java` —— 每线程单例和 6 种消息。
5. `RepositoryCompilerStates.java` —— 链式 DSL 的状态与语义。
6. `RepositoryCompiler.java` —— 编译与校验（L46-447）。
7. `CompiledRepository.java` —— 指令解释器与取消/重启（L39-623）。
8. `Result.java` 与 `samples/BasicSample/.../UsernamesRepository.java` —— 值对象与真实用法。

## 版本注记

该 commit 是 `master` 上的 2018-02-27 快照，根 `README.md` 给出的接入坐标是 `com.google.android.agera:agera:1.4.0`；浅克隆没有 tag，精确版本以 `04045ccbf41ac5fc6806724127b33cbe0dda372c` 为准。此时 `RepositoryCompilerStates` 已按 REventSource / RFrequency / RFlow / RConfig 拆成多段接口，`extensions/` 是标注为 experimental 的扩展，阅读核心只看 `agera/` 模块即可。
