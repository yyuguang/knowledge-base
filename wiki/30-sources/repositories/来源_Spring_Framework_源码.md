---
type: "source"
tags:
  - java
  - spring-framework
  - spring
  - source-code
summary: "Spring Framework 源码学习入口：从 Gradle 模块、Spring Core 元数据底座、配置类解析、依赖解析、IoC 容器刷新、Bean 创建、AOP 代理、声明式事务、数据访问、MVC/WebFlux、测试与消息模块，到 MergedAnnotations、ConfigurationClassParser、advisor chain、WebFlux result handler、TestContext cache 等复杂链路。"
sources:
  - "raw/code/spring-framework"
aliases:
  - "Spring Framework 源码"
  - "Spring 源码"
status: "evolving"
confidence: 0.9
created: "2026-05-12 20:35:53"
updated: "2026-05-13 09:14:32"
---

# 来源：Spring Framework 源码

## 一句话总结

Spring Framework 不是一个单一框架，而是一组围绕 **IoC 容器** 展开的基础设施模块：`spring-core` 提供资源、类型、反射、注解与转换等底座，`spring-beans` 管理 BeanDefinition 与 Bean 生命周期，`spring-context` 把 BeanFactory 升级为可刷新、可事件广播、可国际化、可资源加载的应用上下文，`spring-aop` 与 `spring-tx` 在容器生命周期中织入横切能力，`spring-webmvc` / `spring-webflux` 分别把同一套策略发现思想落到 Servlet 与 Reactive Web 请求分发。

## 来源信息

- 源码路径：`raw/code/spring-framework`
- 构建系统：Gradle，多模块工程，根项目名 `spring`
- 模块清单来自：`raw/code/spring-framework/settings.gradle`
- 关键文档入口：`raw/code/spring-framework/README.md` 与 `raw/code/spring-framework/framework-docs/modules/ROOT/pages/`
- 许可证：Apache 2.0

## 模块地图

| 模块 | 源码数量约略 | 角色 | 源码学习入口 |
|---|---:|---|---|
| `spring-core` | 731 | 框架底座：资源、环境、类型解析、注解、反射、转换、AOT 基础 | `org.springframework.core.*`、`io.*`、`env.*` |
| `spring-beans` | 336 | BeanDefinition、BeanFactory、属性填充、依赖解析、FactoryBean | `DefaultListableBeanFactory`、`AbstractAutowireCapableBeanFactory` |
| `spring-context` | 625 | ApplicationContext、refresh 生命周期、事件、国际化、扫描、注解配置 | `AbstractApplicationContext.refresh()` |
| `spring-aop` | 226 | Advisor/Advice/Pointcut、代理工厂、JDK/CGLIB 代理选择 | `DefaultAopProxyFactory.createAopProxy()` |
| `spring-tx` | 173 | 编程式/声明式事务、传播行为、事务同步、响应式事务 | `TransactionAspectSupport.invokeWithinTransaction()` |
| `spring-jdbc` | 253 | JDBC 模板、异常转换、DataSource 工具、命名参数 | `JdbcTemplate`、`DataSourceTransactionManager` |
| `spring-web` | 789 | HTTP、REST Client、Web 基础设施、数据绑定、过滤器 | `org.springframework.web.*` |
| `spring-webmvc` | 363 | Servlet MVC：HandlerMapping、HandlerAdapter、ViewResolver、异常处理 | `DispatcherServlet.doDispatch()` |
| `spring-webflux` | 275 | Reactive Web：DispatcherHandler、HandlerResultHandler、WebHandler 链 | `DispatcherHandler.handle()` |
| `spring-test` | 460 | TestContext、MockMvc、WebTestClient、上下文缓存 | `org.springframework.test.context.*` |
| `spring-expression` | 123 | SpEL 解析、AST、求值上下文、类型转换 | `SpelExpressionParser` |
| `spring-messaging` | 245 | Message、Channel、STOMP、注解消息处理 | `org.springframework.messaging.*` |
| `spring-jms` | 122 | JMS 模板、监听容器、消息转换 | `JmsTemplate` |
| `spring-r2dbc` | 65 | Reactive 数据库访问与事务 | `DatabaseClient` |
| `spring-orm` | 65 | JPA/Hibernate 集成、事务适配 | `JpaTransactionManager` |
| `spring-oxm` | 28 | XML 编组/解组抽象 | `Marshaller`、`Unmarshaller` |
| `spring-websocket` | 170 | WebSocket/STOMP 支持 | `org.springframework.web.socket.*` |
| `spring-context-support` | 87 | 缓存、邮件、调度、模板等上下文扩展 | `org.springframework.cache.*` |
| `spring-context-indexer` | 13 | 编译期组件索引，加速 classpath scanning | `CandidateComponentsIndexer` |
| `spring-aspects` | 14 | AspectJ 形式的 Spring 切面 | `org.springframework.beans.factory.aspectj.*` |
| `spring-instrument` | 2 | 类加载/织入 instrumentation 支持 | `InstrumentationSavingAgent` |
| `spring-core-test` | 45 | Spring 内部测试支持 | `org.springframework.core.testfixture.*` |

