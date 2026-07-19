---
type: "concept"
tags:
  - java
  - spring-framework
  - aop
  - transaction
summary: "Spring AOP 的运行时核心是 advisor chain：代理对象在方法调用时由 AdvisedSupport/DefaultAdvisorChainFactory 生成拦截器链，再由 ReflectiveMethodInvocation.proceed() 递归推进，TransactionInterceptor 作为 MethodInterceptor 包裹目标方法完成事务边界。"
sources:
  - "[[30-sources/repositories/来源_Spring_Framework_源码]]"
  - "raw/code/spring-framework/spring-aop/src/main/java/org/springframework/aop/framework/JdkDynamicAopProxy.java"
  - "raw/code/spring-framework/spring-aop/src/main/java/org/springframework/aop/framework/ReflectiveMethodInvocation.java"
  - "raw/code/spring-framework/spring-aop/src/main/java/org/springframework/aop/framework/DefaultAdvisorChainFactory.java"
  - "raw/code/spring-framework/spring-tx/src/main/java/org/springframework/transaction/interceptor/TransactionInterceptor.java"
aliases:
  - "Spring AOP 责任链"
  - "advisor chain"
  - "MethodInvocation"
  - "TransactionInterceptor"
status: "evolving"
confidence: 0.9
created: "2026-05-13 09:14:32"
updated: "2026-05-13 09:14:32"
---

# 概念：Spring AOP责任链源码时序

![[Attachments/spring-aop-advisor-chain-sequence.excalidraw]]

## 定义

Spring AOP 的运行时核心是 `advisor chain`。代理对象拦截方法调用后，会根据当前 method、targetClass、advisors 计算出一条 `MethodInterceptor` 链，再由 `ReflectiveMethodInvocation.proceed()` 像责任链一样逐个推进。目标方法本身位于链尾。

声明式事务就是这条链中的一个典型拦截器：`TransactionInterceptor` 在调用 `invocation.proceed()` 前开启事务，在返回后提交，在异常时按 rollback rule 回滚。

相关主线：[[10-domains/java/spring-framework/概念_Spring_Framework_行为型设计模式]] — AOP 是责任链和回调模式的典型源码实例；[[10-domains/java/spring/概念_Spring_AOP模块_动态代理_通知_切点_事务代理]] — AOP 模块面试主线。

## 源码入口

关键类：

- `JdkDynamicAopProxy`：JDK 动态代理调用入口。
- `CglibAopProxy`：CGLIB 代理调用入口。
- `AdvisedSupport`：保存 targetSource、interfaces、advisors、advisorChainFactory。
- `DefaultAdvisorChainFactory`：把 Advisor 转换成 MethodInterceptor 链。
- `ReflectiveMethodInvocation`：运行时方法调用上下文与 proceed 链推进器。
- `TransactionInterceptor`：声明式事务拦截器。
- `TransactionAspectSupport`：事务模板逻辑。

## 运行时主流程

```text
client 调用 proxy.method()
  ↓
JdkDynamicAopProxy.invoke()
  ↓
targetSource.getTarget()
  ↓
advised.getInterceptorsAndDynamicInterceptionAdvice(method, targetClass)
  ↓
new ReflectiveMethodInvocation(proxy, target, method, args, targetClass, chain)
  ↓
invocation.proceed()
  ↓
interceptor1.invoke(invocation)
  ↓
interceptor2.invoke(invocation)
  ↓
...
  ↓
TransactionInterceptor.invoke(invocation)
  ↓
invokeWithinTransaction(...)
  ↓
invocation.proceed()
  ↓
target method
```

## Advisor Chain 如何生成

`AdvisedSupport.getInterceptorsAndDynamicInterceptionAdvice()` 会委托 `advisorChainFactory`，默认是 `DefaultAdvisorChainFactory`。

生成逻辑：

1. 遍历当前代理持有的 advisors。
2. 如果是 `PointcutAdvisor`，先检查 class filter 是否匹配 targetClass。
3. 再检查 method matcher 是否匹配当前 method。
4. 通过 `AdvisorAdapterRegistry` 把 Advisor/Advice 适配成 `MethodInterceptor`。
5. 对 runtime method matcher，包装成 `InterceptorAndDynamicMethodMatcher`，运行时再判断参数。
6. 返回 ordered chain。

这解释了为什么 AOP 匹配既有启动/代理创建时判断，也有方法调用时判断：静态匹配可提前过滤，动态匹配需要拿到运行时参数。

## ReflectiveMethodInvocation.proceed()

`ReflectiveMethodInvocation` 内部维护 `currentInterceptorIndex`。每调用一次 `proceed()`，推进到下一个拦截器：

```text
if currentInterceptorIndex == interceptors.size - 1:
  invokeJoinpoint() // 目标方法
else:
  interceptor = interceptors[++currentInterceptorIndex]
  interceptor.invoke(this)
```

每个 interceptor 决定是否调用 `invocation.proceed()`：

