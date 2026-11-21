# realm-java 源码解析

## 一句话定位

realm-java 是 Realm 数据库的 Android Java 绑定层：对外提供 `Realm`、`RealmObject`、`RealmQuery` 等面向对象 API，对内把每个对象模型映射成 native 的表、行与版本句柄，真正的存储与查询全部在 Realm Core（C++）中执行，Java 侧只负责生命周期、线程约束和句柄管理。

## 源码快照

- 上游仓库：https://github.com/realm/realm-java
- 克隆 commit：`e5f4caac29d2ca36022352ef5c1f59045750e021`
- 版本线索：`version.txt` 为 `11.0.0`，`dependencies.list` 指定 `REALM_CORE=13.26.0`
- 模块划分：`realm/realm-library` 是 Java API 与 JNI 声明主体；`realm-annotations` 提供 `@RealmClass`、`@PrimaryKey` 等注解；`realm-transformer` 与 `gradle-plugin` 负责构建期字节码处理，把模型类替换为代理类。
- 关键包：`io.realm` 放公开 API、注解和对象基类；`io.realm.internal` 放 `Os*` 系列 native 包装（`OsSharedRealm`、`OsResults`、`OsRealmConfig`、`OsObject` 等），是 Java 与 C++ 之间的唯一缝。

## 核心类

| 类 | 职责 | 关键方法（行号） |
|---|---|---|
| `Realm` | 线程内单例门面 | `init`(262)、`getInstance`(459)、`where`(1562)、`executeTransaction`(1623) |
| `BaseRealm` | 持有 `OsSharedRealm`，提供事务入口 | `beginTransaction`(446)、`commitTransaction`(457)、`close`(657) |
| `RealmCache` | 按文件路径复用实例、维护引用计数 | `createRealmOrGetFromCache`(420)、`doCreateRealmOrGetFromCache`(431) |
| `RealmObject` | 受管对象基类 | `deleteFromRealm`(94)、静态版(107) |
| `RealmQuery` | 链式条件构造，内部包住 `TableQuery` | `createQuery`(93)、`equalTo`(291)、`findAll`(2462) |
| `RealmResults` | 对 `OsResults` 的集合包装 | 构造(98)、`where`(114)、`isLoaded`(134) |
| `ProxyState` | 受管对象的运行态（realm + row） | `getRow$realm`(103)、`setRow$realm`(107) |
| `OsSharedRealm` | JNI 句柄包装 | `getInstance`(246)、`nativeGetSharedRealm`(597) |

## 关键流程

1. **初始化**：`Realm.init(Context)` 转入 `initializeRealm`，用 ReLinker 装载 `realm-jni`，补一份默认 `RealmConfiguration`，初始化临时目录，并把 applicationContext 记到 `BaseRealm` 上，供后续所有实例共享。
2. **取实例**：`getInstance` 进入 `RealmCache.createRealmOrGetFromCache`，按 Realm 文件路径找到缓存桶；首次打开时 `BaseRealm` 组装 `OsRealmConfig` 并调用 `OsSharedRealm.getInstance`，最终落到 `nativeGetSharedRealm` 打开文件、跑迁移回调。
3. **写事务**：`executeTransaction` 先做 UI 线程写检查，再 `beginTransaction` → 执行用户 `Transaction` → `commitTransaction`，抛异常时改走 `cancelTransaction`；事务调用只是转发给 `sharedRealm`。
4. **查询**：`where(clazz)` 建出 `RealmQuery`，`equalTo` 等方法只往 `TableQuery` 追加条件；`findAll` 才创建 `OsResults` 并包成 `RealmResults`，行数据要逐条 `get` 才回到 Java 对象。
5. **对象读写**：字段访问被代理成 `ProxyState.getRow$realm()` 上的读写，`RealmObject` 子类只是被代理的壳；对象失效、Realm 关闭或跨线程访问时，都由 `checkValidObject` 与 `BaseRealm.checkIfValid` 统一抛异常，避免上层拿到脏数据。

## 设计要点

- **Java 不碰存储**：数据全在 native 的 Realm Core，Java 侧只持有 `long` 指针；`NativeContext` 与 `NativeObjectReference` 用引用队列在 GC 后释放 native 对象，避免手写 free，也因此所有 `Os*` 对象都必须遵守「谁创建谁持有」的约定。
- **代理是核心机制**：注解处理器为每个模型生成 `xxxRealmProxy`，实现 `RealmObjectProxy` 并持有 `ProxyState`，字段读写被改写成对 `Row` 的操作，`RealmObject` 本身接近空基类。
- **线程一旦绑定不可漂移**：`BaseRealm` 记录 `threadId`，跨线程访问直接抛异常；实例按线程缓存，ThreadLocal 与全局两级计数决定真正 close 的时机。
- **事务靠版本推进通知**：提交后版本号前移，`RealmChangeListener` 由 `OsSharedRealm` 的 notifier 回主线程派发，查询结果自动看到新数据。
- **配置不可变**：`RealmConfiguration` 字段全部 `final`，由 `Builder.build()` 一次固化，避免运行期被改。
- **UI 线程保护**：写事务和查询默认检查线程，要求显式 `allowWritesOnUiThread`，把卡顿风险前置暴露。

## 阅读路径

1. `realm/realm-library/src/main/java/io/realm/Realm.java`
2. `realm/realm-library/src/main/java/io/realm/RealmConfiguration.java`
3. `realm/realm-library/src/main/java/io/realm/BaseRealm.java`
4. `realm/realm-library/src/main/java/io/realm/RealmCache.java`
5. `realm/realm-library/src/main/java/io/realm/internal/OsSharedRealm.java`
6. `realm/realm-library/src/main/java/io/realm/RealmQuery.java`
7. `realm/realm-library/src/main/java/io/realm/RealmResults.java`
8. `realm/realm-library/src/main/java/io/realm/ProxyState.java`

## 版本注记

以 commit `e5f4caa` 为准（`version.txt` 写 11.0.0，Realm Core 13.26.0）。此版本已用 `Os*` 前缀统一包装层命名（`OsSharedRealm`、`OsResults`、`OsRealmConfig`），同时保留同步与异步两套查询 API；`Realm.init(Context)` 仍然存在，未初始化就取默认实例会提示先调用它。文中行号以该 commit 实际文件为准，不同分支可能漂移。
