---
type: "concept"
tags:
  - java
  - spring-framework
  - test
summary: "Spring TestContext 通过静态 ContextCache 复用 ApplicationContext，以 MergedContextConfiguration 作为 cache key，并通过 TestExecutionListener 链编排依赖注入、事务测试、SQL 脚本、MockMvc 与 WebTestClient 等测试能力。"
sources:
  - "[[30-sources/repositories/来源_Spring_Framework_源码]]"
  - "raw/code/spring-framework/spring-test/src/main/java/org/springframework/test/context/cache/DefaultCacheAwareContextLoaderDelegate.java"
  - "raw/code/spring-framework/spring-test/src/main/java/org/springframework/test/context/MergedContextConfiguration.java"
  - "raw/code/spring-framework/spring-test/src/main/java/org/springframework/test/context/transaction/TransactionalTestExecutionListener.java"
aliases:
  - "Spring TestContext"
  - "ContextCache"
  - "TransactionalTestExecutionListener"
  - "MockMvc"
  - "WebTestClient"
status: "evolving"
confidence: 0.88
created: "2026-05-13 09:14:32"
updated: "2026-05-13 09:14:32"
---

# 概念：Spring TestContext缓存与事务测试

## 定义

Spring TestContext 是 Spring 测试框架的核心上下文编排层。它负责根据测试类注解加载 `ApplicationContext`，缓存上下文，执行测试生命周期 listener，并支持依赖注入、事务回滚、SQL 脚本、MockMvc、WebTestClient 等测试能力。

它的核心不是某一个测试工具，而是：

```text
TestContextManager
  ↓
TestExecutionListener 链
  ↓
CacheAwareContextLoaderDelegate
  ↓
ContextCache<ApplicationContext>
```

相关主线：[[10-domains/java/spring-framework/概念_Spring_Framework_Web测试消息与应用层扩展]] — TestContext 属于应用层策略模型；[[10-domains/java/spring-framework/概念_Spring_Framework_IoC容器启动与Bean生命周期]] — 测试上下文缓存复用的是完整 ApplicationContext。

## 源码入口

关键类：

- `TestContextManager`：测试生命周期总控。
- `TestContext`：封装当前测试类、方法、实例和 ApplicationContext。
- `TestExecutionListener`：测试生命周期扩展点。
- `DefaultCacheAwareContextLoaderDelegate`：负责从缓存加载或创建上下文。
- `DefaultContextCache`：默认上下文缓存。
- `MergedContextConfiguration`：上下文缓存 key。
- `TransactionalTestExecutionListener`：事务测试 listener。

## ContextCache：为什么测试上下文能复用

`DefaultCacheAwareContextLoaderDelegate` 中有一个 static 默认缓存：

```text
static final ContextCache defaultContextCache = new DefaultContextCache()
```

这意味着同一个 JVM 进程内，只要测试类的 `MergedContextConfiguration` 相等，就可以复用同一个 `ApplicationContext`。这也是 Spring 集成测试第一次慢、后续快的主要原因。

加载流程：

```text
loadContext(mergedConfig)
  ↓
contextCache.get(mergedConfig)
  ↓
命中：返回已有 ApplicationContext
  ↓
未命中：检查 failure threshold
  ↓
loadContextInternal(mergedConfig)
  ↓
contextCache.put(mergedConfig, context)
```

如果使用 `@DirtiesContext`，测试会标记上下文脏掉，后续不再复用该缓存实例。

## Cache Key：MergedContextConfiguration 比较什么

`MergedContextConfiguration.equals/hashCode()` 决定两个测试能否共享上下文。源码中参与比较的字段包括：

| 字段 | 含义 | 影响 |
|---|---|---|
| `locations` | XML/Groovy 配置位置 | 不同则不能复用 |
| `classes` | 配置类集合 | 不同则不能复用 |
| `contextInitializerClasses` | initializer | 不同则不能复用 |
| `activeProfiles` | 激活 profile | profile 顺序/集合变化会影响 key |
| `propertySourceDescriptors` | 测试属性源描述 | 不同则不能复用 |
| `propertySourceProperties` | inline properties | 常见导致缓存碎片 |
| `contextCustomizers` | Boot/test customizer 等 | mock bean、动态属性等可能影响 |
| `parent` | 父上下文 | 层级上下文参与比较 |
| `contextLoader` | 上下文加载器类名 | loader 不同则不能复用 |

排查测试慢时，一个核心方向就是找 cache key 为什么碎片化：大量测试使用不同 inline property、不同 mock bean、不同 active profile，都会降低复用率。

## TestExecutionListener 顺序

Spring Test 不把所有测试逻辑硬编码在 runner/extension 中，而是用 listener 链扩展生命周期：

```text
beforeTestClass
prepareTestInstance
beforeTestMethod
beforeTestExecution
test method
afterTestExecution
afterTestMethod
afterTestClass
```

常见 listener 职责：

- Servlet 测试上下文准备。
- DirtiesContext 前后处理。
- 应用事件记录。
- 依赖注入。
- 事务测试。
- SQL 脚本执行。

