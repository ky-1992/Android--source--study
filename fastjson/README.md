# fastjson 源码解析

## 一句话定位

fastjson 是阿里巴巴开源的 Java JSON 库，主打“快”：对外是 `JSON` 静态门面，内部把工作拆成词法层（`JSONLexer`）、语法层（`DefaultJSONParser`）和编解码注册表（`ParserConfig` / `SerializeConfig`）。解析时按目标 `Type` 取 `ObjectDeserializer` 填充 Bean，序列化时按运行期 `Class` 取 `ObjectSerializer` 写出文本，JVM 上还会用 ASM 生成专用序列化器。

## 源码快照

- 上游：<https://github.com/alibaba/fastjson>
- commit：`c942c83443117b73af5ad278cc780270998ba3e1`（2023-05-12，提交说明 “Update README.md”）
- 版本：`pom.xml` 声明 `1.2.84-SNAPSHOT`，属于 fastjson 1.x 维护线
- 结构：`src/main/java/com/alibaba/fastjson/` 下按职责分包——`parser/`（`DefaultJSONParser`、`JSONLexer`、`JSONScanner`、`ParserConfig`、`deserializer/`）、`serializer/`（`SerializeConfig`、`JSONSerializer`、`ObjectSerializer`、`JavaBeanSerializer`）、`asm/`（ASM 工厂）、`annotation/`、`support/`、`util/`；根包放 `JSON`、`JSONObject`、`JSONArray`、`TypeReference`、`JSONPath` 等类型。

## 核心类

| 类 | 职责 | 关键位置 |
| --- | --- | --- |
| `JSON` | 静态门面：`parseObject` 建解析器，`toJSONString` 建 `JSONSerializer` | 类声明 L65、`parseObject` L366、`toJSONString` L771 |
| `DefaultJSONParser` | 递归下降解析：持有 lexer 与 config，按 token 分派 | 字段 L42–52、`parseObject` L674、`parse` L1402、`handleResovleTask` L1577 |
| `JSONLexer` | 词法协议：`token` / `nextToken` / `scanSymbol` / `stringVal` | 接口 L23、方法 L100–126 |
| `JSONLexerBase` / `JSONScanner` | 抽象词法骨架与 String 输入实现 | L35、`JSONScanner` L35 |
| `ParserConfig` | 反序列化器缓存、SymbolTable、AutoType 安全策略 | 类 L68、`getDeserializer` L577/608、`checkAutoType` L1311、`setSafeMode` L546 |
| `JavaBeanDeserializer` | 字段匹配、创建实例并反射设值 | 类 L23、`createInstance` L190、`deserialze` L392 |
| `SerializeConfig` | 序列化器缓存，按需创建 Bean 序列化器 | 类 L51、`getObjectWriter` L439、`createJavaBeanSerializer` L121 |
| `ObjectSerializer` / `JavaBeanSerializer` | 写出协议与默认 Bean 实现 | L58、`write` L149 |

## 关键流程

**解析入口**：`JSON.parseObject`（L366）先把 `Feature...` 合并进 `featureValues`，再 `new DefaultJSONParser(input, config, featureValues)`，最后 `parser.parseObject(clazz, null)`。`DefaultJSONParser` 构造时把输入包成 `JSONScanner`，并依据首字符把当前 token 预置为 `LBRACE`/`LBRACKET`（L156–175）。

**取反序列化器**：`parseObject(Type, Object)`（L674）先处理 `NULL`、`byte[]`、`char[]` 等特例，然后 `config.getDeserializer(type)`（L696）；若拿到的是 `JavaBeanDeserializer`，直接走其 `deserialze` 快路径，否则调通用 `ObjectDeserializer.deserialze`。`ParserConfig.getDeserializer`（L577）先用 `IdentityHashMap` 缓存命中，再按 `Class` / 参数化类型 / 通配符逐层解析。

