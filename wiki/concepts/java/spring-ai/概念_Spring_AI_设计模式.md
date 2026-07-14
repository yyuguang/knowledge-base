---
type: "concept"
tags:
  - java
  - spring-ai
  - design-pattern
  - architecture
summary: "Spring AI 设计模式整理：Builder/Fluent Interface、责任链、模板方法、策略、适配器、门面、自动配置工厂、Repository、Observation、Prototype、Command 等模式如何服务于厂商隔离、横切能力组合、RAG 流水线扩展和 Spring Boot 可装配性。"
sources:
  - "raw/sai/spring-ai-client-chat/src/main/java/org/springframework/ai/chat/client/DefaultChatClientBuilder.java"
  - "raw/sai/spring-ai-client-chat/src/main/java/org/springframework/ai/chat/client/advisor/DefaultAroundAdvisorChain.java"
  - "raw/sai/spring-ai-client-chat/src/main/java/org/springframework/ai/chat/client/advisor/api/BaseAdvisor.java"
  - "raw/sai/spring-ai-rag/src/main/java/org/springframework/ai/rag/advisor/RetrievalAugmentationAdvisor.java"
  - "raw/sai/spring-ai-model/src/main/java/org/springframework/ai/model/tool/DefaultToolCallingManager.java"
  - "raw/sai/auto-configurations/"
aliases:
  - Spring AI 设计模式
  - Spring AI patterns
  - Spring AI 架构模式
status: "evolving"
confidence: 0.9
created: "2026-05-12 00:00:00"
updated: "2026-07-14 15:05:00"
---

# 概念：Spring AI 设计模式

![[Attachments/spring-ai-design-patterns.svg]]

## 定义

Spring AI 里的设计模式不是为了“套模式”，而是为了解决 AI 应用框架的四类复杂度：

1. **厂商差异**：OpenAI、Anthropic、Ollama、Bedrock、Mistral 等 API 不同，但业务希望只依赖 `ChatModel`。
2. **横切能力**：日志、观测、记忆、RAG、安全、工具调用都要围绕一次模型调用生效，但不能散落到业务代码里。
3. **流程可变**：RAG、Tool Calling、Structured Output 都是多步骤流程，每个步骤都需要可替换。
4. **Spring Boot 装配**：用户希望引入 starter + 写配置即可使用，同时保留自定义 Bean 覆盖点。

因此 Spring AI 的设计模式核心不是 GoF 清单本身，而是“用熟悉的 Spring 模式把 AI 调用变成可组合、可替换、可观测的应用基础设施”。

## 模式总览

