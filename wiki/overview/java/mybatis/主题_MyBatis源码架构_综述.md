---
type: "overview"
tags: ["java", "mybatis", "source-code", "architecture"]
summary: "MyBatis 源码架构综述：以 SQL 透明化为设计边界，围绕 Configuration、XML mapper 编译、MappedStatement、MapperProxy、Executor、ResultSetHandler、缓存和插件构成从配置解析到对象映射的完整链路。"
sources:
  - "wiki/sources/java/mybatis/来源_MyBatis_3_源码.md"
aliases: ["MyBatis 源码架构", "MyBatis 知识体系"]
status: "evolving"
confidence: 0.89
created: "2026-05-12 21:37:31"
updated: "2026-05-13 09:19:36"
---

# 主题：MyBatis源码架构 综述

![[Attachments/mybatis-source-architecture.excalidraw]]
![[Attachments/mybatis-execution-flow.excalidraw]]

## 当前结论

MyBatis 的源码主线可以概括为：

```text
配置编译 -> Mapper 绑定 -> SQL 执行 -> 结果映射 -> 缓存/插件扩展
```

它的核心竞争力不是替开发者生成所有 SQL，而是把手写 SQL 变成可复用、可映射、可缓存、可拦截、可事务化的运行时对象。

## 总体框架

### 0. 初识 MyBatis

MyBatis 与 JPA/Hibernate 的关键区别是边界选择：

| 框架 | 控制重心 | 适合场景 | 代价 |
| --- | --- | --- | --- |
| MyBatis | 开发者控制 SQL，框架管理映射和执行 | 复杂 SQL、报表、遗留库、性能敏感系统 | 需要维护 SQL 和 mapper |
| JPA | 对象模型和实体状态管理 | 标准 CRUD、领域模型较稳定 | 复杂 SQL 可控性较弱 |
| Hibernate | JPA 实现 + ORM 运行时能力 | 对象关系建模、缓存、脏检查 | 学习和调优成本高 |

MyBatis 的设计理念是 SQL 透明化：框架不把 SQL 藏起来，而是让 SQL 成为一等配置/代码资产。

### 1. 核心 API 与生命周期

- `Resources`：资源加载工具。
- `SqlSessionFactoryBuilder`：构建期对象，用完即丢。
- `SqlSessionFactory`：应用级工厂。
- `SqlSession`：请求/方法级会话，非线程安全。
- `Configuration`：全局运行时注册表。

详见 [[concepts/java/mybatis/概念_MyBatis核心生命周期与Configuration]] — 解释构建顺序、配置注册表和线程安全边界。

### 2. 配置体系

`mybatis-config.xml` 的关键区域：

- `properties`
- `settings`
- `typeAliases`
- `plugins`
- `objectFactory/objectWrapperFactory/reflectorFactory`
- `environments`
- `databaseIdProvider`
- `typeHandlers`
- `mappers`

其中 mapper 加载有四种常见方式：

```xml
<mapper resource="..."/>
<mapper url="..."/>
<mapper class="..."/>
<package name="..."/>
```

配置解析的本质是把 XML 逐步写入 `Configuration`。

### 3. 映射器核心

Mapper XML 的核心元素：

- `cache/cache-ref`
- `resultMap`
- `sql`
- `insert/update/delete/select`

关键判断：

- 简单映射可用 `resultType`。
- 复杂对象图应使用 `resultMap`。
- 一对一用 `association`。
- 一对多用 `collection`。
- 动态 SQL 用 `if/choose/trim/where/set/foreach`。
- 普通参数绑定用 `#{}`。
- SQL 标识符或片段拼接才考虑 `${}`，且必须白名单校验。

### 3.1 XML SQL 到 MappedStatement 的编译链路

第二轮深挖结论：

```text
XMLMapperBuilder
-> XMLStatementBuilder
-> MapperBuilderAssistant
-> MappedStatement.Builder
-> Configuration.mappedStatements
```

