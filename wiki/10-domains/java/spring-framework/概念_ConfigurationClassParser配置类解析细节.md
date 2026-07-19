---
type: "concept"
tags:
  - java
  - spring-framework
  - configuration
summary: "ConfigurationClassParser 是 Spring 注解配置编译器的核心：在 BeanFactoryPostProcessor 阶段解析 @ComponentScan、@Import、DeferredImportSelector、@Bean、@PropertySource、@ImportResource，把配置类语义转换为 BeanDefinition 注册素材。"
sources:
  - "[[30-sources/repositories/来源_Spring_Framework_源码]]"
  - "raw/code/spring-framework/spring-context/src/main/java/org/springframework/context/annotation/ConfigurationClassParser.java"
  - "raw/code/spring-framework/spring-context/src/main/java/org/springframework/context/annotation/ConfigurationClassPostProcessor.java"
aliases:
  - "ConfigurationClassParser"
  - "配置类解析"
  - "@Import"
  - "DeferredImportSelector"
status: "evolving"
confidence: 0.9
created: "2026-05-13 09:14:32"
updated: "2026-05-13 09:14:32"
---

# 概念：ConfigurationClassParser配置类解析细节

![[Attachments/spring-configuration-class-parser-flow.excalidraw]]

## 定义

`ConfigurationClassParser` 是 Spring 注解配置的“编译器前端”。它在 Bean 实例化之前运行，读取配置类上的注解元数据，把 `@ComponentScan`、`@Import`、`DeferredImportSelector`、`@Bean`、`@PropertySource`、`@ImportResource` 等声明式配置转换成后续可注册的 `ConfigurationClass` 模型。

它不是创建 Bean 的地方，而是把“配置类语法”编译为 BeanDefinition 注册素材。真正注册 `@Bean` 方法、import 进来的类、扫描出来的组件，后续由 `ConfigurationClassBeanDefinitionReader` 完成。

## 为什么重要

理解它可以解释很多高频问题：

- `@Configuration` 为什么会被提前处理？
- `@ComponentScan` 扫描出来的配置类为什么还能继续递归解析？
- 普通 `@Import` 和 `DeferredImportSelector` 顺序有什么区别？
- Spring Boot 自动配置为什么要用 deferred import？
- `@Bean` 方法是什么时候被发现的？

相关主线：[[10-domains/java/spring-framework/概念_Spring_Framework_配置类解析与依赖解析]] — 第一层配置编译主线；[[10-domains/java/spring-framework/概念_MergedAnnotations合并注解模型]] — 配置类解析依赖合并注解模型读取组合注解。

## 源码入口

关键类：

- `ConfigurationClassPostProcessor`：BeanFactoryPostProcessor 入口，触发配置类解析。
- `ConfigurationClassParser`：解析配置类结构。
- `ComponentScanAnnotationParser`：执行 `@ComponentScan`。
- `DeferredImportSelectorHandler`：收集和处理延迟导入。
- `DeferredImportSelectorGroupingHandler`：按 group 组织 deferred imports。
- `ConfigurationClassBeanDefinitionReader`：把解析结果注册成 BeanDefinition。

## 总体流程

```text
ConfigurationClassPostProcessor
  ↓
找出候选配置类 BeanDefinition
  ↓
ConfigurationClassParser.parse(candidates)
  ↓
processConfigurationClass()
  ↓
doProcessConfigurationClass()
  ↓
处理内部类 / @PropertySource / @ComponentScan / @Import / @ImportResource / @Bean
  ↓
deferredImportSelectorHandler.process()
  ↓
ConfigurationClassBeanDefinitionReader.loadBeanDefinitions()
```

## `doProcessConfigurationClass()` 的处理顺序

源码主线可以理解为固定顺序：

1. 如果配置类也是 `@Component`，先处理成员类。
2. 处理 `@PropertySource`。
3. 处理 `@ComponentScan`。
4. 处理 `@Import`。
5. 处理 `@ImportResource`。
6. 收集当前类的 `@Bean` 方法。
7. 处理接口默认方法上的 `@Bean`。
8. 继续处理父类。

这个顺序有工程意义：扫描和 Import 可能带来新的配置类，必须尽早加入解析队列；`@Bean` 方法则依附于已经识别出的配置类模型。

## `@ComponentScan` 细节

`ConfigurationClassParser` 会先找本地直接声明的 `@ComponentScan`，再找元注解声明的扫描注解：

```text
sourceClass.getAnnotations().stream(ComponentScans.class, ComponentScan.class)
  ↓
优先 MergedAnnotation::isDirectlyPresent
  ↓
否则 MergedAnnotation::isMetaPresent
```

每个 scan 注解交给 `componentScanParser.parse()`：

```text
@ComponentScan
  ↓
扫描 basePackages / basePackageClasses
  ↓
得到 BeanDefinitionHolder 集合
  ↓
检查每个 scanned BeanDefinition 是否又是配置类候选
  ↓
递归 parse(scannedBeanDefinition)
```