- 调用：链继续向后推进。
- 不调用：短路，目标方法不会执行。
- 前后包裹：典型 around advice。
- try/catch/finally：典型事务、异常、资源清理。

这就是责任链的本质：链节点不只是被顺序调用，还能控制后续链路是否继续。

## TransactionInterceptor 时序

`TransactionInterceptor.invoke()` 主要做两件事：

1. 计算 targetClass。
2. 调用 `invokeWithinTransaction(method, targetClass, invocationCallback)`。

传统事务路径在 `TransactionAspectSupport.invokeWithinTransaction()` 中：

```text
读取 TransactionAttribute
  ↓
确定 TransactionManager
  ↓
createTransactionIfNecessary()
  ↓
try invocation.proceedWithInvocation()
  ↓
异常：completeTransactionAfterThrowing()
  ↓
finally: cleanupTransactionInfo()
  ↓
正常：commitTransactionAfterReturning()
```

因此事务不是围绕代理对象本身，而是围绕 `MethodInvocation.proceed()`。它只要位于 advisor chain 中，就可以包住后面的所有拦截器和目标方法。

## 与通知类型的关系

Spring 最终会把不同 Advice 适配成 `MethodInterceptor`：

| 通知类型 | 链中行为 |
|---|---|
| Before advice | 在 `proceed()` 前执行，再继续链 |
| After returning advice | 先 `proceed()`，正常返回后执行 |
| Throws advice | `proceed()` 抛异常后匹配异常处理 |
| Around advice | 完整控制是否、何时调用 `proceed()` |
| Transaction advice | Around 形态，前置开启事务，后置提交/回滚 |

所以源码层并不存在很多条完全不同的执行机制，最终都收敛到 `MethodInterceptor` 和 `MethodInvocation`。

## 自调用为什么失效

如果同一个目标对象内部 `this.inner()` 调用另一个被增强方法，这次调用没有经过 proxy，因此不会进入 `JdkDynamicAopProxy.invoke()`，也就不会生成并执行 advisor chain。

解决思路：

- 从外部通过代理对象调用。
- 拆到另一个 Bean。
- 暴露代理后通过 `AopContext.currentProxy()` 调用，但这会增加耦合。
- 使用 AspectJ weaving 而非 proxy-based AOP。

## 常见误区

1. **误区：Advisor chain 在每次调用时重新做所有扫描。**  
   代理持有 advisors；方法调用时会按 method/targetClass 取链，并有缓存机制辅助，非全量 classpath 扫描。

2. **误区：事务和 AOP 是两套无关机制。**  
   声明式事务就是一个 advisor + interceptor，通过 AOP 代理进入。

3. **误区：所有 advice 都直接反射调用。**  
   Spring 会通过 adapter 把不同 advice 统一成 `MethodInterceptor`。

4. **误区：责任链只能顺序执行。**  
   每个 interceptor 可以短路、包裹、异常恢复或修改返回值。

## 面试表达

可以这样回答：

> Spring AOP 方法调用时，代理对象先通过 `AdvisedSupport` 和 `DefaultAdvisorChainFactory` 根据当前 method、targetClass 从 advisors 里筛出 `MethodInterceptor` 链，然后创建 `ReflectiveMethodInvocation`。`proceed()` 每次推进一个 interceptor，直到链尾反射调用目标方法。`TransactionInterceptor` 就是链上的一个 around interceptor，它在 `invokeWithinTransaction()` 里读取事务属性、选择事务管理器、开启事务，然后调用 `invocation.proceed()`，正常提交、异常按规则回滚。因此声明式事务本质上是 AOP advisor chain 的一个节点。

## 和其它概念的关系

- [[10-domains/java/spring-framework/概念_Spring_Framework_结构型设计模式]] — proxy 结构让客户端无感调用增强对象。
- [[10-domains/java/spring-framework/概念_Spring_Framework_行为型设计模式]] — advisor chain 是责任链，`invocation.proceed()` 是回调。
- [[10-domains/java/spring-framework/概念_Spring_TestContext缓存与事务测试]] — 测试事务入口是 TestExecutionListener，业务声明式事务入口是 AOP chain。

## 来源

- `raw/code/spring-framework/spring-aop/src/main/java/org/springframework/aop/framework/JdkDynamicAopProxy.java`
- `raw/code/spring-framework/spring-aop/src/main/java/org/springframework/aop/framework/AdvisedSupport.java`
- `raw/code/spring-framework/spring-aop/src/main/java/org/springframework/aop/framework/DefaultAdvisorChainFactory.java`
- `raw/code/spring-framework/spring-aop/src/main/java/org/springframework/aop/framework/ReflectiveMethodInvocation.java`
- `raw/code/spring-framework/spring-tx/src/main/java/org/springframework/transaction/interceptor/TransactionInterceptor.java`
- `raw/code/spring-framework/spring-tx/src/main/java/org/springframework/transaction/interceptor/TransactionAspectSupport.java`
