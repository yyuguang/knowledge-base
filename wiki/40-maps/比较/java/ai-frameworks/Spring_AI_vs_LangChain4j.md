---
type: "comparison"
tags:
  - java
  - spring-ai
  - langchain4j
  - ai
  - framework
summary: "Spring AI 与 LangChain4j 的系统性对比：出身生态、设计哲学、API 风格、编排机制、工具调用、RAG、厂商可移植性、Spring 耦合度、成熟度等多个维度的差异与选择建议"
sources:
  - "[[30-sources/repositories/Spring_AI_源码]]"
aliases:
  - Spring AI vs LangChain4j
  - SpringAI vs LangChain4j
status: "evolving"
confidence: 0.8
created: "2026-05-14 10:36:36"
updated: "2026-05-14 10:36:36"
---

# Spring AI vs LangChain4j

## 一句话结论

**选 Spring AI**：你已经在 Spring Boot 生态中，需要深度 Spring 集成（AutoConfiguration、Micrometer 观测、AOT 编译）、多厂商无缝切换。

**选 LangChain4j**：你需要更成熟的 API、不绑定 Spring、偏好声明式注解（`@AiService`）、或需要脱离 Spring 使用（Quarkus / 纯 Java）。

---

## 比较对象

### A：Spring AI

Spring 官方 AI 应用开发框架（Apache 2.0），将 LLM 抽象为 `ChatModel`/`EmbeddingModel`/`ImageModel` 等接口，通过 ChatClient Fluent API + Advisor AOP 拦截链 + Spring Boot AutoConfiguration 实现"换 Starter 就换厂商"的可移植性。

详见 [[10-domains/java/spring-ai/项目_Spring_AI]] — 项目实体页，含版本、核心能力表、概念对应。

### B：LangChain4j

社区驱动的 Java LLM 框架，灵感来自 Python LangChain。通过 `@AiService` 声明式注解 API + Chain/Agent 编排模式 + 独立于 Spring 的设计，实现多厂商 LLM 集成。也可通过 Spring Boot Starter 与 Spring 集成。

> ⚠️ 当前 wiki 暂无 LangChain4j 专题页面，以下分析基于内部知识。

---

## 相同点

- 都是 Java 生态的 LLM 应用开发框架
- 都提供多模型厂商抽象（OpenAI / Anthropic / Ollama / Azure / Vertex 等）
- 都支持 Tool Calling、RAG、ChatMemory、结构化输出
- 都支持多种向量数据库
- 都用接口 + 策略/适配器模式实现厂商可移植性
- 都有 Spring Boot Starter 集成方式
- 都遵循 Apache 2.0 许可证

---

## 不同点

| 维度 | Spring AI | LangChain4j |
|------|-----------|-------------|
| 出身 | Spring 官方子项目 | 社区驱动（Dmytro Liubarskyi 等） |
| 设计哲学 | Spring 风格：AutoConfiguration + Advisor AOP + Builder + Template | 链式编排：Chain/Agent + AiServices 声明式注解 |
| API 风格 | `chatClient.prompt().user("...").call().content()` Fluent API | `@AiService` 接口 + `@UserMessage("...")` 模板，自动生成实现 |
| 编排机制 | Advisor 责任链（AOP 风格横切关注点统一注入） | Chain 链式组合 + Agent 决策循环（手动编排） |
| Spring 耦合度 | 深度绑定 Spring Boot | 可选集成，可脱离 Spring 独立使用 |
| 厂商切换 | 换 Starter 换厂商（AutoConfiguration 自动装配） | Builder 模式构建实现，运行时切换 |
| 核心抽象 | `Model<TReq, TRes>` 泛型接口 → `ChatModel` / `EmbeddingModel` / `ImageModel` | `ChatLanguageModel` / `EmbeddingModel` / `ImageModel` 接口 |
| RAG | `RetrievalAugmentationAdvisor` 7 步 Modular RAG 流水线（变换→扩展→检索→合并→后处理→增强） | Easy RAG（DocumentLoader→Splitter→Ingestor 一键）+ Advanced RAG（自定义检索+重排+压缩） |
| Tool Calling | `@Tool` → `MethodToolCallback` → `ToolCallAdvisor` ReAct 循环，需注册为 Advisor | `@Tool` 直接标注，`AiServices` 自动发现注册 |
| Memory | `MessageWindowChatMemory` 滑动窗口 + 6 种持久化后端 + `MessageChatMemoryAdvisor` | `ChatMemory` + `ChatMemoryStore`，更灵活的消息管理 |
| 结构化输出 | `BeanOutputConverter` JSON Schema 自动生成 + Jackson 反序列化 | `AiServices` 直接返回 POJO + 内置 JSON 模式支持 |
| 可观测性 | Micrometer Observation 自动埋点（token/延迟/错误全链路） | 需自己集成（但 LangChain4j 无内置观测抽象） |
| AOT / Native | 官方支持 AOT 编译 + Native Image（`AiRuntimeHints`、`ToolRuntimeHints`） | 无官方 AOT 支持 |
| MCP 协议 | 原生 MCP Client/Server + ToolCallback 双向桥接 | 无内置 MCP 支持 |
| 成熟度 | 仍在快速迭代（2.0-SNAPSHOT，适配 Spring Boot 4.x） | 更成熟的版本管理（1.x 稳定版） |
| Prompt 模板 | `PromptTemplate` + `{placeholder}` 风格 | `@UserMessage` 注解 + `{{variable}}` 风格 |

