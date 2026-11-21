# Retrofit 源码解析

> 排行榜 #1 · 类别：网络

## 一句话定位

Retrofit 把「HTTP API 接口」变成一个 Java 接口：你在接口方法上写注解，它在运行时用 **JDK 动态代理** 拦截调用，用 **ServiceMethod（注解解析模型）+ CallAdapter + Converter 双工厂** 把一次方法调用翻译成 OkHttp 请求，并把响应反序列化成声明式返回类型。

## 源码快照

| 项 | 值 |
|---|---|
| 上游仓库 | https://github.com/square/retrofit （已迁移，访问会 301 跳转到 lysine-dev/retrofit） |
| 快照 commit | `987fe7740c4bdcf28131062545df26ba1f9e1430` |
| 版本线 | `3.1.0-SNAPSHOT`（gradle.properties） |
| 克隆深度 | `--depth 1` |

模块结构：

```
retrofit/                     # 核心：注解解析、动态代理、OkHttpCall
retrofit-converters/          # gson / moshi / jackson / protobuf / kotlinx-serialization ...
retrofit-adapters/            # rxjava(1/2/3) / guava / java8 / scala
retrofit-mock/                # BehaviorDelegate 假网络
retrofit-response-type-keeper/# R8 保留泛型信息的处理器
samples/  website/  retrofit-bom/
```

核心源码集中在 `retrofit/src/main/java/retrofit2/`，一共约 30 个类，其中真正撑起框架的是 8~10 个。

## 核心类

| 类 | 职责 | 关键方法 |
|---|---|---|
| `Retrofit` | 框架入口与全局配置持有者 | `create(Class<T>)`、`callAdapter(...)`、`responseBodyConverter(...)`、`loadServiceMethod(...)` |
| `Retrofit.Builder` | 组装 baseUrl / OkHttpClient / 两组工厂 | `baseUrl`、`client`、`addConverterFactory`、`addCallAdapterFactory`、`build` |
| `ServiceMethod<T>` | 服务方法模型抽象（一次解析、多次复用） | `parseAnnotations`、`invoke` |
| `HttpServiceMethod` | 注解解析结果，持有 RequestFactory / CallAdapter / Converter | `parseAnnotations`、`invoke`、`adapt` |
| `CallAdapter<R,T>` | 把 `Call<R>` 适配成方法声明的返回类型 | `responseType()`、`adapt(Call<R>)` |
| `Converter<F,T>` | 请求体序列化与响应体反序列化契约 | `convert(F)` |
| `Call<T>` | 可执行请求句柄 | `execute()`、`enqueue()`、`clone()`、`cancel()` |
| `OkHttpCall<T>` | `Call` 的实现，桥接 okhttp 并解析响应 | `enqueue`、`execute`、`createRawCall`、`parseResponse` |
| `RequestFactory` | 把方法注解 + 实参编译成 `okhttp3.Request` | `parseAnnotations`、`create` |
| `Platform` / `AndroidMainExecutor` | 运行时差异（反射、默认方法、主线程 Executor） | `Platform.reflection`、`handler.post` |

## 关键流程

### 1. 创建代理

```java
GitHubService service = retrofit.create(GitHubService.class);
```

`create()` 先 `validateServiceInterface`（必须是接口、不能有类型参数、不能继承带类型参数的接口），然后 `Proxy.newProxyInstance` 生成代理。所有接口方法调用都会进入一个匿名的 `InvocationHandler`：`Object` 方法与接口 `default` 方法被提前分流，其余走 `loadServiceMethod(service, method).invoke(proxy, args)`。

### 2. 解析 + 缓存 ServiceMethod

`loadServiceMethod` 以 `ConcurrentHashMap<Method, Object>` 做三级状态：**未解析 → 解析中（存一个锁对象）→ 已解析（存 ServiceMethod）**。解析失败会写入一个失败标记，后续线程重试时直接复现异常，避免重复解析同一个坏方法。缓存 key 是 `Method` 对象，因此 **同一个 Method 只解析一次**，这是 Retrofit 的性能核心。

