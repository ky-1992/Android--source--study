# logger 源码解析

## 一句话定位

logger 是 orhanobut 出品的 Android 日志门面：`Logger.d("hello %s", "world")` 这类静态调用统一进入 `LoggerPrinter`，由 `LogAdapter` 决定是否放行，再由可插拔的 `FormatStrategy` 把一条日志美化成带边框、线程名、调用栈、JSON/XML 缩进的文本，最后交给 `LogStrategy` 落到 Logcat 或文件。

## 源码快照

- 上游：<https://github.com/orhanobut/logger>
- commit：`0a0e922ee40ae0904bf29e56b81094920c09a485`（2026-10-04，合并 maintenance 分支）
- 版本：`gradle.properties` 声明 `VERSION_NAME=2.3.0-SNAPSHOT`；README 的依赖示例仍写 `com.orhanobut:logger:2.2.0`
- 形态：本 commit 已是 **Kotlin 重写版**，`logger/src/main/kotlin/com/orhanobut/logger/` 下 13 个 `.kt` 文件共 436 行，不再是早期 Java 包结构

模块只有 `logger/`（库）与 `sample/`（示例）。库用 version catalog 配置 `namespace = "com.orhanobut.logger"`，`api(kotlin-stdlib)` 且排除了 `org.jetbrains:annotations`，因为 Kotlin 元数据自带可空性；测试依赖 `junit` 与一份 JVM 版 `org.json`。

## 核心类

| 类 | 职责 | 关键方法与行号 |
| --- | --- | --- |
| `Logger` | 全局 `object` 门面，六个优先级常量 2~7 直接对齐 `android.util.Log` | `printer()` L14、`addLogAdapter()` L15、`t()` L19、`d()` L24、`e()` L26、`json()` L33、`xml()` L34 |
| `Printer` / `LoggerPrinter` | 打印接口与唯一实现，持有 `ThreadLocal` 标签和适配器列表 | `t()` L18、`logMessage()` L82、`log()` L67、`json()` L32、`xml()` L49（`LoggerPrinter.kt`） |
| `LogAdapter` / `AndroidLogAdapter` / `DiskLogAdapter` | 过滤 + 选格式化策略；前者的默认策略是 Pretty，后者是 CSV | `isLoggable()`、`log()`（`AndroidLogAdapter.kt` L4–10） |
| `FormatStrategy` / `PrettyFormatStrategy` | 单方法接口 + 美化实现：边框、线程、栈头、分段 | `log()` L11、`logHeaderContent()` L32、`logContent()` L61（`PrettyFormatStrategy.kt`） |
| `PrettyFormatStrategy.Builder` | 配置 `methodCount`/`methodOffset`/`showThreadInfo`/`tag` | `build()` L80，默认 `methodCount=2`、`tag="PRETTY_LOGGER"` |
| `CsvFormatStrategy` | 生成 `毫秒,时间,级别,tag,内容` 的 CSV 行 | `log()` L17，默认 `dateFormat` 为 `yyyy.MM.dd HH:mm:ss.SSS` |
| `LogStrategy` / `LogcatLogStrategy` / `DiskLogStrategy` | 真正写出去的出口 | `Log.println(priority, tag, message)` L8；`handler.sendMessage` L13、`WriteHandler.handleMessage` L21–29 |
| `Utils` | 栈转字符串、级别名、数组转字符串 | `getStackTraceString()` L8、`logLevel()` L22、`toString()` L32 |

类图见 [class-diagram.html](./class-diagram.html)，只保留主链路：`Logger` → `Printer` → `LogAdapter` → `FormatStrategy` → `LogStrategy`。

## 关键流程

