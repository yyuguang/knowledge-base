---
type: "concept"
tags:
  - java
  - spring-ai
  - spring-boot
  - auto-configuration
summary: "Spring AI AutoConfiguration 自动配置：Starter 引入 autoconfigure + provider 模块，AutoConfiguration 通过 ConditionalOnClass/ConditionalOnProperty/ConditionalOnMissingBean 创建 ChatModel、ChatClient.Builder、VectorStore 等 Bean，实现依赖即用和厂商切换。"
sources:
  - "raw/code/spring-ai/starters/spring-ai-starter-model-openai/pom.xml"
  - "raw/code/spring-ai/auto-configurations/models/spring-ai-autoconfigure-model-openai/src/main/java/org/springframework/ai/model/openai/autoconfigure/OpenAiChatAutoConfiguration.java"
  - "raw/code/spring-ai/auto-configurations/models/chat/client/spring-ai-autoconfigure-model-chat-client/src/main/java/org/springframework/ai/model/chat/client/autoconfigure/ChatClientAutoConfiguration.java"
  - "raw/code/spring-ai/auto-configurations/vector-stores/spring-ai-autoconfigure-vector-store-pgvector/src/main/java/org/springframework/ai/vectorstore/pgvector/autoconfigure/PgVectorStoreAutoConfiguration.java"
aliases:
  - Spring AI 自动配置
  - Spring AI Starter
  - AutoConfiguration
status: "evolving"
confidence: 0.9
created: "2026-05-12 00:00:00"
updated: "2026-05-12 00:00:00"
---

# 概念：Spring AI AutoConfiguration 自动配置

![[Attachments/spring-ai-autoconfiguration-chain.svg]]

## 定义

Spring AI 的 AutoConfiguration 是它接入 Spring Boot 生态的关键层：用户引入某个 starter 后，classpath 中出现对应模型、向量库或 MCP 模块，Spring Boot 根据 `AutoConfiguration.imports` 加载自动配置类，再按条件创建 `ChatModel`、`EmbeddingModel`、`VectorStore`、`ChatClient.Builder` 等 Bean。

这使 Spring AI 的使用方式从“手写 SDK client + 手动组装模型对象”变成“引入依赖 + 配置属性 + 注入接口”。

## 为什么重要

Spring AI 的“可移植性”不是只靠接口完成的。接口负责隔离代码，AutoConfiguration 负责把具体实现装配进容器。

如果没有自动配置，用户仍然要自己创建 OpenAI client、配置超时和重试、注册 Observation、装配 ToolCallingManager。Spring AI 把这些重复工作放到 Boot 条件装配中，让业务只注入 `ChatModel` 或 `ChatClient.Builder`。

## 核心机制

### 1. Starter 不是实现本体，而是依赖组合

以 `spring-ai-starter-model-openai` 为例，它主要引入：

| 依赖 | 职责 |
|------|------|
| `spring-boot-starter` | Boot 基础能力 |
| `spring-ai-autoconfigure-model-openai` | OpenAI 自动配置 |
| `spring-ai-openai` | OpenAI 模型实现与 API 封装 |
| `spring-ai-autoconfigure-model-chat-client` | ChatClient.Builder 自动配置 |
| `spring-ai-autoconfigure-model-chat-memory` | ChatMemory 自动配置 |

所以 starter 的本质是“可工作的默认依赖包”。真正创建 Bean 的逻辑在 auto-configurations 模块里。

### 2. 模型自动配置：OpenAiChatAutoConfiguration

`OpenAiChatAutoConfiguration` 的条件结构很典型：

```java
@AutoConfiguration
@EnableConfigurationProperties({ OpenAiCommonProperties.class, OpenAiChatProperties.class })
@ConditionalOnProperty(
    name = SpringAIModelProperties.CHAT_MODEL,
    havingValue = SpringAIModels.OPENAI,
    matchIfMissing = true)
class OpenAiChatAutoConfiguration {

    @Bean
    @ConditionalOnMissingBean
    OpenAiChatModel openAiChatModel(...) { ... }
}
```

它表达了三层语义：

- `@EnableConfigurationProperties`：把 `spring.ai.openai.*` 配置绑定成属性对象。
- `@ConditionalOnProperty`：只有当前 chat model 选择 OpenAI 时才启用；`matchIfMissing = true` 让 OpenAI 可作为默认模型。
- `@ConditionalOnMissingBean`：用户自己声明 `OpenAiChatModel` 时，框架不抢控制权。

创建模型时会同时注入 `ToolCallingManager`、`ObservationRegistry`、`ToolExecutionEligibilityPredicate`，说明工具调用和可观测性是模型调用链的一等能力，不是外围补丁。

### 3. ChatClient.Builder 是 prototype

`ChatClientAutoConfiguration` 会创建 `ChatClient.Builder`：

```java
@Bean
@Scope("prototype")
@ConditionalOnMissingBean
ChatClient.Builder chatClientBuilder(...)
```

`prototype` 很重要：每个注入点拿到的是一个新 builder，避免不同业务模块对默认 system prompt、advisor、tool 的配置互相污染。最终 `ChatClient` 仍然应构造成不可变客户端复用。

