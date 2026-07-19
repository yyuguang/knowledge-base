---
type: "concept"
tags:
  - java
  - spring-framework
  - design-pattern
  - structural-pattern
summary: "Spring Framework 结构型设计模式：AOP 与事务用代理模式做低侵入增强，HandlerAdapter 适配不同 handler，Composite 聚合多个策略或配置源，Decorator/Wrapper 在不改变原对象的前提下增加资源感知和上下文能力。"
sources:
  - "[[30-sources/repositories/来源_Spring_Framework_源码]]"
  - "raw/code/spring-framework/spring-aop/src/main/java/org/springframework/aop/framework/JdkDynamicAopProxy.java"
  - "raw/code/spring-framework/spring-webmvc/src/main/java/org/springframework/web/servlet/HandlerAdapter.java"
  - "raw/code/spring-framework/spring-web/src/main/java/org/springframework/web/method/support/HandlerMethodArgumentResolverComposite.java"
  - "raw/code/spring-framework/spring-core/src/main/java/org/springframework/core/env/CompositePropertySource.java"
  - "raw/code/spring-framework/spring-jdbc/src/main/java/org/springframework/jdbc/datasource/TransactionAwareDataSourceProxy.java"
aliases:
  - "Spring 结构型模式"
  - "Spring 代理适配器组合装饰器"
status: "evolving"
confidence: 0.88
created: "2026-05-12 21:14:19"
updated: "2026-05-12 21:14:19"
---

# 概念：Spring Framework 结构型设计模式

## 定义

结构型设计模式关注“对象之间如何组合”。Spring 使用结构型模式把不同对象接到统一框架结构中，或者在对象外层增加能力。

高频模式：

- 代理模式：AOP、事务、数据源事务感知。
- 适配器模式：`HandlerAdapter`。
- 组合模式：`HandlerMethodArgumentResolverComposite`、`CompositePropertySource`。
- 装饰器/包装器：`DelegatingDataSource`、request/response wrapper、各种 proxy/adapter。

## 为什么重要

Spring 作为框架必须面对大量“不统一”的对象：

- 业务 Bean 不应该知道事务代理。
- Web handler 可以是接口实现、注解方法、Servlet 或第三方对象。
- 参数解析器、返回值处理器、PropertySource 都可能有多个。
- 外部资源对象需要增加 Spring 事务感知。

结构型模式让 Spring 在不改原对象的前提下统一调用方式和增强行为。

## 核心模式

### 1. Proxy：低侵入横切增强

Spring AOP 的核心是代理模式：

```text
client
  → proxy object
      → interceptor/advisor chain
      → target method
```

典型源码：

- `JdkDynamicAopProxy`
- `ObjenesisCglibAopProxy`
- `ProxyFactory`
- `TransactionAwareDataSourceProxy`

**为什么这么设计**：事务、缓存、日志、安全等横切逻辑不应该写进业务类。代理对象可以在目标方法前后插入行为。

**好处**：

- 业务代码低侵入。
- 横切逻辑复用。
- 多个 Advice 可以组合成链。
- 可以按 Bean 粒度启用或关闭。

**代价**：

- self-invocation 不经过代理。
- final 类/方法、private 方法等有代理限制。
- 调试调用链更复杂。

### 2. Adapter：DispatcherServlet 不关心 handler 类型

`HandlerAdapter` 的源码注释很典型：它让核心 MVC workflow 可以参数化。`DispatcherServlet` 通过适配器访问所有 handler，不包含特定 handler 类型的代码。

接口方法：

- `supports(Object handler)`
- `handle(request, response, handler)`

**为什么这么设计**：Spring MVC 需要支持多种 handler：注解方法、`Controller` 接口、`HttpRequestHandler`、Servlet、第三方 handler。如果 DispatcherServlet 直接 `if-else` 判断类型，会越来越臃肿。

**好处**：

- DispatcherServlet 主流程稳定。
- 新 handler 类型只需新增 HandlerAdapter。
- 旧编程模型和新注解模型可以共存。

### 3. Composite：多个对象作为一个对象使用