这条链路把 mapper XML 从“可读 SQL 配置”编译成“运行时可执行元数据”：

- `XMLMapperBuilder` 负责 mapper 文件级结构：namespace、cache、resultMap、sql fragment、statement 节点。
- `XMLStatementBuilder` 负责单条语句：`databaseId` 过滤、`include` 展开、`selectKey`、`KeyGenerator`、`SqlSource`、缓存属性和结果属性。
- `MapperBuilderAssistant` 负责 namespace 前缀、跨 namespace 引用、inline resultMap、当前 cache 和 incomplete 依赖保护。
- `MappedStatement.Builder` 最终封装 statement id、`SqlSource`、SQL 类型、参数映射、结果映射、缓存、主键生成、语言驱动等执行元数据。

详见 [[concepts/java/mybatis/概念_MyBatis_XML语句解析与MappedStatement构建]] — 解释一条 XML SQL 如何被编译为 `MappedStatement`。![[Attachments/mybatis-mappedstatement-build-flow.excalidraw]]

### 4. 执行流程源码

四大组件协作：

1. `Executor`：事务、缓存、执行策略。
2. `StatementHandler`：创建和操作 JDBC Statement。
3. `ParameterHandler`：把参数对象绑定到 PreparedStatement。
4. `ResultSetHandler`：把 ResultSet 映射成 Java 对象。

详见 [[concepts/java/mybatis/概念_MyBatis执行器缓存与插件链]] — 覆盖 `SimpleExecutor/ReuseExecutor/BatchExecutor/CachingExecutor`、一级/二级缓存和插件链。

### 4.1 ResultSetHandler 结果映射链路

第三轮深挖结论：

```text
ResultSetWrapper
-> DefaultResultSetHandler.handleResultSets()
-> handleRowValues(simple/nested)
-> createResultObject()
-> applyAutomaticMappings()
-> applyPropertyMappings()
-> applyNestedResultMappings()
-> linkObjects()
```

这条链路把 JDBC 行集转换成对象或对象图：

- 自动映射只处理未显式映射的列，并受 `autoMappingBehavior`、`mapUnderscoreToCamelCase`、`callSettersOnNulls` 影响。
- 显式映射处理普通列、nested query、nested cursor、multiple resultSets；nested resultMap 由专门的嵌套映射阶段处理。
- 对象创建优先级是简单类型 type handler、显式 constructor、默认构造器、构造器自动映射。
- join 嵌套映射依赖 `CacheKey` 和 `<id>` 行去重，将多行合并为 association/collection 对象图。
- nested query 延迟加载依赖 `ResultLoaderMap` 和 `ProxyFactory`，getter 触发真正查询。

详见 [[concepts/java/mybatis/概念_MyBatis_ResultSetHandler结果映射机制]] — 解释自动映射、嵌套映射、延迟加载、对象创建策略。![[Attachments/mybatis-resultsethandler-mapping-flow.excalidraw]]

### 5. Mapper 动态代理

Mapper 接口不需要实现类，是因为：

```text
MapperRegistry
-> MapperProxyFactory
-> JDK Proxy
-> MapperProxy.invoke()
-> MapperMethod.execute()
-> SqlSession
```

详见 [[concepts/java/mybatis/概念_MyBatis_Mapper代理与MappedStatement]] — 解释 statement id、返回值适配和 `MappedStatement` 构建结果。

### 6. 缓存体系

- 一级缓存：`SqlSession` 级别，默认开启，生命周期绑定 executor/session。
- 二级缓存：mapper namespace 级别，需要 mapper cache 配置，并受事务提交控制。
- 缓存 key：由 statement id、分页、SQL、参数值、environment id 等构成。

缓存不是越多越好。频繁更新、跨 namespace 关联强、事务一致性要求高的场景，要谨慎使用二级缓存。

### 7. Spring 整合边界

本次 raw 没有 `mybatis-spring` 源码，因此只保留架构边界：

