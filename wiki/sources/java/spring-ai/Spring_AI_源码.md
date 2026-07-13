---
type: source
tags:
  - java
  - spring-ai
  - ai
  - llm
  - spring-boot
  - framework
summary: "Spring AI 2.0-SNAPSHOT 完整源码分析：Spring 官方 AI 集成框架，提供 Chat/Embedding/Image/Audio/Moderation 多模型统一抽象、ChatClient 流式 API、Advisor 拦截链、Tool Calling、RAG、ChatMemory、VectorStore、MCP 与 Spring Boot AutoConfiguration"
sources:
  - "sai (GitHub spring-projects/spring-ai, main branch, 2.0.0-SNAPSHOT)"
aliases:
  - Spring AI 源码
  - Spring AI 源代码
status: stable
confidence: 0.95
updated: "2026-05-12 00:00:00"
---

# Spring AI 源码

![[Attachments/spring-ai-source-architecture.svg]]

- **一句话总结**：Spring AI 是 Spring 官方出品的 AI 应用开发框架，通过统一抽象（ChatModel/EmbeddingModel/ImageModel 等）隔离 14+ 模型厂商和 22+ 向量数据库的 API 差异，以声明式 ChatClient + Advisor 链 + 自动配置实现 "写一次代码，换 Starter 就换厂商" 的可移植性。
- **版本**：2.0.0-SNAPSHOT（main 分支），适配 Spring Boot 4.x
- **仓库**：`spring-projects/spring-ai`（GitHub）
- **许可证**：Apache 2.0
- **作者团队**：Christian Tzolov, Mark Pollack, Thomas Vitale 等

---

## 核心观点

1. **分层架构，层层抽象**：Spring AI 不是一个大杂烩，而是精确分层的——Commons（基础内容/文档/评估/分词）→ Model（模型抽象/消息/提示词/工具/转换器/记忆）→ Client-Chat（ChatClient/Advisor 链/RAG/评估）→ Auto-Configuration（各厂商自动配置），各层职责清晰。
2. **ChatClient 是核心入口**：对标 Spring 的 RestClient/WebClient，提供声明式 Fluent API。`ChatClient.builder(chatModel).defaultSystem("...").build()` 构建不可变客户端，`prompt().user("...").call().content()` 完成一次调用。
3. **Advisor 链 = AOP 拦截器**：对标 Spring MVC Interceptor，通过 `BaseAdvisor.before()/after()` 模板方法在每次 LLM 调用前后插入横切逻辑（日志、记忆、安全、工具调用、RAG）。
4. **RAG 是 7 步流水线**：`RetrievalAugmentationAdvisor` 精确定义了 Query 变换→扩展→检索（并行）→合并→后处理→增强的完整流程，每个步骤都是可替换的策略接口。
5. **Tool Calling 是 ReAct 模式的 Spring 实现**：`@Tool` 注解声明工具→`ToolCallAdvisor` 检测 LLM 的工具调用→`MethodToolCallback` 反射执行→结果回填→LLM 再次推理，天然支持多轮 Agent 循环。
6. **ChatMemory 是滑动窗口 + 多后端持久化**：`MessageWindowChatMemory` 默认 20 条窗口，保留 SystemMessage 不裁剪，支持 JDBC/Redis/MongoDB/Cassandra/Neo4j/CosmosDB 多后端（通过 `ChatMemoryRepository` 接口隔离）。
7. **StructuredOutput 通过 JSON Schema 注入实现**：`BeanOutputConverter` 自动从 Java 类生成 JSON Schema，注入 System Prompt 约束 LLM 输出格式，再通过 Jackson 反序列化为 DTO，全过程自动。
8. **Retry 区分瞬态/非瞬态异常**：`TransientAiException`（5xx，可重试成功）与 `NonTransientAiException`（4xx，重试无效），`RetryUtils` 提供默认重试策略（最大 10 次、2s 初始、5x 倍数、3min 上限）。
9. **可观测性内建**：ChatClient/ChatModel/Embedding/Tool 全链路集成 Micrometer Observation，每个操作都携带 token 消耗、延迟、错误等指标。
10. **JSpecify Null-Safety**：2.0 版本全面引入 JSpecify `@NullMarked`，包级默认非空，`@Nullable` 显式标注，配合 ErrorProne + NullAway 静态检查。
11. **VectorStore 是 RAG 的第二个可移植抽象**：`VectorStore` 用 `add/delete/similaritySearch` 统一不同向量库，`SearchRequest` 统一 topK、相似度阈值和元数据过滤，`Filter.Expression` 作为跨存储过滤 AST。
12. **AutoConfiguration 是接口抽象的落地层**：Starter 引入实现和自动配置模块，`@ConditionalOnProperty` 选择 provider，`@ConditionalOnMissingBean` 保留用户覆盖点，`ChatClient.Builder` 以 prototype 方式注入。
13. **MCP 把工具扩展到协议生态**：MCP Client 可把远程 server 工具转换为 Spring AI `ToolCallback`，本地 `ToolCallback` 也能转换成 MCP Tool specification，对 Tool Calling 形成双向桥接。

