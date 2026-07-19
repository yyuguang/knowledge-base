---
type: "concept"
tags:
  - java
  - spring
  - spring-mvc
  - dispatcher-servlet
  - web
summary: "Spring Web 层：DispatcherServlet 是中央分发器，HandlerMapping 定位 handler，HandlerAdapter 用适配器模式调用 handler，参数解析器和返回值处理器完成 Controller 方法调用，Filter 属于 Servlet 容器链，Interceptor 属于 Spring MVC handler 链。"
sources:
  - "[[30-sources/repositories/来源_Spring_Framework_源码]]"
  - "raw/code/spring-framework/spring-webmvc/src/main/java/org/springframework/web/servlet/DispatcherServlet.java"
  - "raw/code/spring-framework/spring-webmvc/src/main/java/org/springframework/web/servlet/mvc/method/annotation/RequestMappingHandlerMapping.java"
  - "raw/code/spring-framework/spring-webmvc/src/main/java/org/springframework/web/servlet/mvc/method/annotation/RequestMappingHandlerAdapter.java"
  - "raw/code/spring-framework/spring-web/src/main/java/org/springframework/web/filter/OncePerRequestFilter.java"
aliases:
  - "Spring Web 层"
  - "DispatcherServlet HandlerMapping 参数解析器 返回值处理器"
status: "evolving"
confidence: 0.9
created: "2026-05-12 21:22:48"
updated: "2026-05-12 21:22:48"
---

# 概念：Spring Web层 DispatcherServlet HandlerMapping 参数返回值 过滤器拦截器

## 41_DispatcherServlet处理流程

`DispatcherServlet` 是 Spring MVC 的前端控制器。

核心流程：

```text
HTTP Request
  → Servlet Filter chain
  → DispatcherServlet.doDispatch()
  → checkMultipart()
  → getHandler()                 # HandlerMapping
  → mappedHandler.applyPreHandle()
  → getHandlerAdapter()
  → handlerAdapter.handle()
  → applyDefaultViewName()
  → mappedHandler.applyPostHandle()
  → processDispatchResult()
      → HandlerExceptionResolver
      → ViewResolver / HttpMessageConverter
      → afterCompletion()
```

源码方法：`DispatcherServlet#doDispatch()`

## 42_HandlerMapping与适配器模式

`HandlerMapping` 负责“请求找谁处理”。

常见实现：

- `RequestMappingHandlerMapping`：处理 `@RequestMapping` / `@GetMapping` 等注解。
- `BeanNameUrlHandlerMapping`：按 Bean 名称映射 URL。

`HandlerAdapter` 负责“如何调用这个 handler”。

为什么是适配器模式：

- handler 可能是注解方法。
- handler 可能是 `Controller` 接口。
- handler 可能是 `HttpRequestHandler`。
- handler 可能来自第三方框架。

`DispatcherServlet` 不直接判断所有 handler 类型，而是通过 `HandlerAdapter.supports()` 和 `handle()` 统一调用。

## 43_参数解析器与返回值处理器

真正调用 `@RequestMapping` 方法的是 `RequestMappingHandlerAdapter`。

它维护：

- `HandlerMethodArgumentResolverComposite`
- `HandlerMethodReturnValueHandlerComposite`
- `HttpMessageConverter`
- `WebDataBinderFactory`
- `ControllerAdviceBean` 缓存

参数解析流程：

```text
HandlerMethod
  → 遍历 HandlerMethodArgumentResolver
  → supportsParameter(parameter)
  → resolveArgument(...)
  → 得到方法参数值
  → 反射调用 Controller 方法
```

返回值处理流程：

```text
Controller return value
  → 遍历 HandlerMethodReturnValueHandler
  → supportsReturnType(...)
  → handleReturnValue(...)
  → ModelAndView / ResponseBody / ResponseEntity / async
```

## 44_拦截器与过滤器链

| 对比项 | Filter | HandlerInterceptor |
|---|---|---|
| 所属体系 | Servlet 容器 | Spring MVC |
| 执行位置 | DispatcherServlet 前后 | 找到 handler 之后 |
| 能否拿到 handler | 通常不能 | 能 |
| 常见用途 | 编码、CORS、安全、日志、包装 request | 权限、登录态、业务拦截、埋点 |
| 注册方式 | Servlet FilterRegistration / Boot FilterRegistrationBean | WebMvcConfigurer.addInterceptors |

`OncePerRequestFilter` 是 Spring Web 提供的过滤器基类，用于保证一次请求只执行一次。

`HandlerInterceptor` 有三个典型回调：

- `preHandle`
- `postHandle`
- `afterCompletion`

## 面试表达

> Spring MVC 的核心流程是 DispatcherServlet 作为中央分发器，先通过 HandlerMapping 找到 handler，再通过 HandlerAdapter 适配调用。注解 Controller 的调用由 RequestMappingHandlerAdapter 完成，参数解析器负责把 request 转成方法参数，返回值处理器负责把方法返回值转成响应。Filter 是 Servlet 链，Interceptor 是 Spring MVC 链，二者执行位置不同。

## 常见误区

- DispatcherServlet 不直接反射调用 Controller。
- HandlerMapping 和 HandlerAdapter 是两件事：一个找 handler，一个调 handler。
- Filter 和 Interceptor 不在同一层。
- `@RequestBody` 不是参数解析器单独完成，还需要 HttpMessageConverter。

## 相关链接

- [[10-domains/java/spring-framework/概念_Spring_Framework_Web测试消息与应用层扩展]] — Web 应用层源码。
- [[10-domains/java/spring-framework/概念_Spring_Framework_结构型设计模式]] — HandlerAdapter 适配器模式。
- [[10-domains/java/spring/主题_Spring知识体系_综述]] — 返回大纲。

