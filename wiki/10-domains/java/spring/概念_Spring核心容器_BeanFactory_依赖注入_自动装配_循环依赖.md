---
type: "concept"
tags:
  - java
  - spring
  - bean-factory
  - autowired
  - circular-dependency
summary: "Spring 核心容器：BeanFactory 是最小对象工厂，ApplicationContext 增加应用上下文能力；依赖注入可通过构造器、Setter、字段完成；@Autowired 由 BeanPostProcessor 和 resolveDependency 实现；三级缓存解决部分单例循环依赖。"
sources:
  - "[[30-sources/repositories/来源_Spring_Framework_源码]]"
  - "raw/code/spring-framework/spring-beans/src/main/java/org/springframework/beans/factory/support/DefaultListableBeanFactory.java"
  - "raw/code/spring-framework/spring-beans/src/main/java/org/springframework/beans/factory/annotation/AutowiredAnnotationBeanPostProcessor.java"
  - "raw/code/spring-framework/spring-beans/src/main/java/org/springframework/beans/factory/support/DefaultSingletonBeanRegistry.java"
  - "raw/code/spring-framework/spring-context/src/main/java/org/springframework/context/ApplicationContext.java"
aliases:
  - "Spring 核心容器"
  - "BeanFactory ApplicationContext 自动装配 循环依赖"
status: "evolving"
confidence: 0.9
created: "2026-05-12 21:22:48"
updated: "2026-05-12 21:22:48"
---

# 概念：Spring核心容器 BeanFactory 依赖注入 自动装配 循环依赖

## 11_BeanFactory与ApplicationContext区别

`BeanFactory` 是 Spring 最基础的 IoC 容器接口，核心能力是按 name/type 获取 Bean、管理 Bean 定义和实例。

`ApplicationContext` 在 BeanFactory 之上增加应用级能力：

| 能力 | BeanFactory | ApplicationContext |
|---|---|---|
| Bean 创建/获取 | 支持 | 支持 |
| BeanDefinition 管理 | 支持 | 支持 |
| 国际化 MessageSource | 无 | 有 |
| 事件发布 ApplicationEvent | 无 | 有 |
| 资源加载 ResourceLoader | 基础 | 更完整 |
| 自动注册特殊 Bean | 较弱 | `refresh()` 中自动处理 |
| 生命周期启动/关闭 | 较弱 | 有完整协议 |

面试表达：

> BeanFactory 是对象工厂，ApplicationContext 是应用上下文。ApplicationContext 组合并扩展 BeanFactory，增加事件、国际化、资源加载、生命周期和自动识别后处理器等能力。

## 12_依赖注入方式（构造器、Setter、字段）

| 方式 | 优点 | 缺点 | 推荐场景 |
|---|---|---|---|
| 构造器注入 | 依赖不可变、强制必填、利于测试 | 依赖多时构造器长 | 必需依赖 |
| Setter 注入 | 可选依赖、可后置变更 | 对象可能短暂不完整 | 可选配置 |
| 字段注入 | 写法短 | 不利测试、隐藏依赖、破坏不可变性 | 少用，主要在 demo |

源码位置：

- 构造器注入：`ConstructorResolver`
- 属性/字段注入：`populateBean()` + `AutowiredAnnotationBeanPostProcessor`

## 13_自动装配机制（@Autowired原理）

`@Autowired` 不是 BeanFactory 天生直接识别字段注解，而是由 `AutowiredAnnotationBeanPostProcessor` 作为 BeanPostProcessor 参与 Bean 创建。

流程：

```text
doCreateBean()
  → populateBean()
  → AutowiredAnnotationBeanPostProcessor.postProcessProperties()
  → findAutowiringMetadata()
  → InjectionMetadata.inject()
  → beanFactory.resolveDependency()
```

`DefaultListableBeanFactory.resolveDependency()` 的核心规则：

1. 处理 `Optional`、`ObjectFactory`、`ObjectProvider`。
2. 处理 `@Value` 建议值和 SpEL。
3. 按字段名/参数名/Qualifier 建议名称匹配。
4. 处理数组、集合、Map、Stream。
5. 按类型查找候选 Bean。
6. 多候选时用 `@Primary`、优先级、名称等决策。
7. 校验结果类型。

## 14_循环依赖解决（三级缓存）

Spring 主要解决 singleton setter/field 注入形成的循环依赖。

三级缓存：

| 缓存 | 字段 | 含义 |
|---|---|---|
| 一级缓存 | `singletonObjects` | 完整初始化后的单例 |
| 二级缓存 | `earlySingletonObjects` | 早期引用 |
| 三级缓存 | `singletonFactories` | 早期引用工厂，允许返回代理 |

典型流程：

```text
创建 A
  → 实例化 A
  → A 放入 singletonFactories
  → A 注入 B
      → 创建 B
      → B 注入 A
          → 从 singletonFactories 获取 A 的早期引用
          → 可能通过 getEarlyBeanReference 返回代理
      → B 初始化完成
  → A 完成属性填充和初始化
  → A 进入 singletonObjects
```

不能解决或不推荐依赖：

- 构造器循环依赖。
- prototype 循环依赖。
- 代理不一致导致的 raw bean 注入问题。
- 复杂业务循环依赖，最好通过重构职责或事件解耦。

## 面试表达

> `@Autowired` 的本质是 BeanPostProcessor 在属性填充阶段解析注入元数据，再委托 BeanFactory 的 `resolveDependency()` 按类型、名称、Qualifier、Primary、集合等规则解析依赖。循环依赖则依靠 singleton 的三级缓存提前暴露引用，尤其第三级缓存允许 AOP 提前暴露代理引用。

## 相关链接

- [[10-domains/java/spring-framework/概念_Spring_Framework_配置类解析与依赖解析]] — `resolveDependency()` 源码级解释。
- [[10-domains/java/spring-framework/概念_Spring_Framework_IoC容器启动与Bean生命周期]] — Bean 创建和三级缓存。
- [[10-domains/java/spring/主题_Spring知识体系_综述]] — 返回大纲。