## 核心观点

1. **Spring Core 是元数据底座**：资源、泛型、注解、环境、类型转换不是边角工具，而是配置类解析、依赖注入、Web 参数绑定和测试上下文的基础。
2. **IoC 是中心，不是模块之一**：`spring-core` 提供底座，`spring-beans` 形成最小容器，`spring-context` 通过 `refresh()` 把容器推进到应用级运行态，其它模块大多通过 Bean、后处理器或策略 Bean 接入。
3. **注解配置是两阶段编译**：`ConfigurationClassPostProcessor` 先把 `@Configuration/@Bean/@ComponentScan/@Import` 编译成 BeanDefinition，`DefaultListableBeanFactory.resolveDependency()` 再把注入点编译成对象引用。
4. **Spring 的扩展点分两类**：一类发生在容器启动期，例如 `BeanFactoryPostProcessor` 改 BeanDefinition、`BeanPostProcessor` 改 Bean 实例；另一类发生在请求/调用运行期，例如 AOP `MethodInterceptor`、MVC `HandlerInterceptor`、事务 `TransactionInterceptor`。
5. **策略发现是跨模块复用模式**：`AbstractApplicationContext` 找特殊 Bean，`DispatcherServlet` 找 `HandlerMapping/HandlerAdapter/HandlerExceptionResolver`，`DispatcherHandler` 找 reactive 版本的 mapping/adapter/result handler。Spring 不把流程写死，而是把“流程骨架 + 策略列表”固定下来。
6. **AOP 是事务等横切能力的通道**：声明式事务不是事务模块单独魔法，而是 `@EnableTransactionManagement` 注册 advisor/interceptor，再由 AOP 代理把 `TransactionInterceptor.invoke()` 包在目标方法外。
7. **数据访问的核心是资源上下文**：`JdbcTemplate` 管理 JDBC 样板代码，`TransactionSynchronizationManager` 把 Connection/Session 等资源绑定到当前执行上下文，让 JDBC/JPA/R2DBC/JMS 能参与统一事务。
8. **MVC、WebFlux、Messaging、Test 都是应用层策略框架**：它们把 HTTP、Reactive、消息、测试生命周期转换成上下文对象 + 策略列表 + 可解析参数的方法调用。
9. **MVC 与 WebFlux 同构但执行模型不同**：MVC 的 `DispatcherServlet.doDispatch()` 是同步 Servlet 流程，WebFlux 的 `DispatcherHandler.handle()` 是 `Mono<Void>` 管线；两者都遵循 mapping → adapter → result/exception handling。
10. **模块之间存在清晰依赖方向**：上层模块依赖下层抽象，例如 `spring-webmvc` 依赖 `spring-web/context/beans/core`，但 `spring-core` 不反向知道 Web；这让 Spring 能作为其它 Spring 项目的根基。
11. **设计模式是 Spring 扩展性的语言**：工厂负责接管对象创建，代理负责横切增强，适配器负责统一 handler 调用，模板方法和回调负责资源生命周期，策略和责任链负责多扩展点协作，观察者负责事件解耦。
12. **复杂链路要按源码对象拆解**：注解合并看 `MergedAnnotations/AnnotationTypeMapping`，配置解析看 `ConfigurationClassParser`，AOP 时序看 `ReflectiveMethodInvocation`，WebFlux 看 `DispatcherHandler/HandlerResultHandler`，测试缓存看 `MergedContextConfiguration`。

