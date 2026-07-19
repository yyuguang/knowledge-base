---
type: "overview"
tags:
  - java
  - spring-framework
  - design-pattern
  - interview
summary: "Spring Framework 设计模式综述：Spring 不是为了展示模式而使用模式，而是用模式稳定扩展点、隔离变化、统一生命周期；高频模式包括工厂、单例、建造者、代理、适配器、组合、装饰器、模板方法、策略、观察者、责任链和回调。"
sources:
  - "[[30-sources/repositories/来源_Spring_Framework_源码]]"
  - "raw/code/spring-framework"
aliases:
  - "Spring 设计模式"
  - "Spring Framework 设计模式"
  - "Spring 面试设计模式"
status: "evolving"
confidence: 0.9
created: "2026-05-12 21:14:19"
updated: "2026-05-12 21:14:19"
---

# 主题：Spring Framework设计模式 综述

![[Attachments/spring-framework-design-patterns-map.excalidraw]]

## 当前结论

Spring Framework 中的设计模式不是“套模板”，而是为了解决四类框架问题：

1. **对象怎么创建**：工厂、FactoryBean、Builder、Singleton Registry。
2. **对象怎么包裹和适配**：代理、适配器、组合、装饰器。
3. **流程怎么稳定且可扩展**：模板方法、策略、责任链、回调。
4. **事件和扩展怎么解耦**：观察者、监听器、发布-订阅。

面试时不要只背“Spring 用了工厂、单例、代理”。更好的回答是：Spring 用设计模式把可变点放到接口/回调/策略对象里，把稳定流程留在框架内部，从而实现低侵入扩展。

## 总体框架

```text
创建型
  Factory / FactoryBean / Abstract Factory / Builder / Singleton Registry
  → 解决 Bean、代理、BeanDefinition 如何创建

结构型
  Proxy / Adapter / Composite / Decorator
  → 解决 横切增强、协议适配、多个策略组合、资源包装

行为型
  Template Method / Strategy / Observer / Chain / Callback
  → 解决 固定流程、可替换算法、事件通知、拦截链、用户扩展代码
```

## 核心概念

- [[10-domains/java/spring-framework/概念_Spring_Framework_创建型设计模式]] — FactoryBean、BeanFactory、AopProxyFactory、BeanDefinitionBuilder、Singleton Registry。
- [[10-domains/java/spring-framework/概念_Spring_Framework_结构型设计模式]] — AOP Proxy、HandlerAdapter、Composite、Decorator。
- [[10-domains/java/spring-framework/概念_Spring_Framework_行为型设计模式]] — Template Method、Strategy、Observer、Chain of Responsibility、Callback。
- [[10-domains/java/spring-framework/概念_Spring_Framework_面试高频设计模式问答]] — 面试场景下的高频问题、回答结构和易错点。

## 关键源码映射

| 模式 | Spring 源码例子 | 解决的问题 |
|---|---|---|
| Factory Method / Factory | `BeanFactory#getBean()`、`FactoryBean#getObject()` | 屏蔽对象创建细节 |
| Abstract Factory | `AopProxyFactory#createAopProxy()` | 根据配置创建 JDK/CGLIB 等一族代理对象 |
| Builder | `BeanDefinitionBuilder` | 程序化构建复杂 BeanDefinition |
| Singleton Registry | `DefaultSingletonBeanRegistry` | 管理共享单例和三级缓存 |
| Proxy | `JdkDynamicAopProxy`、`ObjenesisCglibAopProxy`、`TransactionAwareDataSourceProxy` | 在目标对象外增加事务、切面、资源感知 |
| Adapter | `HandlerAdapter` | 让 DispatcherServlet 调用任意 handler 类型 |
| Composite | `HandlerMethodArgumentResolverComposite`、`CompositePropertySource` | 把多个策略或配置源当成一个对象使用 |
| Decorator | `DelegatingDataSource`、各种 request/response wrapper | 包装对象并增强行为 |
| Template Method | `AbstractApplicationContext.refresh()`、`JdbcTemplate.execute()`、`TransactionTemplate.execute()` | 固定主流程，开放局部步骤 |
| Strategy | `InstantiationStrategy`、`HandlerMapping`、`HandlerMethodArgumentResolver`、`TransactionAttributeSource` | 可替换算法或决策 |
| Observer | `ApplicationEventMulticaster`、`ApplicationListener` | 事件发布者与监听者解耦 |
| Chain of Responsibility | MVC interceptor chain、AOP advisor chain、argument resolver list | 请求/调用在一组处理器中传递 |
| Callback | `StatementCallback`、`TransactionCallback`、`ApplicationContextInitializer` | 框架控制流程，用户提供变化逻辑 |

