---
type: "concept"
tags:
  - java
  - spring-framework
  - configuration
  - dependency-injection
summary: "Spring 配置类解析与依赖解析：ConfigurationClassPostProcessor 把 @Configuration/@Bean/@ComponentScan/@Import 编译成 BeanDefinition，DefaultListableBeanFactory.resolveDependency 再按 @Value、名称、类型、集合、@Primary/@Qualifier 等规则解析注入点。"
sources:
  - "raw/code/spring-framework/spring-context/src/main/java/org/springframework/context/annotation/ConfigurationClassPostProcessor.java"
  - "raw/code/spring-framework/spring-context/src/main/java/org/springframework/context/annotation/ConfigurationClassParser.java"
  - "raw/code/spring-framework/spring-beans/src/main/java/org/springframework/beans/factory/support/DefaultListableBeanFactory.java"
  - "raw/code/spring-framework/spring-beans/src/main/java/org/springframework/beans/factory/annotation/AutowiredAnnotationBeanPostProcessor.java"
aliases:
  - "Spring 配置类解析"
  - "Spring 依赖解析"
  - "ConfigurationClassPostProcessor"
  - "resolveDependency"
status: "evolving"
confidence: 0.87
created: "2026-05-12 21:10:00"
updated: "2026-05-12 21:10:00"
---

# 概念：Spring Framework 配置类解析与依赖解析

## 定义

Spring 现代注解配置可以拆成两段编译过程：

1. **配置类解析**：`ConfigurationClassPostProcessor` 在 Bean 实例化前执行，把 `@Configuration`、`@ComponentScan`、`@Import`、`@Bean`、`@PropertySource` 等元数据解析成 BeanDefinition。
2. **依赖解析**：`DefaultListableBeanFactory.resolveDependency()` 在 Bean 创建期间执行，把字段、构造器参数、方法参数上的注入点解析为具体 Bean、集合、Map、Provider、Optional 或配置值。

一句话：配置类解析负责“Bean 从哪里来”，依赖解析负责“Bean 之间怎么连起来”。

## 为什么重要

第一轮只看 `refresh()` 和 `doCreateBean()` 还不够，因为真正的 Spring 应用通常不是手写 BeanDefinition，而是通过注解配置生成 BeanDefinition。

理解本页可以解释：

- `@Configuration` 为什么能拦截同类内部 `@Bean` 方法调用。
- `@ComponentScan` 什么时候扫描。
- `@Import`、`ImportSelector`、`DeferredImportSelector` 为什么能注册额外配置。
- `@Autowired` 为什么能注入 `List<T>`、`Map<String,T>`、`ObjectProvider<T>`。
- 多个同类型 Bean 时，`@Primary`、`@Qualifier`、参数名如何参与决策。

## 使用场景

- 排查 `@Bean` 没注册、配置类没生效、Import 顺序异常。
- 排查 `NoUniqueBeanDefinitionException` 或 `NoSuchBeanDefinitionException`。
- 编写复杂 Starter：用 `ImportSelector`、`BeanDefinitionRegistryPostProcessor` 注册基础设施。
- 设计插件式架构：按类型收集所有策略 Bean，并通过 `@Order` 排序。

## 核心机制

### 1. `ConfigurationClassPostProcessor` 的位置

它实现 `BeanDefinitionRegistryPostProcessor`，因此发生在普通 Bean 实例化之前：

```text
refresh()
  → invokeBeanFactoryPostProcessors()
      → ConfigurationClassPostProcessor.postProcessBeanDefinitionRegistry()
          → 发现配置类候选
          → ConfigurationClassParser 解析
          → ConfigurationClassBeanDefinitionReader 注册 BeanDefinition
```

这就是为什么 `@Bean` 方法定义的 Bean 能参与后续依赖注入：它们在实例化前已经变成 BeanDefinition。

### 2. 配置类 full / lite 模式

Spring 会区分：

- **full configuration**：典型 `@Configuration`，可能被 CGLIB 增强，保证 `@Bean` 方法互相调用时仍从容器取单例。
- **lite configuration**：普通 `@Component` 内声明 `@Bean`，不一定做完整增强。

源码里 `ConfigurationClassPostProcessor` 会收集 full 配置类，并用 `ConfigurationClassEnhancer` 替换 BeanDefinition 的 beanClass。增强后配置类内部调用 `@Bean` 方法时，会通过 BeanFactory 保证容器语义。

### 3. 配置类解析输入

`ConfigurationClassParser` 需要处理：

| 注解/机制 | 结果 |
|---|---|
| `@PropertySource` | 添加 property source |
| `@ComponentScan` | 扫描 classpath，注册候选 BeanDefinition |
| `@Import` | 导入配置类、ImportSelector、ImportBeanDefinitionRegistrar |
| `@Bean` | 把方法转换为 BeanDefinition |
| 条件注解 | 通过 ConditionEvaluator 判断是否跳过 |
| 父类/接口/default method | 递归解析配置来源 |