| 模式                               | Spring AI 源码位置                                                                                                         | 解决的问题                                             | 带来的好处                                        |
| -------------------------------- | ---------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------- | -------------------------------------------- |
| Builder + Fluent Interface       | `ChatClient.Builder`、`DefaultChatClientBuilder`、各类 `*.builder()`                                                       | ChatClient 默认配置很多，直接构造会爆炸                         | 配置可读、可链式组合，构建后对象不可变                          |
| Facade 门面                        | `ChatClient`                                                                                                           | 底层 Prompt、Options、Advisor、Tool、Observation 很复杂    | 给业务一个简洁统一入口                                  |
| Chain of Responsibility 责任链      | `DefaultAroundAdvisorChain`、`Advisor`                                                                                  | 多个横切能力需要按顺序包裹模型调用                                 | 日志/记忆/RAG/工具/安全解耦，可按 order 组合                |
| Template Method 模板方法             | `BaseAdvisor.adviseCall/adviseStream`                                                                                  | 每个 Advisor 都有相似的调用骨架                              | 固定 before → next → after，子类只关心增强点            |
| Strategy 策略                      | RAG 的 `QueryTransformer`、`QueryExpander`、`DocumentRetriever`、`DocumentJoiner`、`DocumentPostProcessor`、`QueryAugmenter` | RAG 流程步骤多且算法可变                                    | 每一步独立替换，形成 Modular RAG                       |
| Adapter 适配器                      | `OpenAiChatModel`、各 provider model、`McpToolUtils`、VectorStore 实现                                                       | 外部模型/向量库/MCP 协议形态各异                               | 统一到 `ChatModel`、`VectorStore`、`ToolCallback` |
| Factory + DI / AutoConfiguration | `OpenAiChatAutoConfiguration`、`ChatClientAutoConfiguration`、各 starter                                                  | 用户不想手动组装 provider client                          | 根据 classpath/property 自动创建 Bean，并允许覆盖        |
| Repository                       | `ChatMemoryRepository`                                                                                                 | 对话记忆要支持 JDBC/Redis/Mongo/Milvus 等后端               | 记忆策略与存储实现解耦                                  |
| Observer / Observation           | Micrometer `Observation`                                                                                               | AI 调用链跨 ChatClient/Advisor/Model/Tool/VectorStore | 全链路观测、指标、日志、trace 一致                         |
| Prototype                        | `@Scope("prototype") ChatClient.Builder`                                                                               | Builder 是可变对象，单例会污染配置                             | 每个注入点拿到独立 builder                            |
| Command                          | `ToolCallback`、`MethodToolCallback`                                                                                    | LLM 要调用外部行为，但行为来源多样                               | 工具被封装成统一可执行对象                                |
| Decorator/AOP 风格                 | Advisor 包裹 `ChatModel` 调用                                                                                              | 横切增强不应侵入模型实现                                      | 功能叠加不改核心接口                                   |

## Builder + Fluent Interface

代表源码：

- `ChatClient.Builder`
- `DefaultChatClientBuilder`
- `DefaultChatClientRequestSpec`
- `ToolCallAdvisor.builder()`
- `RetrievalAugmentationAdvisor.builder()`
- `SearchRequest.builder()`

Spring AI 需要 Builder，因为一次 AI 调用的配置维度太多：system prompt、user prompt、media、tools、tool context、advisors、options、template renderer、observation convention。直接构造函数会变成十几个参数，而且很难表达默认配置与本次配置的差异。

好处：

- **可读性**：`ChatClient.builder(chatModel).defaultSystem(...).defaultTools(...).defaultAdvisors(...)` 像配置 DSL。
- **默认配置复用**：`DefaultChatClientBuilder` 把 default 配置写入 `defaultRequest`，每次 `prompt()` 复制它。
- **构建后不可变**：业务复用的是 `ChatClient`，而不是可变 builder，线程安全边界清晰。
- **渐进增强**：从最简单的 `user().call().content()` 到 tools/RAG/memory 都是同一条 API。

为什么这样设计：Spring AI 对标的是 Spring `RestClient` / `WebClient` 的开发体验。AI 调用本质上比 HTTP 调用多了工具、记忆、输出格式等配置，如果没有 fluent builder，业务侧会快速退化成“配置对象拼装代码”。

## Facade 门面

代表源码：

- `ChatClient`
- `DefaultChatClient`

`ChatClient` 是门面。业务代码不需要直接理解 `Prompt`、`Message`、`ChatOptions`、`ToolCallingChatOptions`、`DefaultAroundAdvisorChain`、`Observation` 的细节，只需要：

```java
chatClient.prompt().user("...").call().content();
```

好处：

- **隐藏复杂性**：底层仍然有 Prompt、Advisor、Tool、Observation，但业务入口很薄。
- **保留逃生口**：需要时仍可拿 `chatResponse()`、`chatClientResponse()`、`entity()`、`stream()`。
- **统一同步/流式体验**：`call()` 和 `stream()` 分支不同，但 API 风格一致。

为什么这样设计：AI 框架如果只暴露底层模型接口，使用者会在每个调用点重复处理上下文、工具、结构化输出和观测。门面把“常见路径”固定下来，同时不剥夺高级用法。

## 责任链 + AOP 风格

代表源码：