它还会接入 `ChatClientCustomizer`，因此团队可以用一个全局 customizer 统一加日志、审计、默认 Advisor 或观测配置。

### 4. VectorStore 自动配置同样遵循条件装配

以 PGVector 为例：

```java
@AutoConfiguration
@ConditionalOnClass({ PgVectorStore.class, DataSource.class, JdbcTemplate.class })
@EnableConfigurationProperties(PgVectorStoreProperties.class)
@ConditionalOnProperty(
    name = SpringAIVectorStoreTypes.TYPE,
    havingValue = SpringAIVectorStoreTypes.PGVECTOR,
    matchIfMissing = true)
class PgVectorStoreAutoConfiguration { ... }
```

它要求 classpath 中有 `PgVectorStore`、`DataSource`、`JdbcTemplate`，并根据 `spring.ai.vectorstore.type` 选择实现。构建 `PgVectorStore` 时还会注入 `EmbeddingModel`、`BatchingStrategy`、`ObservationRegistry` 和向量表参数。

这说明 Spring AI 把 RAG 的检索层也纳入了 Spring Boot 的装配模型：换向量库通常是换 starter + 改配置，而不是重写 RAG 业务代码。

## 使用场景

- 开发环境用 Ollama、本地 embedding、简单向量存储；生产切 OpenAI/Azure + PGVector。
- 统一企业默认配置：用 `ChatClientCustomizer` 加默认 Advisor、观测和安全策略。
- 多模型实验：通过 `spring.ai.model.chat` 等配置选择 provider。
- 接入已有基础设施：复用 Boot 中已有的 `DataSource`、`JdbcTemplate`、`ObservationRegistry`。

## 常见误区

1. **“引入 starter 就等于只引入一个 jar”**：starter 更像依赖菜单，通常同时拉入实现模块、自动配置模块和配套基础能力。
2. **“有多个模型 starter 会自动多模型并存”**：默认自动配置往往按 `spring.ai.model.*` 条件选择，多个 provider 并存时需要明确配置或手工声明 Bean。
3. **“ChatClient.Builder 是单例所以可以随便改”**：它是 prototype，适合每个注入点独立构造自己的不可变 `ChatClient`。
4. **“自定义 Bean 会和自动配置冲突”**：大多数核心 Bean 有 `@ConditionalOnMissingBean`，用户自定义 Bean 通常会覆盖默认装配。

## 和其它概念的关系

- [[concepts/java/spring-ai/概念_Spring_AI_架构设计]] — AutoConfiguration 是“接口抽象”落地为 Spring Bean 的装配层。
- [[concepts/java/spring-ai/概念_Spring_AI_ChatClient_API]] — ChatClient.Builder 由自动配置以 prototype 形式注入。
- [[concepts/java/spring-ai/概念_Spring_AI_VectorStore向量存储]] — 向量存储实现也通过自动配置选择和创建。
- [[overview/java/spring-ai/主题_Spring_AI源码架构_综述]] — 自动配置在完整源码主线中的入口位置。

## 我的理解

Spring AI 的自动配置层把“AI 能力”驯化成了 Spring Boot 应用的普通基础设施。它最重要的不是省几行 new 对象代码，而是把 provider 选择、属性绑定、观测、工具调用管理、默认 builder 和用户覆盖点都放进同一个可预测的 Boot 模型里。

## 源码级判断

自动配置层的源码要按“Bean 依赖图”读，而不是按单个类读：

- `OpenAiChatAutoConfiguration.openAiChatModel()` 同时依赖 provider properties、`ToolCallingManager`、`ObservationRegistry`、`ChatModelObservationConvention` 和 `ToolExecutionEligibilityPredicate`。
- `ChatClientAutoConfiguration.chatClientBuilder()` 依赖 `ChatModel`，所以模型 Bean 必须先能被条件装配出来。
- `ChatClientBuilderConfigurer` 会收集所有 `ChatClientCustomizer`，这是组织级统一默认 Advisor/Tool/Observation 的入口。
- `PgVectorStoreAutoConfiguration.vectorStore()` 依赖 `JdbcTemplate` 和 `EmbeddingModel`，因此向量库自动配置天然绑定 embedding provider。
- `@ConditionalOnMissingBean` 是用户覆盖点，不是边缘细节；生产项目需要知道哪些 Bean 是可以替换的。

## 来源

- `raw/code/spring-ai/starters/spring-ai-starter-model-openai/pom.xml`
- `raw/code/spring-ai/auto-configurations/models/spring-ai-autoconfigure-model-openai/src/main/java/org/springframework/ai/model/openai/autoconfigure/OpenAiChatAutoConfiguration.java`
- `raw/code/spring-ai/auto-configurations/models/chat/client/spring-ai-autoconfigure-model-chat-client/src/main/java/org/springframework/ai/model/chat/client/autoconfigure/ChatClientAutoConfiguration.java`
- `raw/code/spring-ai/auto-configurations/vector-stores/spring-ai-autoconfigure-vector-store-pgvector/src/main/java/org/springframework/ai/vectorstore/pgvector/autoconfigure/PgVectorStoreAutoConfiguration.java`