## 关键源码入口

### 1. 容器刷新：`AbstractApplicationContext.refresh()`

源码路径：`raw/code/spring-framework/spring-context/src/main/java/org/springframework/context/support/AbstractApplicationContext.java`

`refresh()` 是 ApplicationContext 的总控模板方法，关键阶段为：

1. `prepareRefresh()`：设置 active、startupDate、初始化 property sources。
2. `obtainFreshBeanFactory()`：创建或刷新底层 BeanFactory。
3. `prepareBeanFactory(beanFactory)`：注册 ClassLoader、表达式解析器、Aware 处理器等基础设施。
4. `postProcessBeanFactory(beanFactory)`：留给子类扩展。
5. `invokeBeanFactoryPostProcessors(beanFactory)`：让配置类解析、扫描、属性占位符等在 Bean 实例化前修改 BeanDefinition。
6. `registerBeanPostProcessors(beanFactory)`：注册能拦截 Bean 创建的后处理器。
7. `initMessageSource()`、`initApplicationEventMulticaster()`、`registerListeners()`：装配应用级上下文能力。
8. `finishBeanFactoryInitialization(beanFactory)`：实例化剩余非懒加载单例。
9. `finishRefresh()`：发布刷新完成事件，启动生命周期处理器。

对应概念页：[[10-domains/java/spring-framework/概念_Spring_Framework_IoC容器启动与Bean生命周期]] — 从 `refresh()` 进入 Bean 创建细节。

### 2. Bean 创建：`AbstractAutowireCapableBeanFactory.doCreateBean()`

源码路径：`raw/code/spring-framework/spring-beans/src/main/java/org/springframework/beans/factory/support/AbstractAutowireCapableBeanFactory.java`

`doCreateBean()` 的本质是把一个 `RootBeanDefinition` 变成最终可暴露的 Bean：

1. `createBeanInstance()`：构造器解析、工厂方法或 Supplier 实例化。
2. `applyMergedBeanDefinitionPostProcessors()`：例如注入元数据预处理。
3. `addSingletonFactory()`：三级缓存提前暴露单例，用于解决部分循环依赖。
4. `populateBean()`：属性填充与自动注入。
5. `initializeBean()`：Aware 回调、初始化前后 BeanPostProcessor、init 方法。
6. 循环依赖一致性检查：如果早期注入的是 raw bean，而最终变成代理，可能抛异常。
7. `registerDisposableBeanIfNecessary()`：注册销毁逻辑。

### 3. AOP 代理选择：`DefaultAopProxyFactory.createAopProxy()`

源码路径：`raw/code/spring-framework/spring-aop/src/main/java/org/springframework/aop/framework/DefaultAopProxyFactory.java`

核心判断：

- 如果开启 optimize、proxyTargetClass，或没有用户提供接口，则尝试基于目标类代理。
- 如果目标类为空、目标类本身是接口、已经是 JDK Proxy、或是 lambda class，则使用 `JdkDynamicAopProxy`。
- 否则使用 `ObjenesisCglibAopProxy`。
- 如果没有强制类代理，默认使用 JDK 动态代理。

对应概念页：[[10-domains/java/spring-framework/概念_Spring_Framework_AOP事务与Web分发]] — AOP 代理如何承载事务、缓存等横切逻辑。

### 4. 声明式事务：`TransactionAspectSupport.invokeWithinTransaction()`

源码路径：`raw/code/spring-framework/spring-tx/src/main/java/org/springframework/transaction/interceptor/TransactionAspectSupport.java`

事务拦截器的关键流程：

1. 通过 `TransactionAttributeSource` 读取方法级事务属性。
2. 选择 `TransactionManager`，区分响应式与传统事务。
3. 对传统事务，调用 `createTransactionIfNecessary()` 获取或创建事务。
4. 调用 `invocation.proceedWithInvocation()` 执行目标方法。
5. 异常时按 rollback rule 决定回滚或提交。
6. 返回值为 `Future` 或 Vavr `Try` 时，还会检查失败状态并标记 rollback-only。
7. 正常返回后 `commitTransactionAfterReturning()`。

