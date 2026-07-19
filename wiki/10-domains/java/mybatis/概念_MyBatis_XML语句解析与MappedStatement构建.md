---
type: "concept"
tags: ["java", "mybatis", "xml", "mappedstatement", "source-code"]
summary: "MyBatis 将 mapper XML 编译为 MappedStatement 的链路：XMLMapperBuilder 解析 namespace、cache、resultMap、sql 片段和 statement 节点；XMLStatementBuilder 解析单条 select/insert/update/delete；MapperBuilderAssistant 统一命名空间、引用和默认值；MappedStatement.Builder 生成最终 SQL 执行元数据。"
sources:
  - "[[30-sources/repositories/来源_MyBatis_3_源码]]"
  - "raw/code/mybatis-3/src/main/java/org/apache/ibatis/builder/xml/XMLMapperBuilder.java"
  - "raw/code/mybatis-3/src/main/java/org/apache/ibatis/builder/xml/XMLStatementBuilder.java"
  - "raw/code/mybatis-3/src/main/java/org/apache/ibatis/builder/MapperBuilderAssistant.java"
  - "raw/code/mybatis-3/src/main/java/org/apache/ibatis/mapping/MappedStatement.java"
  - "raw/code/mybatis-3/src/main/java/org/apache/ibatis/builder/xml/XMLIncludeTransformer.java"
  - "raw/code/mybatis-3/src/main/java/org/apache/ibatis/scripting/xmltags/XMLLanguageDriver.java"
aliases: ["XMLMapperBuilder", "XMLStatementBuilder", "MapperBuilderAssistant", "MappedStatement Builder", "XML SQL 编译"]
status: "evolving"
confidence: 0.9
created: "2026-05-13 09:14:29"
updated: "2026-05-13 09:19:36"
---

# 概念：MyBatis XML语句解析与MappedStatement构建

![[Attachments/mybatis-mappedstatement-build-flow.excalidraw]]

## 定义

MyBatis 的 mapper XML 解析不是直接把 SQL 字符串塞进 Map，而是经过一条编译链：

```text
XMLMapperBuilder
-> XMLStatementBuilder
-> MapperBuilderAssistant
-> MappedStatement.Builder
-> Configuration.addMappedStatement()
```

这条链路把 `<select|insert|update|delete>` 节点编译成 `MappedStatement`。`MappedStatement` 是后续 Mapper 代理和 Executor 执行的共同入口。

## 为什么重要

这条链路解释了很多 MyBatis 使用和排错问题：

- 为什么 mapper XML 必须有 `namespace`。
- 为什么 `<sql id="...">` 可以被 `<include refid="...">` 引用。
- 为什么同一个 statement 可以按 `databaseId` 覆盖。
- 为什么 `<selectKey>` 会生成额外的 statement 和 key generator。
- 为什么 `resultType` 和 `resultMap` 的处理路径不同。
- 为什么 `flushCache/useCache` 在 select 与 update 类语句中默认值不同。
- 为什么缺失 resultMap/cache-ref 时会进入 incomplete 队列，稍后重试。

## 使用场景

- 排查 `Invalid bound statement`、`Could not find SQL statement to include`、`Could not find result map`。
- 深入理解 mapper XML 与 Mapper 接口绑定。
- 编写代码生成器时确定 XML 元素之间的依赖顺序。
- 开发 MyBatis 插件时判断 `MappedStatement` 里已经包含哪些执行元数据。
- 阅读动态 SQL、缓存、主键回填、databaseId 多数据库适配源码。

## 核心机制

### 1. XMLMapperBuilder：解析整个 mapper 文件

`XMLMapperBuilder#parse()` 先检查 `configuration.isResourceLoaded(resource)`，避免同一 mapper 重复解析。未加载时执行：

```text
configurationElement(/mapper)
-> configuration.addLoadedResource(resource)
-> bindMapperForNamespace()
-> parsePendingResultMaps(false)
-> parsePendingCacheRefs(false)
-> parsePendingStatements(false)
```

`configurationElement()` 的顺序是：

1. 读取并校验 `namespace`。
2. 解析 `cache-ref`。
3. 解析 `cache`。
4. 解析已废弃的 `parameterMap`。
5. 解析 `resultMap`。
6. 收集 `<sql>` 片段。
7. 解析 `select|insert|update|delete`。

这个顺序体现了依赖关系：statement 可以引用 resultMap、cache、sql fragment，所以这些内容要先进入 `Configuration` 或当前 builder 上下文。

### 2. namespace 是所有局部 id 的前缀规则

