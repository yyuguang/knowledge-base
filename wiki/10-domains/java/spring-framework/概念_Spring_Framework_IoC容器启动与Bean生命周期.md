---
type: "concept"
tags:
  - java
  - spring-framework
  - ioc
  - bean-lifecycle
summary: "Spring Framework IoC 容器启动与 Bean 生命周期：以 AbstractApplicationContext.refresh() 和 AbstractAutowireCapableBeanFactory.doCreateBean() 为主线，理解 BeanDefinition 如何变成应用运行时对象图。"
sources:
  - "[[30-sources/repositories/来源_Spring_Framework_源码]]"
  - "raw/code/spring-framework/spring-context/src/main/java/org/springframework/context/support/AbstractApplicationContext.java"
  - "raw/code/spring-framework/spring-beans/src/main/java/org/springframework/beans/factory/support/AbstractAutowireCapableBeanFactory.java"
aliases:
  - "Spring IoC 容器启动"
  - "Spring Bean 生命周期"
  - "ApplicationContext refresh"
status: "evolving"
confidence: 0.9
created: "2026-05-12 20:35:53"
updated: "2026-05-12 20:35:53"
---

# 概念：Spring Framework IoC 容器启动与 Bean 生命周期

## 定义

Spring IoC 容器启动可以分成两层：

- **容器级生命周期**：`AbstractApplicationContext.refresh()` 负责准备环境、创建 BeanFactory、执行后处理器、注册监听器、实例化单例、发布完成事件。
- **Bean 级生命周期**：`AbstractAutowireCapableBeanFactory.doCreateBean()` 负责实例化、依赖注入、初始化、AOP 代理暴露、销毁注册。

一句话：`refresh()` 把配置编译成运行时容器，`doCreateBean()` 把 BeanDefinition 编译成对象实例。

## 为什么重要

绝大多数 Spring 问题都能落到这两条主线上：

- Bean 为什么没注册？看 BeanDefinition 加载和 `BeanFactoryPostProcessor`。
- Bean 为什么注入失败？看依赖解析、构造器选择、`populateBean()`。
- `@Autowired`、`@PostConstruct`、`@Configuration` 为什么生效？看 BeanPostProcessor 与配置类后处理器。
- 事务为什么不生效？看 Bean 初始化后是否被 AOP 代理、调用是否经过代理。
- 循环依赖为什么有时能解决、有时不能？看三级缓存和早期代理引用。

## 使用场景

- Debug Spring Boot 启动异常。
- 自定义框架 Starter，注册 BeanDefinition 或后处理器。
- 理解 `@Configuration`、`@Bean`、`@ComponentScan`、`@Autowired` 的执行时机。
- 排查循环依赖、AOP 代理、初始化顺序、销毁回调问题。

## 核心机制

### 1. `refresh()` 是容器总控模板方法

`AbstractApplicationContext.refresh()` 的源码阶段可以编译成以下流程：

| 阶段 | 方法 | 作用 | 关键影响 |
|---|---|---|---|
| 1 | `prepareRefresh()` | 设置 active/startupDate，准备 environment | 容器开始进入刷新态 |
| 2 | `obtainFreshBeanFactory()` | 创建或刷新 BeanFactory | 得到底层 BeanDefinition 容器 |
| 3 | `prepareBeanFactory()` | 注册 ClassLoader、表达式解析器、Aware 处理器 | 为 Bean 创建准备基础设施 |
| 4 | `postProcessBeanFactory()` | 子类扩展点 | Web/特殊上下文可定制 |
| 5 | `invokeBeanFactoryPostProcessors()` | 执行 BFPP/BDRPP | 配置类解析、扫描、属性占位符等发生在这里 |
| 6 | `registerBeanPostProcessors()` | 注册 BPP | 后续 Bean 创建会被拦截 |
| 7 | `initMessageSource()` | 国际化消息源 | 应用级能力 |
| 8 | `initApplicationEventMulticaster()` | 事件广播器 | 事件发布能力 |
| 9 | `onRefresh()` | 子类特殊初始化 | Web server 等可在子类处理 |
| 10 | `registerListeners()` | 注册监听器 | ApplicationListener 生效 |
| 11 | `finishBeanFactoryInitialization()` | 实例化非懒加载单例 | 大量业务 Bean 真正创建 |
| 12 | `finishRefresh()` | 发布 ContextRefreshedEvent，启动 Lifecycle | 容器进入完成态 |

### 2. BeanDefinition 是“对象的编译前形态”

BeanDefinition 不是 Bean，而是描述 Bean 的元数据：

- beanClass / factoryMethodName
- scope
- constructor arguments
- property values
- init/destroy methods
- lazyInit
- dependsOn
- autowire mode
- role / source

`BeanFactoryPostProcessor` 操作的是 BeanDefinition，因此它发生在对象创建之前。`BeanPostProcessor` 操作的是 Bean 实例，因此它发生在对象创建期间。

### 3. `doCreateBean()` 是 Bean 创建主线

核心步骤：