### 5. MVC 分发：`DispatcherServlet.doDispatch()`

源码路径：`raw/code/spring-framework/spring-webmvc/src/main/java/org/springframework/web/servlet/DispatcherServlet.java`

`doDispatch()` 的主线：

1. `checkMultipart()` 预处理上传请求。
2. `getHandler()` 通过 `HandlerMapping` 定位 handler。
3. `mappedHandler.applyPreHandle()` 执行拦截器前置逻辑。
4. `getHandlerAdapter()` 选择能调用该 handler 的 adapter。
5. `ha.handle()` 调用控制器，得到 `ModelAndView` 或写出响应。
6. 异步请求启动则提前返回。
7. `applyDefaultViewName()` 与 `applyPostHandle()`。
8. `processDispatchResult()` 统一处理异常解析、视图渲染和 afterCompletion。

### 6. WebFlux 分发：`DispatcherHandler.handle()`

源码路径：`raw/code/spring-framework/spring-webflux/src/main/java/org/springframework/web/reactive/DispatcherHandler.java`

Reactive 分发使用 `Flux.fromIterable(handlerMappings)` 逐个查找 handler，找到后 `handleRequestWith(exchange, handler)`，最终由 `HandlerResultHandler` 处理返回值。它与 MVC 的策略角色对应，但把控制流交给 Reactor `Mono/Flux`。

### 7. Core 元数据底座：`ResolvableType` 与 `PathMatchingResourcePatternResolver`

源码路径：

- `raw/code/spring-framework/spring-core/src/main/java/org/springframework/core/ResolvableType.java`
- `raw/code/spring-framework/spring-core/src/main/java/org/springframework/core/io/support/PathMatchingResourcePatternResolver.java`

`ResolvableType` 封装 Java `Type`，让 Spring 可以解析字段、方法参数、父类、接口和集合中的泛型。`PathMatchingResourcePatternResolver` 处理 `classpath*:` 和 Ant-style pattern，支撑组件扫描和资源发现。

对应概念页：[[10-domains/java/spring-framework/概念_Spring_Core资源类型注解与转换底座]] — Spring Core 如何把 Java 元数据统一为框架语义对象。

### 8. 配置类解析与依赖解析：`ConfigurationClassPostProcessor` / `resolveDependency()`

源码路径：

- `raw/code/spring-framework/spring-context/src/main/java/org/springframework/context/annotation/ConfigurationClassPostProcessor.java`
- `raw/code/spring-framework/spring-beans/src/main/java/org/springframework/beans/factory/support/DefaultListableBeanFactory.java`

配置类解析发生在 BeanFactoryPostProcessor 阶段，把注解配置编译成 BeanDefinition。依赖解析发生在 Bean 创建阶段，按 `@Value`、名称、Qualifier、集合、类型、Primary、Priority 等规则找最终依赖。

对应概念页：[[10-domains/java/spring-framework/概念_Spring_Framework_配置类解析与依赖解析]] — 注解配置如何从类元数据变成对象图。

### 8.1 合并注解模型：`MergedAnnotations` / `AnnotationTypeMapping`

源码路径：

- `raw/code/spring-framework/spring-core/src/main/java/org/springframework/core/annotation/MergedAnnotations.java`
- `raw/code/spring-framework/spring-core/src/main/java/org/springframework/core/annotation/AnnotationTypeMapping.java`
- `raw/code/spring-framework/spring-core/src/main/java/org/springframework/core/annotation/AliasFor.java`

合并注解模型负责把直接注解、元注解、组合注解、继承层级和 `@AliasFor` 别名关系合并成统一查询语义。上层配置类解析、Web 映射、条件判断不直接依赖 JDK `getAnnotation()`，而是消费 Spring 编译后的注解视图。

对应概念页：[[10-domains/java/spring-framework/概念_MergedAnnotations合并注解模型]] — 源码级拆 `@AliasFor`、mirror set 与组合注解。

### 8.2 配置类解析细节：`ConfigurationClassParser`

