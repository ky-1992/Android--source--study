# greenDAO 源码解析

## 一句话定位

greenDAO 是面向 Android 的轻量 ORM：编译期用 `DaoGenerator` 把实体 schema 生成 `DaoMaster`/`DaoSession`/`XxxDao` 与 `Xxx` 实体类，运行期 `DaoCore` 只保留 `Abstract*` 基类，通过 `Property` 元数据 + 预编译语句完成对 SQLite 的增删改查，并用 `IdentityScope` 做会话级对象缓存。

## 源码快照

- 上游：<https://github.com/greenrobot/greenDAO>
- commit：`0bbb338e17c4cf28aba5aae62d42f73f50186157`（2024-04-30，README 维护公告）
- 版本：根 `build.gradle` 为 `3.3.0`；本次快照时项目已停止活跃维护
- 仓库模块：`DaoCore/` 运行期核心；`DaoGenerator/` 代码生成器（`Schema`/`Entity` + `src-template/*.ftl` 模板）；`greendao-api/` 提供 `@Entity`、`@Id` 等注解；`tests/`、`examples/` 为测试与示例

运行期核心在 `DaoCore/src/main/java/org/greenrobot/greendao/`；查询相关在 `query/` 子包，内部元数据在 `internal/`，SQLCipher 适配在 `database/`。

## 核心类

| 类 | 职责 | 关键成员（行号取自快照） |
| --- | --- | --- |
| `AbstractDaoMaster` | 持有 `Database` 与 `daoConfigMap`，每登记一个 DAO 就构建一份 `DaoConfig` | `registerDaoClass()` L43、`newSession()` L57 |
| `AbstractDaoSession` | 实体类到 DAO 的映射 + 事务/CRUD 便捷方法 | `getDao()` L138、`insert()` L69、`runInTx()` L149、`callInTx()` L163 |
| `AbstractDao<T,K>` | CRUD 主实现，子类只需实现 `bindValues`/`readEntity` 等抽象方法 | `insert()` L318、`executeInsertInTx()` L271、`executeInsert()` L343、`insertInsideTx()` L363、`updateKeyAfterInsertAndAttach()` L376、`attachEntity()` L834、`queryBuilder()` L792 |
| `DaoConfig` | 由生成类的 `TABLENAME` 与 `$Properties` 反射出的表元数据 | 构造 L53、`reflectProperties()` L98、`initIdentityScope()` L163 |
| `TableStatements` | 按表缓存 insert/update/delete 等预编译语句 | `getInsertStatement()` L49、`getInsertOrReplaceStatement()` L65 |
| `Property` | 列元数据，并作为 `WhereCondition` 工厂 | 字段 L31–L35、`eq()` L46、`between()` L61 |
| `QueryBuilder<T>` | 链式拼 SELECT，用 `Property` 常量代替手写列名 | `where()` L134、`orderAsc()` L215、`build()` L281、`list()` L424 |
| `Query<T>` | 可复用的查询对象，按线程拿副本 | `create()` L57、`forCurrentThread()` L81、`list()` L86 |
| `WhereCondition` | WHERE 片段抽象，`PropertyCondition`/`StringCondition` 负责拼接 | 接口 L29、`PropertyCondition` L71、`appendTo` L138 |
| `DaoGenerator` | 读 `Schema` 生成实体、DAO、Master、Session | `generateAll()` L108、生成 Master L143、生成 Session L145 |

类图见 [class-diagram.html](./class-diagram.html)。需说明：`DaoMaster`、`DaoSession`、具体 `XxxDao` 都由 `DaoGenerator` 用 FreeMarker 模板生成（`dao-master.ftl` L82 调 `registerDaoClass`），仓库里只有 `Abstract` 基类与模板，没有生成产物源码。

## 关键流程