- `Advisor`
- `CallAdvisor` / `StreamAdvisor`
- `DefaultAroundAdvisorChain`
- `ChatModelCallAdvisor` / `ChatModelStreamAdvisor`

Advisor 链让每个横切能力都能围绕模型调用生效：

```text
SimpleLoggerAdvisor
MessageChatMemoryAdvisor
RetrievalAugmentationAdvisor
ToolCallAdvisor
ChatModelCallAdvisor
```

好处：

- **横切能力解耦**：RAG 不需要知道记忆如何实现，日志不需要知道 Tool Calling。
- **顺序可控**：通过 `Ordered` / `getOrder()` 控制执行位置。
- **统一同步和流式扩展**：同一个 Advisor 可以实现 `CallAdvisor` 和 `StreamAdvisor`。
- **递归能力**：`copy(after)` 支持 ToolCallAdvisor 跳过自己、重新进入剩余链。

为什么这样设计：AI 调用天然会长出很多“调用前/调用后”的能力。如果这些逻辑散落在 ChatClient 或 ChatModel 里，框架会变成巨大的条件分支。责任链把复杂度分摊到独立 Advisor 中。

## 模板方法

代表源码：

- `BaseAdvisor.adviseCall()`
- `BaseAdvisor.adviseStream()`
- 子类实现 `before()` / `after()`

`BaseAdvisor` 固定了调用骨架：

```text
before(request)
→ chain.nextCall(processedRequest)
→ after(response)
```

流式调用也固定了“先 before，再 nextStream，最终 chunk 才 after”的骨架。

好处：

- **减少重复代码**：普通 Advisor 不必反复写链式调用模板。
- **降低扩展风险**：开发者主要关注请求改写和响应后处理。
- **行为一致**：日志、记忆、RAG 等 Advisor 都遵守同一生命周期。

为什么这样设计：Advisor 是扩展点，扩展点最怕每个实现都自己写控制流。模板方法把控制流收回框架，只把变化点开放出去。

## 策略模式

代表源码：

- `QueryTransformer`
- `QueryExpander`
- `DocumentRetriever`
- `DocumentJoiner`
- `DocumentPostProcessor`
- `QueryAugmenter`

`RetrievalAugmentationAdvisor` 本身不是某一种 RAG 算法，而是一个策略编排器。它把 RAG 拆成多个可替换策略：

```text
transform → expand → retrieve → join → post-process → augment
```

好处：

- **算法替换成本低**：需要 HyDE、多查询、重排、压缩时替换对应策略。
- **实验友好**：RAG 质量优化可以逐步替换单个环节，而不是重写整条链。
- **生产可控**：不同业务域可以用不同 retriever、joiner、post processor。

为什么这样设计：RAG 没有唯一正确流程。Spring AI 选择 Modular RAG，是为了避免把“默认 RAG 示例代码”固化成框架不可改的流程。

## 适配器模式

代表源码：

- `OpenAiChatModel`、`AnthropicChatModel`、`OllamaChatModel` 等 provider model
- `PgVectorStore`、`RedisVectorStore`、`MilvusVectorStore` 等 vector store
- `McpToolUtils`
- `SyncMcpToolCallbackProvider`

适配器负责把外部世界转成 Spring AI 内部抽象：

| 外部差异 | 内部抽象 |
|----------|----------|
| OpenAI / Anthropic / Ollama API | `ChatModel.call(Prompt)` |
| 各向量数据库查询语法 | `VectorStore.similaritySearch(SearchRequest)` |
| MCP Tool schema/call | `ToolCallback` / `ToolDefinition` |
| Java 方法工具 | `MethodToolCallback` |

好处：

- **业务代码稳定**：切换 provider 不影响 ChatClient 调用方式。
- **生态可接入**：新模型、新向量库、新协议可以作为新适配器加入。
- **统一观测与工具机制**：适配后进入同一套 Observation / Advisor / ToolCallingManager。

