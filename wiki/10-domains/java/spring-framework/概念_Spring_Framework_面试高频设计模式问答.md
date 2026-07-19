---
type: "concept"
tags:
  - java
  - spring-framework
  - design-pattern
  - interview
summary: "Spring Framework 设计模式面试高频问答：围绕工厂、单例、代理、适配器、模板方法、策略、观察者、责任链、回调等模式，给出源码类、为什么这么设计、好处、代价和易错点。"
sources:
  - "[[30-sources/repositories/来源_Spring_Framework_源码]]"
  - "raw/code/spring-framework"
aliases:
  - "Spring 设计模式面试"
  - "Spring 面试高频设计模式"
status: "evolving"
confidence: 0.88
created: "2026-05-12 21:14:19"
updated: "2026-05-12 21:14:19"
---

# 概念：Spring Framework 面试高频设计模式问答

## 定义

这是一页面试导向的 Spring 设计模式速查。它不只列模式名，而是按“源码类 → 解决问题 → 为什么这样设计 → 好处/代价”组织。

## 高频问答

### 1. Spring 中用了哪些设计模式？

高质量回答：

Spring 大量使用设计模式，但不是为了炫技，而是为了稳定扩展点。创建型包括 `BeanFactory`、`FactoryBean`、`AopProxyFactory`、`BeanDefinitionBuilder`、单例注册表；结构型包括 AOP 代理、`HandlerAdapter` 适配器、`HandlerMethodArgumentResolverComposite` 组合、DataSource wrapper；行为型包括 `refresh()` 模板方法、`JdbcTemplate`/`TransactionTemplate` 模板回调、`HandlerMapping` 等策略、`ApplicationListener` 观察者、AOP/MVC 拦截链责任链。

### 2. Spring 单例和 GoF 单例有什么区别？

源码类：

- `DefaultSingletonBeanRegistry`

回答：

GoF 单例强调一个类在 JVM 中通常只有一个实例；Spring 单例强调一个 BeanFactory 中，一个 beanName 对应一个共享实例。也就是说，同一个类可以在 Spring 容器里注册多个 beanName，得到多个“单例 Bean”。Spring 单例还包含创建中状态、三级缓存、销毁依赖等容器语义。

好处：

- 容器统一管理生命周期。
- 可以按 beanName 区分不同配置的同类对象。
- 支持循环依赖处理和销毁顺序。

### 3. BeanFactory 和 FactoryBean 区别？

源码类：

- `BeanFactory`
- `FactoryBean`

回答：

`BeanFactory` 是 Spring 容器，是生产和管理 Bean 的工厂。`FactoryBean` 是容器中的一种特殊 Bean，它自己也是工厂，用 `getObject()` 生产真正暴露给外部的对象。默认 `getBean("x")` 得到 FactoryBean 的产品，`getBean("&x")` 才得到 FactoryBean 本身。

为什么设计：

有些对象创建很复杂，例如 AOP 代理、JNDI 对象、远程代理。FactoryBean 把复杂创建逻辑封装起来，同时让产品对象像普通 Bean 一样被注入。

### 4. Spring AOP 为什么用代理模式？

源码类：

- `JdkDynamicAopProxy`
- `ObjenesisCglibAopProxy`
- `ProxyFactory`
- `DefaultAopProxyFactory`

回答：

AOP 要在业务方法前后增加事务、缓存、日志等横切逻辑。如果直接写在业务类里会强侵入。代理模式让客户端调用代理对象，代理对象执行 interceptor/advisor 链后再调用目标对象。

好处：

- 业务代码不感知横切逻辑。
- 多个 Advice 可组合。
- 可按 Bean、方法、注解控制增强范围。

代价：

- self-invocation 不生效。
- final/private 方法等不适合代理增强。
- 调用链复杂，排查时要知道代理类型。

### 5. JDK 动态代理和 CGLIB 怎么选？

源码类：

- `DefaultAopProxyFactory`

回答：

如果目标对象有用户提供接口且没有强制类代理，通常用 JDK 动态代理；如果强制 `proxyTargetClass` 或没有合适接口，普通类目标通常用 CGLIB/Objenesis 代理。但目标本身是接口、JDK Proxy 或 lambda class 时，仍可能使用 JDK 动态代理。

### 6. Spring MVC 为什么用适配器模式？

源码类：

- `HandlerAdapter`
- `RequestMappingHandlerAdapter`
- `SimpleControllerHandlerAdapter`

回答：

DispatcherServlet 不直接调用 handler，因为 handler 类型很多：注解方法、Controller 接口、HttpRequestHandler、Servlet、第三方 handler。`HandlerAdapter` 用 `supports()` 判断是否支持，用 `handle()` 执行调用。这样 DispatcherServlet 不需要知道具体 handler 类型。

好处：