## 为什么这么设计

### 1. Spring 是框架，不是业务库

业务库可以直接暴露几个方法；框架必须允许用户插入自己的行为。设计模式的作用，是让 Spring 知道“什么时候调用用户代码”，但不需要知道用户代码具体是什么。

### 2. Spring 需要同时支持旧 API 和新模型

例如 MVC 既要支持实现 `Controller` 接口的老 handler，也要支持 `@RequestMapping` 注解方法，还要能接入其它 handler 类型。`HandlerAdapter` 把这些差异隔离掉。

### 3. Spring 需要横切增强但不能侵入业务类

事务、缓存、日志、权限如果写进业务类，会污染业务逻辑。代理模式让增强逻辑包裹在目标对象外部。

### 4. Spring 的稳定流程很多

容器刷新、Bean 创建、JDBC 执行、事务执行、MVC 分发都有固定骨架。模板方法和回调让框架掌控资源和异常处理，用户只填变化部分。

## 好处

- **扩展点稳定**：用户实现接口或注册 Bean，不需要改 Spring 源码。
- **变化隔离**：创建方式、调用方式、资源类型、异常处理都被封装在策略或适配层。
- **生命周期统一**：框架掌控创建、初始化、调用、销毁、异常和资源释放。
- **低侵入增强**：AOP 代理让事务等能力不污染业务类。
- **组合能力强**：多个 BeanPostProcessor、Advisor、HandlerMapping、ArgumentResolver 可以按顺序协同。
- **面向接口编程**：大部分核心点都通过 SPI 暴露，适合第三方生态接入。

## 面试表达模板

回答 Spring 设计模式时可以按这个结构：

1. 先说模式名称。
2. 指出源码类。
3. 说明解决的问题。
4. 解释为什么不用更简单的写法。
5. 总结好处和代价。

示例：

> Spring MVC 使用适配器模式，核心接口是 `HandlerAdapter`。`DispatcherServlet` 不直接调用 Controller，因为 handler 可能是注解方法、Controller 接口、HttpRequestHandler 或第三方 handler。通过 `supports()` 和 `handle()`，DispatcherServlet 只依赖统一适配器接口，从而保持分发流程稳定，并允许新增 handler 类型。代价是调用链更长，需要理解 adapter 选择过程。

## 未解决问题

1. AOP advisor chain 和 MVC interceptor chain 可以继续画源码级调用时序。
2. `MergedAnnotations` 中组合注解和别名解析也可视为解释器/访问者风格，需要单独分析。
3. `BeanPostProcessor` 既像责任链又像插件机制，后续可单独拆它和 `BeanFactoryPostProcessor` 的差异。

## 相关链接

- [[10-domains/java/spring-framework/主题_Spring_Framework源码学习_综述]] — Spring Framework 源码总体路线。
- [[10-domains/java/spring-framework/概念_Spring_Framework_AOP事务与Web分发]] — 代理、事务和 Web 分发主线。
- [[10-domains/java/spring-framework/概念_Spring_Framework_Web测试消息与应用层扩展]] — HandlerAdapter、ArgumentResolver 等应用层策略。
- [[10-domains/java/spring-framework/概念_Spring_Framework_数据访问事务同步与集成模块]] — JdbcTemplate、TransactionTemplate 与资源同步。

