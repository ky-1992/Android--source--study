# OkHttp 源码解析

## 一句话定位

OkHttp 是 Square 出品的 JVM / Android HTTP 客户端：用一条「可插拔的拦截器责任链」把重试、缓存、连接复用、HTTP/2 多路复用等能力串起来，再用 `ConnectionPool` 把 TCP/TLS 连接的生命周期延长到多次请求之间。它同时是 Retrofit、Glide、Coil 等一大批库的底层网络实现，读懂它就等于读懂了 Android 网络栈的地基。

## 源码快照

- 上游仓库：<https://github.com/square/okhttp>
- 本次研读 commit：`aa0888e202350532463981d8e51125f54bdce777`（CHANGELOG 最新条目为 5.5.1，具体版本以该 commit 为准）
- 仓库是 34 个 Gradle 模块的多模块工程，主模块 `okhttp`，周边有 `okhttp-coroutines`、`okhttp-sse`、`okhttp-brotli`、`okhttp-tls`、`okhttp-logging-interceptor`、`mockwebserver` 等。
- 主模块是 Kotlin Multiplatform 布局，核心源码在 `okhttp/src/commonJvmAndroid/kotlin/okhttp3`，另有 `jvmMain` / `androidMain` 放平台实现；仓库 README 注明支持 Android 5.0+（API 21+）与 Java 8+。
- 包结构：包根 `okhttp3`（`OkHttpClient`、`Call`、`Interceptor`、`ConnectionPool` 等公开 API）、`internal/http`（内置拦截器与 `RealInterceptorChain`）、`internal/connection`（`RealCall`、`RealConnection`、`RealConnectionPool`、`Exchange`、`ConnectPlan`、`RealRoutePlanner`）、`internal/cache`（`CacheInterceptor`、`CacheStrategy`、`DiskLruCache`）。

## 核心类

| 类 | 职责 | 关键方法与行号 |
|---|---|---|
| `OkHttpClient` | 不可变配置门面，持有 `Dispatcher`、`ConnectionPool` 和两级拦截器列表 | `newCall()` `OkHttpClient.kt:357`、`newBuilder()` `:380` |
| `RealCall` | `Call` 的唯一实现，组装并执行拦截器链 | `execute()` `RealCall.kt:182`、`enqueue()` `:195`、`getResponseWithInterceptorChain()` `:210` |
| `Dispatcher` | 并发额度（默认 64 个总并发、5 个同 host）、异步队列与线程池 | `enqueue()` `Dispatcher.kt:130`、`promoteAndExecute()` `:170`、`executed()` `:272` |
| `Interceptor` / `Interceptor.Chain` | 责任链节点与拦截器可见的请求上下文接口 | `intercept()` `Interceptor.kt:66`、`Chain.proceed()` `:88` |
| `RealInterceptorChain` | 按 `index` 递进的责任链实现，每步 `copy(index + 1)` | `proceed()` `RealInterceptorChain.kt:312`、`copy()` `:113` |
| 内置拦截器 | 固定顺序：重试、补头、缓存、建连、读写 | `intercept()` `RetryAndFollowUpInterceptor.kt:53`、`BridgeInterceptor.kt:36`、`CacheInterceptor.kt:45`、`ConnectInterceptor.kt:30`、`CallServerInterceptor.kt:32` |
| `ConnectionPool` | 连接池公开壳，全部逻辑委托 `RealConnectionPool` | `idleConnectionCount()` `ConnectionPool.kt:90`、`evictAll()` `:95` |
| `RealConnectionPool` | 真实连接池：复用判定、空闲清理 | `callAcquirePooledConnection()` `RealConnectionPool.kt:84`、`put()` `:134`、`closeConnections()` `:190` |
| `RealConnection` | 一条 TCP/TLS 连接，承载 HTTP/1.1 或 HTTP/2 流 | `newCodec()` `RealConnection.kt:270`、`isHealthy()` `:302` |
| `RealRoutePlanner` / `ConnectPlan` | 选路计划与建连执行 | `plan()` `RealRoutePlanner.kt:65`、`connectTcp()` `ConnectPlan.kt:134`、`connectTlsEtc()` `:168` |
| `Exchange` | 一次请求-响应的收发事务，包住 `ExchangeCodec` | `writeRequestHeaders()` `Exchange.kt:60`、`readResponseHeaders()` `:115` |

## 关键流程