**填充 Bean**：`JavaBeanDeserializer.deserialze`（L392）用 `symbolTable` 与字段名做匹配，命中字段后按类型调用 `lexer.scanInt/scanString/scanBoolean` 等快扫方法；未命中或复杂类型则递归回 `parser.parseObject`。`@type` 一类的引用先记成 `ResolveTask`，等 `JSON.parseObject` 返回前调 `handleResovleTask`（L1577）统一解析。

**序列化输出**：`JSON.toJSONString`（L771）创建 `SerializeWriter` 与 `JSONSerializer`，`serializer.write(object)`；`JSONSerializer.write`（L302）按 `object.getClass()` 调 `getObjectWriter`，再由 `SerializeConfig.getObjectWriter`（L439）查缓存或创建 `ObjectSerializer`，最后回调 `write` 把字段写进 writer。

## 设计要点

- **词法与语法分离**：`JSONLexer` 只负责字符流到 token 的扫描，`DefaultJSONParser` 负责语法结构，二者通过 `JSONToken` 常量协作，便于为 String、char[]、InputStream 提供不同 lexer 实现。
- **两级注册表缓存**：`ParserConfig` 与 `SerializeConfig` 都持有 `IdentityHashMap<Type, ...>`，首次解析/序列化后复用实例，避免每字段反射查表。
- **ASM 加速可关闭**：两个 Config 的 `asm` 字段都以 `!ASMUtils.IS_ANDROID` 初始化，JVM 上由 `ASMSerializerFactory` 生成专用序列化器，Android 上退回普通反射路径。
- **Feature 位掩码**：`DEFAULT_PARSER_FEATURE` 与 `DEFAULT_GENERATE_FEATURE` 在 `JSON` 静态块里用按位或组装，调用方还能按次传入 `Feature`，一个 int 控制全部开关。
- **AutoType 安全边界**：`@type` 键由 `DEFAULT_TYPE_KEY` 定义（L68），`ParserConfig` 用 deny 列表、accept 列表与 `safeMode` 在 `checkAutoType`（L1311）里做准入，这是 1.2.68 之后版本的重点修复面。
- **解析与序列化结构对称**：两侧都是“门面 → 配置注册表 → 编解码器 → 输出”四层，读通反序列化一侧后，序列化侧可以照着类推；`JSONReader`、`JSONWriter` 另提供流式读写，不走本文的整树主链路。

## 阅读路径

1. `src/main/java/com/alibaba/fastjson/JSON.java` —— L65–120 静态配置，L366–402 解析入口，L771–799 序列化入口。
2. `src/main/java/com/alibaba/fastjson/parser/DefaultJSONParser.java` —— L40–180 字段与构造，L674–712、L1402–1440。
3. `src/main/java/com/alibaba/fastjson/parser/JSONLexer.java` 与 `JSONLexerBase.java` —— L23–126、L35–120。
4. `src/main/java/com/alibaba/fastjson/parser/ParserConfig.java` —— L68–145、L577–640、L1299–1330。
5. `src/main/java/com/alibaba/fastjson/parser/deserializer/JavaBeanDeserializer.java` —— L23–60、L190–230、L392–460。
6. `src/main/java/com/alibaba/fastjson/serializer/SerializeConfig.java` —— L51–140、L439–528。
7. `src/main/java/com/alibaba/fastjson/serializer/JavaBeanSerializer.java` —— L45–60、L149–160。
8. `src/main/java/com/alibaba/fastjson/serializer/JSONSerializer.java` —— L302–316。

## 版本注记

该 commit 在 `master` 上，`pom.xml` 版本为 `1.2.84-SNAPSHOT`；此时项目已是 1.x 维护线，AutoType 相关安全开关（`safeMode`、deny/accept 列表）都已具备。fastjson 2.x 是几乎重写的另一套包结构，不在本快照范围内；若要精确对应发布版本，以 commit `c942c83443117b73af5ad278cc780270998ba3e1` 为准。
