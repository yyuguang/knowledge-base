---
type: concept
tags:
  - java
  - spring-ai
  - rag
  - retrieval
  - vector-store
  - document
summary: "Spring AI RAG 检索增强生成：RetrievalAugmentationAdvisor 实现 7 步 Modular RAG 流水线（Query 创建→变换→扩展→检索→合并→后处理→增强），每步都是可替换的策略接口，通过 Advisor 链自动嵌入每次 LLM 调用"
sources:
  - "raw/code/spring-ai/spring-ai-rag/src/main/java/org/springframework/ai/rag/advisor/RetrievalAugmentationAdvisor.java"
  - "raw/code/spring-ai/spring-ai-rag/src/main/java/org/springframework/ai/rag/"
aliases:
  - RAG
  - Retrieval Augmented Generation
  - Spring AI RAG
  - 检索增强生成
status: evolving
confidence: 0.95
updated: "2026-05-12 00:00:00"
---

# 概念_Spring_AI_RAG检索增强生成

![[Attachments/spring-ai-rag-pipeline.svg]]

## 定义

RAG（Retrieval Augmented Generation）是 Spring AI 中通过 `RetrievalAugmentationAdvisor` 实现的**模块化检索增强生成流水线**。它将用户的原始查询经过 7 个可替换的阶段（创建查询→变换→扩展→检索→合并→后处理→增强），将检索到的相关文档注入 Prompt，让 LLM 基于外部知识库回答，解决 LLM 训练数据截止和幻觉问题。

## 为什么重要

RAG 是让 LLM 应用**突破知识截止边界**的核心方案：
1. **私有知识接入**：企业内部文档/知识库不需要微调模型
2. **实时数据**：文档向量化后，新数据立即可检索
3. **可溯源**：回复可附带引用来源，降低幻觉
4. **成本可控**：无需为每个领域微调模型，只需维护向量库

Spring AI 的设计让 RAG 不再是手写的碎片代码，而是**可组合的策略流水线**，每个步骤都可以独立替换。

## 使用场景

- **企业知识库问答**：上传 PDF/Word → 向量化 → ChatClient + RAG Advisor → 智能问答
- **客服系统**：历史工单向量化 → RAG 检索相似案例 → LLM 生成回复
- **代码助手**：代码库 + 文档向量化 → 开发者自然语言查询 → RAG + 代码生成
- **合规审查**：法规文档向量化 → 查询 + RAG → 合规性分析 + 引用出处

## 核心机制

### 1. 7 步 Modular RAG 流水线

```
用户输入："Spring AI 如何配置 RAG？"
    │
    ▼
┌─ RetrievalAugmentationAdvisor.before() ────────────────────────┐
│                                                                  │
│  0️⃣ 创建 Query                                                   │
│     Query.builder()                                              │
│       .text(userMessage)          ← 用户原始文本                  │
│       .history(conversationMessages) ← 对话历史上下文              │
│       .context(advisoryParams)    ← Advisor 链共享参数             │
│       .build()                                                   │
│     │                                                            │
│  1️⃣ Query 变换（可选，链式）                                      │
│     for (QueryTransformer t : queryTransformers)                 │
│       query = t.apply(query)                                     │
│     // 如：RewriteQueryTransformer（用 LLM 重写查询为更精准的表述）  │
│     // 如：TranslationTransformer（翻译为非英文查询）               │
│     │                                                            │
│  2️⃣ Query 扩展（可选）                                            │
│     List<Query> queries = queryExpander.expand(query)             │
│     // 如：MultiQueryExpander（生成多个变体查询提高召回）            │
│     │                                                            │
│  3️⃣ 并行检索                                                      │
│     expandedQueries.stream()                                     │
│       .map(q → CompletableFuture                                 │
│           .supplyAsync(() → documentRetriever.retrieve(q)))      │
│       .map(CompletableFuture::join)                              │
│     // 每个 Query 并行检索，默认线程池 coreSize=4                   │
│     │                                                            │
│  4️⃣ 文档合并                                                      │
│     List<Document> docs = documentJoiner.join(docsForQuery)       │
│     // 默认：ConcatenationDocumentJoiner（简单拼接 + 去重）          │
│     │                                                            │
│  5️⃣ 文档后处理（可选，链式）                                       │
│     for (DocumentPostProcessor p : postProcessors)                │
│       docs = p.process(originalQuery, docs)                      │
│     // 如：重新排序、截断、格式化、去噪                               │
│     │                                                            │
│  6️⃣ Query 增强                                                    │
│     Query augmented = queryAugmenter.augment(original, docs)      │
│     // 默认：ContextualQueryAugmenter                             │
│     //   将文档内容注入 Prompt：                                    │
│     //   "Context: {docs}\n\nQuestion: {userQuery}"               │
│     │                                                            │
│  7️⃣ 更新请求                                                      │
│     return chatClientRequest.mutate()                             │
│       .prompt(prompt.augmentUserMessage(augmented.text()))        │
│       .context(ctx.put(DOCUMENT_CONTEXT, docs))                   │
│       .build()                                                    │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
    │
    ▼
ChatModel.call(augmentedPrompt)  → LLM 基于检索文档回答
    │
    ▼
RetrievalAugmentationAdvisor.after()
    → 将 Documents 注入 ChatResponse.metadata["rag_document_context"]
    → 前端可展示引用来源
```

