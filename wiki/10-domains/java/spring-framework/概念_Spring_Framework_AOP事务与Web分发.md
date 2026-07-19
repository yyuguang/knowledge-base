---
type: "concept"
tags:
  - java
  - spring-framework
  - aop
  - transaction
  - webmvc
  - webflux
summary: "Spring Framework AOP、事务与 Web 分发的共同模型：固定执行骨架、可排序策略列表、拦截链和上下文回调。"
sources:
  - "[[30-sources/repositories/来源_Spring_Framework_源码]]"
  - "raw/code/spring-framework/spring-aop/src/main/java/org/springframework/aop/framework/DefaultAopProxyFactory.java"
  - "raw/code/spring-framework/spring-tx/src/main/java/org/springframework/transaction/interceptor/TransactionAspectSupport.java"
  - "raw/code/spring-framework/spring-webmvc/src/main/java/org/springframework/web/servlet/DispatcherServlet.java"
  - "raw/code/spring-framework/spring-webflux/src/main/java/org/springframework/web/reactive/DispatcherHandler.java"
aliases:
  - "Spring AOP 事务 Web 分发"
  - "Spring DispatcherServlet 源码"
  - "Spring 声明式事务源码"
status: "evolving"
confidence: 0.86
created: "2026-05-12 20:35:53"
updated: "2026-05-12 20:35:53"
---

# 概念：Spring Framework AOP事务与Web分发

## 定义

AOP、声明式事务、MVC/WebFlux 请求分发看似属于不同模块，但在 Spring Framework 源码里共享一个底层思想：

> 框架固定主流程，用户或上层框架提供可排序策略；运行时通过拦截链、回调和上下文对象把横切逻辑插入关键节点。

在方法调用场景，这个模型表现为 AOP `MethodInterceptor` 链；在事务场景，表现为 `TransactionInterceptor` 包裹目标方法；在 Web 场景，表现为 `DispatcherServlet` / `DispatcherHandler` 选择 mapping、adapter、exception resolver、result handler。

## 为什么重要

理解这页能把三个常见问题串起来：

- `@Transactional` 为什么需要代理？因为事务逻辑是 AOP around advice。
- Controller 为什么可以有各种参数和返回值？因为 `HandlerAdapter` 与参数/返回值解析器把方法调用标准化。
- WebFlux 为什么不是 MVC 的简单异步版？因为它保留策略角色，但把执行流改成 Reactor `Mono/Flux`。

## 使用场景

- 排查 `@Transactional` 不生效、自调用失效、final 类/方法代理失败。
- 理解 Spring MVC Controller 调用链、拦截器顺序、异常解析、视图渲染。
- 对比 MVC 与 WebFlux 的源码模型。
- 自定义 AOP advisor、事务属性源、HandlerMethodArgumentResolver、HandlerExceptionResolver。

## 核心机制

### 1. AOP 代理选择

`DefaultAopProxyFactory.createAopProxy()` 的判断规则：

| 条件 | 代理类型 |
|---|---|
| 未强制类代理，且有用户提供接口 | `JdkDynamicAopProxy` |
| 强制类代理，但目标类是接口/JDK Proxy/lambda class | `JdkDynamicAopProxy` |
| 强制类代理，目标是普通类 | `ObjenesisCglibAopProxy` |

AOP 代理的关键不是“生成一个代理类”这么简单，而是生成一个能执行 advisor/interceptor 链的调用入口。

### 2. 声明式事务是 AOP around advice

`TransactionInterceptor.invoke()` 会适配到 `TransactionAspectSupport.invokeWithinTransaction()`：

```text
读取 TransactionAttribute
  ↓
选择 TransactionManager
  ↓
createTransactionIfNecessary()
  ↓
invocation.proceedWithInvocation()
  ↓
异常：completeTransactionAfterThrowing()
正常：commitTransactionAfterReturning()
  ↓
cleanupTransactionInfo()
```

事务传播行为的核心在 `AbstractPlatformTransactionManager.getTransaction()`：

- 已有事务：根据 REQUIRED、REQUIRES_NEW、NESTED、NOT_SUPPORTED 等决定加入、挂起、新建或报错。
- 无事务：MANDATORY 报错；REQUIRED/REQUIRES_NEW/NESTED 新建；其它场景可能只创建同步状态。

### 3. MVC 分发是 Servlet 版策略流水线

`DispatcherServlet.doDispatch()`：

```text
HttpServletRequest
  → checkMultipart()
  → getHandler()                 # HandlerMapping
  → applyPreHandle()             # HandlerInterceptor
  → getHandlerAdapter()
  → ha.handle()
  → applyDefaultViewName()
  → applyPostHandle()
  → processDispatchResult()
      → processHandlerException()
      → render()
      → afterCompletion()
```

关键策略：