为什么这样设计：AI 基础设施处于快速变化期，框架不能假设某个厂商 API 是中心。适配器让 Spring AI 把变化隔离在 provider 层。

## AutoConfiguration 作为工厂

代表源码：

- `OpenAiChatAutoConfiguration`
- `ChatClientAutoConfiguration`
- `PgVectorStoreAutoConfiguration`
- MCP client/server auto configuration

Spring Boot AutoConfiguration 是 Spring AI 的工厂与装配层。它基于 classpath、properties、已有 Bean 决定创建什么对象。

好处：

- **依赖即用**：引入 starter 后自动创建常用 Bean。
- **可覆盖**：`@ConditionalOnMissingBean` 保留用户自定义入口。
- **环境可切换**：用 property 切换模型或向量库，而不是改业务代码。
- **和 Spring 生态一致**：DataSource、JdbcTemplate、ObservationRegistry、Customizer 都能被复用。

为什么这样设计：Spring AI 的目标用户大多在 Spring Boot 应用里。手动 new provider client 会绕开 Boot 的配置、生命周期和观测体系。自动配置让 AI 能力成为 Spring 应用的基础设施 Bean。

## Repository 模式

代表源码：

- `ChatMemoryRepository`
- `InMemoryChatMemoryRepository`
- JDBC / Redis / MongoDB / Cassandra / Neo4j / CosmosDB memory repository

`ChatMemory` 处理“怎么保留对话窗口”，`ChatMemoryRepository` 处理“消息存在哪里”。这是典型 Repository 分离。

好处：

- **策略与存储解耦**：`MessageWindowChatMemory` 不关心底层是 Redis 还是 JDBC。
- **生产迁移方便**：从内存切到持久化后端，不必重写 Advisor。
- **测试简单**：单测用 in-memory repository，生产换分布式存储。

为什么这样设计：对话记忆既有算法问题，也有存储问题。把两者绑死会让简单场景变重，也让生产场景不可靠。

## Observer / Observation

代表源码：

- `ChatClientObservationDocumentation`
- `AdvisorObservationDocumentation`
- `ToolCallingObservationDocumentation`
- `VectorStoreObservationDocumentation`

Micrometer Observation 让 Spring AI 的调用链可观测：

```text
ChatClient
  Advisor
    ChatModel
    ToolCallback
    VectorStore
```

好处：

- **问题定位**：能区分慢在模型、RAG 检索、工具调用还是向量库。
- **成本治理**：ChatModel metadata 可关联 token、usage、provider。
- **生产可运维**：接入 tracing/metrics 后，AI 调用不再是黑盒。

为什么这样设计：AI 应用的失败经常不是异常，而是慢、贵、召回差、工具不稳定。没有观测，框架很难进入生产系统。

## Prototype 模式

代表源码：

- `ChatClientAutoConfiguration.chatClientBuilder()`
- `@Scope("prototype")`

`ChatClient.Builder` 是可变对象。如果作为单例 Bean 注入，A 模块设置的 default advisor 可能污染 B 模块。Spring AI 将 builder 声明为 prototype，每个注入点拿到独立 builder。

好处：

- **避免配置串扰**：不同业务模块可以独立构造 ChatClient。
- **保持 builder 易用性**：builder 仍然可变，但生命周期很短。
- **最终对象安全**：构建出的 `ChatClient` 才是可复用不可变对象。

为什么这样设计：Spring 里可变 builder 做单例通常是陷阱。prototype 是保留开发体验与线程安全边界之间的折中。

## Command 模式

代表源码：

- `ToolCallback`
- `MethodToolCallback`
- `FunctionToolCallback`
- `DefaultToolCallingManager`

`ToolCallback` 把一个可执行动作封装成统一对象：有名字、描述、输入 schema、元数据和 `call()` 方法。LLM 不关心这个工具来自 Java 方法、函数对象、MCP server 还是 Spring Bean。

好处：