1. **创建与调度**：`OkHttpClient.newCall()` 只是把 `Request` 包成 `RealCall`；真正的执行入口是 `RealCall.execute()`（阻塞）或 `enqueue()`（异步）。异步路径把 `AsyncCall` 交给 `Dispatcher.enqueue()`，由 `promoteAndExecute()` 在额度内投进无界线程池。
2. **装配责任链**：`getResponseWithInterceptorChain()` 把应用拦截器、`RetryAndFollowUpInterceptor`、`BridgeInterceptor`、`CacheInterceptor`、`ConnectInterceptor`、网络拦截器、`CallServerInterceptor` 依次装进 `interceptors` 列表，然后从 `index = 0` 开始 `proceed(originalRequest)`。
3. **递进与回溯**：`RealInterceptorChain.proceed()` 每次取 `interceptors[index]`，`copy(index + 1)` 出下一条链再调 `intercept(chain)`，形成嵌套调用栈；响应沿栈逐层返回，所以任一环都能改写请求或短路响应。
4. **找连接**：`ConnectInterceptor` 触发 `RealCall.enterNetworkInterceptorExchange()`，由 `RealRoutePlanner.plan()` 依次尝试复用当前连接、向 `RealConnectionPool.callAcquirePooledConnection()` 要空闲连接，最后才 `planConnect()`。
5. **建连与收发**：`ConnectPlan.connectTcp()` / `connectTlsEtc()` 完成握手，`RealConnection.newCodec()` 生成编解码器；`CallServerInterceptor` 通过 `Exchange.writeRequestHeaders()` / `readResponseHeaders()` 完成一次请求-响应，结果回到 `AsyncCall.run()` 的 `onResponse` / `onFailure`。

## 设计要点

- **不可变客户端**：`OkHttpClient` 字段全是 `val`，改超时、换拦截器都走 `newBuilder()` 复制；共享同一个实例才能共享连接池与线程池。
- **拦截器即架构**：缓存、重定向、gzip、连接管理都不是硬编码分支，而是链上的独立节点，顺序本身就是语义（缓存要在建连之前，读写必须在最后）。
- **计划与执行分离**：`RealRoutePlanner` 只产出 `Plan`（`ReusePlan`、`ConnectPlan`、`FailedPlan`），真正连接动作在 `Plan` 的实现里，失败重试用 `RouteDatabase` 记录坏路由。
- **两级连接池**：公开的 `ConnectionPool` 只是壳，`RealConnectionPool` 用引用计数（`allocationLimit`）与 `closeConnections()` 做 keep-alive 清理；HTTP/2 下一条连接可承载多路流。
- **平台抽象**：TLS、DNS、线程、日志差异集中在 `internal/platform` 的 `Platform` 子类，主流程保持平台无关，这也是它能做 KMP 的前提。
- **内部 API 显式化**：`@OkHttpInternalApi` 标注的类（如 `RealConnection`）只对测试与扩展模块开放，避免公开 API 面膨胀。
- **可观测性内建**：`EventListener` 把 DNS、连接、请求头、响应体的每个阶段都暴露成回调，线上排查慢请求时不用改源码就能定位卡在哪一环。

## 阅读路径

1. `okhttp/src/commonJvmAndroid/kotlin/okhttp3/OkHttpClient.kt` —— 先看清一个客户端持有什么。
2. `okhttp/src/commonJvmAndroid/kotlin/okhttp3/Call.kt` —— 理解 Call 的契约（只能执行一次、可取消）。
3. `okhttp/src/commonJvmAndroid/kotlin/okhttp3/internal/connection/RealCall.kt` —— 执行入口与拦截器链装配。
4. `okhttp/src/commonJvmAndroid/kotlin/okhttp3/Dispatcher.kt` —— 异步额度与队列。
5. `okhttp/src/commonJvmAndroid/kotlin/okhttp3/Interceptor.kt` 与 `internal/http/RealInterceptorChain.kt` —— 责任链机制的核心。
6. `okhttp/src/commonJvmAndroid/kotlin/okhttp3/internal/connection/ExchangeFinder.kt`、`RealRoutePlanner.kt` —— 连接从哪来。
7. `okhttp/src/commonJvmAndroid/kotlin/okhttp3/internal/connection/RealConnectionPool.kt`、`ConnectPlan.kt` —— 连接池复用与建连细节。
8. `okhttp/src/commonJvmAndroid/kotlin/okhttp3/internal/http/CallServerInterceptor.kt` —— 真正把字节写到 socket 上。

## 版本注记

该 commit 处于 5.x 线：代码已从旧 Java 实现整体改写为 Kotlin，采用 `commonJvmAndroid` + `jvmMain` + `androidMain` 的 KMP 源码集，`RealConnection` / `Exchange` / `RealConnectionPool` 等连接层类被拆到 `internal/connection` 并标注 `@OkHttpInternalApi`（老版本这些类在 `okhttp3.internal.http` 下）。4.x 时代的 `RealCall` 里的 `retryAndFollowUpInterceptor` 字段也已被 `RealRoutePlanner` 计划模型取代，阅读老博客时注意对照。其余细节以该 commit 源码为准。