### 4. `@Autowired` 的执行位置

`AutowiredAnnotationBeanPostProcessor` 是 `SmartInstantiationAwareBeanPostProcessor`。它不负责全局扫描配置类，而是在 Bean 创建过程中处理注入元数据：

```text
doCreateBean()
  → populateBean()
      → AutowiredAnnotationBeanPostProcessor.postProcessProperties()
          → findAutowiringMetadata()
          → InjectionMetadata.inject()
          → beanFactory.resolveDependency()
```

### 5. `resolveDependency()` 六步

`DefaultListableBeanFactory.doResolveDependency()` 的源码步骤非常清晰：

| 步骤 | 作用 | 例子 |
|---|---|---|
| Step 1 | shortcut | 预解析快捷路径 |
| Step 2 | suggested value | `@Value("${x}")`、SpEL |
| Step 3 | dependency name / qualifier name | 参数名、字段名、`@Qualifier` 建议名称 |
| Step 4a | multiple beans | `List<T>`、`T[]`、`Stream<T>`、`Map<String,T>` |
| Step 4b | direct bean matches | 按类型找候选 Bean |
| Step 4c | fallback collection/map | 兜底集合匹配 |
| Step 5 | determine single candidate | `@Primary`、优先级、名称等 |
| Step 6 | validate result | 类型校验、NullBean 处理 |

这说明 `@Autowired` 不是简单 “by type”。它是 value、name、qualifier、type、collection、primary、priority、fallback 的组合算法。

## 例子

### 多个同类型 Bean 的决策顺序

```java
interface SmsClient {}

@Bean
@Primary
SmsClient aliyunSmsClient() { ... }

@Bean
SmsClient tencentSmsClient() { ... }

@Autowired
SmsClient smsClient;
```

`resolveDependency()` 会先找到两个候选，再进入单候选决策。`@Primary` 让 `aliyunSmsClient` 胜出。如果没有 `@Primary`，字段名 `smsClient` 也不匹配任何 beanName，就可能报不唯一。

### 集合注入是策略扩展的基础

```java
@Autowired
List<PaymentHandler> handlers;
```

Spring 会找到所有 `PaymentHandler` 类型 Bean，转换成 List，并按依赖比较器排序。这就是很多框架扩展点天然支持“注册多个策略 Bean”的原因。

## 常见误区

1. **误区：`@Configuration` 只是一个特殊 `@Component`。**  
   full 模式下它会被增强，以保证 `@Bean` 方法调用符合容器单例语义。

2. **误区：`@Autowired` 就是按类型找 Bean。**  
   真正算法包含 `@Value`、Optional、ObjectProvider、延迟代理、名称、Qualifier、集合、Primary、Priority、Fallback。

3. **误区：配置类解析发生在 Bean 创建时。**  
   配置类解析是 BeanFactoryPostProcessor 阶段，早于普通 Bean 实例化。

4. **误区：`List<T>` 注入只是把已有 Bean 随机塞进去。**  
   它会按类型、泛型和排序规则处理，顺序不是偶然。

## 和其它概念的关系

- [[concepts/java/spring-framework/概念_Spring_Core资源类型注解与转换底座]] — 配置类扫描和依赖解析依赖 Resource、MergedAnnotations、ResolvableType。
- [[concepts/java/spring-framework/概念_Spring_Framework_IoC容器启动与Bean生命周期]] — 配置解析发生在 `refresh()` 的后处理器阶段，依赖注入发生在 `doCreateBean()` 的属性填充阶段。
- [[concepts/java/spring-framework/概念_Spring_Framework_模块体系与扩展点]] — 配置类解析和依赖解析是 Spring 扩展点体系的中枢。

## 我的理解

Spring 注解配置的本质不是“扫描注解然后反射创建对象”，而是一个两阶段编译器：

1. 配置阶段把类、方法、Import、扫描结果编译为 BeanDefinition。
2. 创建阶段把注入点编译为对象引用、集合、Provider 或配置值。

所以排查 Spring 问题时，先问“目标 BeanDefinition 是否存在”，再问“注入点如何解析”。这比直接盯着 `@Autowired` 更有效。

## 来源

- `raw/code/spring-framework/spring-context/src/main/java/org/springframework/context/annotation/ConfigurationClassPostProcessor.java`
- `raw/code/spring-framework/spring-context/src/main/java/org/springframework/context/annotation/ConfigurationClassParser.java`
- `raw/code/spring-framework/spring-beans/src/main/java/org/springframework/beans/factory/support/DefaultListableBeanFactory.java`
- `raw/code/spring-framework/spring-beans/src/main/java/org/springframework/beans/factory/annotation/AutowiredAnnotationBeanPostProcessor.java`