源码路径：`raw/code/spring-framework/spring-context/src/main/java/org/springframework/context/annotation/ConfigurationClassParser.java`

`ConfigurationClassParser` 在 `doProcessConfigurationClass()` 中按顺序处理成员类、`@PropertySource`、`@ComponentScan`、`@Import`、`@ImportResource`、`@Bean` 方法。`@ComponentScan` 扫描出的配置类候选会递归解析；普通 `ImportSelector` 立即展开；`DeferredImportSelector` 先收集，待普通配置解析完成后按 order 和 group 统一处理。

对应概念页：[[10-domains/java/spring-framework/概念_ConfigurationClassParser配置类解析细节]] — 含 `@ComponentScan`、`@Import`、`DeferredImportSelector` 流程图。

### 8.3 AOP 责任链时序：`ReflectiveMethodInvocation` / `TransactionInterceptor`

源码路径：

- `raw/code/spring-framework/spring-aop/src/main/java/org/springframework/aop/framework/DefaultAdvisorChainFactory.java`
- `raw/code/spring-framework/spring-aop/src/main/java/org/springframework/aop/framework/ReflectiveMethodInvocation.java`
- `raw/code/spring-framework/spring-tx/src/main/java/org/springframework/transaction/interceptor/TransactionInterceptor.java`

代理方法调用时，`DefaultAdvisorChainFactory` 把 Advisor 筛选并适配为 `MethodInterceptor` 链，`ReflectiveMethodInvocation.proceed()` 逐个推进。`TransactionInterceptor` 是链上的 around interceptor，在 `invocation.proceed()` 前后开启、提交或回滚事务。

对应概念页：[[10-domains/java/spring-framework/概念_Spring_AOP责任链源码时序]] — 含 advisor chain 与事务拦截时序图。

### 8.4 WebFlux 响应式处理链：`DispatcherHandler`

源码路径：`raw/code/spring-framework/spring-webflux/src/main/java/org/springframework/web/reactive/DispatcherHandler.java`

WebFlux 中央分发器按 `HandlerMapping → HandlerAdapter → HandlerResultHandler` 执行。异常以 Reactor 信号形式通过 `DispatchExceptionHandler` 和 result 内置 exception handler 恢复；请求上下文通过 Reactor Context 传播，例如 `ServerWebExchangeContextFilter` 会把 exchange 写入 context。

对应概念页：[[10-domains/java/spring-framework/概念_Spring_WebFlux响应式处理链]] — 深挖 result handler、exception handler 与 Reactor Context。

### 8.5 Spring TestContext：缓存与事务测试

源码路径：

- `raw/code/spring-framework/spring-test/src/main/java/org/springframework/test/context/MergedContextConfiguration.java`
- `raw/code/spring-framework/spring-test/src/main/java/org/springframework/test/context/cache/DefaultCacheAwareContextLoaderDelegate.java`
- `raw/code/spring-framework/spring-test/src/main/java/org/springframework/test/context/transaction/TransactionalTestExecutionListener.java`

TestContext 使用静态 `DefaultContextCache` 复用上下文，以 `MergedContextConfiguration` 的配置类、locations、profiles、property sources、context customizers、parent、loader 等字段作为 cache key。事务测试由 `TransactionalTestExecutionListener` 在测试方法前后开启和结束事务。

对应概念页：[[10-domains/java/spring-framework/概念_Spring_TestContext缓存与事务测试]] — 拆 cache key、listener 顺序、MockMvc/WebTestClient。

### 9. 数据访问与事务同步：`JdbcTemplate` / `TransactionSynchronizationManager`

源码路径：

- `raw/code/spring-framework/spring-jdbc/src/main/java/org/springframework/jdbc/core/JdbcTemplate.java`
- `raw/code/spring-framework/spring-tx/src/main/java/org/springframework/transaction/support/TransactionSynchronizationManager.java`

`JdbcTemplate` 用模板方法管理 JDBC 固定流程，`TransactionSynchronizationManager` 用上下文绑定事务资源。数据访问模块的核心不是 SQL 拼接，而是统一资源生命周期、异常转换和事务参与方式。