解析链路：`ServiceMethod.parseAnnotations` → 校验返回类型（不能是 void、不能含类型变量/通配符）→ `HttpServiceMethod.parseAnnotations` → 用 `retrofit.callAdapter(returnType, annotations)` 选中 `CallAdapter`，用 `retrofit.responseBodyConverter(responseType, annotations)` 选中响应体 `Converter`，最后按返回类型生成 `CallAdapted` / `SuspendForResponse` / `SuspendForBody` 三个变体之一。

### 3. invoke 一次调用

`HttpServiceMethod.invoke` 只做两件事：

```java
Call<ResponseT> call = new OkHttpCall<>(requestFactory, instance, args, callFactory, responseConverter);
return adapt(call, args);   // 交给 CallAdapter 决定返回什么
```

- 返回 `Call<T>` → `DefaultCallAdapterFactory` 直接返回 `OkHttpCall`；
- 返回 `Observable`/`Flowable` → RxJava 适配器包一层；
- `suspend fun` → `SuspendForBody` 用 `KotlinExtensions.await()` 挂起协程。

注意 `OkHttpCall` 是**懒创建 rawCall** 的：真正的 `okhttp3.Call` 要等到 `execute()`/`enqueue()` 时才由 `createRawCall()` 生成（`callFactory.newCall(requestFactory.create(instance, args))`），所以同一个 `Call` 被 cancel 或从未执行都不会产生网络开销。

### 4. 网络与响应

`OkHttpCall.enqueue` 把请求交给 okhttp 的线程池，回调里调用 `parseResponse`：包装响应体（避免响应流被提前消费）、判断 2xx/204/205/错误码、必要时缓冲 body 生成 `Response.error`，成功路径再用 `responseConverter` 反序列化。异步回调最终由 `callbackExecutor`（Android 上就是 `AndroidMainExecutor` 的 `Handler(Looper.getMainLooper())`）切回主线程。

## 设计要点

1. **两组工厂，两级责任链**：`Converter.Factory` / `CallAdapter.Factory` 都是「遍历工厂列表，谁先返回非 null 谁生效」，所以自定义适配器和转换器不需要改框架代码。
2. **ServiceMethod 缓存 + 失败标记**：用 `ConcurrentHashMap` 的 value 类型做状态机，避免重复解析，同时保证并发安全。
3. **接口即声明**：注解 → `RequestFactory` 的 `ParameterHandler` 列表，运行时把实参逐个写入请求。
4. **平台差异收敛到 Platform**：Java 8 默认方法、Android 主线程 Executor、反射优化都从这里分叉。
5. **与 OkHttp 解耦**：核心只依赖 `okhttp3.Call.Factory` 接口，测试可用 `retrofit-mock` 替换。

## 阅读路径

1. `Retrofit.java` — `create()` / `loadServiceMethod()`（第 158~262 行）
2. `ServiceMethod.java` — 解析入口与返回类型校验
3. `HttpServiceMethod.java` — 工厂选择、三个 adapt 变体
4. `CallAdapter.java` / `Converter.java` — 两个扩展点契约
5. `OkHttpCall.java` — `enqueue` / `createRawCall` / `parseResponse`
6. `RequestFactory.java` — 注解与实参如何变成请求
7. `Platform.java` + `AndroidMainExecutor.java` — 主线程回调
8. `retrofit-adapters/rxjava3` 或 `KotlinExtensions.kt` — 看适配器/协程如何接入

## 版本注记

- 该 commit 位于 3.x 开发线（`3.1.0-SNAPSHOT`），核心仍是 Java；`retrofit-adapters/rxjava3`、`kotlinx-serialization` 转换器等模块已拆分。
- 上游仓库已从 `square/retrofit` 迁移（本快照的 gradle.properties POM 指向 lysine-dev），旧链接会重定向，git 克隆不受影响。
- 排行榜成文于 2017 年（当时版本 2.3.0），本目录图表对应的是当前快照，类名与包结构保持一致（`ServiceMethod`、`OkHttpCall`、`CallAdapter` 等自 2.x 起未变）。
