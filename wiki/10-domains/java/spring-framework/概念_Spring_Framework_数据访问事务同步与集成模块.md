---
type: "concept"
tags:
  - java
  - spring-framework
  - jdbc
  - transaction
  - r2dbc
  - orm
summary: "Spring 数据访问层的核心不是 ORM，而是模板方法、异常转换、资源绑定和事务同步：JdbcTemplate 管理 JDBC 样板代码，TransactionSynchronizationManager 把连接等资源绑定到当前执行上下文，JDBC/R2DBC/ORM/JMS 等模块复用这套资源协调模型。"
sources:
  - "[[30-sources/repositories/来源_Spring_Framework_源码]]"
  - "raw/code/spring-framework/spring-jdbc/src/main/java/org/springframework/jdbc/core/JdbcTemplate.java"
  - "raw/code/spring-framework/spring-jdbc/src/main/java/org/springframework/jdbc/datasource/DataSourceTransactionManager.java"
  - "raw/code/spring-framework/spring-tx/src/main/java/org/springframework/transaction/support/TransactionSynchronizationManager.java"
  - "raw/code/spring-framework/spring-orm/src/main/java/org/springframework/orm/jpa/JpaTransactionManager.java"
aliases:
  - "Spring 数据访问"
  - "Spring 事务同步"
  - "JdbcTemplate 源码"
status: "evolving"
confidence: 0.84
created: "2026-05-12 21:10:00"
updated: "2026-05-12 21:10:00"
---

# 概念：Spring Framework 数据访问事务同步与集成模块

## 定义

Spring 数据访问层由三类能力组成：

1. **模板方法**：`JdbcTemplate`、`JmsTemplate` 等负责打开/关闭资源、执行回调、异常转换。
2. **事务抽象**：`PlatformTransactionManager`、`TransactionDefinition`、`TransactionStatus` 统一事务边界。
3. **资源同步**：`TransactionSynchronizationManager` 把连接、Session、同步回调绑定到当前线程或响应式上下文。

它的目标不是替代 JDBC/JPA/R2DBC/JMS，而是统一资源生命周期、异常模型和事务参与方式。

## 为什么重要

很多人把 Spring 数据访问理解成 “JdbcTemplate 简化 JDBC”。这只看到表层。真正关键是：

- 同一个事务中多次 JDBC 操作为什么能复用同一个 Connection。
- `@Transactional` 如何让 JDBC、JPA、MyBatis 等参与同一事务。
- SQL 异常为什么能转成 Spring 的 `DataAccessException` 层级。
- 事务完成后资源如何提交、回滚、释放。
- Reactive 事务为什么需要另一套上下文绑定方式。

## 使用场景

- 排查连接泄露、事务未生效、只读事务、嵌套事务。
- 理解 `DataSourceTransactionManager`、`JpaTransactionManager`、`R2dbcTransactionManager` 差异。
- 写自定义资源管理器，让它参与 Spring 事务同步。
- 理解 MyBatis-Spring、Spring Data 如何接入 Spring 事务。

## 核心机制

### 1. `JdbcTemplate` 是模板方法，不是 SQL 构造器

`JdbcTemplate` 负责：

- 从 DataSource 获取 Connection。
- 创建 Statement / PreparedStatement。
- 设置 fetchSize、maxRows、queryTimeout。
- 执行 SQL。
- 用 RowMapper / ResultSetExtractor 处理结果。
- 捕获 SQLException，交给 SQLExceptionTranslator。
- 释放 JDBC 资源。

业务代码只提供变化部分：SQL、参数、结果映射。固定流程由模板掌控。

### 2. DataAccessException：统一异常模型

JDBC 的 `SQLException` 是厂商错误码 + SQLState 混合体，不适合业务层直接处理。Spring 把它翻译成非受检异常层级：

- `BadSqlGrammarException`
- `DuplicateKeyException`
- `DataIntegrityViolationException`
- `CannotAcquireLockException`
- `QueryTimeoutException`

这样上层代码不依赖具体数据库驱动异常。

### 3. `TransactionSynchronizationManager` 是资源绑定中心

传统事务同步使用 ThreadLocal：

| ThreadLocal | 含义 |
|---|---|
| `resources` | 当前线程绑定的资源，如 DataSource → ConnectionHolder |
| `synchronizations` | 事务完成前后的回调 |
| `currentTransactionName` | 当前事务名 |
| `currentTransactionReadOnly` | 当前事务只读标记 |
| `currentTransactionIsolationLevel` | 隔离级别 |
| `actualTransactionActive` | 是否存在真实事务 |