对应概念页：[[10-domains/java/spring-framework/概念_Spring_Framework_数据访问事务同步与集成模块]] — JDBC/R2DBC/ORM/JMS 如何接入 Spring 事务和资源同步。

### 10. 应用层扩展：`RequestMappingHandlerAdapter` / `TestContextManager`

源码路径：

- `raw/code/spring-framework/spring-webmvc/src/main/java/org/springframework/web/servlet/mvc/method/annotation/RequestMappingHandlerAdapter.java`
- `raw/code/spring-framework/spring-test/src/main/java/org/springframework/test/context/TestContextManager.java`

`RequestMappingHandlerAdapter` 通过参数解析器、返回值处理器和消息转换器调用 Controller 方法；`TestContextManager` 通过 listener 编排测试生命周期与上下文缓存。Messaging 模块也复用类似的注解方法处理模型。

对应概念页：[[10-domains/java/spring-framework/概念_Spring_Framework_Web测试消息与应用层扩展]] — Web、Test、Messaging 如何复用策略链和方法调用模型。

### 11. 设计模式专题：Factory / Proxy / Adapter / Template / Strategy / Observer / Chain

关键源码路径：

- `raw/code/spring-framework/spring-beans/src/main/java/org/springframework/beans/factory/FactoryBean.java`
- `raw/code/spring-framework/spring-beans/src/main/java/org/springframework/beans/factory/support/DefaultSingletonBeanRegistry.java`
- `raw/code/spring-framework/spring-aop/src/main/java/org/springframework/aop/framework/AopProxyFactory.java`
- `raw/code/spring-framework/spring-webmvc/src/main/java/org/springframework/web/servlet/HandlerAdapter.java`
- `raw/code/spring-framework/spring-jdbc/src/main/java/org/springframework/jdbc/core/JdbcTemplate.java`
- `raw/code/spring-framework/spring-context/src/main/java/org/springframework/context/event/ApplicationEventMulticaster.java`

对应专题：[[10-domains/java/spring-framework/主题_Spring_Framework设计模式_综述]] — 以面试高频问题为导向，把设计模式绑定到源码类、解决问题、好处和代价。

## 重要概念

- [[10-domains/java/spring-framework/概念_Spring_Framework_模块体系与扩展点]] — 理解每个模块为什么存在、依赖谁、提供哪些扩展点。
- [[10-domains/java/spring-framework/概念_Spring_Core资源类型注解与转换底座]] — Core 元数据底座：资源、泛型、合并注解、配置源和类型转换。
- [[10-domains/java/spring-framework/概念_Spring_Framework_配置类解析与依赖解析]] — 注解配置编译成 BeanDefinition，注入点解析为对象引用。
- [[10-domains/java/spring-framework/概念_Spring_Framework_IoC容器启动与Bean生命周期]] — `refresh()` 与 `doCreateBean()` 是阅读 Spring 源码的第一条主线。
- [[10-domains/java/spring-framework/概念_Spring_Framework_AOP事务与Web分发]] — AOP、事务、MVC、WebFlux 共同展示 Spring 的“策略骨架 + 拦截链”思想。
- [[10-domains/java/spring-framework/概念_Spring_Framework_数据访问事务同步与集成模块]] — JdbcTemplate、事务同步、JDBC/R2DBC/ORM/JMS 集成。
- [[10-domains/java/spring-framework/概念_Spring_Framework_Web测试消息与应用层扩展]] — Web、Test、Messaging 的应用层策略模型。
- [[10-domains/java/spring-framework/主题_Spring_Framework设计模式_综述]] — Spring Framework 设计模式总览与面试表达。
- [[10-domains/java/spring-framework/概念_Spring_Framework_创建型设计模式]] — 工厂、FactoryBean、抽象工厂、Builder、单例注册表。
- [[10-domains/java/spring-framework/概念_Spring_Framework_结构型设计模式]] — 代理、适配器、组合、装饰器。
- [[10-domains/java/spring-framework/概念_Spring_Framework_行为型设计模式]] — 模板方法、策略、观察者、责任链、回调。
- [[10-domains/java/spring-framework/概念_Spring_Framework_面试高频设计模式问答]] — 高频问答与易错点。
- [[10-domains/java/spring-framework/主题_Spring_Framework源码学习_综述]] — 后续逐模块深度学习路线。

