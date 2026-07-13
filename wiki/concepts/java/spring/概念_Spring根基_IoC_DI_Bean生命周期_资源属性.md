---
type: "concept"
tags:
  - java
  - spring
  - ioc
  - dependency-injection
  - bean-lifecycle
summary: "Spring 根基：IoC/DI 解决对象创建和依赖管理问题，Bean 生命周期描述从 BeanDefinition 到销毁的全过程，Resource/Environment/PropertySource/占位符/YAML 负责把外部资源和配置接入容器。"
sources:
  - "raw/code/spring-framework/spring-context/src/main/java/org/springframework/context/support/AbstractApplicationContext.java"
  - "raw/code/spring-framework/spring-beans/src/main/java/org/springframework/beans/factory/support/AbstractAutowireCapableBeanFactory.java"
  - "raw/code/spring-framework/spring-context/src/main/java/org/springframework/context/support/PropertySourcesPlaceholderConfigurer.java"
  - "raw/code/spring-framework/spring-beans/src/main/java/org/springframework/beans/factory/config/YamlPropertiesFactoryBean.java"
aliases:
  - "Spring 根基"
  - "IoC DI Bean 生命周期 资源属性"
status: "evolving"
confidence: 0.9
created: "2026-05-12 21:22:48"
updated: "2026-05-12 21:22:48"
---

# 概念：Spring根基 IoC/DI Bean生命周期 资源属性

## 00_IoC与DI思想（为什么需要容器）

IoC（控制反转）的核心不是“使用 XML 或注解”，而是把对象创建权、依赖装配权、生命周期管理权从业务代码转移到容器。

如果没有容器：

```java
class OrderService {
    private final PayService payService = new PayService(new PayClient(...));
}
```

问题：

- 依赖创建散落在业务代码里。
- 替换实现困难，例如 mock、灰度实现、多环境配置。
- 对象生命周期不可统一管理。
- 事务、缓存、事件、AOP 等横切能力很难插入。

有了容器：

```text
配置/注解
  → BeanDefinition
  → BeanFactory 创建对象
  → 依赖注入
  → 生命周期回调
  → 后置处理器/AOP
  → 应用使用 Bean
```

DI（依赖注入）是 IoC 的主要实现方式：对象不主动创建依赖，而是由容器把依赖注入进来。

## 01_Bean的生命周期（从定义到销毁全流程）

源码主线：

- `AbstractApplicationContext.refresh()`
- `AbstractAutowireCapableBeanFactory.doCreateBean()`

流程：

```text
BeanDefinition 注册
  → BeanFactoryPostProcessor 修改定义
  → createBeanInstance 实例化
  → applyMergedBeanDefinitionPostProcessors
  → 需要时 addSingletonFactory 提前暴露
  → populateBean 属性填充/依赖注入
  → Aware 回调
  → BeanPostProcessor before initialization
  → init method / InitializingBean / @PostConstruct
  → BeanPostProcessor after initialization
  → AOP 代理可能在此返回
  → registerDisposableBeanIfNecessary
  → destroy / DisposableBean / @PreDestroy
```

关键点：

- BeanDefinition 是对象的“编译前形态”。
- BeanFactoryPostProcessor 处理 BeanDefinition。
- BeanPostProcessor 处理 Bean 实例。
- AOP 代理通常在初始化后产生。
- 销毁只对容器管理的 Bean 生效。

## 02_资源与属性解析（Properties、YAML、占位符）

Spring 把外部资源和配置拆成几个层次：

| 层 | 关键类 | 作用 |
|---|---|---|
| 资源定位 | `Resource`、`ResourceLoader` | 统一 classpath、file、URL、JAR 资源 |
| 模式扫描 | `PathMatchingResourcePatternResolver` | 支持 `classpath*:` 与 Ant 风格路径 |
| 配置源 | `Environment`、`PropertySource` | 多配置源按优先级查询 |
| 占位符 | `PropertySourcesPlaceholderConfigurer`、`PropertyPlaceholderHelper` | 解析 `${...}` |
| YAML | `YamlPropertiesFactoryBean`、`YamlMapFactoryBean` | YAML 转 Properties/Map |
| 类型转换 | `ConversionService` | String 转目标类型 |

典型过程：

```text
application.yml / properties / env
  → Resource 定位
  → PropertySource 加入 Environment
  → ${server.port} 占位符解析
  → ConversionService 转成 int/Duration/List 等目标类型
  → 注入 @Value 或绑定属性对象
```

## 源码解释

`PropertySourcesPlaceholderConfigurer` 是 BeanFactoryPostProcessor 类型的扩展点，因此它在 Bean 实例化前处理 BeanDefinition 中的占位符。

YAML 在 Spring Framework 中由 `YamlProcessor` 系列支持，但 Spring Boot 的 `application.yml` 加载、Profile 文档拆分和配置绑定还需要 Spring Boot 源码才能完整解释。

## 面试表达

可以这样说：

> IoC 解决的是对象控制权问题，DI 是实现 IoC 的主要方式。Spring 先把配置或注解解析成 BeanDefinition，再由 BeanFactory 负责实例化、属性填充、初始化、后置处理和销毁。资源和属性解析则通过 Resource、Environment、PropertySource、PlaceholderConfigurer 和 ConversionService 把外部配置接入容器。

## 常见误区

- IoC 不等于依赖注入，DI 是 IoC 的实现方式之一。
- Bean 生命周期不只是 init/destroy，还包括 BeanDefinition、后处理器、属性填充、代理。
- `@Value` 的解析不是字段注入时才随便替换字符串，很多占位符处理发生在 BeanFactoryPostProcessor 阶段。
- Spring Framework 支持 YAML 工厂 Bean，但 Spring Boot 的配置加载体系更复杂，不能混为一谈。

## 相关链接

- [[concepts/java/spring-framework/概念_Spring_Framework_IoC容器启动与Bean生命周期]] — Framework 源码级 Bean 生命周期。
- [[concepts/java/spring-framework/概念_Spring_Core资源类型注解与转换底座]] — Resource、Environment、ConversionService。
- [[overview/java/spring/主题_Spring知识体系_综述]] — 返回大纲。