1. **标签与参数**：`Logger.t(tag)` 把标签写进 `ThreadLocal`，随即返回同一个 `Printer` 支持链式调用；下一条 `d/e/i/w/v/wtf` 走 `logMessage()`，先取标签再立刻 `localTag.remove()`（`LoggerPrinter.kt` L82–87），所以标签只影响本线程的下一条日志。参数用 `String.format(message, *args)` 展开，`Logger.d(obj)` 则交给 `Utils.toString()` 处理基本类型数组与 `Array<*>`（`Utils.kt` L32–44）。
2. **适配器分发**：`log()` 先把 `throwable` 拼成 `message : stackTrace`（`message` 为空时只用栈），空内容补 `"Empty/NULL log message"`，再遍历 `logAdapters`，`isLoggable` 通过才 `adapter.log(...)`（`LoggerPrinter.kt` L66–76）。`log` 与 `logMessage` 都标了 `@Synchronized`，同一 printer 上的调用串行。
3. **美化头部**：`PrettyFormatStrategy.log()` 先算输出标签（默认 `PRETTY_LOGGER`，外部传入不同 tag 时拼成 `PRETTY_LOGGER-xxx`），接着打印 `┌` 上边框、`│ Thread: 线程名`、`├` 栈头（L13–15、L32–36）。栈帧取 `Thread.currentThread().stackTrace`，用 `indexOfLast` 找到调用链里最后一个 logger 自身类，再 `+1` 得到真实调用方（L38–47）。
4. **分段与内容**：消息按 UTF-8 字节切成 4000 字节一段，切分点若落在多字节字符中间就向前回退（`0xC0 == 0x80` 判断续字节，L22–28），这是为了避免 Logcat 4KB 上限把中文截断；`logContent()` 按行拆分、去掉尾部空行、每行加 `│ ` 前缀（L61–66），最后补 `└` 下边框。
5. **JSON/XML 美化**：`json()` 按首字符分派到 `JSONObject`/`JSONArray` 的 `toString(2)`，解析失败打 `"Invalid Json"`；`xml()` 用 `TransformerFactory` 设置 `OutputKeys.INDENT=yes` 与 `indent-amount=2`（`LoggerPrinter.kt` L32–64）。
6. **文件通路**：`DiskLogAdapter` 默认 `CsvFormatStrategy`，后者把换行替换成 ` <br> ` 并拼上时间戳与级别，默认 `LogStrategy` 是一个后台 `HandlerThread` 上的 `DiskLogStrategy`；`WriteHandler` 用追加 `FileWriter` 写入 `logs_N.csv`，超过 500KB 自动换文件（`DiskLogStrategy.kt` L21–42）。

流程图见 [flow-diagram.html](./flow-diagram.html)，覆盖入口 → 适配器 → 美化头部 → 分段输出 → Logcat，以及 JSON 与文件两条支路。

## 设计要点

- **四层接口全部可替换**：`Printer` 管组织方式，`LogAdapter` 管过滤与路由，`FormatStrategy` 管文本生成，`LogStrategy` 管落地，任意一层都能单独换掉而不动其它层。
- **ThreadLocal 短命标签**：标签用完即删，既保证线程隔离，又让 `Logger.t()` 不会污染下一条日志。
- **优先级零映射**：常量值 2~7 与 `android.util.Log.VERBOSE..ASSERT` 相同，`LogcatLogStrategy` 直接把 `priority` 传给 `Log.println`。
- **字节级切帧**：不能按字符数切，否则 UTF-8 中文会被拦腰截断；回退续字节的做法值得复用。
- **日志不能弄崩应用**：写文件时 `IOException` 被捕获忽略（`DiskLogStrategy.kt` L24–28），`UnknownHostException` 的栈直接返回空串，避免网络库刷屏（`Utils.kt` L10–14）。
- **文件轮转简单可靠**：按 `logs_N.csv` 递增序号找最后一个文件，未达上限就续写，绝不重写历史。

## 阅读路径

1. `logger/src/main/kotlin/com/orhanobut/logger/Logger.kt`——门面与常量，先看调用面。
2. `Printer.kt`、`LogAdapter.kt`、`FormatStrategy.kt`、`LogStrategy.kt`——四个接口各只有几行，先建立分层心智。
3. `LoggerPrinter.kt`——标签、参数、`throwable` 拼接与遍历适配器。
4. `PrettyFormatStrategy.kt`——边框、线程、栈头、4000 字节切帧，本文档最值得细读的文件。
5. `AndroidLogAdapter.kt`、`LogcatLogStrategy.kt`——默认装配与 Logcat 出口。
6. `CsvFormatStrategy.kt`、`DiskLogStrategy.kt`、`DiskLogAdapter.kt`——文件通路与轮转。
7. `Utils.kt`——栈与数组的边角处理。
8. `logger/src/test/kotlin/.../PrettyFormatStrategyTest.kt`——看期望输出长什么样。

## 版本注记

该 commit 是 logger 的 Kotlin 化版本，`gradle.properties` 已到 `2.3.0-SNAPSHOT`，而 README 的依赖坐标还停在 `2.2.0`，说明发布说明与代码线有滞后。与早期 Java 版相比，这一版的类数量大幅收缩（13 个文件、436 行），`CsvFormatStrategy` 内部保留 `logLevel()` 映射，而 Logcat 通路直接复用 Android 的优先级常量。仓库为浅克隆，未取到 tag，精确版本以 commit `0a0e922ee40ae0904bf29e56b81094920c09a485` 为准。