- `SqlSessionFactoryBean`：在 Spring 中构建 `SqlSessionFactory`。
- `MapperScannerConfigurer/@MapperScan`：扫描接口并注册 Mapper bean。
- `MapperFactoryBean`：通过 FactoryBean 暴露 Mapper 代理。
- `SqlSessionTemplate`：线程安全的 `SqlSession` 门面。
- `SpringManagedTransactionFactory`：把事务交给 Spring 管理。

相关已有页面：[[concepts/java/spring/概念_Spring数据访问_JdbcTemplate_事务传播_MyBatis_多数据源]] — Spring 侧的整合机制入口。

## 核心概念

- [[concepts/java/mybatis/概念_MyBatis核心生命周期与Configuration]] — 从资源解析到 `Configuration` 注册表。
- [[concepts/java/mybatis/概念_MyBatis_XML语句解析与MappedStatement构建]] — 从 mapper XML statement 到 `MappedStatement`。
- [[concepts/java/mybatis/概念_MyBatis_Mapper代理与MappedStatement]] — 从接口方法到 SQL 元数据。
- [[concepts/java/mybatis/概念_MyBatis执行器缓存与插件链]] — 从执行器到 JDBC、缓存和插件。
- [[concepts/java/mybatis/概念_MyBatis_ResultSetHandler结果映射机制]] — 从 JDBC `ResultSet` 到对象、集合、嵌套对象图和 lazy proxy。

## 关键来源

- [[sources/java/mybatis/来源_MyBatis_3_源码]] — MyBatis 3 本地源码编译页。
- `raw/code/mybatis-3/src/site/markdown/getting-started.md`
- `raw/code/mybatis-3/src/site/markdown/configuration.md`
- `raw/code/mybatis-3/src/site/markdown/sqlmap-xml.md`
- `raw/code/mybatis-3/src/site/markdown/dynamic-sql.md`

## 重要关系

```text
Configuration
  owns MapperRegistry
  owns MappedStatement registry
  owns TypeHandler/TypeAlias/LanguageDriver
  owns InterceptorChain
  creates Executor and handlers

MapperProxy
  depends on SqlSession
  resolves MapperMethod
  delegates to SqlSession

Executor
  depends on Transaction
  uses MappedStatement and BoundSql
  manages cache
  delegates JDBC details to handlers

DefaultResultSetHandler
  uses ResultMap and ResultMapping
  creates result object via ObjectFactory
  applies automatic and explicit mappings
  merges nested result objects via CacheKey
  creates lazy proxy for nested query
```

## 当前判断

从源码学习顺序看，优先级应是：

1. 生命周期与 `Configuration`
2. XML mapper 编译与 `MappedStatement`
3. Mapper 代理与 `MappedStatement`
4. Executor 和四大组件
5. 参数绑定、动态 SQL、结果映射
6. 缓存
7. 插件
8. Spring/MyBatis-Plus/PageHelper 等生态整合

这条路线比按包目录阅读更稳定，因为它贴近一次 SQL 调用的真实运行路径。

## 未解决问题

- 需要引入 `mybatis-spring` raw 后再更新 Spring 整合章节。
- 需要引入 MyBatis-Plus/PageHelper 源码后再整理 Wrapper、AutoGenerator、分页插件。

## 下一步

- 已完成：第二轮已深挖 `XMLMapperBuilder -> XMLStatementBuilder -> MapperBuilderAssistant -> MappedStatement.Builder`，见 [[concepts/java/mybatis/概念_MyBatis_XML语句解析与MappedStatement构建]]。
- 已完成：第三轮已深挖 `DefaultResultSetHandler` 的自动映射、嵌套映射、延迟加载和对象创建策略，见 [[concepts/java/mybatis/概念_MyBatis_ResultSetHandler结果映射机制]]。
- 第四轮建议：ingest `mybatis-spring`，补齐 `SqlSessionTemplate` 与 Spring 事务同步。
