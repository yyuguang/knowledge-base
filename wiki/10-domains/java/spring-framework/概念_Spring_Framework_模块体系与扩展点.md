---
type: "concept"
tags:
  - java
  - spring-framework
  - architecture
  - extension-points
summary: "Spring Framework 模块体系与扩展点：core/beans/context 是内核，aop/tx/web/test 等模块围绕 BeanFactory/ApplicationContext 的生命周期和策略发现机制扩展。"
sources:
  - "[[30-sources/repositories/来源_Spring_Framework_源码]]"
  - "raw/code/spring-framework/settings.gradle"
  - "raw/code/spring-framework/README.md"
aliases:
  - "Spring Framework 模块体系"
  - "Spring Framework 扩展点"
status: "evolving"
confidence: 0.88
created: "2026-05-12 20:35:53"
updated: "2026-05-12 20:35:53"
---

# 概念：Spring Framework 模块体系与扩展点

## 定义

Spring Framework 的模块体系可以理解为一个分层内核：

```text
spring-core
  ↓
spring-beans
  ↓
spring-context
  ↓
spring-aop / spring-tx / spring-expression / spring-context-support
  ↓
spring-web / spring-webmvc / spring-webflux / spring-jdbc / spring-test / integration modules
```

其中 `spring-core`、`spring-beans`、`spring-context` 构成最小理解闭环；其它模块不是孤立功能，而是通过容器、代理、模板方法、策略 Bean 和后处理器接入这个闭环。

## 为什么重要

读 Spring 源码最容易迷路的原因是模块太多、类太多。如果没有模块地图，会把 `DispatcherServlet`、`TransactionInterceptor`、`BeanPostProcessor`、`ApplicationEventMulticaster` 看成一堆分散机制。

更准确的读法是：

- `spring-core` 提供“语言增强”：资源、注解、类型、反射、转换、环境。
- `spring-beans` 提供“对象工厂”：BeanDefinition → Bean 实例。
- `spring-context` 提供“应用运行时”：refresh、事件、国际化、资源加载、生命周期。
- `spring-aop` 提供“对象外壳”：在方法调用外包一层拦截链。
- `spring-tx`、cache、security 等能力通常借助 AOP 或 BeanPostProcessor 接入。
- `spring-webmvc` / `spring-webflux` 把策略发现与拦截链模式用于 HTTP 请求。

## 使用场景

- 做源码学习计划：先按模块依赖顺序读，而不是按业务兴趣随机跳。
- 排查 Spring Boot 启动问题：区分是 BeanDefinition 阶段、Bean 实例化阶段、BeanPostProcessor 阶段，还是 Web 分发阶段。
- 写框架扩展：判断该用 `BeanFactoryPostProcessor`、`BeanPostProcessor`、`FactoryBean`、`Advisor`、`HandlerMethodArgumentResolver` 还是 `HandlerInterceptor`。
- 理解 Spring Boot：Boot 的 AutoConfiguration 本质上是在 Spring Framework 容器扩展点上批量注册配置。

## 核心机制

### 1. 三层内核

| 层 | 核心模块 | 关键对象 | 负责什么 |
|---|---|---|---|
| 基础层 | `spring-core` | `Resource`、`Environment`、`ResolvableType`、`MergedAnnotations`、`ConversionService` | 让 Java 语言和 JDK 反射更适合做框架 |
| 容器层 | `spring-beans` | `BeanDefinition`、`BeanFactory`、`BeanWrapper`、`FactoryBean`、`BeanPostProcessor` | 把元数据编译成对象图 |
| 应用层 | `spring-context` | `ApplicationContext`、`ApplicationEventMulticaster`、`MessageSource`、`LifecycleProcessor` | 把对象图升级为应用运行时 |

### 2. 模块不是平铺，而是围绕扩展点挂载

| 模块 | 挂载点 | 典型扩展方式 |
|---|---|---|
| `spring-aop` | Bean 创建后期 | `BeanPostProcessor` 创建代理对象 |
| `spring-tx` | AOP 方法调用 | `TransactionInterceptor` 包裹目标方法 |
| `spring-webmvc` | Servlet 请求入口 | `DispatcherServlet` 发现策略 Bean |
| `spring-webflux` | Reactive WebHandler | `DispatcherHandler` 发现 mapping/adapter/result handler |
| `spring-jdbc` | 事务同步与模板方法 | `JdbcTemplate`、`TransactionSynchronizationManager` |
| `spring-test` | TestContext 生命周期 | 上下文缓存、监听器、测试执行回调 |
| `spring-expression` | BeanDefinition/注解属性解析 | `BeanExpressionResolver`、SpEL AST |