`MapperBuilderAssistant#setCurrentNamespace()` 会锁定当前 mapper 的命名空间。如果后续尝试切换到另一个 namespace，会抛出错误。

`applyCurrentNamespace(base, isReference)` 是理解 XML 解析的关键：

| 场景 | 规则 |
| --- | --- |
| 定义本 mapper 元素，如 `<select id="find">` | 不允许 id 内含点号，最终变成 `namespace.find` |
| 引用元素，如 `resultMap="BaseMap"` | 如果未带点号，补成 `namespace.BaseMap` |
| 跨 namespace 引用 | 引用场景允许写完整限定名 |

因此，MyBatis 的短 id 只是 XML 编写便利，进入 `Configuration` 后基本都变成全限定 id。

### 3. `<sql>` 与 `<include>` 是静态片段展开

`XMLMapperBuilder#sqlElement()` 把 `<sql id="...">` 放进 `configuration.sqlFragments`，key 是带 namespace 的 id。

`XMLStatementBuilder#parseStatementNode()` 在解析 SQL 前调用：

```text
XMLIncludeTransformer.applyIncludes(context.getNode())
```

`XMLIncludeTransformer` 会：

1. 找到 `<include refid="...">`。
2. 解析 refid 中的 `${}` 属性变量。
3. 按 namespace 查找 sql fragment。
4. clone 对应 DOM 节点。
5. 用 include 中声明的 `<property>` 替换片段里的变量。
6. 把 include 节点替换成片段子节点。

所以 `<include>` 是构建期 DOM 展开，不是运行期动态 SQL。

### 4. XMLStatementBuilder：解析单条语句

`parseStatementNode()` 处理一条 `<select|insert|update|delete>`：

1. 读取 `id` 和 `databaseId`，按当前 database 规则决定是否跳过。
2. 根据节点名得到 `SqlCommandType`。
3. 计算默认缓存策略：
   - select：`flushCache=false`，`useCache=true`
   - insert/update/delete：`flushCache=true`，`useCache=false`
4. 展开 `<include>`。
5. 解析 `parameterType`，必要时从 mapper 方法推断参数类型和 `ParamNameResolver`。
6. 选择 `LanguageDriver`。
7. 先解析并移除 `<selectKey>`。
8. 选择 `KeyGenerator`。
9. 用 language driver 创建 `SqlSource`。
10. 解析 statementType、fetchSize、timeout、resultType、resultMap、resultSetType、keyProperty、keyColumn、resultSets、affectData。
11. 调用 `builderAssistant.addMappedStatement(...)`。

这一阶段把 XML 属性转成强类型枚举、类、布尔值和执行策略。

### 5. databaseId 的覆盖规则

statement 和 sql fragment 都有类似规则：

- 如果当前要求特定 `databaseId`，只接受匹配的节点。
- 如果当前解析通用节点，带 `databaseId` 的节点会跳过。
- 如果同一全限定 id 已有专用 databaseId 版本，通用版本不会覆盖它。

这使同一 mapper 可以同时放通用 SQL 和数据库特定 SQL。

### 6. `<selectKey>` 会变成额外 MappedStatement

`processSelectKeyNodes()` 会把子节点 `<selectKey>` 解析为独立 statement，id 是：

```text
parentId + "!selectKey"
```

随后：

1. 用 `builderAssistant.addMappedStatement()` 创建 key statement。
2. 从 `Configuration` 取回这个 key statement。
3. 注册 `SelectKeyGenerator(keyStatement, executeBefore)`。
4. 从原 DOM 中移除 `<selectKey>`，避免主 SQL 再解析到它。

主 insert 语句随后会使用这个 key generator。

### 7. LanguageDriver 决定 SqlSource 类型

默认 `XMLLanguageDriver` 会用 `XMLScriptBuilder` 解析脚本节点。

`XMLScriptBuilder` 的核心判断：

- 如果包含动态标签或 `${}` 动态文本，生成 `DynamicSqlSource`。
- 否则生成 `RawSqlSource`。

动态标签包括：

```text
trim / where / set / foreach / if / choose / when / otherwise / bind
```

因此，`MappedStatement` 保存的不是最终 SQL 字符串，而是可在运行期根据参数生成 `BoundSql` 的 `SqlSource`。

### 8. MapperBuilderAssistant：收口命名空间、引用和默认值

`MapperBuilderAssistant#addMappedStatement()` 是 XML 解析到最终元数据的收口方法。它负责：

