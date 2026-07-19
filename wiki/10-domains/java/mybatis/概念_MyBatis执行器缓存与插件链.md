---
type: "concept"
tags: ["java", "mybatis", "executor", "cache", "plugin"]
summary: "MyBatis 执行链由 Executor 统筹事务、缓存和执行策略，再委托 StatementHandler、ParameterHandler、ResultSetHandler 完成 JDBC 操作；一级缓存绑定 SqlSession，二级缓存绑定 namespace，插件通过 InterceptorChain 包裹四类核心接口。"
sources:
  - "[[30-sources/repositories/来源_MyBatis_3_源码]]"
  - "raw/code/mybatis-3/src/main/java/org/apache/ibatis/executor/BaseExecutor.java"
  - "raw/code/mybatis-3/src/main/java/org/apache/ibatis/executor/CachingExecutor.java"
  - "raw/code/mybatis-3/src/main/java/org/apache/ibatis/executor/SimpleExecutor.java"
  - "raw/code/mybatis-3/src/main/java/org/apache/ibatis/executor/ReuseExecutor.java"
  - "raw/code/mybatis-3/src/main/java/org/apache/ibatis/executor/BatchExecutor.java"
  - "raw/code/mybatis-3/src/main/java/org/apache/ibatis/plugin/InterceptorChain.java"
  - "raw/code/mybatis-3/src/main/java/org/apache/ibatis/plugin/Plugin.java"
aliases: ["MyBatis Executor", "MyBatis 一级缓存", "MyBatis 二级缓存", "MyBatis 插件机制"]
status: "evolving"
confidence: 0.9
created: "2026-05-12 21:37:31"
updated: "2026-05-13 09:19:36"
---

# 概念：MyBatis执行器缓存与插件链

![[Attachments/mybatis-execution-flow.excalidraw]]

## 定义

MyBatis 执行器链是 SQL 从 `SqlSession` 进入 JDBC 的核心路径：

```text
Executor
  -> StatementHandler
  -> ParameterHandler
  -> JDBC Statement
  -> ResultSetHandler
```

`Executor` 决定执行策略、事务边界、一级缓存和二级缓存；三类 handler 负责具体 JDBC 细节。插件机制通过动态代理包裹这四类接口，在关键方法前后插入扩展逻辑。

## 为什么重要

理解这条链路可以回答 MyBatis 最常见的源码问题：

- 一级缓存什么时候命中、什么时候清空？
- 二级缓存为什么要等 commit 才真正写入？
- `SIMPLE/REUSE/BATCH` 三种执行器差异是什么？
- PageHelper 等分页插件为什么能改写 SQL？
- 插件为什么只能拦截少数类型，而不是任意类？
- 批处理为什么查询前会 flush？

## 使用场景

- 性能调优：选择 `ExecutorType.REUSE/BATCH`。
- 缓存排错：分析脏读、缓存不生效、update 后缓存失效。
- 插件开发：分页、审计、SQL 日志、数据权限、多租户。
- 结果映射排错：定位参数绑定或结果处理阶段。
- 面试表达：串起四大组件协作流程。

## 核心机制

### 1. Executor 类型

`Configuration#newExecutor()` 根据配置创建基础执行器：

| 执行器 | 特征 | 适合场景 |
| --- | --- | --- |
| `SimpleExecutor` | 每次执行都创建、使用、关闭 Statement | 默认通用场景 |
| `ReuseExecutor` | 按 SQL 文本缓存 Statement | 同一会话重复执行相同 SQL |
| `BatchExecutor` | 合并同 SQL/同 MappedStatement 的更新，统一 flush | 批量 insert/update/delete |
| `CachingExecutor` | 装饰基础执行器，处理二级缓存 | `cacheEnabled=true` 时默认包裹 |

### 2. 一级缓存

一级缓存位于 `BaseExecutor.localCache`，底层是 `PerpetualCache`。

查询流程：

1. `MappedStatement#getBoundSql(parameter)` 生成 `BoundSql`。
2. `createCacheKey()` 生成 `CacheKey`。
3. 如果不是自定义 `ResultHandler`，先查 `localCache`。
4. 未命中则查数据库，并把执行占位和结果放入 local cache。
5. 如果 `localCacheScope=STATEMENT`，顶层查询结束后清空。

`CacheKey` 包含：

- statement id
- `RowBounds` offset/limit
- 最终 SQL
- 参数值
- environment id

update/commit/rollback 会清理一级缓存，避免同一 session 内读到过期数据。