---

## 关键论证

Spring AI 的核心论证是 **"接口抽象 + Starter 隔离 = 厂商无关的可移植性"**：

- `ChatModel` 接口只定义 `call(Prompt): ChatResponse` 和 `stream(Prompt): Flux<ChatResponse>`
- 所有厂商实现（OpenAI/Anthropic/Ollama/DeepSeek 等）都实现同一接口
- 用户代码只依赖 `ChatModel` 接口，通过 Spring Boot Auto-Configuration 注入具体实现
- 切换厂商 = 换 Maven Starter + 改配置前缀，**零业务代码修改**

对比 LangChain/LlamaIndex 的 Python 生态，Spring AI 的优势在于：
- **类型安全**：编译期检查，而非 Python 的运行时错误
- **Spring 生态集成**：无缝集成 Spring Boot/Spring Security/Spring Data/Spring Cloud
- **Auto-Configuration**：零 Java 配置，依赖即用
- **GraalVM/Project Leyden**：原生编译支持（AOT hints）

---

## 重要概念

| 概念 | 模块 | 一句话 |
|------|------|--------|
| `ChatModel` | spring-ai-model | LLM 对话的统一抽象接口 |
| `ChatClient` | spring-ai-client-chat | 声明式 Fluent API 客户端 |
| `Advisor` | spring-ai-client-chat | AOP 拦截器，环绕每次 LLM 调用 |
| `ToolCallback` | spring-ai-model | 工具/函数调用的回调接口 |
| `ChatMemory` | spring-ai-model | 多轮对话历史管理 |
| `RetrievalAugmentationAdvisor` | spring-ai-rag | 7 步 Modular RAG 流水线 |
| `StructuredOutputConverter` | spring-ai-model | JSON→DTO 自动转换 |
| `VectorStore` | spring-ai-vector-store | 跨厂商向量存储抽象 |
| `Filter.Expression` | spring-ai-vector-store | 可移植的元数据过滤 AST |
| `ToolCallingManager` | spring-ai-model | 工具调用编排管理器 |
| `Prompt` | spring-ai-model | 请求标准：消息列表 + ChatOptions |
| `Usage` | spring-ai-model | Token 用量统计（含缓存命中） |
| `MCP` | mcp/ | Model Context Protocol 的 Spring 实现 |
| `AutoConfiguration` | auto-configurations/ | Spring Boot 条件装配层，创建模型、ChatClient、VectorStore、MCP 等 Bean |

---

## 我的理解

Spring AI 的设计哲学是 **"把 AI 当作一种新的数据源"**——就像 Spring Data 把数据库抽象为 Repository，Spring AI 把 LLM 抽象为 ChatModel/ChatClient。

这种类比的工程价值非常实在：
- **ChatClient.Builder → RestClient.Builder**：都是 Builder 模式构建不可变客户端
- **Advisor → HandlerInterceptor**：都是责任链模式的拦截器
- **ToolCallback → @Service 方法**：都被容器管理，通过注解声明
- **ChatMemory → Cache**：都是 key-value 存储 + 淘汰策略
- **RAG Pipeline → Spring Batch**：都是可组合的 ETL 流水线

整个框架最精巧的设计是 **ToolCallAdvisor 的循环机制**：它在 Advisor 链中通过 `after()` 检查 LLM 返回是否需要工具调用，如果需要就执行工具、回填结果、再次调用 ChatModel，形成 "思考→行动→观察" 的 Agent 循环，无需业务代码手写循环逻辑。

RAG 的 `RetrievalAugmentationAdvisor` 也同样精巧：它在 `before()` 中完成完整的 7 步流水线（变换→扩展→并行检索→合并→后处理→增强），用户只需配置各个策略组件，整个 RAG 流程自动嵌入每次调用。

## 源码级阅读路线

建议按这条路径读源码，而不是从 provider 目录直接开始：