`HandlerMethodArgumentResolverComposite` 实现了 `HandlerMethodArgumentResolver`，内部维护多个 resolver。

工作方式：

1. 遍历 resolver 列表。
2. 找到 `supportsParameter()` 为 true 的 resolver。
3. 缓存 MethodParameter → resolver。
4. 委托该 resolver 解析参数。

另一个例子是 `CompositePropertySource`：多个 PropertySource 组合成一个 PropertySource，按顺序查找属性。

**为什么这么设计**：框架扩展点经常是一组策略，而上层希望像使用一个对象一样使用它们。

**好处**：

- 上层接口保持简单。
- 多个扩展可以按顺序组合。
- 可以缓存选择结果，提高性能。

### 4. Decorator / Wrapper：在原对象外增加能力

`TransactionAwareDataSourceProxy` 包装目标 DataSource，让普通 JDBC 代码也能参与 Spring 管理的事务。

它的关键点：

- 对外仍表现为 `DataSource`。
- 内部委托目标 DataSource。
- `getConnection()` 和 `close()` 通过 Spring 的 `DataSourceUtils` 参与线程绑定事务。

**为什么这么设计**：有些第三方代码只认识标准 JDBC API，不会主动调用 Spring 的 DataSourceUtils。包装器让它在不改代码的情况下接入 Spring 事务。

**好处**：

- 保持原接口不变。
- 增加 Spring 上下文感知能力。
- 兼容第三方或遗留代码。

## 面试高频说法

### Spring AOP 为什么用代理模式？

因为代理能在目标对象外部织入横切逻辑，业务对象无需感知事务、缓存、日志等基础设施。Spring 根据目标是否有接口、是否强制类代理等条件选择 JDK 动态代理或 CGLIB 代理。

### Spring MVC 为什么用适配器模式？

因为 DispatcherServlet 需要调用多种 handler 类型。用 HandlerAdapter 后，DispatcherServlet 只依赖统一接口，新增 handler 类型不需要修改中央分发器。

### ArgumentResolverComposite 是什么模式？

它是组合模式，也带一点责任链味道。它把多个参数解析器组合成一个解析器，对外仍暴露 `supportsParameter/resolveArgument`，内部按顺序找到合适解析器。

## 常见误区

1. **误区：代理模式只用于 AOP。**  
   `TransactionAwareDataSourceProxy` 也是代理/包装，用于资源感知。

2. **误区：适配器就是转换数据格式。**  
   在 MVC 中，适配器是把不同 handler 调用协议统一成 `handle()`。

3. **误区：Composite 和 Chain 完全一样。**  
   Composite 强调“一组对象作为一个对象”；Chain 强调“请求沿链传递并可能短路”。Spring 很多实现同时有两者味道。

## 和其它概念的关系

- [[10-domains/java/spring-framework/概念_Spring_Framework_AOP事务与Web分发]] — AOP 代理和 HandlerAdapter 是结构型模式的核心例子。
- [[10-domains/java/spring-framework/概念_Spring_Framework_Web测试消息与应用层扩展]] — ArgumentResolverComposite 位于 MVC 方法调用层。
- [[10-domains/java/spring-framework/主题_Spring_Framework设计模式_综述]] — 设计模式总览。

## 我的理解

Spring 结构型模式的核心是“不要让中心流程认识所有具体对象”。代理把目标对象包起来，适配器把不同对象统一成一种调用协议，组合对象把多个策略折叠成一个入口。这样 Spring 主流程才能稳定地演进很多年。

## 来源

- `raw/code/spring-framework/spring-webmvc/src/main/java/org/springframework/web/servlet/HandlerAdapter.java`
- `raw/code/spring-framework/spring-web/src/main/java/org/springframework/web/method/support/HandlerMethodArgumentResolverComposite.java`
- `raw/code/spring-framework/spring-core/src/main/java/org/springframework/core/env/CompositePropertySource.java`
- `raw/code/spring-framework/spring-jdbc/src/main/java/org/springframework/jdbc/datasource/TransactionAwareDataSourceProxy.java`

