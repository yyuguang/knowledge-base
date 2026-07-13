---
type: "concept"
tags:
  - java
  - spring-framework
  - webflux
  - reactive
summary: "Spring WebFlux 的响应式处理链以 DispatcherHandler 为中央分发器，通过 HandlerMapping、HandlerAdapter、HandlerResultHandler、DispatchExceptionHandler 和 Reactor Context 组成非阻塞请求管线。"
sources:
  - "raw/code/spring-framework/spring-webflux/src/main/java/org/springframework/web/reactive/DispatcherHandler.java"
  - "raw/code/spring-framework/spring-webflux/src/main/java/org/springframework/web/reactive/HandlerResultHandler.java"
  - "raw/code/spring-framework/spring-web/src/main/java/org/springframework/web/filter/reactive/ServerWebExchangeContextFilter.java"
aliases:
  - "Spring WebFlux"
  - "DispatcherHandler"
  - "HandlerResultHandler"
  - "Reactor Context"
status: "evolving"
confidence: 0.88
created: "2026-05-13 09:14:32"
updated: "2026-05-13 09:14:32"
---

# 概念：Spring WebFlux响应式处理链

## 定义

Spring WebFlux 是 Spring 的响应式 Web 栈。它的中央分发器是 `DispatcherHandler`，与 MVC 的 `DispatcherServlet` 同构，但控制流不是同步方法调用，而是 Reactor `Mono` / `Flux` 管线。

核心链路：

```text
WebHandler
  ↓
DispatcherHandler.handle(ServerWebExchange)
  ↓
HandlerMapping 找 handler
  ↓
HandlerAdapter 调用 handler
  ↓
HandlerResultHandler 处理返回值
  ↓
Mono<Void> 写出响应
```

相关主线：[[concepts/java/spring-framework/概念_Spring_Framework_AOP事务与Web分发]] — MVC/WebFlux 共享 mapping/adapter/result 的策略骨架；[[concepts/java/spring-framework/概念_Spring_Framework_Web测试消息与应用层扩展]] — WebFlux 与测试工具 WebTestClient 有直接关系。

## 源码入口

关键类：

- `DispatcherHandler`：WebFlux 中央分发器。
- `HandlerMapping`：根据 `ServerWebExchange` 找 handler。
- `HandlerAdapter`：把不同形态 handler 适配成统一调用。
- `HandlerResult`：handler 返回值和上下文信息。
- `HandlerResultHandler`：把返回值写成响应。
- `DispatchExceptionHandler`：在分发链中把异常恢复成 `HandlerResult`。
- `ServerWebExchangeContextFilter`：把当前 `ServerWebExchange` 写入 Reactor Context。

## DispatcherHandler 主流程

`DispatcherHandler.handle()` 的核心形态：

```text
Flux.fromIterable(handlerMappings)
  ↓ concatMap(mapping.getHandler(exchange))
  ↓ next()
  ↓ switchIfEmpty(404)
  ↓ onErrorResume(ex -> handleResultMono(exchange, Mono.error(ex)))
  ↓ flatMap(handler -> handleRequestWith(exchange, handler))
```

几个源码级细节：

1. `concatMap` 表示按 `HandlerMapping` 顺序查找，而不是并发抢占。
2. `next()` 只取第一个匹配 handler。
3. 找不到 handler 会进入 not found error。
4. 查找 handler 阶段的异常会交给 `handleResultMono()`，尝试走统一异常处理路径。
5. 找到 handler 后，`handleRequestWith()` 再选择 `HandlerAdapter`。

## HandlerAdapter：把 handler 变成 HandlerResult

`handleRequestWith(exchange, handler)` 会遍历所有 `HandlerAdapter`：

```text
for adapter in handlerAdapters:
  if adapter.supports(handler):
    resultMono = adapter.handle(exchange, handler)
    return handleResultMono(exchange, resultMono)
```

这与 MVC 的 HandlerAdapter 模式一致：中央分发器不关心 handler 是注解 controller、函数式路由，还是其它扩展形态，只要 adapter 能把它转成 `Mono<HandlerResult>`。

## HandlerResultHandler：返回值处理器

`HandlerResultHandler` 接口只有两个关键方法：

```java
boolean supports(HandlerResult result);
Mono<Void> handleResult(ServerWebExchange exchange, HandlerResult result);
```

它解决的问题是：controller 返回值可能是 `Mono<T>`、`Flux<T>`、`ResponseEntity<T>`、`ServerResponse`、`String` view name、void 等，分发器不应该写死每种返回类型如何输出。

典型 result handler：

- `ResponseEntityResultHandler`：处理 `ResponseEntity`。
- `ResponseBodyResultHandler`：处理 `@ResponseBody` 和响应体编码。
- `ViewResolutionResultHandler`：处理视图名和模型。
- `ServerResponseResultHandler`：处理函数式端点的 `ServerResponse`。

源码主线是：

```text
handleResult(exchange, result, description)
  ↓
遍历 resultHandlers
  ↓
第一个 supports(result) 的 handler 执行 handleResult()
  ↓
checkpoint(description)
```

如果没有任何 result handler 支持，会返回 error。这说明返回值处理是一个有序策略链，不是 if/else 大杂烩。

## Exception Handler：异常恢复点

WebFlux 的异常处理有两个层次。

### 1. HandlerAdapter 级异常处理

`handleResultMono()` 会先检查 `HandlerAdapter` 是否同时实现 `DispatchExceptionHandler`。如果是，会给 `resultMono` 加 `onErrorResume()`：