### 3. Spring 常见扩展点分类

| 扩展点 | 发生阶段 | 能改变什么 | 风险 |
|---|---|---|---|
| `BeanDefinitionRegistryPostProcessor` | BeanDefinition 加载后、普通 BFPP 前 | 新增/修改 BeanDefinition | 影响范围最大，顺序敏感 |
| `BeanFactoryPostProcessor` | Bean 实例化前 | 修改 BeanDefinition 属性 | 不能依赖普通 Bean 已创建 |
| `BeanPostProcessor` | Bean 实例初始化前后 | 改 Bean 实例、创建代理 | 容易影响 AOP、循环依赖 |
| `SmartInstantiationAwareBeanPostProcessor` | 实例化前、类型预测、早期引用 | 参与构造器选择、提前暴露代理 | 是 AOP 和循环依赖交汇点 |
| `FactoryBean` | `getBean()` 获取对象时 | 把一个工厂 Bean 暴露成另一个产品对象 | `&beanName` 与 `beanName` 语义不同 |
| `Advisor/Advice` | 方法调用时 | 在目标方法前后插入行为 | self-invocation 不经过代理 |
| `HandlerMapping/HandlerAdapter` | Web 请求分发时 | 决定请求由谁处理、如何调用 | 排序和匹配条件影响结果 |

## 例子

### 声明式事务如何跨模块协作

1. `spring-context` 解析 `@EnableTransactionManagement`，注册事务相关 BeanDefinition。
2. `spring-aop` 通过 auto-proxy creator 在 Bean 初始化后创建代理。
3. `spring-tx` 提供 `TransactionInterceptor` 和事务属性解析。
4. 业务调用进入代理后，AOP 拦截链执行事务逻辑。
5. JDBC/JPA/R2DBC 模块提供具体 `TransactionManager` 实现。

这说明事务不是一个模块独立完成的，而是 `context + aop + tx + data access` 的协作结果。

### MVC 请求如何复用容器策略发现

`DispatcherServlet` 启动时从 ApplicationContext 中按类型查找：

- `HandlerMapping`
- `HandlerAdapter`
- `HandlerExceptionResolver`
- `ViewResolver`
- `LocaleResolver`
- `MultipartResolver`

这和 `ApplicationContext.refresh()` 查找 `BeanFactoryPostProcessor`、`BeanPostProcessor`、`ApplicationListener` 是同一思想：框架提供流程骨架，用户通过 Bean 提供策略。

## 常见误区

1. **误区：Spring Core 就是工具类集合。**  
   更准确地说，`spring-core` 是框架编程底座，它提供类型推断、注解合成、资源抽象、环境抽象等能力。

2. **误区：ApplicationContext 只是 BeanFactory 的增强版。**  
   它不只是加功能，而是通过 `refresh()` 定义应用启动协议，负责把容器状态推进到可运行态。

3. **误区：AOP、事务、MVC 是不同设计。**  
   它们共享同一种骨架：固定主流程 + 可排序策略列表 + 拦截/回调扩展点。

## 和其它概念的关系

- [[10-domains/java/spring-framework/概念_Spring_Framework_IoC容器启动与Bean生命周期]] — 模块体系的核心执行主线是 `refresh()` 与 Bean 生命周期。
- [[10-domains/java/spring-framework/概念_Spring_Framework_AOP事务与Web分发]] — AOP、事务和 Web 展示了运行期扩展点如何工作。
- [[30-sources/repositories/来源_Spring_Framework_源码]] — 当前概念来自 Spring Framework 源码模块清单与关键类阅读。

## 我的理解

Spring Framework 的模块边界体现了一种框架设计审美：底层模块只提供抽象能力，上层模块把抽象能力组合成业务基础设施。它不是靠继承层级把所有东西绑定死，而是大量使用“接口 + 策略列表 + 后处理器 + 模板方法”。这也是 Spring 能长期演进的关键。

学习时应把每个模块都问成三个问题：

1. 它依赖哪个底层抽象？
2. 它向上暴露哪个稳定接口？
3. 它通过哪个生命周期阶段或策略点接入容器？

## 来源

- `raw/code/spring-framework/settings.gradle`
- `raw/code/spring-framework/README.md`
- [[30-sources/repositories/来源_Spring_Framework_源码]] — 源码入口总览。

