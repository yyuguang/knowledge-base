---
type: "overview"
tags:
  - java
  - spring
  - spring-framework
  - spring-boot
  - interview
summary: "按 0-6 编号组织的 Spring 知识体系：根基、核心容器、AOP、数据访问、Web、Spring Boot 内核、扩展进阶；把每个知识点扩充为核心问题、源码锚点、流程图、面试表达与后续源码待补。"
sources:
  - "raw/code/spring-framework"
  - "raw/code/spring-ai"
aliases:
  - "Spring 知识大纲"
  - "Spring 面试知识体系"
status: "evolving"
confidence: 0.82
created: "2026-05-12 21:22:48"
updated: "2026-05-12 21:22:48"
---

# 主题：Spring知识体系 综述

![[Attachments/spring-knowledge-outline.excalidraw]]

![[Attachments/spring-runtime-core-flows.excalidraw]]

## 当前结论

这套 Spring 大纲可以被整理成四条主线：

1. **对象线**：IoC/DI → BeanDefinition → BeanFactory → Bean 生命周期 → 自动装配 → 循环依赖。
2. **增强线**：动态代理 → AOP 通知/切点 → Advisor/Interceptor 链 → `@Transactional`。
3. **请求线**：DispatcherServlet → HandlerMapping → HandlerAdapter → 参数解析 → 返回值处理 → Interceptor/Filter。
4. **工程化线**：资源属性解析 → Spring Boot 自动配置/条件注解/Starter → 事件、FactoryBean、后置处理器、SPI。

## 结构化目录

### 0_Spring根基

- `00_IoC与DI思想`：容器存在的原因不是“帮忙 new 对象”，而是接管对象创建、依赖关系、生命周期和扩展点。
- `01_Bean的生命周期`：BeanDefinition 被加载后，经过实例化、属性填充、Aware、初始化、后置处理、代理、销毁注册。
- `02_资源与属性解析`：`Resource`、`Environment`、`PropertySource`、占位符和 YAML 让外部配置进入容器。

详见：[[concepts/java/spring/概念_Spring根基_IoC_DI_Bean生命周期_资源属性]]

### 1_核心容器

- `11_BeanFactory与ApplicationContext区别`：BeanFactory 是对象工厂，ApplicationContext 是应用上下文协议。
- `12_依赖注入方式`：构造器、Setter、字段注入的生命周期差异。
- `13_自动装配机制`：`AutowiredAnnotationBeanPostProcessor` + `DefaultListableBeanFactory.resolveDependency()`。
- `14_循环依赖解决`：三级缓存解决部分 singleton setter/field 循环依赖。

详见：[[concepts/java/spring/概念_Spring核心容器_BeanFactory_依赖注入_自动装配_循环依赖]]

### 2_AOP模块

- `21_动态代理`：JDK 动态代理基于接口，CGLIB/Objenesis 基于类代理。
- `22_通知类型与执行顺序`：Advice 被转换成拦截器链，按 order 和调用栈执行。
- `23_切入点表达式与@Aspect`：Pointcut 决定哪些 join point 需要增强。
- `24_@Transactional代理原理`：事务是 `TransactionInterceptor` 包裹目标方法。

详见：[[concepts/java/spring/概念_Spring_AOP模块_动态代理_通知_切点_事务代理]]

### 3_数据访问

- `31_JdbcTemplate与数据源配置`：模板方法管理 Connection/Statement/异常转换/资源释放。
- `32_声明式事务传播行为`：`AbstractPlatformTransactionManager` 根据传播行为加入、新建、挂起、嵌套或拒绝事务。
- `33_MyBatis整合原理`：核心是扫描 Mapper 接口，注册 MapperFactoryBean，生成代理对象；本仓库缺 `mybatis-spring` 源码，后续需补 raw。
- `34_多数据源与读写分离`：`AbstractRoutingDataSource` 根据 lookup key 路由到目标 DataSource。

详见：[[concepts/java/spring/概念_Spring数据访问_JdbcTemplate_事务传播_MyBatis_多数据源]]

### 4_Web层

- `41_DispatcherServlet处理流程`：中央分发，负责查找 handler、adapter、异常处理和视图/响应。
- `42_HandlerMapping与适配器模式`：Mapping 找 handler，Adapter 屏蔽 handler 调用差异。
- `43_参数解析器与返回值处理器`：Controller 方法调用由 resolver/handler 组合完成。
- `44_拦截器与过滤器链`：Filter 属于 Servlet 容器链，Interceptor 属于 Spring MVC handler 链。

详见：[[concepts/java/spring/概念_Spring_Web层_DispatcherServlet_HandlerMapping_参数返回值_过滤器拦截器]]

### 5_Spring Boot内核

- `51_自动配置原理`：通过 AutoConfiguration 导入候选配置类，再由条件注解决定是否生效。
- `52_条件注解`：`@ConditionalOnClass`、`@ConditionalOnMissingBean`、`@ConditionalOnProperty` 等用于“有条件地注册 Bean”。
- `53_启动过程`：`SpringApplication.run()` 准备环境、创建 ApplicationContext、加载 BeanDefinition、refresh、发布事件。
- `54_自定义Starter`：依赖 + auto-configuration + properties + metadata/imports。

详见：[[concepts/java/spring/概念_Spring_Boot内核_自动配置_条件注解_启动过程_Starter]]

注意：本仓库当前没有 `raw/code/spring-boot`，Boot 部分先基于 Spring 机制和 `raw/code/spring-ai` 中的 Boot auto-configuration 样本整理，源码级深挖需要后续引入 Spring Boot 源码。

### 6_扩展与进阶

- `61_事件监听机制`：`ApplicationEventPublisher` + `ApplicationEventMulticaster` + `ApplicationListener`。
- `62_FactoryBean`：容器管理工厂 Bean，注入时暴露其产品对象。
- `63_后置处理器`：`BeanFactoryPostProcessor` 改 BeanDefinition，`BeanPostProcessor` 改 Bean 实例。
- `64_SPI与Spring Factories`：`SpringFactoriesLoader` 从 `META-INF/spring.factories` 加载扩展。

详见：[[concepts/java/spring/概念_Spring扩展进阶_事件_FactoryBean_后置处理器_SPI]]

## 学习顺序建议

1. 先学 0/1：理解容器为什么存在，以及 Bean 如何从定义变成对象。
2. 再学 2/3：理解代理如何把事务和数据访问接入业务方法。
3. 再学 4：理解 HTTP 请求如何变成 Controller 方法调用。
4. 再学 5：理解 Boot 如何在 Framework 基础上做自动装配。
5. 最后学 6：理解如何扩展 Spring，而不是只会使用注解。

## 源码边界

已读源码：

- `raw/code/spring-framework`：IoC、AOP、事务、JDBC、Web、事件、资源、SPI。
- `raw/code/spring-ai`：Spring Boot auto-configuration 的局部样本。

待补源码：

- `raw/code/spring-boot`：启动流程、自动配置导入、条件注解实现、Starter 元数据。
- `raw/code/mybatis-spring`：MapperScan、MapperFactoryBean、SqlSessionTemplate、事务同步细节。

## 相关链接

- [[sources/java/spring-framework/来源_Spring_Framework_源码]] — Spring Framework 源码入口。
- [[overview/java/spring-framework/主题_Spring_Framework源码学习_综述]] — 已有源码学习综述。
- [[overview/java/spring-framework/主题_Spring_Framework设计模式_综述]] — 设计模式专题。
- [[concepts/java/spring-ai/概念_Spring_AI_AutoConfiguration自动配置]] — Spring AI 中的 Boot 自动配置样本。

