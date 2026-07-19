---
type: "concept"
tags:
  - java
  - spring
  - jdbc
  - transaction
  - mybatis
  - datasource
summary: "Spring 数据访问：JdbcTemplate 用模板方法封装 JDBC 样板代码，声明式事务由传播行为决定加入/新建/挂起/嵌套事务，MyBatis 整合核心是 Mapper 接口扫描和 MapperFactoryBean，多数据源与读写分离通常基于 AbstractRoutingDataSource。"
sources:
  - "[[30-sources/repositories/来源_Spring_Framework_源码]]"
  - "raw/code/spring-framework/spring-jdbc/src/main/java/org/springframework/jdbc/core/JdbcTemplate.java"
  - "raw/code/spring-framework/spring-tx/src/main/java/org/springframework/transaction/support/AbstractPlatformTransactionManager.java"
  - "raw/code/spring-framework/spring-jdbc/src/main/java/org/springframework/jdbc/datasource/lookup/AbstractRoutingDataSource.java"
aliases:
  - "Spring 数据访问大纲"
  - "JdbcTemplate 事务传播 MyBatis 多数据源"
status: "draft"
confidence: 0.76
created: "2026-05-12 21:22:48"
updated: "2026-05-12 21:22:48"
---

# 概念：Spring数据访问 JdbcTemplate 事务传播 MyBatis 多数据源

## 31_JdbcTemplate与数据源配置

`JdbcTemplate` 是模板方法模式：

```text
JdbcTemplate.query/update/execute
  → DataSourceUtils.getConnection()
  → 创建 Statement/PreparedStatement
  → 执行 SQL
  → RowMapper / ResultSetExtractor 处理结果
  → SQLExceptionTranslator 异常转换
  → 释放 Statement/Connection
```

它解决的是 JDBC 样板代码问题：

- 获取和释放连接。
- 关闭 Statement/ResultSet。
- 处理 SQLException。
- 参与 Spring 事务同步。

数据源配置通常把 `DataSource`、`JdbcTemplate`、`PlatformTransactionManager` 注册为 Bean。Spring Boot 场景下通常由 `DataSourceAutoConfiguration` 自动配置，本仓库暂缺 Spring Boot 源码。

## 32_声明式事务传播行为

声明式事务核心在 `TransactionInterceptor` 和 `AbstractPlatformTransactionManager`。

常见传播行为：

| 传播行为 | 有事务 | 无事务 |
|---|---|---|
| REQUIRED | 加入当前事务 | 新建事务 |
| REQUIRES_NEW | 挂起当前事务，新建事务 | 新建事务 |
| SUPPORTS | 加入当前事务 | 非事务执行 |
| NOT_SUPPORTED | 挂起当前事务，非事务执行 | 非事务执行 |
| MANDATORY | 加入当前事务 | 抛异常 |
| NEVER | 抛异常 | 非事务执行 |
| NESTED | 保存点嵌套事务 | 新建事务 |

源码主线：

```text
TransactionInterceptor
  → invokeWithinTransaction()
  → PlatformTransactionManager.getTransaction()
  → existing transaction?
      → handleExistingTransaction()
      → startTransaction()
  → commit / rollback
```

## 33_MyBatis整合原理（MapperScan）

本仓库当前没有 `mybatis-spring` 源码，因此这里先记录机制级理解，后续需要补 `raw/code/mybatis-spring` 做源码验证。

MyBatis 与 Spring 整合的典型机制：

1. `@MapperScan` 扫描 Mapper 接口。
2. 为每个 Mapper 接口注册 BeanDefinition。
3. BeanDefinition 的 beanClass 通常指向 `MapperFactoryBean`。
4. `MapperFactoryBean#getObject()` 从 `SqlSession` 获取 Mapper 代理。
5. `SqlSessionTemplate` 负责线程安全和事务同步。
6. Spring 事务通过 DataSource/SqlSession 资源绑定生效。

面试表达：

> Mapper 接口本身没有实现类，MyBatis-Spring 通过扫描接口并注册 FactoryBean，由 FactoryBean 暴露 MyBatis 生成的 Mapper 代理对象。这个过程和 Spring 的 FactoryBean、BeanDefinitionRegistryPostProcessor、事务同步机制密切相关。

## 34_多数据源与读写分离

Spring Framework 提供 `AbstractRoutingDataSource`，它根据 lookup key 路由到目标 DataSource。

典型结构：

```text
业务方法
  → AOP/注解/上下文 设置当前数据源 key
  → AbstractRoutingDataSource.determineCurrentLookupKey()
  → 选择 writeDataSource / readDataSource
  → JdbcTemplate/MyBatis 获取连接
```

关键问题：

- lookup key 通常放在线程上下文中。
- 读写分离要注意事务：一个写事务中继续读，通常应走主库，避免主从延迟。
- 多数据源事务如果跨库，普通本地事务不够，需要分布式事务或业务补偿。
- 数据源路由要发生在获取 Connection 之前，已经拿到连接后再切 key 没意义。

## 面试表达

> Spring 数据访问层的核心不是 ORM，而是模板方法、异常转换、资源绑定和事务同步。JdbcTemplate 封装 JDBC 样板代码；声明式事务通过 AOP 进入 TransactionManager；MyBatis 整合靠 MapperScan + FactoryBean 暴露 Mapper 代理；多数据源通常通过 AbstractRoutingDataSource 按上下文 key 路由。

## 源码待补

- `raw/code/mybatis-spring`
- `raw/code/spring-boot` 中 DataSource 自动配置源码

## 相关链接

- [[10-domains/java/spring-framework/概念_Spring_Framework_数据访问事务同步与集成模块]] — Spring Framework 数据访问源码页。
- [[10-domains/java/spring-framework/概念_Spring_Framework_行为型设计模式]] — JdbcTemplate/TransactionTemplate 模板方法。
- [[10-domains/java/spring/主题_Spring知识体系_综述]] — 返回大纲。

