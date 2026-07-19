---
type: entity
entity_type: project
tags:
  - java
  - spring-ai
  - spring
  - ai
  - framework
summary: "Spring AI 项目实体：Spring 官方 AI 集成框架，v2.0.0-SNAPSHOT，Apache 2.0 许可证，提供 Chat/Embedding/Image/Audio/Moderation 多模型统一抽象，ChatClient + Advisor 链 + Tool Calling + RAG + ChatMemory + StructuredOutput + VectorStore + MCP，支持多模型厂商和多向量数据库"
sources:
  - "[[30-sources/repositories/Spring_AI_源码]]"
  - "raw/sai"
aliases:
  - Spring AI
  - spring-ai
  - SpringAI
status: stable
confidence: 0.95
updated: "2026-07-14 15:05:00"
---

# 项目_Spring_AI

- **项目名**：Spring AI
- **类型**：Java 框架 / Spring 生态子项目
- **版本**：2.0.0-SNAPSHOT（main 分支），适配 Spring Boot 4.x
- **仓库**：`spring-projects/spring-ai`（GitHub）
- **许可证**：Apache 2.0
- **核心作者**：Christian Tzolov, Mark Pollack, Thomas Vitale 等
- **网站**：https://spring.io/projects/spring-ai

## 简介

Spring AI 是 Spring 官方出品的 AI 应用开发框架，将 LLM 抽象为 `ChatModel`/`EmbeddingModel`/`ImageModel` 等接口，通过 ChatClient 声明式 Fluent API + Advisor AOP 拦截链 + Spring Boot Auto-Configuration，实现 "写一次代码，换 Starter 就换厂商" 的可移植性。

## 核心能力

| 能力 | 抽象 | 厂商 |
|------|------|------|
| Chat（对话） | `ChatModel` | OpenAI / Anthropic / Ollama / DeepSeek / DashScope / Azure / Bedrock / Vertex / Gemini / Mistral / Watson / MiniMax / Moonshot / ZhiPu |
| Embedding（向量化） | `EmbeddingModel` | 同上 + Transformers（本地 ONNX） |
| Image（图像生成） | `ImageModel` | OpenAI DALL-E / Stability AI / Azure |
| Audio（语音） | `TranscriptionModel` / `TextToSpeechModel` | OpenAI Whisper / TTS |
| Moderation（内容审核） | `ModerationModel` | OpenAI Moderation |
| VectorStore（向量存储） | `VectorStore` | 22+ 向量数据库（PGVector / Redis / Elasticsearch / Milvus / Qdrant / Pinecone / Weaviate / Chroma / MongoDB 等） |
| MCP（模型上下文协议） | `MCP` | STDIO / SSE / Streamable HTTP Transport |

## 与本知识库的概念对应

- [[10-domains/java/spring-ai/概念_Spring_AI_架构设计]] — Generic Model 抽象 + ChatClient + Advisor 链的整体架构
- [[10-domains/java/spring-ai/概念_Spring_AI_ChatClient_API]] — ChatClient Fluent API 的完整接口分析
- [[10-domains/java/spring-ai/概念_Spring_AI_Advisor拦截链]] — Advisor 链机制、12 个内置 Advisor、自定义扩展
- [[10-domains/java/spring-ai/概念_Spring_AI_ToolCalling工具调用]] — @Tool 注解到 ToolCallAdvisor 编排全链路
- [[10-domains/java/spring-ai/概念_Spring_AI_RAG检索增强生成]] — 7 步 Modular RAG 流水线
- [[10-domains/java/spring-ai/概念_Spring_AI_ChatMemory对话记忆]] — 滑动窗口记忆与多后端持久化
- [[10-domains/java/spring-ai/概念_Spring_AI_StructuredOutput结构化输出]] — BeanOutputConverter 的 JSON Schema 注入机制
- [[10-domains/java/spring-ai/概念_Spring_AI_VectorStore向量存储]] — RAG 知识库检索、SearchRequest 和元数据过滤抽象
- [[10-domains/java/spring-ai/概念_Spring_AI_AutoConfiguration自动配置]] — Starter 和 Spring Boot 条件装配如何创建核心 Bean
- [[10-domains/java/spring-ai/概念_Spring_AI_MCP集成]] — MCP 工具如何接入 Spring AI Tool Calling 体系
- [[30-sources/repositories/Spring_AI_源码.md]] — 完整源码分析
