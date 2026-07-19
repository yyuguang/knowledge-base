---
type: "source"
tags: ["java", "mybatis", "orm", "sql-mapper"]
summary: "MyBatis 3 源码入口：以 SQL 透明化为核心，把配置解析、Mapper 代理、MappedStatement、Executor 执行链、缓存和插件串成一个轻量 SQL Mapper 框架。"
sources:
  - "raw/code/mybatis-3"
  - "https://github.com/mybatis/mybatis-3/releases"
  - "https://mybatis.org/mybatis-3/summary.html"
aliases: ["MyBatis 3 源码", "mybatis-3"]
status: "evolving"
confidence: 0.89
created: "2026-05-12 21:37:31"
updated: "2026-05-13 09:19:36"
---

# 来源：MyBatis 3 源码

![[Attachments/mybatis-source-architecture.excalidraw]]
![[Attachments/mybatis-execution-flow.excalidraw]]

## 一句话总结

MyBatis 不是全自动 ORM，而是以 `MappedStatement` 为中心的 SQL Mapper：开发者保留 SQL 控制权，框架负责把配置、参数、JDBC 执行、结果映射、缓存和插件扩展编排起来。

## 来源信息

- 本地源码：`raw/code/mybatis-3`
- 本地源码版本：`pom.xml` 显示 `3.6.0-SNAPSHOT`，部分代码和文档包含 3.6.0 预备内容。
- 稳定发布边界：官方发布页和项目 summary 显示 `3.5.19` 是当前稳定发布，Java Version 为 8；发布说明中 `3.5.18/3.5.19` 标注这是预期最后一个 Java 8 支持线。
- 主要源码入口：
  - `org.apache.ibatis.session.SqlSessionFactoryBuilder`
  - `org.apache.ibatis.session.Configuration`
  - `org.apache.ibatis.session.defaults.DefaultSqlSession`
  - `org.apache.ibatis.builder.xml.XMLConfigBuilder`
  - `org.apache.ibatis.builder.xml.XMLMapperBuilder`
  - `org.apache.ibatis.builder.xml.XMLStatementBuilder`
  - `org.apache.ibatis.builder.MapperBuilderAssistant`
  - `org.apache.ibatis.binding.MapperProxy`
  - `org.apache.ibatis.mapping.MappedStatement`
  - `org.apache.ibatis.executor.*`
  - `org.apache.ibatis.executor.resultset.DefaultResultSetHandler`
  - `org.apache.ibatis.executor.loader.ResultLoaderMap`
  - `org.apache.ibatis.plugin.*`

## 核心观点

1. **MyBatis 的核心不是“对象持久化”，而是 SQL 调用编译器。** XML/注解中的 SQL 会被解析成 `MappedStatement`，调用时再用参数生成 `BoundSql`，交给执行器链处理。
2. **`Configuration` 是运行时世界的注册表。** 它保存 mapped statements、result maps、caches、type handlers、type aliases、language drivers、mapper registry、interceptor chain 等全局配置。
3. **`SqlSession` 是短生命周期门面，不是线程安全组件。** `DefaultSqlSession` 持有 `Executor`、dirty 状态、cursor 列表和一级缓存相关执行器状态，每个线程/请求应独占。
4. **Mapper 接口没有实现类，是因为 JDK 动态代理把方法调用转成 `SqlSession` 调用。** `MapperProxy` 缓存方法调用器，普通方法走 `MapperMethod.execute()`，default 方法走 `MethodHandle`。
5. **四大执行组件是 MyBatis 的 JDBC 编排骨架。** `Executor` 管事务、缓存和执行策略；`StatementHandler` 管 JDBC Statement；`ParameterHandler` 管参数绑定；`ResultSetHandler` 管结果映射。
6. **缓存分两层。** 一级缓存属于 `SqlSession/Executor`，以 `CacheKey` 定位；二级缓存属于 mapper namespace，经 `CachingExecutor + TransactionalCacheManager` 延迟到 commit 生效。
7. **插件只拦截四类核心接口。** `Executor`、`StatementHandler`、`ParameterHandler`、`ResultSetHandler` 创建后都会经过 `InterceptorChain.pluginAll()` 包装。

## 关键论证

### 生命周期链路