```text
RootBeanDefinition
  → createBeanInstance()
  → applyMergedBeanDefinitionPostProcessors()
  → addSingletonFactory()        # 需要时提前暴露早期引用
  → populateBean()
  → initializeBean()
  → circular reference consistency check
  → registerDisposableBeanIfNecessary()
  → exposedObject
```

其中最值得深挖的是三处：

1. `createBeanInstance()`：构造器选择、工厂方法解析、Supplier 实例化。
2. `populateBean()`：属性注入、`@Autowired` 依赖解析、类型转换。
3. `initializeBean()`：Aware、初始化回调、BeanPostProcessor 前后置处理，AOP 代理通常在这里产生。

### 4. 循环依赖与三级缓存

Spring 对单例 Bean 的循环依赖处理依赖三个层次的缓存：

| 缓存 | 含义 | 典型内容 |
|---|---|---|
| `singletonObjects` | 一级缓存，完整单例 | 已初始化完成的 Bean |
| `earlySingletonObjects` | 二级缓存，早期单例引用 | 尚未初始化完成但可被依赖注入的引用 |
| `singletonFactories` | 三级缓存，早期引用工厂 | 可生成早期引用，AOP 可在这里提前暴露代理 |

`doCreateBean()` 中的 `addSingletonFactory(beanName, () -> getEarlyBeanReference(...))` 是关键点：它不是直接把 raw bean 塞出去，而是给后处理器机会返回早期代理引用。

### 5. BeanPostProcessor 是 Bean 生命周期的切面

常见处理器：

- `AutowiredAnnotationBeanPostProcessor`：处理 `@Autowired`、`@Value`。
- `CommonAnnotationBeanPostProcessor`：处理 JSR-250 注解。
- `InitDestroyAnnotationBeanPostProcessor`：处理初始化/销毁注解。
- AOP auto-proxy creator：在初始化后返回代理对象。
- `ApplicationContextAwareProcessor`：处理各种 Aware 回调。

这些处理器的共同点是：它们不是业务 Bean，却能影响所有业务 Bean 的创建。

## 例子

### 一个 Bean 从注解到实例的大致路径

```text
@Component class UserService
  ↓ classpath scanning
ScannedGenericBeanDefinition
  ↓ ConfigurationClassPostProcessor / registry
DefaultListableBeanFactory.beanDefinitionMap
  ↓ finishBeanFactoryInitialization()
getBean("userService")
  ↓ doCreateBean()
constructor → dependency injection → init callbacks → proxy wrapping
  ↓
singletonObjects["userService"]
```

### `@Autowired` 不是构造器魔法

`@Autowired` 能工作，是因为容器注册了相关 `BeanPostProcessor`。它在 Bean 创建过程中读取注入元数据，并通过 BeanFactory 的依赖解析能力找到候选 Bean。

因此，如果一个对象不是由 Spring 创建的，`@Autowired` 默认不会生效；它没有进入 `doCreateBean()` 流程。

## 常见误区

1. **误区：BeanFactoryPostProcessor 和 BeanPostProcessor 都是在 Bean 创建后执行。**  
   前者操作 BeanDefinition，发生在实例化前；后者操作 Bean 实例，发生在实例化/初始化期间。

2. **误区：循环依赖总能解决。**  
   Spring 主要解决单例 setter/field 注入循环依赖；构造器循环依赖、prototype 循环依赖、代理不一致场景都可能失败。

3. **误区：`refresh()` 只是启动方法。**  
   它是 Spring 应用上下文的启动协议，定义了所有扩展点的相对顺序。

4. **误区：AOP 代理是 Bean 创建完成后随便包一层。**  
   它必须嵌入 Bean 生命周期，并和早期引用、依赖注入、类型预测协作。

## 和其它概念的关系

- [[10-domains/java/spring-framework/概念_Spring_Framework_模块体系与扩展点]] — IoC 生命周期是各模块接入 Spring 的中心坐标。
- [[10-domains/java/spring-framework/概念_Spring_Framework_AOP事务与Web分发]] — AOP 代理通常由 BeanPostProcessor 在 Bean 初始化阶段创建。
- [[30-sources/repositories/来源_Spring_Framework_源码]] — 本页关键源码入口来自 source page。

## 我的理解

Spring IoC 的本质不是简单的对象容器，而是一个“对象图编译器”：

1. 输入是配置类、注解、XML、扫描结果等元数据。
2. 中间产物是 BeanDefinition。
3. 编译期优化和改写由 BeanFactoryPostProcessor 完成。
4. 运行期实例由 BeanFactory 创建。
5. 横切增强由 BeanPostProcessor/AOP 在实例创建过程中嵌入。
6. 最终输出是一个带事件、资源、生命周期、国际化和扩展能力的应用上下文。

理解这条编译链，比背诵 Bean 生命周期步骤更重要。

## 来源

- `raw/code/spring-framework/spring-context/src/main/java/org/springframework/context/support/AbstractApplicationContext.java`
- `raw/code/spring-framework/spring-beans/src/main/java/org/springframework/beans/factory/support/AbstractAutowireCapableBeanFactory.java`
- [[30-sources/repositories/来源_Spring_Framework_源码]] — Spring Framework 源码入口。

