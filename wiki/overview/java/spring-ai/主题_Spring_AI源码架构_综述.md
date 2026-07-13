---
type: "overview"
tags:
  - java
  - spring-ai
  - source-code
  - architecture
summary: "Spring AI 源码级架构综述：从 AutoConfiguration 装配 Bean，到 ChatClient 构建请求、Advisor 链执行、Tool Calling 递归循环、RAG 检索增强、VectorStore 可移植检索和 MCP 工具桥接。"
sources:
  - "raw/code/spring-ai/spring-ai-client-chat/src/main/java/org/springframework/ai/chat/client/DefaultChatClient.java"
  - "raw/code/spring-ai/spring-ai-client-chat/src/main/java/org/springframework/ai/chat/client/advisor/DefaultAroundAdvisorChain.java"
  - "raw/code/spring-ai/spring-ai-client-chat/src/main/java/org/springframework/ai/chat/client/advisor/ToolCallAdvisor.java"
  - "raw/code/spring-ai/spring-ai-rag/src/main/java/org/springframework/ai/rag/advisor/RetrievalAugmentationAdvisor.java"
  - "raw/code/spring-ai/auto-configurations/"
aliases:
  - Spring AI 源码架构
  - Spring AI 源码综述
status: "evolving"
confidence: 0.9
created: "2026-05-12 00:00:00"
updated: "2026-05-12 00:00:00"
---

# 主题：Spring AI 源码架构 综述

![[Attachments/spring-ai-source-architecture.svg]]

## 当前结论

Spring AI 的源码不是“LLM SDK 包装器”，而是一个典型的 Spring 风格应用框架：底层用 `Model<TReq,TRes>` 抽象统一不同 AI 能力，中层用 `ChatClient` 和 `Advisor` 管理请求编排，上层用 Spring Boot AutoConfiguration 把具体 provider 装进容器。RAG、Tool Calling、ChatMemory、StructuredOutput、MCP 都围绕 `Prompt → AdvisorChain → ChatModel → ChatResponse` 这条主路径挂接。

最重要的理解路径：

1. **装配期**：starter 引入 provider + autoconfigure，Spring Boot 条件装配出 `ChatModel`、`ChatClient.Builder`、`VectorStore`、MCP clients。
2. **构建期**：`ChatClient.Builder` 保存默认 system/user/tools/advisors/options，`build()` 得到不可变 `DefaultChatClient`。
3. **请求期**：每次 `prompt()` 复制默认请求规格，用户再追加本次输入。
4. **执行期**：`call()` / `stream()` 构建 `DefaultAroundAdvisorChain`，临时追加 `ChatModelCallAdvisor` / `ChatModelStreamAdvisor` 作为终端节点。
5. **增强期**：Advisor 在模型调用前后改写请求、维护记忆、执行 RAG、处理工具调用、记录观测。

## 总体框架

| 层 | 关键模块 | 关键类 | 职责 |
|----|----------|--------|------|
| Boot 装配层 | `auto-configurations/*`、`starters/*` | `OpenAiChatAutoConfiguration`、`ChatClientAutoConfiguration`、`PgVectorStoreAutoConfiguration` | 把 provider、client、vector store、MCP transport 变成 Spring Bean |
| Client 编排层 | `spring-ai-client-chat` | `DefaultChatClient`、`DefaultChatClientRequestSpec`、`DefaultAroundAdvisorChain` | 构造请求、组织 Advisor 链、包裹 Observation |
| Model 抽象层 | `spring-ai-model` | `ChatModel`、`Prompt`、`ChatResponse`、`ToolCallingManager` | 定义模型请求/响应、消息、工具和结构化输出 |
| RAG 层 | `spring-ai-rag`、`spring-ai-vector-store` | `RetrievalAugmentationAdvisor`、`VectorStore`、`SearchRequest` | 查询变换、检索、合并、后处理、Prompt 增强 |
| Provider 层 | `models/*`、`vector-stores/*`、`mcp/*` | `OpenAiChatModel`、`PgVectorStore`、`McpToolUtils` | 对接外部模型、向量数据库和 MCP 协议 |

## 核心调用链

![[Attachments/spring-ai-chatclient-call-flow.svg]]

源码关键点：

- `DefaultChatClient.prompt()` 会复制 `defaultChatClientRequest`，让每次请求拥有自己的可变 `RequestSpec`。
- `DefaultChatClientRequestSpec.call()` / `stream()` 会调用 `buildAdvisorChain()`。
- `buildAdvisorChain()` 会在用户配置的 Advisor 后追加 `ChatModelCallAdvisor` 和 `ChatModelStreamAdvisor`。这两个终端 Advisor 不需要用户手工注册。
- `DefaultAroundAdvisorChain` 内部把 Advisor 拆成 `CallAdvisor` 和 `StreamAdvisor` 两条队列，并用 `OrderComparator.sort()` 排序。
- `nextCall()` 使用 `Deque.pop()` 取出当前 Advisor，所以链本身是一次执行的消费型对象；需要递归时使用 `copy(after)` 重新构造剩余链。

## Tool Calling 主线

![[Attachments/spring-ai-tool-calling-loop.svg]]

`ToolCallAdvisor` 的源码设计比普通 Advisor 更特殊：它不是简单的 before/after 包装，而是递归执行循环。

关键机制：

