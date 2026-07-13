---
type: "concept"
tags: ["java", "mybatis", "configuration", "lifecycle"]
summary: "MyBatis 生命周期主线：SqlSessionFactoryBuilder 解析 XML/Configuration，SqlSessionFactory 作为应用级工厂，SqlSession 作为非线程安全的短生命周期执行门面，Configuration 则保存运行时注册表和策略。"
sources:
  - "raw/code/mybatis-3/src/main/java/org/apache/ibatis/session/SqlSessionFactoryBuilder.java"
  - "raw/code/mybatis-3/src/main/java/org/apache/ibatis/builder/xml/XMLConfigBuilder.java"
  - "raw/code/mybatis-3/src/main/java/org/apache/ibatis/session/Configuration.java"
  - "raw/code/mybatis-3/src/main/java/org/apache/ibatis/session/defaults/DefaultSqlSession.java"
aliases: ["SqlSession 生命周期", "Configuration 注册表", "MyBatis 生命周期"]
status: "evolving"
confidence: 0.9
created: "2026-05-12 21:37:31"
updated: "2026-05-12 21:37:31"
---

# 概念：MyBatis核心生命周期与Configuration

![[Attachments/mybatis-source-architecture.excalidraw]]

## 定义

MyBatis 的核心生命周期是：

`Resources/InputStream -> SqlSessionFactoryBuilder -> XMLConfigBuilder -> Configuration -> DefaultSqlSessionFactory -> DefaultSqlSession -> Executor`

其中 `Configuration` 是最重要的运行时对象。它不是普通配置 DTO，而是保存 MyBatis 全局注册表、执行策略、插件链、Mapper 注册器和 SQL 元数据的中心仓库。

## 为什么重要

很多 MyBatis 问题本质上是生命周期错误：

- 把 `SqlSession` 放到单例对象里，导致线程安全与连接泄漏问题。
- 重复构建 `SqlSessionFactory`，导致配置解析和资源初始化成本过高。
- 在 mapper 加载前没有正确注册 type handler、plugin、environment，导致 SQL 执行期才暴露错误。
- 混淆 `Configuration` 全局状态和 `SqlSession` 会话状态，误判缓存边界。

## 使用场景

- 手写 MyBatis 启动代码。
- 排查 mapper XML 加载失败、statement not found、type handler 找不到。
- 解释 `SqlSession` 为什么不线程安全。
- 设计 MyBatis 插件或自定义语言驱动时判断注册点。
- 阅读 MyBatis-Spring 时理解 `SqlSessionFactoryBean` 的职责。

## 核心机制

### 1. Builder 用完即丢

`SqlSessionFactoryBuilder` 的职责很窄：接收 `Reader/InputStream/Configuration`，构造 `XMLConfigBuilder`，调用 `parse()` 得到 `Configuration`，再创建 `DefaultSqlSessionFactory`。

它在 finally 中关闭输入流并 reset `ErrorContext`，说明它是构建期工具，不应该成为运行时依赖。

### 2. Configuration 是注册表

`Configuration` 内部维护多类核心结构：

| 注册项 | 作用 |
| --- | --- |
| `mappedStatements` | statement id 到 `MappedStatement` 的映射 |
| `resultMaps` | 结果映射定义 |
| `caches` | namespace 缓存 |
| `typeHandlerRegistry` | Java/JDBC 类型转换策略 |
| `typeAliasRegistry` | 别名解析 |
| `languageRegistry` | XML/RAW 等脚本语言驱动 |
| `mapperRegistry` | Mapper 接口到代理工厂 |
| `interceptorChain` | 插件链 |
| `environment` | DataSource + TransactionFactory |

默认构造器还会注册大量内置别名，例如 `JDBC`、`MANAGED`、`POOLED`、`UNPOOLED`、`PERPETUAL`、`LRU`、`XML`、`RAW`、`SLF4J` 等。

### 3. XMLConfigBuilder 的解析顺序体现依赖关系

`parseConfiguration()` 先解析属性和全局设置，再解析别名、插件、对象工厂、环境、database id、类型处理器，最后才解析 mappers。

这个顺序不是偶然：

- mapper SQL 需要 type alias/type handler。
- `MappedStatement` 需要语言驱动。
- mapper cache 需要 cache alias。
- 数据库厂商差异需要 database id。
- 插件需要在运行时组件创建时统一包装。

### 4. SqlSession 是短生命周期门面

`DefaultSqlSession` 持有：

- `Configuration`
- `Executor`
- `autoCommit`
- `dirty`
- `cursorList`

源码注释直接说明它不是线程安全类。`dirty`、cursor 列表、底层 executor 的一级缓存和事务连接都绑定到当前会话，因此不能跨线程共享。

### 5. SqlSessionFactory 是应用级工厂

`SqlSessionFactory` 基于稳定的 `Configuration` 创建多个 `SqlSession`。一次应用通常只需要一个工厂；多环境或多数据源场景则通常对应多个工厂或在 Spring 层做路由。

## 例子

典型手写启动代码：

```java
String resource = "mybatis-config.xml";
try (InputStream in = Resources.getResourceAsStream(resource)) {
  SqlSessionFactory factory = new SqlSessionFactoryBuilder().build(in);
  try (SqlSession session = factory.openSession()) {
    UserMapper mapper = session.getMapper(UserMapper.class);
    mapper.selectById(1L);
  }
}
```

这段代码背后发生了三次转换：

1. XML 资源被解析成 `Configuration`。
2. `Configuration` 被封装进 `DefaultSqlSessionFactory`。
3. `openSession()` 创建执行器和会话，Mapper 调用才真正触发 SQL 执行。

## 常见误区

- **误区：`SqlSessionFactoryBuilder` 应该单例。** 实际上它只是构建工具，用完即可丢。
- **误区：`SqlSession` 可以作为 DAO 字段复用。** 它不线程安全，且绑定事务、一级缓存、cursor 和 dirty 状态。
- **误区：`Configuration` 只是配置值集合。** 它实际是运行时注册表，负责创建 executor、statement handler、parameter handler、result set handler，并套用插件。
- **误区：多个 environment 会自动切换。** 一个 `SqlSessionFactory` 构建时只会选定一个 environment。

## 和其它概念的关系

- [[concepts/java/mybatis/概念_MyBatis_Mapper代理与MappedStatement]] — Mapper 方法最终要从 `Configuration.mappedStatements` 找到对应 SQL 元数据。
- [[concepts/java/mybatis/概念_MyBatis执行器缓存与插件链]] — `Configuration#newExecutor()` 决定执行器类型、二级缓存装饰和插件包装。
- [[concepts/java/spring/概念_Spring数据访问_JdbcTemplate_事务传播_MyBatis_多数据源]] — Spring 环境中 `SqlSessionFactoryBean` 负责构造这里描述的 `SqlSessionFactory`。

## 我的理解

`Configuration` 是 MyBatis 的“编译产物容器”。XML/注解解析完成后，运行时不再频繁读配置文件，而是围绕 `Configuration` 中的注册表执行。理解这一点后，很多 MyBatis 行为会变得统一：Mapper 为什么能代理、插件为什么能织入、缓存为什么按 statement/namespace 生效，都因为它们最终都被挂到了 `Configuration` 这张运行时地图上。

## 来源

- [[sources/java/mybatis/来源_MyBatis_3_源码]] — 本概念的源码入口和整体架构。
- `raw/code/mybatis-3/src/site/markdown/getting-started.md`
- `raw/code/mybatis-3/src/main/java/org/apache/ibatis/session/Configuration.java`
