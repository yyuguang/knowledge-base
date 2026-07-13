---
type: "concept"
tags:
  - java
  - spring-framework
  - design-pattern
  - creational-pattern
summary: "Spring Framework 创建型设计模式：BeanFactory/FactoryBean 隔离 Bean 创建，AopProxyFactory 根据配置创建代理族，BeanDefinitionBuilder 构建复杂 BeanDefinition，DefaultSingletonBeanRegistry 管理共享单例和三级缓存。"
sources:
  - "raw/code/spring-framework/spring-beans/src/main/java/org/springframework/beans/factory/FactoryBean.java"
  - "raw/code/spring-framework/spring-aop/src/main/java/org/springframework/aop/framework/AopProxyFactory.java"
  - "raw/code/spring-framework/spring-beans/src/main/java/org/springframework/beans/factory/support/BeanDefinitionBuilder.java"
  - "raw/code/spring-framework/spring-beans/src/main/java/org/springframework/beans/factory/support/DefaultSingletonBeanRegistry.java"
aliases:
  - "Spring 创建型模式"
  - "Spring 工厂模式"
  - "Spring 单例模式"
status: "evolving"
confidence: 0.9
created: "2026-05-12 21:14:19"
updated: "2026-05-12 21:14:19"
---

# 概念：Spring Framework 创建型设计模式

## 定义

创建型设计模式关注“对象如何被创建”。Spring 是一个对象容器，所以创建型模式贯穿 Bean 定义、Bean 创建、代理创建和单例管理。

Spring 中最重要的创建型模式包括：

- 工厂模式：`BeanFactory`、`FactoryBean`
- 抽象工厂：`AopProxyFactory`
- 建造者：`BeanDefinitionBuilder`
- 单例注册表：`DefaultSingletonBeanRegistry`

## 为什么重要

Spring 的核心价值之一就是把对象创建从业务代码中拿出来。业务对象不再自己 `new` 依赖，而是由容器根据 BeanDefinition、Scope、FactoryBean、后处理器和代理规则来创建。

如果没有创建型模式，Spring 会遇到三个问题：

- 对象创建逻辑散落在业务代码中，依赖关系不可管理。
- 代理对象、JNDI 对象、复杂基础设施对象难以用普通构造器表达。
- 单例缓存、循环依赖、销毁顺序无法统一管理。

## 核心模式

### 1. BeanFactory：容器级工厂

`BeanFactory#getBean()` 是最典型的工厂入口。调用者只关心 beanName 或类型，不关心：

- BeanDefinition 从哪里来。
- 构造器如何选择。
- 依赖如何注入。
- 是否需要 AOP 代理。
- 是 singleton 还是 prototype。
- 是否来自 FactoryBean。

**为什么这么设计**：对象创建是框架最复杂的变化点。把创建统一收口到 BeanFactory，Spring 才能统一生命周期、依赖解析、后处理器和作用域。

**好处**：

- 业务代码和对象创建解耦。
- 容器可以插入 `BeanPostProcessor`、AOP、Scope 等横切行为。
- 第三方框架可以注册 BeanDefinition，而不是直接创建对象。

### 2. FactoryBean：Bean 自己是工厂，暴露另一个对象

`FactoryBean<T>` 的关键方法：

- `getObject()`：返回真正暴露给容器使用的对象。
- `getObjectType()`：提前暴露产品类型，方便依赖注入和类型查询。
- `isSingleton()`：声明产品对象是否可缓存。

源码注释明确指出：实现 `FactoryBean` 的 Bean 不作为普通 Bean 暴露，容器引用默认拿到的是 `getObject()` 的返回值。

典型例子：

- `ProxyFactoryBean`：创建 AOP 代理对象。
- `JndiObjectFactoryBean`：暴露 JNDI 查找到的对象。

**为什么这么设计**：有些对象不是简单构造器能创建的，例如代理对象、远程对象、JNDI 对象。它们需要一段工厂逻辑，但又希望作为普通 Bean 被注入。

**好处**：

- 复杂对象创建逻辑封装在工厂 Bean 内。
- 调用方仍按普通 Bean 使用产品对象。
- `getObjectType()` 允许不实例化产品也参与类型匹配。

**面试易错点**：