- **工具来源统一**：本地方法、函数、MCP 工具都进入同一执行器。
- **执行可观测**：每个 tool call 都能被 Observation 包裹。
- **元数据可治理**：`returnDirect`、tool name、schema 都随命令对象一起携带。

为什么这样设计：Tool Calling 的核心是“模型选择一个动作，框架执行它”。Command 模式正好把动作变成一等对象。

## 组合后的整体收益

这些模式组合起来，带来的不是单点优雅，而是系统性收益：

- **开闭原则**：新增模型厂商、新向量库、新 Advisor、新 RAG 策略时，尽量新增实现而不是改核心调用链。
- **关注点分离**：ChatClient 管入口，Advisor 管横切，ChatModel 管模型，VectorStore 管知识库，ToolCallback 管动作。
- **生产友好**：AutoConfiguration、Observation、Repository、Prototype 都是为了让框架进入真实 Spring Boot 服务。
- **学习成本可控**：用户可以先学 ChatClient，再逐步理解 Advisor、Tool、RAG、VectorStore。

## 常见误区

1. **“Spring AI 只是大量 Builder”**：Builder 只是入口体验；真正的架构核心是 ChatClient 门面 + Advisor 责任链 + provider 适配器。
2. **“Advisor 就是简单拦截器”**：普通 Advisor 接近拦截器，但 ToolCallAdvisor 会递归进入剩余链，复杂度明显更高。
3. **“RAG 是固定流程”**：Spring AI 的 RAG 是策略流水线，默认实现只是可替换策略之一。
4. **“AutoConfiguration 只是省配置”**：它还定义了 Bean 生命周期、覆盖点、provider 选择和组织级 customizer 接入位置。
5. **“模式越多越好”**：Spring AI 的模式都围绕可替换、可组合、可观测服务；没有这些目标，模式就是噪音。

## 和其它概念的关系

- [[concepts/java/spring-ai/概念_Spring_AI_架构设计]] — 设计模式服务于 Spring AI 的分层架构和 provider 可移植性。
- [[concepts/java/spring-ai/概念_Spring_AI_ChatClient_API]] — Builder、Fluent Interface、Facade 在 ChatClient 上集中体现。
- [[concepts/java/spring-ai/概念_Spring_AI_Advisor拦截链]] — 责任链、模板方法、AOP 风格是 Advisor 的核心设计。
- [[concepts/java/spring-ai/概念_Spring_AI_RAG检索增强生成]] — RAG 的 QueryTransformer、Retriever、Joiner 等是策略模式。
- [[concepts/java/spring-ai/概念_Spring_AI_ToolCalling工具调用]] — ToolCallback 体现 Command，MethodToolCallback 体现适配与反射封装。
- [[concepts/java/spring-ai/概念_Spring_AI_AutoConfiguration自动配置]] — AutoConfiguration 是工厂、依赖注入和覆盖点设计的集中位置。
- [[overview/java/spring-ai/主题_Spring_AI源码架构_综述]] — 设计模式是源码主线背后的结构性解释。

## 来源

- `raw/sai/spring-ai-client-chat/src/main/java/org/springframework/ai/chat/client/DefaultChatClientBuilder.java`
- `raw/sai/spring-ai-client-chat/src/main/java/org/springframework/ai/chat/client/DefaultChatClient.java`
- `raw/sai/spring-ai-client-chat/src/main/java/org/springframework/ai/chat/client/advisor/DefaultAroundAdvisorChain.java`
- `raw/sai/spring-ai-client-chat/src/main/java/org/springframework/ai/chat/client/advisor/api/BaseAdvisor.java`
- `raw/sai/spring-ai-rag/src/main/java/org/springframework/ai/rag/advisor/RetrievalAugmentationAdvisor.java`
- `raw/sai/spring-ai-model/src/main/java/org/springframework/ai/model/tool/DefaultToolCallingManager.java`
- `raw/sai/spring-ai-model/src/main/java/org/springframework/ai/tool/method/MethodToolCallback.java`