`TransactionalTestExecutionListener.ORDER = 4000`。源码注释说明它位于 common caches 之后、SQL scripts 之前。这让事务可以先开启，后续 SQL 脚本和测试方法能参与同一个测试事务。

## 事务测试流程

`TransactionalTestExecutionListener.beforeTestMethod()`：

```text
读取测试方法/类上的 @Transactional
  ↓
如果传播行为是 NOT_SUPPORTED 或 NEVER：不启动事务
  ↓
查找 PlatformTransactionManager
  ↓
创建 TransactionContext
  ↓
执行 @BeforeTransaction 方法
  ↓
txContext.startTransaction()
  ↓
绑定到 TransactionContextHolder
```

`afterTestMethod()`：

```text
从 TransactionContextHolder 移除 txContext
  ↓
如果事务仍 active
  ↓
txContext.endTransaction()
  ↓
执行 @AfterTransaction 方法
```

默认语义通常是测试事务回滚；可以通过 `@Commit`、`@Rollback` 或 `TestTransaction` 改变行为。关键是测试方法外层被 listener 包进事务，而不是业务代码自己显式开启。

## MockMvc 与 WebTestClient

### MockMvc

`MockMvc` 面向 Servlet MVC，无需真实 Servlet 容器即可测试 `DispatcherServlet` 分发链。它适合：

- 测试 MVC controller。
- 验证 filter、interceptor、argument resolver、return value handler。
- 与 Spring MVC 应用上下文集成。

核心价值：在内存中驱动 MVC 请求，保持接近真实 `DispatcherServlet` 的执行路径。

### WebTestClient

`WebTestClient` 是响应式测试客户端，既可以测试 WebFlux，也可以绑定到不同目标：

- 真实 server。
- `ApplicationContext`。
- `RouterFunction`。
- Controller。
- 通过 MockMvc bridge 测试 Spring MVC。

它的价值是统一“客户端式断言”：

```text
exchange()
  ↓
expectStatus()
  ↓
expectHeader()
  ↓
expectBody()
```

对于 WebFlux，它更贴近响应式链路；对于 MVC，它也可以作为更现代的测试 DSL。

## 工程建议

1. 大型项目应减少测试类之间的上下文差异，提升 ContextCache 命中率。
2. 谨慎使用 `@DirtiesContext`，它会破坏缓存复用。
3. 避免每个测试类都写不同 inline properties。
4. 事务测试适合数据库状态隔离，但不适合覆盖异步线程中提交事务的行为。
5. Controller 层轻量测试优先 MockMvc/WebTestClient；全链路验证再启动真实 server。

## 常见误区

1. **误区：每个测试类都会启动新 Spring 容器。**  
   只要 cache key 相同，同 JVM 内会复用 ApplicationContext。

2. **误区：`@Transactional` 测试等同生产事务。**  
   测试事务由 listener 包裹测试方法，默认回滚，边界与生产调用链可能不同。

3. **误区：MockMvc 是 mock controller。**  
   MockMvc mock 的是 Servlet 环境，不是简单 mock controller 方法。

4. **误区：WebTestClient 只能测 WebFlux。**  
   它也可以通过 bridge 测 MVC，区别在绑定方式和底层执行链。

## 面试表达

可以这样回答：

> Spring TestContext 会把测试配置合并成 `MergedContextConfiguration`，并用它作为 `ContextCache` 的 key。key 包括配置类、locations、active profiles、property sources、context customizers、parent 和 context loader 等。只要 key 相等，同 JVM 内就复用 ApplicationContext。事务测试由 `TransactionalTestExecutionListener` 在测试方法前读取 `@Transactional`、找事务管理器并开启事务，测试后结束事务并默认回滚。MockMvc 驱动 Servlet MVC 的 DispatcherServlet 链，WebTestClient 用客户端式 API 测 WebFlux，也可以绑定到 MockMvc 测 MVC。

## 和其它概念的关系

- [[10-domains/java/spring-framework/概念_Spring_WebFlux响应式处理链]] — WebTestClient 是验证 WebFlux 响应式链路的主要工具。
- [[10-domains/java/spring-framework/概念_Spring_Framework_IoC容器启动与Bean生命周期]] — ContextCache 缓存的是已经 refresh 完成的 ApplicationContext。
- [[10-domains/java/spring-framework/概念_Spring_AOP责任链源码时序]] — 测试事务和声明式业务事务都依赖 Spring transaction 抽象，但入口不同。

## 来源

- `raw/code/spring-framework/spring-test/src/main/java/org/springframework/test/context/cache/DefaultCacheAwareContextLoaderDelegate.java`
- `raw/code/spring-framework/spring-test/src/main/java/org/springframework/test/context/MergedContextConfiguration.java`
- `raw/code/spring-framework/spring-test/src/main/java/org/springframework/test/context/transaction/TransactionalTestExecutionListener.java`
- `raw/code/spring-framework/spring-test/src/main/java/org/springframework/test/web/servlet/MockMvc.java`
- `raw/code/spring-framework/spring-test/src/main/java/org/springframework/test/web/reactive/server/WebTestClient.java`
