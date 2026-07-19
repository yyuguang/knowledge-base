---
type: "concept"
tags:
  - java
  - spring-framework
  - design-pattern
  - behavioral-pattern
summary: "Spring Framework 行为型设计模式：模板方法固定容器、JDBC、事务等主流程；策略模式替换创建、映射、解析算法；观察者模式解耦事件发布和监听；责任链组织拦截器、Advisor 和解析器；回调模式让框架控制资源生命周期、用户提供变化逻辑。"
sources:
  - "[[30-sources/repositories/来源_Spring_Framework_源码]]"
  - "raw/code/spring-framework/spring-context/src/main/java/org/springframework/context/support/AbstractApplicationContext.java"
  - "raw/code/spring-framework/spring-jdbc/src/main/java/org/springframework/jdbc/core/JdbcTemplate.java"
  - "raw/code/spring-framework/spring-tx/src/main/java/org/springframework/transaction/support/TransactionTemplate.java"
  - "raw/code/spring-framework/spring-context/src/main/java/org/springframework/context/event/ApplicationEventMulticaster.java"
aliases:
  - "Spring 行为型模式"
  - "Spring 模板方法策略观察者责任链"
status: "evolving"
confidence: 0.9
created: "2026-05-12 21:14:19"
updated: "2026-05-12 21:14:19"
---

# 概念：Spring Framework 行为型设计模式

## 定义

行为型设计模式关注“对象之间如何协作、流程如何变化”。Spring 大量使用行为型模式来固定主流程，同时允许用户或框架模块替换局部行为。

高频模式：

- 模板方法：`AbstractApplicationContext.refresh()`、`JdbcTemplate.execute()`、`TransactionTemplate.execute()`
- 策略：`InstantiationStrategy`、`HandlerMapping`、`HandlerMethodArgumentResolver`、`TransactionAttributeSource`
- 观察者：`ApplicationEventMulticaster`、`ApplicationListener`
- 责任链：AOP advisor chain、MVC interceptor chain、argument resolver list
- 回调：`StatementCallback`、`TransactionCallback`

## 为什么重要

Spring 是“流程型框架”：容器刷新、Bean 创建、JDBC 执行、事务提交、Web 分发都有固定步骤。用户不能随意接管所有步骤，否则资源释放、异常转换、事务一致性都会出问题。

行为型模式的作用是：

- 框架掌控不可变流程。
- 用户提供可变行为。
- 多个扩展按顺序协作。
- 事件发布者和监听者解耦。

## 核心模式

### 1. Template Method：固定主流程，开放步骤

典型例子：`AbstractApplicationContext.refresh()`

```text
prepareRefresh()
obtainFreshBeanFactory()
prepareBeanFactory()
postProcessBeanFactory()
invokeBeanFactoryPostProcessors()
registerBeanPostProcessors()
initMessageSource()
initApplicationEventMulticaster()
onRefresh()
registerListeners()
finishBeanFactoryInitialization()
finishRefresh()
```

其中 `postProcessBeanFactory()`、`onRefresh()` 等留给子类扩展，但整体顺序由父类控制。

**为什么这么设计**：容器启动顺序必须稳定，否则配置类解析、后处理器、监听器、单例创建会互相踩踏。

**好处**：

- 启动流程可预测。
- 子类可扩展局部步骤。
- 错误处理和资源清理由框架统一控制。

### 2. JdbcTemplate / TransactionTemplate：模板方法 + 回调

`JdbcTemplate.execute()` 固定：

1. 获取 Connection。
2. 创建 Statement。
3. 执行用户回调 `StatementCallback#doInStatement`。
4. 异常转换。
5. 释放资源。

`TransactionTemplate.execute()` 固定：

1. 获取事务。
2. 执行 `TransactionCallback#doInTransaction`。
3. 异常则回滚。
4. 正常则提交。

**为什么这么设计**：JDBC 和事务最容易出错的是资源释放、异常处理、提交回滚。模板把这些固定下来，用户只写业务 SQL 或事务内逻辑。

**好处**：

- 避免重复 try/finally。
- 统一异常和资源处理。
- 用户代码更聚焦。

### 3. Strategy：替换算法和决策

Spring 中的策略非常多：

| 策略接口 | 变化点 |
|---|---|
| `InstantiationStrategy` | Bean 如何实例化 |
| `AutowireCandidateResolver` | 候选 Bean 如何判断 |
| `HandlerMapping` | 请求如何映射到 handler |
| `HandlerAdapter` | handler 如何调用 |
| `HandlerMethodArgumentResolver` | 方法参数如何解析 |
| `HandlerMethodReturnValueHandler` | 返回值如何处理 |
| `TransactionAttributeSource` | 事务属性从哪里读取 |
| `SQLExceptionTranslator` | SQL 异常如何翻译 |

