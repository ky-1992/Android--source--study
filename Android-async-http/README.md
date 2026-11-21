# android-async-http 源码解析

## 一句话定位

android-async-http 是 loopj 出品的 Android 异步 HTTP 客户端：用 `AsyncHttpClient` 统一封装 GET/POST/PUT 等方法，请求体经 `RequestParams` 变成 `HttpEntity`，任务丢进 `ExecutorService`，响应再由 `AsyncHttpResponseHandler` 通过 `Looper` 切回主线程回调。它建立在 HttpClient 4.x 的 Android 移植版 `cz.msebera.android.httpclient` 之上，是 2013 年前后 Android 网络层的主流写法。

## 源码快照

- 上游：<https://github.com/loopj/android-async-http>
- commit：`018a0b8d96a0dd569de9f8128cfe5d030e0423ef`（2021-01-18，提交说明 “Apache license 2.0 (#1363)”）
- 版本：`gradle.properties` 的 `VERSION_NAME=1.4.11`
- 结构：`library/src/main/java/com/loopj/android/http/` 是全部核心代码（约 30 个类）；`sample/` 演示各种用法；`releases/` 存放发布 jar

## 核心类

| 类 | 职责 | 关键位置 |
| --- | --- | --- |
| `AsyncHttpClient` | 门面：方法族、连接管理、线程池、按 `Context` 记录请求 | 构造 L184、线程池 L574、`sendRequest` L1534、`cancelRequests` L917 |
| `AsyncHttpRequest` | 一次请求的 `Runnable`，含重试循环 | `run` L84、`makeRequest` L131、`makeRequestWithRetries` L170 |
| `RequestParams` | 五类参数容器，统一转 `HttpEntity` | `getEntity` L517、`createFormEntity` L565、`createMultipartEntity` L574 |
| `RetryHandler` | 异常白名单/黑名单驱动的重试策略 | 静态表 L46–58、`retryRequest` L77 |
| `RequestHandle` | 取消句柄，弱引用执行体 | 构造 L31、`cancel` L51 |
| `ResponseHandlerInterface` | 生命周期回调协议 | 方法声明 L38–87 |
| `AsyncHttpResponseHandler` | 把结果编成 `Message` 投给 `Handler` | 构造 L109、`handleMessage` L346、`sendMessage` L402、`sendResponseMessage` L440 |
| `SyncHttpClient` | 同步版，覆盖 `sendRequest` 直接执行 | L82–100 |

## 关键流程

**提交流程**：`get/post` 最终都落到 `sendRequest`（L1534）。它先拒绝空 handler 和“同步 handler 用在异步客户端”的误用，补 `Content-Type`，再调 `newAsyncHttpRequest`（L1518）创建 `AsyncHttpRequest`，`threadPool.submit(request)` 后返回 `RequestHandle`；若传了 `Context`，句柄还会按 `WeakHashMap<Context, List<RequestHandle>>` 登记，供 `cancelRequests` 批量取消。

**执行与重试**：`AsyncHttpRequest.run`（L84）依次做预处理、`sendStartMessage`、`makeRequestWithRetries`、`sendFinishMessage`。`makeRequest` 调 `client.execute`，响应经 `onPreProcessResponse` 后交给 `responseHandler.sendResponseMessage`（L160）。重试循环对 `UnknownHostException` 特殊处理（首次不重试），`NullPointerException` 被包成 `IOException`，其余 `IOException` 交给 `RetryHandler` 判定，重试时先回调 `onRetry`。

**回调切线程**：`AsyncHttpResponseHandler` 构造时取当前线程的 `Looper` 建 `ResponderHandler`；`sendResponseMessage` 读取响应体，状态码 ≥300 发失败消息，否则发成功消息；`sendMessage`（L402）在同步模式或 `handler == null` 时直接执行，否则 `handler.sendMessage`。真正的 `onSuccess/onFailure` 发生在 `handleMessage`（L346）。`getResponseData`（L464）边读边发上传/下载进度。

**参数与请求体**：`RequestParams` 用 `ConcurrentSkipListMap` 存字符串、流、文件、文件数组与对象参数；`getEntity`（L517）按开关选择 JSON 流式实体、`UrlEncodedFormEntity` 或 `SimpleMultipartEntity`，多部分实体通过 `ResponseHandlerInterface` 上报写入进度。

## 设计要点

- **两级线程模型**：网络阻塞全部收敛在池线程的 `AsyncHttpRequest.run` 里，UI 回调统一走创建 handler 时的 `Looper`，调用方不用自己 `new Thread`。
- **取消是协作式的**：`AsyncHttpRequest.cancel` 只置 `AtomicBoolean` 并 `abort`，`run` 与 `makeRequest` 在多个检查点提前返回，`sendMessage` 在中断时直接丢弃消息，避免注销后回调。
- **引用与泄漏控制**：`RequestHandle` 用 `WeakReference` 持执行体，`requestMap` 是 `WeakHashMap`，handler 的 TAG 也是 `WeakReference`，三层都在避免 Activity 被请求链长期持有。
- **重试策略可插拔**：白/黑名单是静态 `HashSet`，并暴露 `allowRetryExceptionClass`、`blockRetryExceptionClass`（L318、L324），默认 5 次、每次间隔 1500ms。
- **同步异步共享实现**：`SyncHttpClient` 只覆盖 `sendRequest`，把 `useSynchronousMode` 置真后直接 `run()`，返回不可取消的句柄，其余代码零改动。
- **大响应体走内存**：`getResponseData`（L464）把整个响应读进 `ByteArrayBuffer` 再转 `byte[]`，`contentLength` 超过 `Integer.MAX_VALUE` 会直接抛异常；大文件下载应交由 `FileAsyncHttpResponseHandler` 或 `RangeFileAsyncHttpResponseHandler` 处理。

## 阅读路径

1. `library/src/main/java/com/loopj/android/http/AsyncHttpClient.java` —— L115–280 构造与配置，L1534–1610 `sendRequest`。
2. `library/src/main/java/com/loopj/android/http/AsyncHttpRequest.java` —— L84–210 主循环与重试。
3. `library/src/main/java/com/loopj/android/http/RetryHandler.java` —— L42–122。
4. `library/src/main/java/com/loopj/android/http/RequestParams.java` —— L94–113、L517–610。
5. `library/src/main/java/com/loopj/android/http/AsyncHttpResponseHandler.java` —— L83–150、L346–462。
6. `library/src/main/java/com/loopj/android/http/RequestHandle.java` —— 全文 120 行。
7. `library/src/main/java/com/loopj/android/http/SyncHttpClient.java` —— L82–100 对比同步/异步差异。

## 版本注记

该 commit 位于 1.4.x 发布线（`VERSION_NAME=1.4.11`），已把底层 HttpClient 换成 `cz.msebera` 的 Android 兼容分支，并引入 Conscrypt 处理 Android 4.4 以下 SNI。上游 2021 年后基本停止功能演进，读到的是 1.4.11 的最终形态；若需对齐发布版本，以 commit `018a0b8d96a0dd569de9f8128cfe5d030e0423ef` 为准。