关键点：扫描不是单纯注册类名。扫描出的类如果本身带 `@Configuration`、`@Component`、`@Import` 等配置语义，会继续进入配置类解析，因此配置类图可以递归扩展。

## `@Import` 细节

`processImports()` 会把 import 候选分成四类：

| Import 类型 | 源码行为 | 适合场景 |
|---|---|---|
| `ImportSelector` | 实例化 selector，调用 `selectImports()`，再递归处理返回类名 | 根据注解属性选择一组配置 |
| `DeferredImportSelector` | 暂存到 handler，当前配置类解析结束后再统一处理 | 自动配置、需要晚于普通配置的导入 |
| `ImportBeanDefinitionRegistrar` | 实例化 registrar，后续注册自定义 BeanDefinition | 框架级动态注册，例如 Mapper 扫描 |
| 普通类 | 当作配置类调用 `processConfigurationClass()` | 直接导入配置类 |

`ImportSelector` 是立即执行；`DeferredImportSelector` 是延迟执行。这是 Spring Boot 自动配置能够“用户配置优先、自动配置兜底”的关键机制之一。

## `DeferredImportSelector` 细节

`DeferredImportSelectorHandler` 有两个阶段：

### 1. 收集阶段

解析普通配置类时遇到 deferred selector，不马上执行：

```text
processImports()
  ↓
candidate is DeferredImportSelector
  ↓
deferredImportSelectorHandler.handle(configClass, selector)
  ↓
加入 deferredImports 列表
```

### 2. 处理阶段

所有普通配置类解析完成后：

```text
deferredImportSelectorHandler.process()
  ↓
按 AnnotationAwareOrderComparator 排序
  ↓
按 group 分组
  ↓
group.process(annotationMetadata, selector)
  ↓
group.selectImports()
  ↓
对每个 Entry 再调用 processImports()
```

分组让一类 selector 能统一排序、去重、过滤和决策。Boot 的自动配置导入就是利用这个晚处理阶段，让用户显式配置、扫描配置先进入容器模型，再导入自动配置候选。

## 关键流程图说明

图中三条线要分清：

1. `@ComponentScan`：从配置类出发扫描 classpath，返回 BeanDefinition，再把其中的配置类候选递归解析。
2. 普通 `@Import`：立即展开，可能导入配置类、selector 结果或 registrar。
3. `DeferredImportSelector`：先收集，等普通解析结束后统一排序分组处理。

这三条线最终都会回到 `ConfigurationClass` 集合，交给 reader 注册 BeanDefinition。

## 常见误区

1. **误区：`@ComponentScan` 只负责把类注册成 Bean。**  
   它还会触发被扫描配置类的递归解析。

2. **误区：所有 `@Import` 顺序一样。**  
   普通 ImportSelector 立即执行，DeferredImportSelector 晚于普通配置类执行。

3. **误区：`ConfigurationClassParser` 会创建 Bean 实例。**  
   它只解析配置模型；Bean 实例化发生在 `finishBeanFactoryInitialization()` 之后。

4. **误区：`@Bean` 方法扫描是反射调用。**  
   Spring 先用元数据读取方法信息，后续注册为 BeanDefinition，再在 Bean 创建阶段通过增强后的配置类调用。

## 面试表达

可以这样回答：

> `ConfigurationClassParser` 是 Spring 注解配置的编译器。它在 `ConfigurationClassPostProcessor` 中运行，先解析配置类上的 `@ComponentScan`，扫描出的配置类会递归进入解析；再处理 `@Import`，普通 ImportSelector 立即展开，DeferredImportSelector 会先收集，等普通配置类都解析完后按 order 和 group 统一处理；最后收集 `@Bean`、`@ImportResource` 等信息。它的输出不是 Bean 实例，而是 `ConfigurationClass` 模型，随后由 reader 注册 BeanDefinition。

## 和其它概念的关系

- [[10-domains/java/spring-framework/概念_MergedAnnotations合并注解模型]] — 负责把组合注解解析为配置语义。
- [[10-domains/java/spring-framework/概念_Spring_Framework_IoC容器启动与Bean生命周期]] — 配置类解析发生在 `refresh()` 的 `invokeBeanFactoryPostProcessors()` 阶段。
- [[10-domains/java/spring/概念_Spring_Boot内核_自动配置_条件注解_启动过程_Starter]] — Boot 自动配置依赖 deferred import 机制。

## 来源

- `raw/code/spring-framework/spring-context/src/main/java/org/springframework/context/annotation/ConfigurationClassParser.java`
- `raw/code/spring-framework/spring-context/src/main/java/org/springframework/context/annotation/ConfigurationClassPostProcessor.java`
- `raw/code/spring-framework/spring-context/src/main/java/org/springframework/context/annotation/ComponentScanAnnotationParser.java`