- `HandlerMapping`：请求到 handler 的映射。
- `HandlerAdapter`：把不同类型 handler 统一调用。
- `HandlerExceptionResolver`：把异常转换为响应或视图。
- `ViewResolver`：把逻辑视图名转换成 View。
- `HandlerInterceptor`：请求调用前后与完成后回调。

### 4. WebFlux 分发是 Reactive 版策略流水线

`DispatcherHandler.handle()`：

```text
ServerWebExchange
  → Flux.fromIterable(handlerMappings)
  → mapping.getHandler(exchange)
  → next()
  → handleRequestWith(exchange, handler)
      → HandlerAdapter.handle()
      → HandlerResultHandler.handleResult()
  → Mono<Void>
```

与 MVC 对应关系：

| MVC | WebFlux | 含义 |
|---|---|---|
| `DispatcherServlet` | `DispatcherHandler` | 中央分发器 |
| `HttpServletRequest/Response` | `ServerWebExchange` | 请求上下文 |
| `HandlerMapping` | `HandlerMapping` | 找 handler |
| `HandlerAdapter` | `HandlerAdapter` | 调用 handler |
| `ModelAndView/ViewResolver` | `HandlerResultHandler` | 处理返回值 |
| `HandlerExceptionResolver` | `DispatchExceptionHandler` / WebExceptionHandler | 异常处理 |

## 例子

### `@Transactional` 自调用为什么失效

```java
class UserService {
    public void outer() {
        inner(); // this.inner()，没有经过代理对象
    }

    @Transactional
    public void inner() {
        // 事务拦截器不会执行
    }
}
```

原因：声明式事务在代理对象的方法调用入口生效。`this.inner()` 是目标对象内部调用，没有进入 AOP interceptor 链。

### Controller 参数为什么能自动注入

`@RequestMapping` 方法并不是由反射裸调用完成的。MVC 通过 `RequestMappingHandlerAdapter` 建立一套参数解析器和返回值处理器：

- 参数解析：request param、path variable、request body、model attribute、session、principal 等。
- 返回值处理：`ModelAndView`、`String` view name、`@ResponseBody`、`ResponseEntity`、异步类型等。

这和 IoC 注入一样，都是“把多种输入形式标准化为一次方法调用”。

## 常见误区

1. **误区：AOP 只和日志有关。**  
   在 Spring 里，事务、缓存、异步、部分安全能力都可以通过 AOP 代理接入。

2. **误区：`proxyTargetClass=true` 就一定是 CGLIB。**  
   如果目标类是接口、JDK Proxy 或 lambda class，Spring 仍可能选择 JDK 动态代理。

3. **误区：MVC 的核心是注解扫描。**  
   注解扫描只是发现 handler；真正的请求执行核心是 `HandlerMapping + HandlerAdapter + HandlerExceptionResolver`。

4. **误区：WebFlux 只是把 MVC 方法返回值换成 Mono。**  
   WebFlux 的执行上下文、背压模型、结果处理与异常处理都围绕 Reactor 管线组织。

## 和其它概念的关系

- [[10-domains/java/spring-framework/概念_Spring_Framework_IoC容器启动与Bean生命周期]] — AOP 代理通常由 BeanPostProcessor 在 Bean 初始化阶段产生。
- [[10-domains/java/spring-framework/概念_Spring_Framework_模块体系与扩展点]] — AOP、事务、Web 都是模块通过扩展点接入内核的例子。
- [[30-sources/repositories/来源_Spring_Framework_源码]] — 本页关键源码入口来自 source page。

## 我的理解

Spring Framework 的高级模块并不是各自发明机制，而是在重复使用一种架构语法：

```text
上下文对象
  + 可排序策略列表
  + 模板方法主流程
  + 前后置拦截/异常处理
  = 可扩展框架能力
```

事务和 Web 的共同点尤其明显：都先把“当前调用”包装成上下文，再查找策略，再执行主逻辑，最后按成功/异常路径做收尾。掌握这个模式以后，读缓存、消息、测试上下文、WebSocket 都会更顺。

## 来源

- `raw/code/spring-framework/spring-aop/src/main/java/org/springframework/aop/framework/DefaultAopProxyFactory.java`
- `raw/code/spring-framework/spring-tx/src/main/java/org/springframework/transaction/interceptor/TransactionAspectSupport.java`
- `raw/code/spring-framework/spring-tx/src/main/java/org/springframework/transaction/support/AbstractPlatformTransactionManager.java`
- `raw/code/spring-framework/spring-webmvc/src/main/java/org/springframework/web/servlet/DispatcherServlet.java`
- `raw/code/spring-framework/spring-webflux/src/main/java/org/springframework/web/reactive/DispatcherHandler.java`
- [[30-sources/repositories/来源_Spring_Framework_源码]] — Spring Framework 源码入口。