- `beanName` 获取的是产品对象。
- `&beanName` 获取的是 FactoryBean 本身。
- 容器管理 FactoryBean 生命周期，不自动管理产品对象的销毁，除非 FactoryBean 自己委托处理。

### 3. AopProxyFactory：抽象工厂

`AopProxyFactory#createAopProxy(AdvisedSupport config)` 根据 AOP 配置创建具体代理：

- `JdkDynamicAopProxy`
- `ObjenesisCglibAopProxy`

`DefaultAopProxyFactory` 根据 `proxyTargetClass`、接口、目标类等条件选择代理类型。

**为什么这么设计**：AOP 代理不是一个固定类。不同目标对象、不同配置、不同运行环境需要不同代理实现，但上层只需要“一个可执行 advisor 链的代理”。

**好处**：

- 上层不依赖 JDK Proxy 或 CGLIB 细节。
- 代理选择策略可以集中管理。
- 事务、缓存等横切能力复用同一代理创建体系。

### 4. BeanDefinitionBuilder：建造者模式

`BeanDefinitionBuilder` 用链式 API 构建复杂 BeanDefinition。

BeanDefinition 包含很多可选属性：

- beanClass / beanClassName
- factoryMethodName
- constructorArgumentValues
- propertyValues
- scope
- lazyInit
- instanceSupplier
- role

**为什么这么设计**：BeanDefinition 是复杂对象，如果用构造器传参会非常难维护。Builder 让创建过程可读、可组合，也适合 XML namespace、自定义注册器、框架内部程序化注册。

**好处**：

- 避免构造器参数爆炸。
- 构建过程表达清晰。
- 适合只设置部分属性。

### 5. DefaultSingletonBeanRegistry：单例注册表

Spring 的 singleton 不是 GoF 意义上的“类只能有一个实例”，而是“一个 BeanFactory 内一个 beanName 对应一个共享实例”。

`DefaultSingletonBeanRegistry` 维护：

- `singletonObjects`：完整单例。
- `singletonFactories`：早期对象工厂。
- `earlySingletonObjects`：早期引用。
- `registeredSingletons`：注册顺序。
- disposable bean 与依赖关系。

**为什么这么设计**：Spring 需要管理的不只是“拿到同一个对象”，还包括循环依赖、创建中状态、销毁顺序和依赖关系。

**好处**：

- 单例生命周期由容器统一管理。
- 三级缓存支持部分循环依赖。
- 销毁顺序可按依赖关系处理。

## 常见误区

1. **误区：Spring 单例就是 GoF 单例。**  
   GoF 单例强调类级别唯一；Spring 单例强调容器内 beanName 级别共享。

2. **误区：FactoryBean 和 BeanFactory 是一回事。**  
   BeanFactory 是容器；FactoryBean 是容器中的一个特殊 Bean，用来生产另一个对象。

3. **误区：CGLIB/JDK 代理选择是业务代码控制的。**  
   业务可以给配置倾向，但最终由 `AopProxyFactory` 按目标类和接口等条件决定。

## 和其它概念的关系

- [[concepts/java/spring-framework/概念_Spring_Framework_IoC容器启动与Bean生命周期]] — 创建型模式服务于 BeanDefinition 到 Bean 实例的生命周期。
- [[concepts/java/spring-framework/概念_Spring_Framework_AOP事务与Web分发]] — AOP 代理创建依赖 `AopProxyFactory`。
- [[overview/java/spring-framework/主题_Spring_Framework设计模式_综述]] — 本页属于设计模式第三轮学习的一部分。

## 我的理解

Spring 创建型模式的核心不是“优雅创建对象”，而是“容器接管对象创建权”。一旦创建权归容器，Spring 才能统一处理依赖注入、作用域、代理、循环依赖、初始化和销毁。

## 来源

- `raw/code/spring-framework/spring-beans/src/main/java/org/springframework/beans/factory/FactoryBean.java`
- `raw/code/spring-framework/spring-aop/src/main/java/org/springframework/aop/framework/AopProxyFactory.java`
- `raw/code/spring-framework/spring-beans/src/main/java/org/springframework/beans/factory/support/BeanDefinitionBuilder.java`
- `raw/code/spring-framework/spring-beans/src/main/java/org/springframework/beans/factory/support/DefaultSingletonBeanRegistry.java`