1. `ChatClientAutoConfiguration`：理解 `ChatClient.Builder` 为什么是 prototype，以及 customizer 如何统一改 builder。
2. `DefaultChatClientBuilder`：理解默认配置都落在 `DefaultChatClientRequestSpec defaultRequest` 上。
3. `DefaultChatClient.prompt()`：理解每次请求为什么是复制默认 request spec。
4. `DefaultChatClientRequestSpec.call()/stream()`：理解何时构建 Advisor 链，何时把终端 `ChatModelCallAdvisor` 加进去。
5. `DefaultAroundAdvisorChain.nextCall()/nextStream()`：理解 Advisor 排序、`Deque.pop()` 消费式执行、Observation 包裹。
6. `ToolCallAdvisor`：理解工具调用为什么要 `chain.copy(this)`，以及如何形成 ReAct 循环。
7. `RetrievalAugmentationAdvisor`：理解 RAG 如何在模型调用前改写 user message，而不是响应后追加资料。
8. `VectorStore` / `SearchRequest` / `Filter.Expression`：理解知识库检索的抽象边界。
9. `McpToolUtils` / MCP autoconfigure：理解远程工具如何桥接为 Spring AI ToolCallback。

配套图：

- ![[Attachments/spring-ai-chatclient-call-flow.svg]]
- ![[Attachments/spring-ai-tool-calling-loop.svg]]
- ![[Attachments/spring-ai-rag-pipeline.svg]]
- ![[Attachments/spring-ai-autoconfiguration-chain.svg]]

---

## 未解决问题

1. **流式调用中 Tool Calling 的行为**：`ToolCallAdvisor` 如何处理流式 chunk 中的工具调用信号？需要聚合多个 chunk 才能判断 finishReason 是否为 tool_calls。
2. **并行工具调用的去重与冲突**：`parallelToolCalls` 时，多个工具返回值可能互相矛盾，框架如何处理？
3. **RAG 中多查询扩展的并发控制**：`MultiQueryExpander` 生成多个变体查询，`RetrievalAugmentationAdvisor` 用 `CompletableFuture` 并行检索，但默认线程池只有 4 核心线程——大规模时可能成为瓶颈。
4. **ChatMemory 的存储一致性**：`MessageWindowChatMemory` 每次 `add()` 都是"读→合并→写"三步，在高并发下存在竞态条件。生产环境是否需要分布式锁？
5. **GraalVM native-image 的兼容性**：框架提供了 AOT hints（`AiRuntimeHints` / `ToolRuntimeHints`），但 22 个 VectorStore 和 14 个 Model 厂商的 native 兼容性覆盖到什么程度？

---

## 相关链接

- [[concepts/java/spring-ai/概念_Spring_AI_架构设计]] — Generic Model 抽象 + ChatClient + Advisor 链的整体架构
- [[concepts/java/spring-ai/概念_Spring_AI_ChatClient_API]] — ChatClient Fluent API 的完整接口分析
- [[concepts/java/spring-ai/概念_Spring_AI_Advisor拦截链]] — Advisor 链机制、内置 Advisor、自定义扩展
- [[concepts/java/spring-ai/概念_Spring_AI_ToolCalling工具调用]] — @Tool 注解到 ToolCallAdvisor 编排全链路
- [[concepts/java/spring-ai/概念_Spring_AI_RAG检索增强生成]] — 7 步 Modular RAG 流水线
- [[concepts/java/spring-ai/概念_Spring_AI_ChatMemory对话记忆]] — 滑动窗口记忆与多后端持久化
- [[concepts/java/spring-ai/概念_Spring_AI_StructuredOutput结构化输出]] — BeanOutputConverter 的 JSON Schema 注入机制
- [[concepts/java/spring-ai/概念_Spring_AI_VectorStore向量存储]] — SearchRequest、Filter.Expression 与向量数据库可移植检索
- [[concepts/java/spring-ai/概念_Spring_AI_AutoConfiguration自动配置]] — Starter、条件装配、ChatClient.Builder prototype 与 provider 选择
- [[concepts/java/spring-ai/概念_Spring_AI_MCP集成]] — MCP Client/Server 与 Spring AI ToolCallback 的双向桥接
- [[concepts/java/spring-ai/概念_Spring_AI_设计模式]] — Builder、责任链、模板方法、策略、适配器等模式如何支撑源码架构
- [[overview/java/spring-ai/主题_Spring_AI源码架构_综述]] — 源码级主线、图表和跨模块关系综述
- [[entities/java/spring-ai/项目_Spring_AI]] — Spring AI 项目实体