## 可复用表达

- Spring 的核心不是“自动装配”，而是 **用 core 元数据模型和容器生命周期，把对象创建、依赖关系、横切增强、外部资源和运行期策略连接到同一套扩展点上**。
- `refresh()` 负责把配置世界编译成运行世界，`doCreateBean()` 负责把 BeanDefinition 编译成对象实例，`DispatcherServlet` 负责把 HTTP 请求编译成 handler 调用。
- `ConfigurationClassPostProcessor` 负责把注解配置编译成 BeanDefinition，`resolveDependency()` 负责把注入点编译成对象引用。
- `TransactionSynchronizationManager` 负责让外部资源拥有当前调用上下文，`RequestMappingHandlerAdapter` 负责让 HTTP 请求变成可解析参数的方法调用。
- 设计模式在 Spring 中不是装饰性术语，而是扩展点设计的边界：谁创建、谁包装、谁适配、谁控制流程、谁接收事件。
- 理解 Spring 源码时，不要先追所有实现类；先抓住模板方法、策略接口、后处理器、缓存结构四类东西。

## 我的理解

Spring Framework 的源码有一个非常稳定的阅读顺序：

1. 先看 `spring-core` 的资源、类型、注解、环境和转换，因为后面所有模块都在用。
2. 再看配置类解析，弄清 `@Configuration/@Bean/@Import/@ComponentScan` 如何变成 BeanDefinition。
3. 再看 `spring-beans` 的 BeanDefinition、BeanFactory 和依赖解析，这是容器最小内核。
4. 接着看 `spring-context` 的 `refresh()`，它把 BeanFactory 升级为应用上下文。
5. 然后看 `spring-aop` 与 `spring-tx`，理解“容器如何在对象外面套行为”。
6. 再看 `spring-jdbc/r2dbc/orm/jms`，理解外部资源如何通过事务同步进入 Spring 上下文。
7. 最后看 `spring-webmvc` / `spring-webflux` / `spring-test` / `spring-messaging`，观察同一套扩展思想如何落到应用层。

这份源码最值得深挖的不是 API 用法，而是它如何把大量可变需求压缩成几个稳定抽象：BeanDefinition、BeanPostProcessor、Advisor、HandlerMapping、HandlerAdapter、TransactionManager。掌握这些抽象之后，再读具体模块会轻很多。

## 未解决问题

1. `MergedAnnotations` 的搜索策略、距离计算、`@AliasFor` 冲突处理还需要源码级展开。
2. `ConfigurationClassParser` 对 `DeferredImportSelector` 的分组排序还需要单独深挖。
3. `AutowiredAnnotationBeanPostProcessor`、`CommonAnnotationBeanPostProcessor`、`InitDestroyAnnotationBeanPostProcessor` 的协作需要源码级流程图。
4. WebFlux 的异常处理、返回值处理和函数式路由还没有展开到源码级别。
5. TestContext 的 cache key 构造、事务测试回滚、MockMvc/WebTestClient 需要独立源码页继续展开。

## 相关链接

- [[10-domains/java/spring-framework/概念_Spring_Framework_模块体系与扩展点]] — 模块依赖和扩展点地图。
- [[10-domains/java/spring-framework/概念_Spring_Core资源类型注解与转换底座]] — Spring Core 元数据底座。
- [[10-domains/java/spring-framework/概念_Spring_Framework_配置类解析与依赖解析]] — 注解配置与依赖注入算法。
- [[10-domains/java/spring-framework/概念_Spring_Framework_IoC容器启动与Bean生命周期]] — 容器启动主线。
- [[10-domains/java/spring-framework/概念_Spring_Framework_AOP事务与Web分发]] — 横切与 Web 分发主线。
- [[10-domains/java/spring-framework/概念_Spring_Framework_数据访问事务同步与集成模块]] — 数据访问、事务同步和资源绑定。
- [[10-domains/java/spring-framework/概念_Spring_Framework_Web测试消息与应用层扩展]] — Web、测试、消息的应用层策略模型。
- [[10-domains/java/spring-framework/主题_Spring_Framework源码学习_综述]] — 逐模块深度学习路线。