- 中央分发流程稳定。
- 支持新旧编程模型共存。
- 新增 handler 类型不改 DispatcherServlet。

### 7. Spring 中模板方法模式有哪些？

源码类：

- `AbstractApplicationContext.refresh()`
- `JdbcTemplate.execute()`
- `TransactionTemplate.execute()`

回答：

`refresh()` 固定容器启动流程，留出 `postProcessBeanFactory()`、`onRefresh()` 等扩展点。`JdbcTemplate` 固定获取连接、执行、异常转换、释放资源，用户只实现 SQL 回调。`TransactionTemplate` 固定开启事务、执行回调、提交/回滚。

为什么设计：

资源和生命周期流程不能交给用户随意处理，否则容易泄露连接、事务不一致或启动顺序错乱。

### 8. Spring 中策略模式有哪些？

源码例子：

- `InstantiationStrategy`
- `AutowireCandidateResolver`
- `HandlerMapping`
- `HandlerMethodArgumentResolver`
- `HandlerMethodReturnValueHandler`
- `TransactionAttributeSource`
- `SQLExceptionTranslator`

回答：

策略模式把可替换算法封装为接口。例如请求如何找到 handler 由 `HandlerMapping` 决定，方法参数如何解析由 `HandlerMethodArgumentResolver` 决定，SQL 异常如何翻译由 `SQLExceptionTranslator` 决定。Spring 通过注册不同策略对象扩展功能，而不是修改核心流程。

### 9. Spring 观察者模式体现在哪里？

源码类：

- `ApplicationEvent`
- `ApplicationListener`
- `ApplicationEventMulticaster`
- `ApplicationEventPublisher`

回答：

ApplicationContext 发布事件，ApplicationEventMulticaster 负责广播给匹配的 ApplicationListener。发布者不需要知道有哪些监听者，监听者也可以独立注册或移除。`@EventListener` 是注解方式接入，底层仍会适配到事件监听机制。

### 10. Spring 中责任链模式有哪些？

源码/场景：

- AOP advisor/interceptor chain
- MVC `HandlerInterceptor` chain
- `HandlerMethodArgumentResolverComposite`
- `BeanPostProcessor` 列表
- `HandlerExceptionResolver` 列表

回答：

Spring 很多扩展点是一组有序处理器，请求或调用沿着处理器列表执行。AOP 拦截链可以在目标方法前后增强；MVC 拦截器链可以 preHandle/postHandle/afterCompletion；参数解析器链按 supports 判断谁能处理当前参数。

### 11. Spring 为什么大量使用回调？

源码类：

- `StatementCallback`
- `PreparedStatementCallback`
- `TransactionCallback`
- `ApplicationContextInitializer`

回答：

回调让框架掌控主流程和资源生命周期，用户只提供变化逻辑。比如 JdbcTemplate 控制连接、Statement、异常和释放，用户只写 `doInStatement`；TransactionTemplate 控制事务边界，用户只写事务内代码。

### 12. BeanPostProcessor 是什么模式？

回答：

它不是一个单纯 GoF 模式，更像插件扩展点，也有责任链/拦截器风格。多个 BeanPostProcessor 按顺序作用在 Bean 初始化前后，可以注入 Aware、处理注解、创建 AOP 代理。面试中可以说它体现了责任链思想，但不要强行说它就是标准责任链。

## 统一回答公式

```text
Spring 使用 X 模式。
源码中对应 A/B/C。
它解决的问题是 Y。
如果不用这个模式，核心流程会因为 Z 变得耦合或不可扩展。
好处是扩展点稳定、业务低侵入、生命周期可控。
代价是调用链更长，需要理解框架约定。
```

## 易错点

- 不要说“Spring 默认所有 Bean 都是 JVM 单例”，应说“容器内 singleton scope”。
- 不要把 `FactoryBean` 和 `BeanFactory` 混为一谈。
- 不要说 CGLIB 一定用于没有接口的类，实际还要看配置和目标类型。
- 不要说 `@Transactional` 任何内部方法调用都生效，self-invocation 不经过代理。
- 不要把所有列表遍历都说成责任链，要说明是否存在 supports/短路/传递。

## 和其它概念的关系

- [[10-domains/java/spring-framework/主题_Spring_Framework设计模式_综述]] — 设计模式总览。
- [[10-domains/java/spring-framework/概念_Spring_Framework_创建型设计模式]] — 创建型模式详解。
- [[10-domains/java/spring-framework/概念_Spring_Framework_结构型设计模式]] — 结构型模式详解。
- [[10-domains/java/spring-framework/概念_Spring_Framework_行为型设计模式]] — 行为型模式详解。

## 来源

- `raw/code/spring-framework`
- [[30-sources/repositories/来源_Spring_Framework_源码]] — Spring Framework 源码入口。

