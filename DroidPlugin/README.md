# DroidPlugin 源码解析

## 一句话定位

DroidPlugin 是一个「免安装、免修改、免重打包」就能运行第三方 apk 的 Android 插件框架：它用反射加动态代理 Hook 系统服务，再用宿主清单里预注册的空壳组件（占坑）骗过系统，最后在插件进程里把组件还原成插件自己的实现。

## 源码快照

- 上游：<https://github.com/DroidPluginTeam/DroidPlugin>
- commit：`c6ebf652e0f73aa0e5746766e117e51efaf41dbd`（2019-12-14，仓库最后一次提交）
- 工程结构：核心库在 `project/Libraries/DroidPlugin/`，包 `com.morgoo.droidplugin` 下分 `am`（组件与进程调度）、`core`（类加载与进程）、`hook`（`proxy` / `binder` / `handle` 三层）、`pm`（插件包服务与 `parser`）、`stub`（占坑组件）、`reflect`；仓库约 186 个 Java 文件。构建侧 `compileSdkVersion` 为 28、`minSdkVersion` 为 9、Gradle 插件 3.5.3，模块 `versionName` 是 `1.0`。

## 核心类

| 类 | 职责 | 关键方法（行号） |
|---|---|---|
| `PluginHelper` | 框架初始化入口 | `applicationOnCreate`(PluginHelper.java:62)、`initPlugin`(:69) |
| `PluginManager` | 插件包服务客户端，AIDL 代理 | `installPackage`(PluginManager.java:871)、`selectStubActivityInfo`(:661) |
| `IPluginManagerImpl` | 服务端，维护 `mPluginCache` | `installPackage`(:819)、`selectStubActivityInfo`(:1117) |
| `PluginPackageParser` | 用 `PackageParser` 解析插件 apk | 构造器(:90)、`collectCertificates`(:231) |
| `PluginClassLoader` | `DexClassLoader` 子类 | `loadClass`(:50) |
| `HookFactory` | Hook 装配与开关 | `installHook`(:125) |
| `IActivityManagerHook` | AMS 动态代理 Hook | `onInstall`(:75)、`invoke`(:63) |
| `IActivityManagerHookHandle` | 按方法名分发改写逻辑 | `init`(:88)、`selectProxyActivity`(:2104) |
| `MyActivityManagerService` | 占坑组件与进程选择 | `selectStubActivityInfo`(:277) |
| `PluginCallback` | 拦 `ActivityThread.mH` 消息 | `handleLaunchActivity`(:352) |
| `ActivityStub` | 预注册的空壳 Activity | 类体(:31) |

## 关键流程

### 1. 初始化并装 Hook

`PluginHelper.initPlugin` 依次执行 `fixMiUiLbeSecurity`（清掉小米安全中心的 ClassLoader 与消息队列钩子）、`PluginPatchManager.init`、`PluginProcessManager.installHook`，再按 `isPluginProcess` 决定 Hook 开关，最后 `PluginManager.init` 连接服务。`HookFactory` 在主进程只装 `IActivityManagerHook` 与 `IPackageManagerHook`；在插件进程则装一大串 Binder Hook，外加 `PluginCallbackHook`、`InstrumentationHook`、`LibCoreHook`、`SQLiteDatabaseHook`。

### 2. 安装插件

`PluginManager.installPackage` 走 Binder 到 `IPluginManagerImpl`：先用宿主 `PackageManager.getPackageArchiveInfo` 取包名，把 apk 复制到 `PluginDirHelper` 目录，再 `new PluginPackageParser` 解析四大组件与 `IntentFilter`，`collectCertificates` 后把签名落到私有目录，接着 `copyNativeLibs`、`dexOpt`（内部 `new PluginClassLoader` 触发 dex 优化），最后 `mPluginCache.put` 并 `onPkgInstalled`、发安装广播。

### 3. 启动拦截与替换

`IActivityManagerHook.onInstall` 按 SDK 分支读 `ActivityManager.IActivityManagerSingleton` 或 `ActivityManagerNative.gDefault`，用 `MyProxy.newProxyInstance` 生成代理并写回字段。代理 `invoke` 时按方法名查 handler；`startActivity` 系列由 `doReplaceIntentForStartActivityAPIHigh` 把插件组件换成桩组件，原 Intent 塞进 `Env.EXTRA_TARGET_INTENT`，同时替换 `callingPackage`。

### 4. 占坑组件选择

`IActivityManagerHookHandle.selectProxyActivity` 调 `PluginManager.selectStubActivityInfo`，服务端落在 `MyActivityManagerService.selectStubActivityInfo`：先按运行中进程找可复用的桩，再从 `StaticProcessList` 挑未被占用的桩，并登记 pid/uid。`ActivityStub` 用 `P00`~`P08` 与四种 `launchMode` 组合出成批空壳，这些类必须提前写进宿主清单。

### 5. 插件进程还原

`PluginCallback` 作为 `Handler.Callback` 拦截 `LAUNCH_ACTIVITY`：读出 `EXTRA_TARGET_INTENT`，`preLoadApk` 注入 `LoadedApk`，`PluginProcessManager.getPluginClassLoader` 拿到类加载器并设到 Intent 上，再改写 `msg.obj` 的 `intent` 与 `activityInfo`；`PluginInstrumentation.callActivityOnCreate` 继续做系统服务替换与 `mOpPackageName` 修正。

## 设计要点

1. 插件包服务独立进程：通过 `IPluginManager.Stub` 的 AIDL 暴露能力，插件元数据与签名不污染宿主进程。
2. 不依赖 root 与 Xposed：全部靠反射写 `ActivityThread`、`ActivityManagerNative` 等内部字段，再配动态代理拦截。
3. 占坑换真身：桩组件是系统唯一看得见的「合法入口」，插件真身通过 Intent extra 传递，代价是桩数量有上限、需按 `launchMode` 分桶。
4. 类加载隔离：`PluginClassLoader` 继承 `DexClassLoader`，并把插件 `LoadedApk` 注入 `ActivityThread.mPackages`，让资源与类都能按包名解析。
5. 版本适配密集：大量 `Build.VERSION.SDK_INT` 分支处理字段与方法差异，可读性差但兼容面广。
6. 边界：仓库 README 明确写了未 Hook native 层，带 native 代码的 apk 无法作为插件运行。

## 阅读路径

1. `project/Libraries/DroidPlugin/src/main/java/com/morgoo/droidplugin/PluginHelper.java`
2. `.../hook/HookFactory.java`
3. `.../hook/proxy/IActivityManagerHook.java`
4. `.../hook/handle/IActivityManagerHookHandle.java`
5. `.../pm/PluginManager.java` 与 `.../pm/IPluginManagerImpl.java`
6. `.../pm/parser/PluginPackageParser.java`
7. `.../am/MyActivityManagerService.java` 与 `.../stub/ActivityStub.java`
8. `.../hook/handle/PluginCallback.java` 与 `.../hook/handle/PluginInstrumentation.java`

## 版本注记

本目录所有结论以 commit `c6ebf65…` 为准：这是仓库 2019 年 12 月合并 #448 后的状态，此后上游没有新提交；模块 `versionName` 停留在 `1.0`，编译目标 `compileSdkVersion` 28。框架面向的是 Android 2.3~8 一代的系统实现（大量 `getDeclaredField` 读隐藏字段），高版本系统上能否工作取决于这些字段是否还存在。