### 2. 核心策略接口

| 步骤 | 接口 | 默认实现 | 职责 |
|------|------|----------|------|
| Query 变换 | `QueryTransformer` | 无（空链） | 重写/翻译/压缩查询 |
| Query 扩展 | `QueryExpander` | 无（直接用原查询） | 生成多个变体提高召回 |
| 文档检索 | `DocumentRetriever` | **必须提供**（VectorStore 实现） | 从向量库检索相关文档 |
| 文档合并 | `DocumentJoiner` | `ConcatenationDocumentJoiner` | 合并多查询/多源的文档 |
| 文档后处理 | `DocumentPostProcessor` | 无（空链） | 重排序/去噪/截断 |
| Query 增强 | `QueryAugmenter` | `ContextualQueryAugmenter` | 将文档注入 Prompt |

### 3. DocumentRetriever → VectorStore

`DocumentRetriever` 是一个函数式接口：`Query → List<Document>`，最常用的实现是 `VectorStoreDocumentRetriever`：

```java
// VectorStore 自动适配为 DocumentRetriever
DocumentRetriever retriever = VectorStoreDocumentRetriever.builder()
    .vectorStore(vectorStore)     // 22+ 向量库统一抽象
    .topK(5)                      // 返回 Top 5 相似文档
    .similarityThreshold(0.75)    // 相似度阈值
    .build();
```

### 4. 并行检索机制

RAG Advisor 使用 `CompletableFuture` + `TaskExecutor` 实现并行检索：

```java
// RetrievalAugmentationAdvisor: 默认线程池配置
private static TaskExecutor buildDefaultTaskExecutor() {
    ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
    executor.setThreadNamePrefix("ai-advisor-");
    executor.setCorePoolSize(4);   // 4 核心线程
    executor.setMaxPoolSize(16);   // 最大 16 线程
    executor.setTaskDecorator(new ContextPropagatingTaskDecorator()); // 传递 Spring Context
    executor.initialize();
    return executor;
}
```

### 5. 文档上下文传递

检索到的文档通过两种方式传递：
1. **Prompt 注入**：`QueryAugmenter` 将文档内容注入用户消息（LLM 直接看到）
2. **Response Metadata**：`after()` 将文档写入 `ChatResponse.metadata["rag_document_context"]`（前端引用）

## 例子

### 基础 RAG 配置

```java
// 1. 准备向量库（已预先向量化文档）
VectorStore vectorStore = new SimpleVectorStore(embeddingModel);

// 2. 构建 RAG Advisor
RetrievalAugmentationAdvisor ragAdvisor = RetrievalAugmentationAdvisor.builder()
    .documentRetriever(                      // 必须：检索器
        VectorStoreDocumentRetriever.builder()
            .vectorStore(vectorStore)
            .topK(3)
            .build())
    .queryAugmenter(                         // 可选：增强器（有默认值）
        ContextualQueryAugmenter.builder()
            .allowEmptyContext(true)
            .build())
    .build();

// 3. 注册到 ChatClient
ChatClient client = ChatClient.builder(chatModel)
    .defaultAdvisors(ragAdvisor)
    .build();

// 4. 使用（RAG 自动生效）
String answer = client.prompt()
    .user("Spring AI 支持哪些向量数据库？")
    .call().content();
// Prompt 被自动增强为：
// "Context: [文档1: Spring AI 支持 22+ 向量库...]\n\n
//  Question: Spring AI 支持哪些向量数据库？"
```

### 带 Query 扩展的 RAG

```java
RetrievalAugmentationAdvisor advisor = RetrievalAugmentationAdvisor.builder()
    .queryExpander(                          // 查询扩展：生成多个变体
        MultiQueryExpander.builder()
            .chatModel(chatModel)            // 用 LLM 生成变体
            .numberOfQueries(3)
            .build())
    .documentRetriever(retriever)
    .documentPostProcessors(                 // 后处理：重排序
        new RerankDocumentPostProcessor(rerankModel))
    .build();
```

### 从 Response 中获取引用来源

