---
type: "concept"
tags:
  - java
  - spring
  - spring-boot
  - auto-configuration
  - starter
summary: "Spring Boot 内核：自动配置通过候选 AutoConfiguration 类和条件注解决定注册哪些 Bean；SpringApplication.run 负责准备环境、创建上下文、加载 BeanDefinition、刷新上下文；自定义 Starter 由依赖、自动配置类、条件注解、属性类和 AutoConfiguration.imports 组成。"
sources:
  - "sai/auto-configurations"
  - "sai/starters"
aliases:
  - "Spring Boot 内核"
  - "自动配置 条件注解 启动过程 Starter"
status: "draft"
confidence: 0.7
created: "2026-05-12 21:22:48"
updated: "2026-05-12 21:22:48"
---

# 概念：Spring Boot内核 自动配置 条件注解 启动过程 Starter

## 来源边界

本仓库当前没有 `raw/code/spring-boot`，因此本页先按 Spring Boot 机制和 `sai` 中的 auto-configuration/starter 样本整理。`SpringApplication.run()`、`AutoConfigurationImportSelector`、`ConditionEvaluator` 等源码级深挖需要后续引入 Spring Boot 源码。

## 51_自动配置原理（@EnableAutoConfiguration）

自动配置的核心思想：

```text
引入 starter
  → classpath 出现某些类
  → Boot 发现 AutoConfiguration 候选
  → 条件注解判断是否生效
  → 注册默认 Bean
  → 用户自定义 Bean 可覆盖默认 Bean
```

现代 Spring Boot 通常通过：

```text
META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports
```

列出自动配置类。`sai` 中大量 auto-configuration 模块使用这种方式，例如向量库、模型、tool calling 等自动配置。

## 52_条件注解（@Conditional族）

常见条件：

| 条件注解 | 作用 |
|---|---|
| `@ConditionalOnClass` | classpath 存在某类时生效 |
| `@ConditionalOnMissingClass` | classpath 不存在某类时生效 |
| `@ConditionalOnBean` | 容器存在某 Bean 时生效 |
| `@ConditionalOnMissingBean` | 容器不存在某 Bean 时注册默认 Bean |
| `@ConditionalOnProperty` | 配置属性满足条件时生效 |
| `@ConditionalOnResource` | 资源存在时生效 |
| `@ConditionalOnWebApplication` | Web 应用环境生效 |

为什么重要：

- Starter 可以“引入即用”。
- 用户提供 Bean 时自动配置退让。
- 同一套 starter 可适配不同 classpath 和配置环境。

## 53_启动过程（SpringApplication run方法）

机制级流程：

```text
main()
  → SpringApplication.run()
  → 创建 SpringApplication
  → 推断应用类型
  → 加载 initializer/listener
  → 准备 Environment
  → 打印 Banner
  → 创建 ApplicationContext
  → 加载 BeanDefinition
  → refresh()
  → 调用 runners
  → 发布启动完成事件
```

其中 `refresh()` 仍然回到 Spring Framework 的 `AbstractApplicationContext.refresh()`，所以 Boot 不是替代 Framework，而是在 Framework 之上做约定、环境、自动配置和启动编排。

## 54_自定义Starter

一个 Starter 通常包含：

1. starter 依赖模块：聚合依赖，不写太多逻辑。
2. autoconfigure 模块：提供 `@AutoConfiguration` 类。
3. properties 类：绑定 `spring.xxx.*` 配置。
4. 条件注解：控制默认 Bean 是否注册。
5. `AutoConfiguration.imports`：声明自动配置入口。
6. metadata：辅助 IDE 提示。

典型自动配置类结构：

```java
@AutoConfiguration
@ConditionalOnClass(SomeClient.class)
@ConditionalOnProperty(prefix = "xxx", name = "enabled", matchIfMissing = true)
public class XxxAutoConfiguration {

    @Bean
    @ConditionalOnMissingBean
    XxxClient xxxClient(XxxProperties properties) {
        return new XxxClient(properties);
    }
}
```

## 面试表达

> Spring Boot 自动配置不是魔法，本质是根据 classpath、已有 Bean 和配置属性，有条件地导入一批配置类。Starter 负责把依赖和自动配置放到 classpath，AutoConfiguration 负责注册默认 Bean，Conditional 注解负责判断是否生效，`@ConditionalOnMissingBean` 保证用户自定义 Bean 优先。

## 常见误区

- Starter 本身通常只是依赖聚合，真正配置逻辑在 autoconfigure 模块。
- 自动配置不是一定生效，条件不满足会跳过。
- Boot 启动最终仍会调用 Spring Framework 的 `refresh()`。
- 本页目前不是 Spring Boot 本体源码级页面，后续需补 `raw/code/spring-boot`。

## 相关链接

- [[concepts/java/spring-ai/概念_Spring_AI_AutoConfiguration自动配置]] — Spring AI 自动配置样本。
- [[concepts/java/spring-framework/概念_Spring_Framework_配置类解析与依赖解析]] — Framework 配置类解析基础。
- [[overview/java/spring/主题_Spring知识体系_综述]] — 返回大纲。