- 检查 `cache-ref` 是否已解决。
- 给 statement id 补 namespace。
- 创建 `MappedStatement.Builder`。
- 绑定 resource、statementType、keyGenerator、databaseId、language driver、resultMaps、cache、dirtySelect、paramNameResolver 等。
- 解析或创建 `ParameterMap`。
- `build()` 后加入 `Configuration.mappedStatements`。

其中 `getStatementResultMaps()` 有两条路径：

- 指定 `resultMap`：按引用找到已有 `ResultMap`。
- 指定 `resultType`：创建空的 inline `ResultMap`，id 为 `statementId + "-Inline"`。

### 9. MappedStatement.Builder：最终执行元数据

`MappedStatement.Builder` 最少需要：

```text
Configuration + id + SqlSource + SqlCommandType
```

构造器会设置默认值：

- `statementType = PREPARED`
- `resultSetType = DEFAULT`
- 空 `parameterMap`
- 空 `resultMaps`
- insert 且全局 useGeneratedKeys 时默认 `Jdbc3KeyGenerator`
- `statementLog`
- 默认 `LanguageDriver`

`build()` 前 `MapperBuilderAssistant` 会继续填充缓存、结果映射、参数映射、主键生成、数据库厂商、结果集、影响数据标记等字段。

## 例子

XML：

```xml
<mapper namespace="com.example.UserMapper">
  <sql id="BaseColumns">id, name, age</sql>

  <select id="selectById" resultType="User" useCache="true">
    select <include refid="BaseColumns"/>
    from user
    where id = #{id}
  </select>
</mapper>
```

构建结果可以理解为：

```text
namespace = com.example.UserMapper
sqlFragments["com.example.UserMapper.BaseColumns"] = XNode(sql)
statement id = com.example.UserMapper.selectById
sqlCommandType = SELECT
flushCacheRequired = false
useCache = true
resultMaps = [ResultMap("com.example.UserMapper.selectById-Inline", User)]
sqlSource = RawSqlSource 或 DynamicSqlSource
Configuration.mappedStatements[id] = MappedStatement
```

## 常见误区

- **误区：`<include>` 是运行期拼接。** 它是在 statement 解析前展开 DOM。
- **误区：`id="a.b"` 可以用于本 mapper 内元素。** 定义本地元素时 id 不能带点号，点号意味着命名空间边界。
- **误区：`resultType` 不会产生 resultMap。** MyBatis 会创建 inline empty `ResultMap`。
- **误区：`<selectKey>` 只是 insert 的一个属性。** 它会变成独立 `MappedStatement` 和 `SelectKeyGenerator`。
- **误区：`MappedStatement` 保存最终 SQL。** 它保存 `SqlSource`；最终 SQL 是运行时根据参数生成的 `BoundSql`。

## 和其它概念的关系

- [[10-domains/java/mybatis/概念_MyBatis核心生命周期与Configuration]] — `XMLConfigBuilder#mappersElement()` 会启动 `XMLMapperBuilder`。
- [[10-domains/java/mybatis/概念_MyBatis_Mapper代理与MappedStatement]] — Mapper 方法最终按 statement id 查找这里构建出的 `MappedStatement`。
- [[10-domains/java/mybatis/概念_MyBatis执行器缓存与插件链]] — Executor 执行时依赖 `MappedStatement` 中的 `SqlSource`、缓存、resultMaps 和 keyGenerator。
- [[10-domains/java/mybatis/概念_MyBatis_ResultSetHandler结果映射机制]] — 这里编译出的 `ResultMap/ResultMapping` 是结果映射阶段的输入。
- [[10-domains/java/mybatis/主题_MyBatis源码架构_综述]] — 本页补齐综述中的第二轮深挖主题。

## 我的理解

这条链路是 MyBatis “SQL 透明化”成立的基础。XML 保留了开发者可读、可控的 SQL；builder 链则把它编译成统一的运行时对象。`XMLMapperBuilder` 关注文件级结构，`XMLStatementBuilder` 关注语句级属性，`MapperBuilderAssistant` 负责命名空间和引用一致性，`MappedStatement.Builder` 负责最终不可变执行元数据。分层很薄，但边界清楚。

## 来源

- [[30-sources/repositories/来源_MyBatis_3_源码]] — MyBatis 3 源码总入口。
- `raw/code/mybatis-3/src/main/java/org/apache/ibatis/builder/xml/XMLMapperBuilder.java`
- `raw/code/mybatis-3/src/main/java/org/apache/ibatis/builder/xml/XMLStatementBuilder.java`
- `raw/code/mybatis-3/src/main/java/org/apache/ibatis/builder/MapperBuilderAssistant.java`
- `raw/code/mybatis-3/src/main/java/org/apache/ibatis/mapping/MappedStatement.java`