```java
ChatClientResponse response = client.prompt()
    .user("Spring AI 的 RAG 如何工作？")
    .call()
    .chatClientResponse();

@SuppressWarnings("unchecked")
List<Document> sources = (List<Document>) response.context()
    .get(RetrievalAugmentationAdvisor.DOCUMENT_CONTEXT);
// sources 包含被检索到的原始文档，可展示引用
```

## 常见误区

1. **"RAG Advisor 自动处理对话历史"**：不。RAG Advisor 只增强当前查询的 Prompt。对话历史管理需要 `MessageChatMemoryAdvisor` 配合使用。
2. **"DocumentRetriever 只能接 VectorStore"**：错。`DocumentRetriever` 是函数式接口，可以实现任何检索逻辑（ES 全文搜索、数据库查询、API 调用等）。
3. **"并行检索对性能无影响"**：当 `QueryExpander` 生成多个变体时，并行检索可显著提升吞吐。但默认线程池只有 4 核心线程，高并发时可能成为瓶颈。
4. **"ContextualQueryAugmenter 会自动引用文档"**：不完全。它只是把文档内容拼接进 Prompt，实际引用需要前端从 Response Metadata 中提取 `DOCUMENT_CONTEXT`。

## 源码级执行链

`RetrievalAugmentationAdvisor.before()` 是 RAG 的核心源码入口。它不是在模型响应后做补充，而是在模型调用前改写 `ChatClientRequest`：

| 步骤 | 源码对象 | 说明 |
|------|----------|------|
| 0 | `Query.builder()` | 从 user text、prompt instructions、request context 构造 originalQuery |
| 1 | `List<QueryTransformer>` | 顺序执行查询改写，前一个 transformer 的输出是后一个输入 |
| 2 | `QueryExpander` | 可选，一问变多问；没有配置时就是单查询 |
| 3 | `CompletableFuture.supplyAsync()` | 每个 expanded query 并发调用 `documentRetriever.retrieve(query)` |
| 4 | `DocumentJoiner` | 默认 `ConcatenationDocumentJoiner`，负责合并多查询/多源文档 |
| 5 | `DocumentPostProcessor` | 顺序后处理文档，可重排、过滤、压缩 |
| 6 | `context.put(DOCUMENT_CONTEXT, documents)` | 把检索结果放进上下文，供 after 和应用读取 |
| 7 | `queryAugmenter.augment()` | 默认 `ContextualQueryAugmenter` 把文档内容注入用户问题 |
| 8 | `prompt().augmentUserMessage()` | 返回新的 `ChatClientRequest`，后续 Advisor 和模型看到的是增强后的用户消息 |

`after()` 的职责很小：把 `DOCUMENT_CONTEXT` 从 response context 写回 `ChatResponse.metadata`。因此引用来源展示不应该靠 LLM 自己编，应从 `DOCUMENT_CONTEXT` 里取。

## 并发与生产调优点

默认 `TaskExecutor` 由 `buildDefaultTaskExecutor()` 创建：

```text
threadNamePrefix = ai-advisor-
corePoolSize = 4
maxPoolSize = 16
taskDecorator = ContextPropagatingTaskDecorator
```

这意味着：

- `QueryExpander` 生成多个 query 后，检索是并发的。
- 上下文传播被显式考虑，适合 Observation / tracing 场景。
- 高并发服务里，默认线程池可能和业务线程池、向量库连接池一起成为瓶颈。
- 如果检索后处理很重，应显式提供 `TaskExecutor` 和 `DocumentPostProcessor` 的性能预算。

## 和其它概念的关系

- [[concepts/java/spring-ai/概念_Spring_AI_Advisor拦截链]] — RetrievalAugmentationAdvisor 在 Advisor 链中自动嵌入，before() 执行 7 步流水线
- [[concepts/java/spring-ai/概念_Spring_AI_ChatClient_API]] — `defaultAdvisors(ragAdvisor)` 注册到 ChatClient
- [[concepts/java/spring-ai/概念_Spring_AI_架构设计]] — RAG 是架构中 Client-Chat 层的 Advisors 模块之一
- [[concepts/java/spring-ai/概念_Spring_AI_ChatMemory对话记忆]] — RAG + ChatMemory 组合实现 "基于历史的检索增强"
- [[concepts/java/spring-ai/概念_Spring_AI_VectorStore向量存储]] — VectorStore 是最常见的 DocumentRetriever 数据源
- [[overview/java/spring-ai/主题_Spring_AI源码架构_综述]] — RAG 在完整源码主线中的位置

## 来源

- `raw/code/spring-ai/spring-ai-rag/src/main/java/org/springframework/ai/rag/advisor/RetrievalAugmentationAdvisor.java`
- `raw/code/spring-ai/spring-ai-rag/src/main/java/org/springframework/ai/rag/Query.java`
- `raw/code/spring-ai/spring-ai-rag/src/main/java/org/springframework/ai/rag/retrieval/search/DocumentRetriever.java`