JDBC 操作获取连接时，会先看当前线程是否已有 DataSource 绑定资源；如果有，就复用事务连接。

### 4. `DataSourceTransactionManager` 管理 JDBC 本地事务

它的核心职责：

1. 从 DataSource 获取 Connection。
2. 设置 autoCommit、isolation、readOnly。
3. 把 ConnectionHolder 绑定到 `TransactionSynchronizationManager`。
4. 提交或回滚。
5. 恢复连接状态并释放连接。

源码里 `setDataSource()` 会 unwrap `TransactionAwareDataSourceProxy`，说明事务管理器需要操作真实目标 DataSource，而不是给数据访问代码看的代理。

### 5. ORM/R2DBC/JMS 是同一思想的不同资源实现

| 模块 | 管理资源 | 事务管理器/模板 |
|---|---|---|
| `spring-jdbc` | JDBC Connection | `DataSourceTransactionManager`、`JdbcTemplate` |
| `spring-orm` | JPA EntityManager / Hibernate Session | `JpaTransactionManager` |
| `spring-r2dbc` | Reactive Connection | `R2dbcTransactionManager`、`DatabaseClient` |
| `spring-jms` | JMS Connection/Session | `JmsTemplate`、listener container |

Reactive 场景不能依赖普通 ThreadLocal，因此 `spring-tx` 也提供 reactive `TransactionSynchronizationManager`，用 Reactor context 承载事务状态。

## 例子

### 同一事务内复用连接

```text
@Transactional serviceMethod()
  → TransactionInterceptor
  → DataSourceTransactionManager.doBegin()
      → 获取 Connection
      → bindResource(dataSource, connectionHolder)
  → jdbcTemplate.query(...)
      → DataSourceUtils.getConnection(dataSource)
      → 发现线程已绑定 ConnectionHolder
      → 复用同一个 Connection
  → commit / rollback
  → unbindResource(dataSource)
  → release Connection
```

这解释了为什么不要在事务中绕过 Spring 直接手动管理连接，否则可能破坏资源同步。

### `JdbcTemplate` 和事务的关系

`JdbcTemplate` 本身不开始事务。它只是通过 Spring 的 DataSourceUtils 获取连接；如果外层有事务，它自然参与；如果没有事务，它每次操作独立获取和释放连接。

## 常见误区

1. **误区：`JdbcTemplate` 替你管理事务。**  
   它管理 JDBC 样板代码，不负责事务边界；事务边界由 transaction manager 或 `TransactionTemplate` 控制。

2. **误区：事务就是 ThreadLocal。**  
   ThreadLocal 只是传统同步事务的资源上下文载体；事务语义还包括传播行为、隔离级别、提交回滚、同步回调。

3. **误区：`readOnly=true` 一定由数据库强制。**  
   默认通常是 JDBC hint。`DataSourceTransactionManager#setEnforceReadOnly(true)` 才可能发出数据库级只读语句。

4. **误区：JPA 和 JDBC 事务模型完全不同。**  
   资源不同，但都通过 transaction manager 和 synchronization 参与 Spring 事务边界。

## 和其它概念的关系

- [[10-domains/java/spring-framework/概念_Spring_Framework_AOP事务与Web分发]] — 声明式事务通过 AOP 进入 transaction manager，再由本页的资源同步机制落地。
- [[10-domains/java/spring-framework/概念_Spring_Framework_模块体系与扩展点]] — 数据访问模块展示 Spring 如何把外部资源纳入统一生命周期。
- [[10-domains/java/spring-framework/概念_Spring_Framework_配置类解析与依赖解析]] — DataSource、JdbcTemplate、TransactionManager 通常通过配置类或自动配置注册为 Bean。

## 我的理解

Spring 数据访问层的核心设计不是“封装数据库 API”，而是“让资源有上下文”。没有上下文，JDBC 调用只是散落的方法；有了 `TransactionSynchronizationManager`，Connection、Session、同步回调、事务属性就能围绕一次业务调用聚合起来。

## 来源

- `raw/code/spring-framework/spring-jdbc/src/main/java/org/springframework/jdbc/core/JdbcTemplate.java`
- `raw/code/spring-framework/spring-jdbc/src/main/java/org/springframework/jdbc/datasource/DataSourceTransactionManager.java`
- `raw/code/spring-framework/spring-tx/src/main/java/org/springframework/transaction/support/TransactionSynchronizationManager.java`
- `raw/code/spring-framework/spring-orm/src/main/java/org/springframework/orm/jpa/JpaTransactionManager.java`

