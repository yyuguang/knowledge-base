---
type: "concept"
tags:
  - java
  - spring
  - aop
  - transaction
  - proxy
summary: "Spring AOP 模块：动态代理负责生成代理对象，通知被组织成拦截器链，切入点决定增强范围，@Transactional 通过 TransactionInterceptor 在代理调用中开启、提交或回滚事务。"
sources:
  - "raw/code/spring-framework/spring-aop/src/main/java/org/springframework/aop/framework/DefaultAopProxyFactory.java"
  - "raw/code/spring-framework/spring-aop/src/main/java/org/springframework/aop/framework/autoproxy/AbstractAutoProxyCreator.java"
  - "raw/code/spring-framework/spring-tx/src/main/java/org/springframework/transaction/interceptor/TransactionInterceptor.java"
  - "raw/code/spring-framework/spring-tx/src/main/java/org/springframework/transaction/interceptor/BeanFactoryTransactionAttributeSourceAdvisor.java"
aliases:
  - "Spring AOP 模块"
  - "动态代理 通知 切点 事务代理"
status: "evolving"
confidence: 0.88
created: "2026-05-12 21:22:48"
updated: "2026-05-12 21:22:48"
---

# 概念：Spring AOP模块 动态代理 通知 切点 事务代理

## 21_动态代理（JDK vs CGLIB）

Spring AOP 基于代理对象工作。客户端拿到的通常不是原始 Bean，而是代理 Bean。

选择规则核心在 `DefaultAopProxyFactory`：

- 有接口且未强制类代理：通常使用 JDK 动态代理。
- 强制 `proxyTargetClass` 或没有合适接口：普通类目标通常使用 CGLIB/Objenesis。
- 目标类是接口、JDK Proxy、lambda class 等特殊情况时仍可能使用 JDK 动态代理。

对比：

| 代理类型 | 基础 | 优点 | 限制 |
|---|---|---|---|
| JDK 动态代理 | 接口 | JDK 原生、简单 | 只能代理接口方法 |
| CGLIB/Objenesis | 子类 | 可代理类 | final 类/方法受限 |

## 22_通知类型与执行顺序

Spring AOP 最终会把不同 Advice 适配成方法拦截器链。

常见通知：

- Before：目标方法前执行。
- AfterReturning：正常返回后执行。
- AfterThrowing：异常后执行。
- After/finally：无论正常或异常都执行。
- Around：包裹目标方法，能决定是否继续执行。

执行顺序关键：

```text
proxy.invoke()
  → advisor/interceptor 1 before
  → advisor/interceptor 2 before
  → target method
  → advisor/interceptor 2 after
  → advisor/interceptor 1 after
```

排序由 Advisor/Advice 的 `Ordered`、`@Order`、注册顺序等共同影响。

## 23_切入点表达式与@Aspect

切入点（Pointcut）回答“哪些方法需要增强”。

`@Aspect` 常见表达：

- `execution(...)`：按方法签名匹配。
- `within(...)`：按类型范围匹配。
- `@annotation(...)`：按方法注解匹配。
- `@within(...)`：按类注解匹配。
- `args(...)`：按运行期参数匹配。

`@Aspect` 本身不是代理，Spring 需要把它解析成 Advisor，再由 auto-proxy creator 为目标 Bean 创建代理。

## 24_事务注解@Transactional的代理原理

`@Transactional` 是 Spring AOP 的经典应用。

关键对象：

- `BeanFactoryTransactionAttributeSourceAdvisor`
- `TransactionAttributeSource`
- `TransactionInterceptor`
- `PlatformTransactionManager`

流程：

```text
调用代理方法
  → AOP advisor chain
  → TransactionInterceptor.invoke()
  → 读取 @Transactional 属性
  → 获取/创建事务
  → invocation.proceed()
  → 正常：commit
  → 异常：按 rollback rule 判断 rollback/commit
  → 清理 TransactionInfo
```

为什么自调用失效：

```java
public void outer() {
    inner(); // this.inner()，未经过代理
}

@Transactional
public void inner() {}
```

事务增强在代理对象入口生效，`this.inner()` 不经过代理，因此不会进入 `TransactionInterceptor`。

## 面试表达

> Spring AOP 通过代理模式实现，容器在 Bean 初始化后可能用 auto-proxy creator 返回代理对象。调用代理方法时进入 advisor/interceptor 链，`@Transactional` 对应的核心拦截器是 `TransactionInterceptor`，它负责读取事务属性、创建或加入事务、执行目标方法、提交或回滚。

## 相关链接

- [[concepts/java/spring-framework/概念_Spring_Framework_AOP事务与Web分发]] — AOP、事务和 Web 分发源码主线。
- [[concepts/java/spring-framework/概念_Spring_Framework_结构型设计模式]] — 代理模式。
- [[concepts/java/spring-framework/概念_Spring_Framework_行为型设计模式]] — 拦截器链和责任链。
- [[overview/java/spring/主题_Spring知识体系_综述]] — 返回大纲。