```text
HandlerAdapter.handle()
  ↓
Mono<HandlerResult> resultMono
  ↓
onErrorResume(exceptionHandler.handleError(exchange, ex))
```

这让“handler 调用阶段”的异常可以恢复成另一个 `HandlerResult`。

### 2. HandlerResult 内置异常处理

`HandlerResult` 还可以携带 `exceptionHandler`。在 result handler 写响应时如果异常，也可以再次恢复：

```text
handleResult(...)
  ↓
Mono<Void> 写响应失败
  ↓ onErrorResume(ex)
  ↓ result.getExceptionHandler().handleError(exchange, ex)
  ↓ 得到新的 HandlerResult
  ↓ 再次 handleResult(...)
```

这比 MVC 更明显地体现了“异常是响应式管线中的信号”，不是单纯 catch 后跳转。

## Reactor Context

响应式链路里不能依赖 ThreadLocal 传递请求上下文，因为同一个请求可能跨线程调度。WebFlux 使用 Reactor Context 传递链路内上下文。

`ServerWebExchangeContextFilter` 的源码行为：

```text
chain.filter(exchange)
  .contextWrite(context -> context.put(EXCHANGE_CONTEXT_ATTRIBUTE, exchange))
```

之后组件可以通过：

```text
ServerWebExchangeContextFilter.getExchange(ContextView)
```

从 Reactor Context 中取出当前 exchange。

这解释了响应式编程中的一个关键差异：MVC 常见的 `RequestContextHolder` / ThreadLocal 思维，在 WebFlux 中必须谨慎。上下文应通过 Reactor Context 随信号传播。

## 和 MVC 的对比

| 维度 | Spring MVC | Spring WebFlux |
|---|---|---|
| 中央分发器 | `DispatcherServlet` | `DispatcherHandler` |
| 请求上下文 | `HttpServletRequest/Response` | `ServerWebExchange` |
| handler 查找 | `HandlerMapping` | `HandlerMapping` |
| handler 调用 | `HandlerAdapter` | `HandlerAdapter` |
| 返回值处理 | `HandlerMethodReturnValueHandler` 等 | `HandlerResultHandler` |
| 异常处理 | `HandlerExceptionResolver` | `DispatchExceptionHandler` / result exception handler |
| 执行模型 | 同步 Servlet，可异步 | Reactor `Mono/Flux` |
| 上下文传播 | ThreadLocal 常见 | Reactor Context |

## 调优与排查点

1. 检查 handler mapping 顺序：谁先匹配，谁就终止查找。
2. 检查 result handler 顺序：返回值由第一个 `supports()` 的处理器接管。
3. 响应体编码问题通常落在 codec、`ResponseBodyResultHandler` 或 content negotiation。
4. 异常没有被处理时，要区分是 handler 调用阶段、返回值写出阶段，还是 WebExceptionHandler 层。
5. Reactor Context 丢失通常来自手动创建异步边界或没有在同一 reactive chain 中传播。

## 常见误区

1. **误区：WebFlux 就是把 MVC 返回值换成 Mono。**  
   真正变化在执行模型、上下文传播、异常信号和背压语义。

2. **误区：WebFlux 仍可依赖 ThreadLocal 传递 request。**  
   响应式链路可能跨线程，应该优先使用 Reactor Context。

3. **误区：HandlerResultHandler 只是序列化响应体。**  
   它覆盖多类返回值，包括响应实体、body、视图、函数式响应。

4. **误区：异常只能靠全局异常处理。**  
   WebFlux 在 adapter 和 result 两层都有异常恢复点。

## 面试表达

可以这样回答：

> WebFlux 的核心是 `DispatcherHandler`。它先按顺序遍历 `HandlerMapping` 找第一个 handler，再用支持该 handler 的 `HandlerAdapter` 调用，得到 `Mono<HandlerResult>`，最后由第一个支持该返回值的 `HandlerResultHandler` 写响应。异常不是简单 catch，而是在 `Mono` 管线中通过 `onErrorResume` 恢复，`HandlerAdapter` 和 `HandlerResult` 都可以提供 `DispatchExceptionHandler`。因为响应式链路可能跨线程，请求上下文通过 Reactor Context 传播，例如 `ServerWebExchangeContextFilter` 会把 exchange 写入 context。

## 和其它概念的关系

- [[concepts/java/spring-framework/概念_Spring_TestContext缓存与事务测试]] — `WebTestClient` 可以测试 WebFlux，也可以通过 MockMvc bridge 测 MVC。
- [[concepts/java/spring-framework/概念_MergedAnnotations合并注解模型]] — 注解 controller 的 mapping 依赖合并注解读取 `@RequestMapping` 语义。
- [[concepts/java/spring/概念_Spring_Web层_DispatcherServlet_HandlerMapping_参数返回值_过滤器拦截器]] — MVC 和 WebFlux 都体现 mapping/adapter/result 策略链。

## 来源

- `raw/code/spring-framework/spring-webflux/src/main/java/org/springframework/web/reactive/DispatcherHandler.java`
- `raw/code/spring-framework/spring-webflux/src/main/java/org/springframework/web/reactive/HandlerResultHandler.java`
- `raw/code/spring-framework/spring-webflux/src/main/java/org/springframework/web/reactive/DispatchExceptionHandler.java`
- `raw/code/spring-framework/spring-web/src/main/java/org/springframework/web/filter/reactive/ServerWebExchangeContextFilter.java`