**为什么这么设计**：Spring 要支持多种算法，并允许用户替换。把算法封装为策略对象，比在框架里写一堆条件判断更可维护。

### 4. Observer：应用事件

`ApplicationEventMulticaster` 管理多个 `ApplicationListener`，负责把事件广播给匹配监听器。

参与者：

- 发布者：`ApplicationEventPublisher` / `ApplicationContext`
- 事件：`ApplicationEvent` 或 payload event
- 监听者：`ApplicationListener`
- 广播器：`ApplicationEventMulticaster`

**为什么这么设计**：容器刷新、关闭、业务事件不应该要求发布者知道所有订阅者。观察者模式让发布和监听解耦。

**好处**：

- 发布者不依赖监听者。
- 监听者可动态增减。
- 支持同步/异步执行。
- `@EventListener` 可通过适配器接入。

### 5. Chain of Responsibility：拦截链和解析链

Spring 中的链非常多：

- AOP advisor/interceptor chain
- MVC `HandlerInterceptor` chain
- `HandlerMethodArgumentResolverComposite`
- `BeanPostProcessor` 列表
- `HandlerExceptionResolver` 列表

以 MVC 参数解析为例：

```text
MethodParameter
  → resolver1.supports?
  → resolver2.supports?
  → resolver3.supports?
  → resolver3.resolveArgument()
```

以 AOP 为例：

```text
proxy.invoke()
  → interceptor1
  → interceptor2
  → TransactionInterceptor
  → target method
```

**为什么这么设计**：扩展点不是一个，而是一组。链式结构允许多个横切逻辑按顺序协作，也允许短路。

## 面试高频说法

### Spring 哪里用了模板方法？

`AbstractApplicationContext.refresh()` 是容器级模板方法，定义应用上下文启动固定顺序；`JdbcTemplate` 和 `TransactionTemplate` 是资源操作模板，固定资源获取、异常处理、释放或提交回滚，把变化点交给回调。

### Spring 哪里用了观察者模式？

ApplicationContext 通过 `ApplicationEventMulticaster` 发布事件，监听器实现 `ApplicationListener` 或使用 `@EventListener`。这样事件发布者不需要知道谁在监听。

### Spring 哪里用了责任链？

AOP advisor chain、MVC interceptor chain、BeanPostProcessor 列表、HandlerExceptionResolver 列表都体现责任链。它们把多个处理器按顺序组织起来，每个处理器处理一部分逻辑，必要时继续向后传递。

## 常见误区

1. **误区：模板方法只存在于抽象类。**  
   Spring 里的 `JdbcTemplate`、`TransactionTemplate` 也体现模板思想：固定流程 + 回调变化点。

2. **误区：策略模式和工厂模式分不清。**  
   工厂关注创建哪个对象；策略关注运行时使用哪个算法或处理逻辑。

3. **误区：所有 list 遍历都是责任链。**  
   只有当请求沿处理器列表传递，并由处理器决定是否处理/继续时，才更接近责任链。

## 和其它概念的关系

- [[10-domains/java/spring-framework/概念_Spring_Framework_IoC容器启动与Bean生命周期]] — `refresh()` 是模板方法的核心例子。
- [[10-domains/java/spring-framework/概念_Spring_Framework_数据访问事务同步与集成模块]] — JdbcTemplate、TransactionTemplate 展示模板+回调。
- [[10-domains/java/spring-framework/概念_Spring_Framework_Web测试消息与应用层扩展]] — Web 参数解析、返回值处理、拦截器链体现策略和责任链。
- [[10-domains/java/spring-framework/主题_Spring_Framework设计模式_综述]] — 设计模式总览。

## 我的理解

Spring 行为型模式可以归纳为一句话：框架负责流程，用户负责变化。这个边界非常关键。如果让用户负责流程，资源释放和异常一致性就会失控；如果框架不给变化点，Spring 又无法成为通用框架。

## 来源

- `raw/code/spring-framework/spring-context/src/main/java/org/springframework/context/support/AbstractApplicationContext.java`
- `raw/code/spring-framework/spring-jdbc/src/main/java/org/springframework/jdbc/core/JdbcTemplate.java`
- `raw/code/spring-framework/spring-tx/src/main/java/org/springframework/transaction/support/TransactionTemplate.java`
- `raw/code/spring-framework/spring-context/src/main/java/org/springframework/context/event/ApplicationEventMulticaster.java`