`Resources` 加载配置资源，`SqlSessionFactoryBuilder` 使用 `XMLConfigBuilder` 解析 XML 得到 `Configuration`，再创建 `DefaultSqlSessionFactory`。之后每次 `openSession()` 创建事务、执行器和 `DefaultSqlSession`。

这个链路解释了为什么：

- `SqlSessionFactoryBuilder` 可用完即丢；
- `SqlSessionFactory` 应是应用级单例；
- `SqlSession` 必须方法/请求级关闭；
- Mapper 实例不能脱离创建它的 `SqlSession` 生命周期。

### 配置解析链路

`XMLConfigBuilder#parseConfiguration()` 的解析顺序很关键：

1. `properties`
2. `settings`
3. `typeAliases`
4. `plugins`
5. object/reflector factories
6. `environments`
7. `databaseIdProvider`
8. `typeHandlers`
9. `mappers`

这个顺序意味着 mapper 解析发生在全局策略、事务环境、类型处理器之后。否则 SQL 语句无法正确绑定语言驱动、类型处理器、缓存和环境。

### XML SQL 编译链路

一条 mapper XML SQL 变成 `MappedStatement` 的关键链路是：

```text
XMLMapperBuilder
-> XMLStatementBuilder
-> MapperBuilderAssistant
-> MappedStatement.Builder
-> Configuration.addMappedStatement()
```

`XMLMapperBuilder` 负责 mapper 文件级解析：校验 `namespace`，解析 `cache-ref/cache`、`parameterMap`、`resultMap`、`sql` 片段，再把 `select|insert|update|delete` 交给 `XMLStatementBuilder`。

`XMLStatementBuilder` 负责单条 statement：过滤 `databaseId`，展开 `<include>`，解析 `<selectKey>`，选择 `KeyGenerator`，创建 `SqlSource`，收集 statement 属性。

`MapperBuilderAssistant` 负责统一命名空间、解析 resultMap/parameterMap 引用、挂接当前 namespace cache，并调用 `MappedStatement.Builder` 生成最终执行元数据。

详见：[[10-domains/java/mybatis/概念_MyBatis_XML语句解析与MappedStatement构建]] — 第二轮深挖：一条 XML SQL 如何被编译成 `MappedStatement`。

### Mapper 代理链路

`MapperRegistry` 只接受接口，注册 `MapperProxyFactory`，并触发注解解析。运行时 `DefaultSqlSession#getMapper()` 委托 `Configuration#getMapper()`，最终由 `MapperProxyFactory` 创建 JDK 动态代理。

调用 `userMapper.selectById(id)` 时：

1. `MapperProxy#invoke()` 拦截方法。
2. 非 `Object` 方法进入 `cachedInvoker()`。
3. 普通接口方法构建或复用 `MapperMethod`。
4. `MapperMethod.SqlCommand` 通过 `mapperInterface.getName() + "." + methodName` 找 `MappedStatement`。
5. `MethodSignature` 解析返回值、`RowBounds`、`ResultHandler`、`@MapKey` 和参数名。
6. 根据 SQL 类型调用 `SqlSession.selectOne/selectList/insert/update/delete`。

### 执行器链路

`Configuration#newExecutor()` 根据 `ExecutorType` 创建：

- `SimpleExecutor`：每次执行创建并关闭 Statement。
- `ReuseExecutor`：按 SQL 复用 Statement。
- `BatchExecutor`：聚合同 SQL/同 MappedStatement 的批处理，查询前会 flush。
- `CachingExecutor`：在全局 `cacheEnabled=true` 时装饰基础执行器，处理二级缓存。

一次查询的核心链路是：

`SqlSession -> Executor.query -> MappedStatement.getBoundSql -> CacheKey -> localCache/二级缓存 -> StatementHandler.prepare -> ParameterHandler.setParameters -> StatementHandler.query -> ResultSetHandler.handleResultSets`

### 结果映射链路

`DefaultResultSetHandler` 负责把 JDBC `ResultSet` 映射成 Java 对象、集合或对象图。它会先根据 `ResultMap` 判断走简单映射还是嵌套映射，再组合：

- `ResultSetWrapper`：缓存列名、JDBC 类型和 type handler。
- `createResultObject()`：处理简单值、默认构造器、显式 constructor、构造器自动映射。
- `applyAutomaticMappings()`：把未显式映射的列按属性名自动 set。
- `applyPropertyMappings()`：处理显式 result/association/collection/nested query。
- `applyNestedResultMappings()`：用 `CacheKey` 合并 join 结果中的父子对象。
- `ResultLoaderMap + ProxyFactory`：支持 nested query 延迟加载。

