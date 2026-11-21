# stetho 源码解析

## 一句话定位

stetho 是运行在 Android 进程内的调试桥：它只绑一条以 `_devtools_remote` 结尾的本地套接字，把 Chrome DevTools 协议和 `dumpapp` 命令行都接到这条连接上，让宿主应用无需改动业务代码就能被 `chrome://inspect` 和命令行工具观察。它的价值不在数据存储，而在于把「进程内状态」翻译成 Chrome 能渲染的协议消息，并且尽量把开销推迟到真正有调试器连入之后。

## 源码快照

- 上游仓库：https://github.com/facebook/stetho
- 克隆 commit：`2198797c0ff961dd2d9b87efa711a7d734c0e4d5`
- 版本线索：`gradle.properties` 中 `VERSION_NAME=1.6.1-SNAPSHOT`，README 示例依赖 `com.facebook.stetho:stetho:1.6.0`
- 模块划分：`stetho`（核心：本地 socket、协议分发、DevTools 域、dumpapp）、`stetho-okhttp3` 与 `stetho-urlconnection`（网络拦截）、`stetho-js-rhino`、`stetho-timber`（可选插件）
- 关键包：`server` 放套接字与权限校验，`inspector` 放 Chrome 协议实现，`dumpapp` 放 CLI 帧协议，`inspector/network` 放网络事件上报

## 核心类

| 类 | 职责 | 关键方法（行号） |
|---|---|---|
| `Stetho` | 初始化入口与插件装配 | `initializeWithDefaults`(103)、`initialize`(123) |
| `Stetho.Initializer` | 懒加载钩子，负责启动服务 | `start`(443)、`create`(457) |
| `DumperPluginsProvider` | 惰性提供 dump 插件 | `get`(18) |
| `ServerManager` | 用监听线程托管服务 | `start`(25)、`startServer`(33) |
| `LocalSocketServer` | accept 循环与 worker 线程 | `run`(64)、`listenOnAddress`(78) |
| `LazySocketHandler` | 首连时才创建真实处理器 | `onAccepted`(33)、`getSocketHandler`(38) |
| `ProtocolDetectingSocketHandler` | 按魔数分流 | `addHandler`(40)、`onSecured`(45) |
| `DumpappSocketLikeHandler` | DUMP 帧协议处理 | `onAccepted`(37) |
| `DevtoolsSocketHandler` | DevTools HTTP/WS 入口 | `createServer`(33)、`onAccepted`(49) |
| `ChromeDevtoolsServer` | JSON-RPC 端点 | `onMessage`(77)、`handleRemoteRequest`(117) |
| `Dumper` | 按插件名分派 CLI 命令 | `dump`(65) |
| `StethoInterceptor` | OkHttp 网络事件搬运 | `intercept`(39) |

## 关键流程

1. **启动**：`Stetho.initialize` 先尝试挂钩 `ActivityTracker`，再让 `Initializer.start()` 用 `AddressNameHelper.createCustomAddress` 拼出 `stetho_<进程名>_<userId>_devtools_remote` 地址，交给 `ServerManager` 起名为 `StethoListener-*` 的线程跑 accept 循环。
2. **首个连接**：`LocalSocketServer` 每接受一个连接就开一个 `WorkerThread`；处理器是 `LazySocketHandler`，第一次回调时才同步创建真正的 `ProtocolDetectingSocketHandler`，之后复用，初始化阶段不做重活。
3. **权限校验**：处理器继承 `SecureSocketHandler`，`onAccepted` 里先取对端 uid/pid 并检查是否拥有 `android.permission.DUMP`，不通过只记日志、不服务。
4. **协议分流**：`onSecured` 用 `LeakyBufferedInputStream` 预读 256 字节再 reset：`DUMP` 魔数进 `DumpappSocketLikeHandler`，`GET/POST /dumpapp` 进旧 HTTP 兼容处理器，其余全部落到 `DevtoolsSocketHandler`。
5. **DevTools 会话**：`/inspector` 上挂 `WebSocketHandler` 包住 `ChromeDevtoolsServer`，每个会话配一个 `JsonRpcPeer`；收到带 `method` 的消息交给 `MethodDispatcher`，按「域名.方法名」反射分派，结果序列化后 `sendText` 回球。
6. **网络上报**：`StethoInterceptor` 在请求发出、响应头到达、读 body 时把事件交给 `NetworkEventReporterImpl`，`isEnabled` 只在 Chrome 打开 Network 面板且注册了 peer 后为真，否则整个拦截近乎空转。

## 设计要点

- **一个 socket 两种协议**：不额外占端口，靠魔数与路径前缀在字节层面分流，`dumpapp` 与 DevTools 可同时使用。
- **能懒就懒**：`DumperPluginsProvider`、`InspectorModulesProvider` 都是单方法接口，`Stetho.initialize` 只保存引用，真正的 `Dumper`、分发表在首个连接才构造。
- **权限而非开关**：用本地 socket 对端凭据校验 `DUMP` 权限，保证只有 adb/shell 这类调试方可用，普通第三方应用无法连入。
- **协议实现分层**：`DevtoolsSocketHandler` 只做 HTTP 路由，`ChromeDevtoolsServer` 只做 JSON-RPC，具体域能力由 `ChromeDevtoolsDomain` 列表注入，便于增删模块。
- **dump 走帧协议**：`Framer` 把 argv、stdin、stdout、stderr、退出码分帧传输，CLI 与进程内插件像本地管道一样交互。
- **按需装插件**：`DefaultDumperPluginsBuilder.finish()` 默认只补 Hprof、SharedPreferences、Crash、Files 四个插件，其余能力由调用方 `provide`，避免 debug 包体积与启动开销失控。

## 阅读路径

1. `stetho/src/main/java/com/facebook/stetho/Stetho.java`
2. `stetho/src/main/java/com/facebook/stetho/DumperPluginsProvider.java`
3. `stetho/src/main/java/com/facebook/stetho/server/LocalSocketServer.java`
4. `stetho/src/main/java/com/facebook/stetho/server/LazySocketHandler.java`
5. `stetho/src/main/java/com/facebook/stetho/server/ProtocolDetectingSocketHandler.java`
6. `stetho/src/main/java/com/facebook/stetho/inspector/DevtoolsSocketHandler.java`
7. `stetho/src/main/java/com/facebook/stetho/inspector/ChromeDevtoolsServer.java`
8. `stetho/src/main/java/com/facebook/stetho/dumpapp/DumpappSocketLikeHandler.java`

## 版本注记

以 commit `2198797` 为准（`VERSION_NAME=1.6.1-SNAPSHOT`，README 对应 1.6.0 发布线）。此版本 `stetho-okhttp` 已标记 deprecated，OkHttp 3 请用 `stetho-okhttp3` 的 `StethoInterceptor`；行号基于该 commit 的实际文件，换分支后可能漂移。