### 3. 二级缓存

二级缓存挂在 `MappedStatement.cache`，通常对应 mapper namespace。

`CachingExecutor` 的查询逻辑：

1. 如果 statement 有 cache，先按 `flushCacheRequired` 清理缓存。
2. 只有 `useCache=true` 且没有自定义 `ResultHandler` 时尝试二级缓存。
3. 不允许缓存带 OUT 参数的存储过程。
4. 未命中则委托基础 executor 查询。
5. 查询结果先放入 `TransactionalCacheManager`。
6. commit 时提交到真实 cache，rollback 时丢弃。

这个事务延迟写入设计，是为了避免事务未提交的数据提前污染 namespace 缓存。

### 4. StatementHandler / ParameterHandler / ResultSetHandler

`SimpleExecutor#doQuery()` 展示了标准 JDBC 执行模板：

```text
configuration.newStatementHandler(...)
-> prepareStatement()
   -> getConnection()
   -> handler.prepare(connection, timeout)
   -> handler.parameterize(stmt)
-> handler.query(stmt, resultHandler)
```

`Configuration` 创建 handler 时都会调用 `interceptorChain.pluginAll()`，因此插件可以包裹：

- `Executor`
- `StatementHandler`
- `ParameterHandler`
- `ResultSetHandler`

### 5. 插件链

`InterceptorChain#pluginAll(target)` 按注册顺序把目标对象层层交给 interceptor 包装。

`Plugin.wrap()` 的核心规则：

- 插件类必须有 `@Intercepts`。
- `@Signature` 指明要拦截的接口、方法名和参数类型。
- 只有目标对象实现了匹配接口，才创建 JDK 动态代理。
- 调用匹配方法时进入 `interceptor.intercept(new Invocation(target, method, args))`。
- 不匹配的方法直接反射调用目标对象。

## 例子

分页插件常拦截 `StatementHandler.prepare(Connection, Integer)`：

```text
Executor.query()
-> Configuration.newStatementHandler()
-> InterceptorChain.pluginAll(statementHandler)
-> Plugin proxy
-> prepare() 被插件拦截
-> 改写 BoundSql.sql 为 limit/offset SQL
-> 原始 StatementHandler.prepare()
```

这解释了为什么分页插件通常要在 SQL 发送给 JDBC 前动手，而不是在结果返回后截断列表。

## 常见误区

- **误区：一级缓存是全局缓存。** 它绑定 `SqlSession/Executor`，session 关闭即失效。
- **误区：二级缓存查询后立即写入真实缓存。** MyBatis 用 `TransactionalCacheManager` 延迟到 commit。
- **误区：插件可以拦截任意类。** 默认插件机制只包裹四类核心接口。
- **误区：BatchExecutor 每次 update 都返回真实影响行数。** 它返回批处理占位值，真实结果在 flush/commit 后取得。
- **误区：`${}` 可以用来绑定普通参数。** `${}` 是 SQL 文本替换，容易注入；普通值应使用 `#{}`。

## 和其它概念的关系

- [[10-domains/java/mybatis/概念_MyBatis核心生命周期与Configuration]] — 执行器和 handler 都由 `Configuration` 创建。
- [[10-domains/java/mybatis/概念_MyBatis_Mapper代理与MappedStatement]] — Mapper 调用最终定位 `MappedStatement` 后进入执行器链。
- [[10-domains/java/mybatis/概念_MyBatis_ResultSetHandler结果映射机制]] — `StatementHandler.query()` 后，`ResultSetHandler` 负责把 JDBC 行集映射为对象图。
- [[10-domains/java/spring-framework/概念_Spring_Framework_数据访问事务同步与集成模块]] — Spring 事务整合会影响 MyBatis 事务和连接获取方式。

## 我的理解

MyBatis 的执行链很克制：`Executor` 只统筹，handler 各司其职，插件只切入少数稳定接口。这种设计让框架既能保持 SQL 透明，又能在分页、数据权限、日志、缓存等横切需求上留出足够扩展空间。它不像全自动 ORM 那样试图接管查询模型，而是把扩展点放在“SQL 即将执行”和“结果即将映射”的关键节点。

## 来源

- [[30-sources/repositories/来源_MyBatis_3_源码]] — 源码入口与整体说明。
- `raw/code/mybatis-3/src/site/markdown/configuration.md`
- `raw/code/mybatis-3/src/site/markdown/sqlmap-xml.md`