详见：[[10-domains/java/mybatis/概念_MyBatis_ResultSetHandler结果映射机制]] — 第三轮深挖：自动映射、嵌套映射、延迟加载和对象创建策略。

### 缓存链路

一级缓存位于 `BaseExecutor.localCache`，底层是 `PerpetualCache("LocalCache")`。`CacheKey` 由 statement id、分页 offset/limit、最终 SQL、参数值、environment id 组成，因此同一 `SqlSession` 内相同语义查询可以命中。

二级缓存挂在 `MappedStatement.cache`，通常由 mapper namespace 的 `<cache/>` 或注解启用。`CachingExecutor` 不直接写入最终缓存，而是经 `TransactionalCacheManager` 暂存，只有 commit 时才提交，rollback 时丢弃。

### 插件链路

插件不是任意 AOP，它只包裹 MyBatis 明确暴露的四类目标。`Plugin.wrap()` 会根据 `@Intercepts/@Signature` 查找目标对象实现的接口；只有匹配方法会进入 `interceptor.intercept()`，否则原样反射调用目标方法。

## 重要概念

- [[10-domains/java/mybatis/概念_MyBatis核心生命周期与Configuration]] — 理解配置如何变成运行时注册表。
- [[10-domains/java/mybatis/概念_MyBatis_Mapper代理与MappedStatement]] — 理解接口方法如何映射到 SQL 语句。
- [[10-domains/java/mybatis/概念_MyBatis_XML语句解析与MappedStatement构建]] — 理解 mapper XML statement 如何编译为 `MappedStatement`。
- [[10-domains/java/mybatis/概念_MyBatis执行器缓存与插件链]] — 理解 SQL 执行、缓存、插件扩展的源码主线。
- [[10-domains/java/mybatis/概念_MyBatis_ResultSetHandler结果映射机制]] — 理解 `ResultSet` 如何映射成对象图、集合和延迟加载代理。
- [[10-domains/java/mybatis/主题_MyBatis源码架构_综述]] — 汇总架构主线、学习路径和后续待补主题。

## 可复用表达

> MyBatis 的“半自动”并不是能力不足，而是边界选择：SQL 的表达权交给开发者，JDBC 的重复工作交给框架。

> `MappedStatement` 是 MyBatis 的最小执行单元；Mapper 方法只是到 `MappedStatement` 的类型安全入口。

> 一级缓存解决同一会话内重复查询和嵌套查询循环引用；二级缓存解决 namespace 级别复用，但必须服从事务提交边界。

## 我的理解

MyBatis 的设计像一个小型 SQL 编译与执行运行时：配置解析阶段把 XML/注解“编译”为 `Configuration` 与 `MappedStatement`；调用阶段把 Java 方法“链接”到 statement id；执行阶段再把参数对象、动态 SQL、JDBC statement、结果集映射和缓存策略组合起来。

学习 MyBatis 源码时，不应按包名孤立阅读，而应抓住三条线：

1. **构建线**：XML/注解 → `Configuration` → `MappedStatement`
2. **调用线**：Mapper 接口 → `MapperProxy` → `MapperMethod` → `SqlSession`
3. **执行线**：`Executor` → `StatementHandler` → `ParameterHandler` → `ResultSetHandler`

## 未解决问题

- 本次 raw 只有 `mybatis-3`，没有 `mybatis-spring` 与 `mybatis-plus` 源码；Spring 整合、`SqlSessionTemplate`、`MapperFactoryBean`、MyBatis-Plus Wrapper 只能作为后续 ingest 主题。
- 当前源码为 `3.6.0-SNAPSHOT`，需要在讲解版本特性时区分 snapshot 与 `3.5.19` 稳定版。

## 相关链接

- [[10-domains/java/mybatis/主题_MyBatis源码架构_综述]] — 以源码组件组织的整体学习地图。
- [[10-domains/java/spring/概念_Spring数据访问_JdbcTemplate_事务传播_MyBatis_多数据源]] — Spring 数据访问中 MyBatis 整合的上下文入口。
- [[10-domains/java/spring-framework/概念_Spring_Framework_数据访问事务同步与集成模块]] — 理解 MyBatis-Spring 如何接入 Spring 事务同步。
