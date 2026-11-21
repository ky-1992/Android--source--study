# Tinker 源码解析

## 一句话定位

Tinker 是腾讯开源的 Android 热修复框架：构建期用 dex 差分打出小体积补丁包，运行期在独立 `:patch` 进程里校验并合成补丁产物，下一次进程启动时由 `TinkerLoader` 把补丁 dex 注入 `ClassLoader`，从而在不重装 APK 的前提下替换类、so 与资源。

## 源码快照

- 上游：<https://github.com/Tencent/tinker>
- 本目录对应 commit：`c72fce285ce10edb244d4142685d40b91cb1625d`（1.9.15.5 稳定版）
- 模块划分：`tinker-android`（`tinker-android-lib` 补丁逻辑与 Service、`tinker-android-loader` 启动加载、`tinker-android-anno` 注解生成、no-op 空实现）、`tinker-build`（`tinker-patch-lib` 差分与打包、`tinker-patch-gradle-plugin` 构建插件、`tinker-patch-cli`）、`tinker-commons`（公共工具）
- 关键目录：`com.tencent.tinker.lib`（tinker/listener/service/patch/reporter）、`com.tencent.tinker.loader`（loader、`shareutil` 共享校验逻辑）、`com.tencent.tinker.build.decoder`（构建期 diff）

## 核心类

| 类 | 职责 | 关键方法（行号） |
|---|---|---|
| `TinkerInstaller` | 对外安装入口，组装 `Tinker.Builder` 并触发安装 | `install` `TinkerInstaller.java:43`，`onReceiveUpgradePatch` `:109` |
| `Tinker` | 单例门面，持有补丁目录、报告器、`TinkerLoadResult`，安装时把 `AbstractPatch` 交给服务 | `with` `Tinker.java:105`，`install` `Tinker.java:141`，`isTinkerLoaded` |
| `TinkerPatchService` | 独立 `:patch` 进程里的 `IntentService`，串行执行补丁并回调结果服务 | `runPatchService` `TinkerPatchService.java:62`，`setPatchProcessor` `:79`，`doApplyPatch` `:201` |
| `AbstractPatch` / `UpgradePatch` | 补丁处理器抽象与默认实现：校验包、拷贝产物、恢复 dex/so/资源、写 `patch.info` | `tryPatch` `UpgradePatch.java:45`，`tryRecoverDexFiles` `:174`，`rewritePatchInfoFileWithLock` `:214` |
| `DexDiffPatchInternal` | 按 `dex-meta` 解出补丁 dex、等待 dexopt，处理 classN 与 Dalvik/ART 差异 | `tryRecoverDexFiles` `DexDiffPatchInternal.java:73`，`extractDexDiffInternals` `:462` |
| `ShareTinkerInternals` | loader 与服务共用的判定：进程、VM、tinkerId、签名与包属性校验、清理 | `checkTinkerPackage` `ShareTinkerInternals.java:209`，`fastGetPatchPackageMeta` `:285`，`cleanPatch` `:738` |
| `TinkerLoader` / `TinkerDexLoader` | 启动时读 `patch.info` 与 guard 锁，校验通过后装载补丁 dex | `tryLoad` `TinkerLoader.java:51`，`loadTinkerJars` `TinkerDexLoader.java:66`，`checkComplete` `:208` |
| `SystemClassLoaderAdder` | 反射把补丁 dex 插到 `PathClassLoader` 的 dex 列表前面，按 API 分版本实现 | `installDexes` `SystemClassLoaderAdder.java:54`，`injectDexesInternal` `:78` |

## 关键流程

1. **构建期生成补丁**：`DexDiffDecoder.patch` 逐对比较新旧 dex，收集变化类并由 dex 差分算法生成补丁 dex，`generatePatchInfoFile` 写出 `dex-meta.txt`；`ApkDecoder` 把 dex、so、资源差分结果与 `package_meta` 一起打包成 patch APK。
2. **接收补丁**：`TinkerInstaller.onReceiveUpgradePatch` 交给 `PatchListener`，默认实现 `DefaultPatchListener` 做快速检查后调用 `TinkerPatchService.runPatchService`，把补丁路径通过 Intent 送进 `:patch` 进程。
3. **合成补丁**：`doApplyPatch` 调 `upgradePatchProcessor.tryPatch`；`UpgradePatch` 先校验签名、tinkerId、MD5 与旧补丁状态，再用 `DexDiffPatchInternal.tryRecoverDexFiles` 把补丁应用到新的补丁版本目录，最后写 guard 锁和 `patch.info`。
4. **启动加载**：应用启动时 `TinkerLoader.tryLoad` 读取 `patch.info`、判断版本变化，`TinkerDexLoader.loadTinkerJars` 校验 MD5 后调用 `SystemClassLoaderAdder.installDexes` 注入 dex；加载结果通过 Intent 回传给 `Tinker.install` 解析成 `TinkerLoadResult`。

## 设计要点

- **补丁目录是唯一事实源**：所有差异产物、`patch.info`、guard 锁都落在 `tinker/` 下，加载与回滚只读这一份状态，避免服务与启动路径各存一份。
- **应用与加载分离**：补丁合成为独立进程的同步任务，类替换留到下次启动，降低主进程内存与崩溃风险。
- **多重校验**：签名、tinkerId、包属性、MD5 分别在服务端合成前和启动加载时校验，任何一步失败只上报 `PatchReporter`，不替换线上产物。
- **分版本注入**：`SystemClassLoaderAdder` 针对 API 4/14/19/23/24 提供不同反射实现，Android 8.0 之后用 `NewClassLoaderInjector` 处理。
- **失败隔离与回滚**：`UpgradePatchRetry`、guard 锁与 safe mode 计数限制连续加载失败，配合 `rollbackPatch` 让坏补丁不会卡死启动。
- **共享校验下沉**：`ShareTinkerInternals` 同时被 `tinker-android-lib` 与 loader 依赖，保证构建期写出的元信息与运行期读取口径一致。

## 阅读路径

1. `tinker-android/tinker-android-lib/src/main/java/com/tencent/tinker/lib/tinker/TinkerInstaller.java`——安装与打补丁的对外入口。
2. `.../lib/tinker/Tinker.java`——单例门面与 `install` 对加载结果的解析。
3. `.../lib/service/TinkerPatchService.java`——`:patch` 进程与 `doApplyPatch` 串行化。
4. `.../lib/patch/UpgradePatch.java`——一次补丁应用的全部校验与产物落盘。
5. `.../lib/patch/DexDiffPatchInternal.java`——dex 差异如何被还原。
6. `tinker-android/tinker-android-loader/src/main/java/com/tencent/tinker/loader/shareutil/ShareTinkerInternals.java`——共享校验逻辑。
7. `.../loader/TinkerLoader.java`——启动时读 `patch.info` 的主流程。
8. `.../loader/TinkerDexLoader.java` 与 `.../loader/SystemClassLoaderAdder.java`——dex 装载与类加载器注入。

## 版本注记

本目录依据 1.9.15.5 稳定版 commit；1.9.x 已支持 Android 8.0 以上的类加载器注入，并保留 classN 与 Dalvik/ART 双路径处理，协议为 BSD-3。构建插件与 `tinker-patch-cli` 的差分实现位于 `tinker-build/tinker-patch-lib`，本解析以运行期 lib 与 loader 为主；`TinkerApplication` 在 `attachBaseContext` 阶段触发加载，因此列表中的加载流程先于 `Tinker.install` 的解析发生。
