---
type: "concept"
tags:
  - java
  - spring
  - application-event
  - factory-bean
  - bean-post-processor
  - spi
summary: "Spring 扩展进阶：事件监听机制用 ApplicationEventPublisher/EventMulticaster/ApplicationListener 解耦发布和订阅；FactoryBean 封装复杂对象创建；BeanFactoryPostProcessor 改 BeanDefinition，BeanPostProcessor 改 Bean 实例；SpringFactoriesLoader 从 spring.factories 加载 SPI 扩展。"
sources:
  - "raw/code/spring-framework/spring-context/src/main/java/org/springframework/context/event/ApplicationEventMulticaster.java"
  - "raw/code/spring-framework/spring-beans/src/main/java/org/springframework/beans/factory/FactoryBean.java"
  - "raw/code/spring-framework/spring-beans/src/main/java/org/springframework/beans/factory/config/BeanPostProcessor.java"
  - "raw/code/spring-framework/spring-core/src/main/java/org/springframework/core/io/support/SpringFactoriesLoader.java"
aliases:
  - "Spring 扩展进阶"
  - "ApplicationEvent FactoryBean BeanPostProcessor SpringFactories"
status: "evolving"
confidence: 0.86
created: "2026-05-12 21:22:48"
updated: "2026-05-12 21:22:48"
---

# 概念：Spring扩展进阶 事件 FactoryBean 后置处理器 SPI

## 61_事件监听机制（ApplicationEvent）

核心角色：

| 角色 | 作用 |
|---|---|
| `ApplicationEventPublisher` | 发布事件 |
| `ApplicationEvent` | 事件对象 |
| `ApplicationListener` | 监听事件 |
| `ApplicationEventMulticaster` | 广播事件给匹配监听器 |
| `@EventListener` | 注解式监听方式 |

流程：

```text
publishEvent(event)
  → ApplicationEventMulticaster
  → 找到支持该事件类型的 listener
  → 同步或异步调用 listener
```

好处：

- 发布方不依赖订阅方。
- 可以增加多个监听器。
- 适合业务解耦、领域事件、生命周期扩展。

## 62_工厂Bean（FactoryBean）

`FactoryBean` 是“生产另一个对象的 Bean”。

关键方法：

- `getObject()`：返回产品对象。
- `getObjectType()`：返回产品类型。
- `isSingleton()`：产品是否单例。

规则：

- `getBean("x")` 返回产品对象。
- `getBean("&x")` 返回 FactoryBean 本身。

适用场景：

- AOP 代理对象。
- JNDI 对象。
- MyBatis Mapper 代理。
- 复杂第三方客户端。

## 63_后置处理器（BeanPostProcessor / BeanFactoryPostProcessor）

两类后处理器要分清：

| 类型 | 处理对象 | 执行时机 | 例子 |
|---|---|---|---|
| `BeanFactoryPostProcessor` | BeanDefinition / BeanFactory | Bean 实例化前 | `PropertySourcesPlaceholderConfigurer`、`ConfigurationClassPostProcessor` |
| `BeanPostProcessor` | Bean 实例 | Bean 初始化前后 | `AutowiredAnnotationBeanPostProcessor`、AOP auto-proxy creator |

面试关键：

- BFPP 不能依赖普通 Bean 已经创建。
- BPP 可以改变返回 Bean，例如返回代理对象。
- AOP、`@Autowired`、`@PostConstruct` 等都依赖后处理器机制。

## 64_SPI与Spring Factories

`SpringFactoriesLoader` 从 classpath 下的：

```text
META-INF/spring.factories
```

加载指定接口对应的实现类。

用途：

- 框架内部扩展发现。
- 早期 Spring Boot 自动配置入口。
- AOT hints 等扩展发现。

与 JDK SPI 对比：

| 对比 | JDK SPI | Spring Factories |
|---|---|---|
| 文件位置 | `META-INF/services/<interface>` | `META-INF/spring.factories` |
| 格式 | 一个接口一个文件 | properties key-value |
| 加载方式 | `ServiceLoader` | `SpringFactoriesLoader` |
| Spring 集成 | 普通 | 更适合 Spring 扩展生态 |

现代 Boot 自动配置更多使用 `AutoConfiguration.imports`，但 `spring.factories` 在 Spring Framework 和部分生态中仍然重要。

## 面试表达

> Spring 扩展点分启动期和运行期。事件机制用于运行期解耦；FactoryBean 用于封装复杂对象创建；BeanFactoryPostProcessor 在 Bean 创建前修改定义，BeanPostProcessor 在 Bean 初始化前后处理实例甚至返回代理；SpringFactoriesLoader 则提供 classpath 级 SPI 扩展发现。

## 常见误区

- FactoryBean 不是 BeanFactory。
- BeanFactoryPostProcessor 和 BeanPostProcessor 的处理对象不同。
- `@Autowired` 不是 Java 原生能力，而是后处理器处理的。
- `spring.factories` 和 Boot 现代 `AutoConfiguration.imports` 不是同一个文件。

## 相关链接

- [[concepts/java/spring-framework/概念_Spring_Framework_创建型设计模式]] — FactoryBean 和 Singleton Registry。
- [[concepts/java/spring-framework/概念_Spring_Framework_行为型设计模式]] — 观察者、回调、责任链。
- [[overview/java/spring/主题_Spring知识体系_综述]] — 返回大纲。