---

## 选择建议

- **团队已用 Spring Boot** → Spring AI。AutoConfiguration 零摩擦接入，Micrometer 自动埋点，与现有 Spring 基础设施无缝衔接。
- **需要独立于 Spring** → LangChain4j。Quarkus / Micronaut / 纯 Java 项目都能用，不强制依赖 Spring 容器。
- **偏好声明式注解** → LangChain4j。`@AiService` / `@UserMessage` / `@Tool` 的注解驱动体验更简洁，类似 Python LangChain 的装饰器风格。
- **需要 AOT / Native Image** → Spring AI。官方支持的反射提示和条件装配让 GraalVM Native Image 编译成为可能。
- **需要 MCP 协议** → Spring AI。原生支持 MCP Client/Server，通过 ToolCallback 双向桥接外部工具生态。
- **需要成熟稳定 API** → LangChain4j（1.x 稳定版）。Spring AI 仍在快速迭代，API 可能存在 breaking changes。
- **需要丰富内置 DocumentLoader** → LangChain4j。内置更多文档解析器（PDF、HTML、GitHub Issues 等），Spring AI 需要自己实现或借助社区。
- **两个都要？** → 可以共存，但通常没必要。除非需要各自独有的集成（如 LangChain4j 的某 Embedding Store + Spring AI 的 MCP）。

---

## 源码级差异（Spring AI 侧基于 wiki)

Spring AI 的核心架构（来自 [[10-domains/java/spring-ai/主题_Spring_AI源码架构_综述]]）是典型 Spring 风格：

```
Boot 装配层（AutoConfiguration / Starter）
    ↓
Client 编排层（ChatClient / Advisor Chain）
    ↓
Model 抽象层（ChatModel / Prompt / ChatResponse）
    ↓
Provider 层（OpenAiChatModel / OllamaChatModel ...）
```

LangChain4j 的核心架构更接近 Python LangChain：

```
AiServices 注解层（自动生成代理实现）
    ↓
Chain / Agent 编排层（SequentialChain / RouterChain / Agent）
    ↓
Model 抽象层（ChatLanguageModel / EmbeddingModel）
    ↓
Provider 层（OpenAiChatModel / OllamaChatModel ...）
```

关键差异：
- Spring AI 的编排靠 **Advisor 责任链**（AOP 风格，横切关注点统一注入），LangChain4j 靠 **Chain 组合**（链式组装，显式编排）。
- Spring AI 的模型抽象是泛型 `Model<TReq, TRes>`，LangChain4j 是具体接口。
- Spring AI 的装配靠 Spring Boot 机制，LangChain4j 的装配靠 Builder 模式。

---

## 相关链接

- [[10-domains/java/spring-ai/项目_Spring_AI]] — Spring AI 项目实体，含核心能力表和概念对应
- [[10-domains/java/spring-ai/概念_Spring_AI_架构设计]] — Generic Model 抽象 + ChatClient + Advisor 链架构
- [[10-domains/java/spring-ai/主题_Spring_AI源码架构_综述]] — Spring AI 源码级主线综述
- [[10-domains/java/spring-ai/概念_Spring_AI_设计模式]] — Spring AI 使用的设计模式全景
