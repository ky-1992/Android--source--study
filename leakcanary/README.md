# LeakCanary 源码解析

> 排行榜 #6 · 类别：内存检查

## 一句话定位

LeakCanary 是 Square 出品的内存泄漏检测库：用 `ObjectWatcher` 给对象挂 `KeyedWeakReference` + `ReferenceQueue`，延迟检查对象是否被回收；保留对象超过阈值就触发 heap dump，再用 Shark 在堆里做支配树分析（GcRoot → 引用链），最终在 App 内给出泄漏报告。

## 源码快照

| 项 | 值 |
|---|---|
| 上游仓库 | https://github.com/square/leakcanary |
| 快照 commit | `eb60b0be53732aa016c79b7774b497770ad1e4f3` |
| 克隆方式 | `--depth 1` |

多模块结构（只看关键部分）：

```
object-watcher/     # AppWatcher + ObjectWatcher + KeyedWeakReference（纯 JVM，可单测）
leakcanary/         # Android 接入层：InternalLeakCanary、HeapDumpTrigger、UI、通知
shark/              # 堆分析引擎：HeapAnalyzer、对象图、支配树、GcRoot
plumber/            # 已知 Android 框架泄漏的修复器（与检测主链解耦）
```

## 核心类

| 类 | 职责 | 关键位置 |
|---|---|---|
| `AppWatcher` | 门面：install 时注册 Activity/Fragment/View 等监听，对外暴露 `watch()` | `AppWatcher.kt` L19、L92、L129 |
| `ObjectWatcher` | 记录被观察对象的弱引用，入队后延迟检查 `retained`；用 `retainTrigger` 记账 | `ObjectWatcher.kt` L21、L69、L99 |
| `RetainedObjectTracker` | 跟踪器接口，决定“谁还活着” | `RetainedObjectTracker.kt` L16 |
| `ReferenceQueueRetainedObjectTracker` | 默认实现：`ReferenceQueue` + `keyedWeakReferences` 表，回收时清理 | `ReferenceQueueRetainedObjectTracker.kt` L18、L83、L137 |
| `KeyedWeakReference` | 带 key、时间戳、描述（类名）的弱引用，泄漏报告的最小单元 | `KeyedWeakReference.kt` L30、L65 |
| `InternalLeakCanary` | 把检测结果接到 dump 流程与 UI | `InternalLeakCanary.kt` L36、L117 |
| `HeapDumpTrigger` | 阈值与节流：保留对象数、GC 次数、dump 间隔 | `HeapDumpTrigger.kt` L35、L97、L167 |
| `HeapDumper` | dump 抽象，Android 实现是 Debug 包里的 Debug.dumpHprofData | `HeapDumper.kt` L5 |
| `BackgroundThreadHeapAnalyzer` | 后台线程调度 Shark 分析 | `BackgroundThreadHeapAnalyzer.kt` L13 |
| `HeapAnalyzer` | Shark 入口：解析 HPROF → 找泄漏对象 → 推导引用链 | `HeapAnalyzer.kt` L40、L121 |
| `HeapAnalysisSuccess` | 分析结果模型（泄漏列表、统计） | `HeapAnalysis.kt` L82 |

## 关键流程

### 1. watch：把对象纳入观察

`AppWatcher.watch(obj)` 把对象交给 `ObjectWatcher`。`ObjectWatcher.watch()` 会先移除已不存在的记录，然后在追踪器里为对象建 `KeyedWeakReference`（key 是自增 UUID，附描述与时间戳），最后调用 `retainTrigger.markRetainedIfStronglyReachable()` 做一次“当前是否强可达”的记账。

### 2. 延迟检查：什么时候算 retained

`ObjectWatcher` 的 `watch` 之后不会立刻判定泄漏，而是排一个延迟检查（`checkRetainedExecutor`，默认约 5 秒后）：届时 `ReferenceQueue` 里被 GC 的引用已经被清理，剩余仍留在 `keyedWeakReferences` 表里的对象就是 **retained**（保留）对象——它们可能泄漏，也可能是 GC 还没回收。

### 3. 阈值与转储

`HeapDumpTrigger` 周期检查：只有“确实发生过 GC 且 retained 数达到阈值（默认保留了 5 个对象）且距上次 dump 超过最短间隔”，才执行 `dumpHeap()`。dump 完成后发出 `HeapDump` 事件，交给 `InternalLeakCanary` → 后台分析器。

### 4. Shark 分析

`HeapAnalyzer` 读取 HPROF，先定位所有 `KeyedWeakReference` 实例，按 key 关联到应用对象，再从 GC Roots 反向推导最短强引用路径，产出 `HeapAnalysisSuccess`（含泄漏对象与引用链）。UI 层把结果落库并展示，形成“泄漏列表 → 详情 → 引用链”的闭环。

## 设计要点

1. **检测与分析解耦**：`object-watcher` 是纯 JVM 模块，`shark` 独立成引擎，LeakCanary 只是编排；两者都能单独复用与测试。
2. **弱引用 + 引用队列**：不依赖反射扫描堆，正常路径零成本；只有 `KeyedWeakReference` 被 GC 后才暴露“保留”事实。
3. **阈值与节流**：dump 很贵，所以 `HeapDumpTrigger` 用“保留数 + GC + 时间窗”三重条件避免频繁 dump。
4. **可插拔追踪器**：`RetainedObjectTracker` 接口允许替换检测策略（默认 `ReferenceQueueRetainedObjectTracker`）。
5. **结果可观测**：`HeapAnalysisSuccess` 是纯数据，便于上报和二次分析。

## 阅读路径

1. `object-watcher/object-watcher-android-core/.../AppWatcher.kt`
2. `object-watcher/object-watcher/.../ObjectWatcher.kt`
3. `object-watcher/object-watcher/.../ReferenceQueueRetainedObjectTracker.kt`
4. `object-watcher/object-watcher/.../KeyedWeakReference.kt`
5. `leakcanary/leakcanary-android-core/.../internal/HeapDumpTrigger.kt`
6. `leakcanary/leakcanary-android-core/.../internal/InternalLeakCanary.kt`
7. `shark/shark/src/main/java/shark/HeapAnalyzer.kt`

## 版本注记

- 该 commit 属于 3.x 开发线（master），已把 `plumber`（已知框架泄漏修复）拆成独立模块，`shark` 也有独立 CLI（`shark-cli.sh`）。
- 与排行榜成文的 2017 年版本（1.x，`RefWatcher` 时代）相比，类名已从 `RefWatcher` 演进为 `ObjectWatcher`，核心思想（弱引用 + 引用队列 + 延迟检查 + Shark 分析）未变；本目录图表对应当前快照。