- 它要求 `Prompt.options` 是 `ToolCallingChatOptions`。
- 它复制 options 并设置 `internalToolExecutionEnabled(false)`，避免 ChatModel provider 自己执行工具。
- 它调用 `callAdvisorChain.copy(this).nextCall(processedRequest)`，跳过当前 ToolCallAdvisor，让链后续部分和 ChatModel 执行。
- 如果 `ChatResponse.hasToolCalls()` 为 true，则交给 `ToolCallingManager.executeToolCalls()`。
- `DefaultToolCallingManager` 根据 tool name 查找 `ToolCallback`，用 Observation 包裹 `toolCallback.call(arguments, toolContext)`。
- 执行后构造 `AssistantMessage(toolCalls)` + `ToolResponseMessage(results)`，作为下一轮 instructions 再喂给模型。
- `returnDirect=true` 会中断循环，把工具结果直接构造成最终 generations 返回应用。

## RAG 主线

![[Attachments/spring-ai-rag-pipeline.svg]]

`RetrievalAugmentationAdvisor.before()` 是完整 Modular RAG 流水线，实际有 8 个源码步骤：

| 步骤 | 源码动作 | 结果 |
|------|----------|------|
| 0 | 从 `chatClientRequest.prompt().getUserMessage().getText()`、history、context 构造 `Query` | 得到 originalQuery |
| 1 | 顺序执行 `queryTransformers` | 得到 transformedQuery |
| 2 | `queryExpander.expand()` 或单查询 | 得到 expandedQueries |
| 3 | 对每个 query 用 `CompletableFuture.supplyAsync()` 调 `documentRetriever.retrieve()` | 并发检索文档 |
| 4 | `documentJoiner.join()` | 合并多查询结果 |
| 5 | 顺序执行 `documentPostProcessors` | 重排、过滤或压缩文档 |
| 6 | `context.put(DOCUMENT_CONTEXT, documents)` | 将引用文档放入上下文 |
| 7 | `queryAugmenter.augment()` + `prompt().augmentUserMessage()` | 改写用户消息 |

默认线程池是 `ThreadPoolTaskExecutor`，线程名前缀 `ai-advisor-`，核心线程 4，最大线程 16，并使用 `ContextPropagatingTaskDecorator` 保留上下文。

## AutoConfiguration 主线

![[Attachments/spring-ai-autoconfiguration-chain.svg]]

自动配置体现了 Spring AI 的框架性：

- provider starter 引入 provider 实现、autoconfigure 和 ChatClient/Memory 默认配置。
- 模型自动配置通过 `@ConditionalOnProperty(name = SpringAIModelProperties.CHAT_MODEL, ...)` 选择 provider。
- `@ConditionalOnMissingBean` 允许用户用自定义 Bean 覆盖默认装配。
- `ChatClientAutoConfiguration` 创建 `ChatClient.Builder`，并标注 `@Scope("prototype")`，避免不同注入点共享可变 builder 状态。
- VectorStore 自动配置还会把 `EmbeddingModel`、`BatchingStrategy`、`ObservationRegistry`、数据源配置一起注入。

## 当前判断

Spring AI 的核心价值是“把 AI 应用的横切复杂度放进 Spring 体系”：模型切换靠 AutoConfiguration，Prompt 请求靠 ChatClient，记忆/RAG/工具/安全靠 Advisor，工具执行靠 ToolCallingManager，知识库靠 VectorStore，跨应用工具生态靠 MCP。源码级学习时不应该只看 provider 适配类，而要沿着 `DefaultChatClientRequestSpec.call()` 这条主线往下追。

从设计模式看，这条主线其实是多种模式的组合：ChatClient 是门面，Builder 负责复杂配置，Advisor 是责任链和模板方法，RAG 是策略流水线，provider 是适配器，AutoConfiguration 是 Spring 风格工厂。详见 [[concepts/java/spring-ai/概念_Spring_AI_设计模式]] — 用模式解释这些源码结构为什么这样组织。

## 未解决问题

- 流式 Tool Calling 的聚合与递归虽然源码清楚，但不同 provider 对 tool call chunk 的表示可能有差异，需要结合具体模型实现验证。
- RAG 默认 DocumentJoiner 是拼接式，缺少默认去重/重排，生产质量取决于用户是否配置 post processor。
- VectorStore 的 `similarityThreshold` 是客户端语义，不能假设所有存储都做服务端下推。
- 多 provider 并存时，自动配置默认选择和用户自定义 Bean 的交互需要按具体 starter 验证。

## 下一步

- 深挖 provider 实现页：以 OpenAI 或 Ollama 为样本，追踪 `ChatModel.call()` 如何把 Spring AI `Prompt` 转换为供应商 API 请求。
- 深挖 Observation：整理 ChatClient / Advisor / ChatModel / Tool / VectorStore 的指标命名和上下文字段。
- 深挖 AOT / Native Image：整理 `AiRuntimeHints`、`ToolRuntimeHints` 和 MCP hints 的反射注册逻辑。

## 相关链接

- [[concepts/java/spring-ai/概念_Spring_AI_ChatClient_API]] — ChatClient 请求构建与执行入口。
- [[concepts/java/spring-ai/概念_Spring_AI_Advisor拦截链]] — Advisor 链的排序、消费式执行和复制机制。
- [[concepts/java/spring-ai/概念_Spring_AI_ToolCalling工具调用]] — ToolCallAdvisor 递归循环和 ToolCallingManager。
- [[concepts/java/spring-ai/概念_Spring_AI_RAG检索增强生成]] — RetrievalAugmentationAdvisor 的 Modular RAG 实现。
- [[concepts/java/spring-ai/概念_Spring_AI_AutoConfiguration自动配置]] — Spring Boot 条件装配与 starter 依赖结构。
- [[concepts/java/spring-ai/概念_Spring_AI_设计模式]] — 从设计模式角度解释 Spring AI 为什么这样组织源码。