1. **装配**：生成的 `DaoMaster` 构造时对每个 DAO 调 `registerDaoClass()`（L43），`DaoConfig` 构造（L53）反射读表名与属性数组，按属性是否 `primaryKey` 拆出 `pkColumns`/`nonPkColumns`，并创建 `TableStatements`；`newSession(type)` 把 `daoConfigMap` 交给生成的 `DaoSession`，其构造里对每个配置 `initIdentityScope(type)`（L163）并克隆给具体 DAO（模板 `dao-session.ftl` L55–L61）。
2. **写入**：`AbstractDao.insert()`（L318）取 `statements.getInsertStatement()`，进入 `executeInsert()`（L343）。若当前线程已持有 DB 锁，直接 `insertInsideTx()`（L363）；否则 `beginTransaction` 后执行并 `setTransactionSuccessful`。`insertInsideTx` 在 `synchronized(stmt)` 内 `bindValues` 再 `executeInsert`，拿到 rowId 后 `updateKeyAfterInsertAndAttach()`（L376）回填主键并 `attachEntity()`（L834）写入 `IdentityScope`。
3. **查询**：`dao.queryBuilder()`（L792）→ `where(property.eq(...))` 把条件交给 `WhereCollector` → `build()`（L281）拼出 SELECT/WHERE/ORDER BY/LIMIT/OFFSET，值进 `values` 列表，最后 `Query.create()`（L57）缓存 `QueryData` 并 `forCurrentThread()` 返回线程副本；`list()`（L86）执行 `rawQuery` 后由 `InternalQueryDaoAccess` 逐行 `readEntity`。
4. **批量与事务**：`insertInTx`（L271）只开一次事务，循环绑定执行；`AbstractDaoSession.runInTx`（L149）与 `callInTx`（L163）提供外部事务包装。

流程图见 [flow-diagram.html](./flow-diagram.html)，覆盖一次 `insert` 的准备、执行、提交与挂载。

## 设计要点

- **生成期取代运行期反射**：表名、列名、属性类型在编译期固化进 `XxxDao` 与 `Properties` 内部类，运行期只做一次反射装配，避免每次查询解析注解。
- **语句缓存**：`TableStatements` 用双检锁按表缓存 `DatabaseStatement`，`bindValues` 与 `executeInsert` 整体在 `synchronized(stmt)` 内，兼顾线程安全与复用。
- **事务判断避免嵌套**：`executeInsert` 先用 `db.isDbLockedByCurrentThread()` 判断，已在事务中就直接写，减少无谓的 begin/end。
- **IdentityScope 保证对象同一性**：同一主键在会话内只有一份实例；`attachEntity` 的 `lock` 参数与 `putNoLock` 区分是否已持有锁。
- **查询对象可复用**：`QueryData` + `forCurrentThread` 让一条 SQL 在多线程各持参数副本，`checkThread`（`AbstractQuery` L97）阻止跨线程误用。

## 阅读路径

1. `DaoCore/src/main/java/org/greenrobot/greendao/AbstractDaoMaster.java`
2. `DaoCore/src/main/java/org/greenrobot/greendao/AbstractDaoSession.java`
3. `DaoCore/src/main/java/org/greenrobot/greendao/internal/DaoConfig.java`
4. `DaoCore/src/main/java/org/greenrobot/greendao/internal/TableStatements.java`
5. `DaoCore/src/main/java/org/greenrobot/greendao/AbstractDao.java`（先 L62–L400 的字段与写入路径）
6. `DaoCore/src/main/java/org/greenrobot/greendao/Property.java` 与 `query/WhereCondition.java`
7. `DaoCore/src/main/java/org/greenrobot/greendao/query/QueryBuilder.java`、`query/Query.java`
8. `DaoGenerator/src/org/greenrobot/greendao/generator/DaoGenerator.java` 与 `DaoGenerator/src-template/*.ftl`

## 版本注记

快照对应 3.3.0 代码线，3.x 已用注解 + 生成器取代 1.x/2.x 的 `DaoMaster` 手写与 `DaoGenerator` 独立工程模式；`DaoCore` 与 `DaoGenerator` 仍分模块维护。所有行号以该 commit 为准；若切到更高版本需重新核对。
