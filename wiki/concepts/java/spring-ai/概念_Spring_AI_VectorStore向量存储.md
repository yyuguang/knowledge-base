---
type: "concept"
tags:
  - java
  - spring-ai
  - vector-store
  - rag
summary: "Spring AI VectorStore 向量存储抽象：以 add/delete/similaritySearch 统一不同向量数据库，SearchRequest 封装 query/topK/similarityThreshold/filterExpression，Filter.Expression 提供跨存储可移植的元数据过滤 AST。"
sources:
  - "sai/spring-ai-vector-store/src/main/java/org/springframework/ai/vectorstore/VectorStore.java"
  - "sai/spring-ai-vector-store/src/main/java/org/springframework/ai/vectorstore/SearchRequest.java"
  - "sai/spring-ai-vector-store/src/main/java/org/springframework/ai/vectorstore/filter/Filter.java"
  - "sai/spring-ai-vector-store/src/main/java/org/springframework/ai/vectorstore/observation/AbstractObservationVectorStore.java"
aliases:
  - Spring AI VectorStore
  - VectorStore
  - 向量存储
status: "evolving"
confidence: 0.92
created: "2026-05-12 00:00:00"
updated: "2026-05-12 00:00:00"
---

# 概念：Spring AI VectorStore 向量存储

## 定义

`VectorStore` 是 Spring AI 对向量数据库的统一抽象。它把 RAG 中最核心的知识库操作压缩成三类能力：写入 `Document`、按 id 或元数据过滤删除、按语义相似度检索。

这个接口同时继承 `DocumentWriter` 和 `VectorStoreRetriever`，因此它既是文档写入端，也是检索端。具体实现可以是 PGVector、Redis、Milvus、Qdrant、Pinecone、Weaviate、Elasticsearch、MongoDB Atlas 等，但上层 RAG Advisor 只依赖 `VectorStore`。

## 为什么重要

Spring AI 的 RAG 可移植性不只来自 ChatModel，也来自 VectorStore。业务代码如果只依赖 `VectorStore` 和 `SearchRequest`，就可以把本地开发的简单向量库替换为生产环境的数据库实现，而不重写检索增强逻辑。

它还解决了一个容易被忽略的问题：不同向量数据库的过滤语法完全不同。Spring AI 用 `Filter.Expression` 作为中间 AST，再由各存储实现转换成自己的原生过滤表达式。

## 使用场景

- RAG 文档入库：把切分后的 `Document` 写入向量库。
- 语义检索：用用户问题做 embedding 相似度搜索，返回 topK 文档。
- 元数据过滤：按租户、知识域、年份、权限标签过滤候选文档。
- 长期记忆：`VectorStoreChatMemoryAdvisor` 可把对话内容当作可检索记忆。
- 多存储切换：开发阶段用轻量实现，生产阶段切换 PGVector / Redis / Milvus 等。

## 核心机制

### 1. VectorStore 接口

核心方法很少：

```java
void add(List<Document> documents);
void delete(List<String> idList);
void delete(Filter.Expression filterExpression);
List<Document> similaritySearch(SearchRequest request);
```

`delete(String filterExpression)` 会先构造 `SearchRequest`，用文本解析器把 SQL-like 过滤语法转成 `Filter.Expression`，再委托给 `delete(Filter.Expression)`。

`getNativeClient()` 是一个逃生口：大多数业务代码不应依赖它，但当确实需要访问底层客户端的高级能力时，可以由具体实现返回原生 client。

### 2. SearchRequest 检索参数

`SearchRequest` 的默认值表达了 Spring AI 对 RAG 检索的保守默认：

| 参数 | 默认值 | 含义 |
|------|--------|------|
| `query` | `""` | 用于 embedding 相似度比较的查询文本 |
| `topK` | `4` | 默认返回 4 个相似文档 |
| `similarityThreshold` | `0.0` | 接受所有相似度；设为 1.0 表示近似精确匹配 |
| `filterExpression` | `null` | 不启用元数据过滤 |

一个典型请求：

```java
SearchRequest request = SearchRequest.builder()
    .query("Spring AI 的 Advisor 链如何执行？")
    .topK(6)
    .similarityThreshold(0.72)
    .filterExpression("area == 'spring-ai' && status == 'stable'")
    .build();
```

需要注意：`similarityThreshold` 是客户端侧后处理，不一定是向量数据库原生过滤。它适合统一行为，但不能假设所有性能都下推到存储层。

### 3. Filter.Expression 可移植过滤 AST

`Filter.Expression` 支持比较、集合、组合和空值判断：

| 类型 | 用途 |
|------|------|
| `EQ / NE / GT / GTE / LT / LTE` | 常量比较 |
| `IN / NIN` | 集合包含 / 不包含 |
| `AND / OR / NOT` | 组合表达式 |
| `ISNULL / ISNOTNULL` | 空值判断 |

它可以通过三种方式产生：

```java
// 方式一：手写 AST
new Filter.Expression(EQ, new Filter.Key("country"), new Filter.Value("UK"));

// 方式二：FilterExpressionBuilder DSL
var b = new FilterExpressionBuilder();
var exp = b.and(b.eq("country", "UK"), b.gte("year", 2020));

// 方式三：SQL-like 文本解析
SearchRequest.builder()
    .filterExpression("country == 'UK' && year >= 2020")
    .build();
```

这层 AST 的价值是把“业务过滤条件”与“数据库过滤语法”分离。比如 Pinecone、Milvus、Redis、Elasticsearch 的表达式格式不同，但上层可以保留同一套语义。

### 4. Observation 包装

`AbstractObservationVectorStore` 是具体向量库实现的模板基类。它把 `add/delete/similaritySearch` 包在 Micrometer Observation 中，然后把真正操作留给：

```java
doAdd(documents);
doDelete(idList);
doDelete(filterExpression);
doSimilaritySearch(request);
```

这让不同存储实现共享统一的指标结构：操作名、数据库系统、查询请求、查询响应文档等。RAG 排查时，这些指标能回答“检索慢在哪里”“召回了多少文档”“哪类向量库错误最多”。

## 例子

```java
List<Document> docs = vectorStore.similaritySearch(
    SearchRequest.builder()
        .query("ToolCallAdvisor 如何形成 ReAct 循环？")
        .topK(5)
        .similarityThreshold(0.7)
        .filterExpression("project == 'spring-ai' && type == 'concept'")
        .build()
);
```

这个查询表达的是：只在 Spring AI 概念页中检索相似文档，并过滤掉低相似度结果。

## 常见误区

1. **把 VectorStore 当作普通数据库 Repository**：它的主查询能力是相似度，不是精确条件查询。元数据过滤应服务于召回约束，而不是替代业务数据库。
2. **以为 similarityThreshold 一定下推到数据库**：源码注释明确它是客户端侧后处理语义，性能和召回行为不能完全按数据库索引理解。
3. **忽略 embedding 维度一致性**：写入和查询必须使用兼容的 `EmbeddingModel`，否则相似度没有意义，很多存储还会在维度上直接报错。
4. **过早使用 native client**：`getNativeClient()` 是必要逃生口，但一旦业务逻辑依赖原生 API，就会丢失 Spring AI 的可移植性。

## 和其它概念的关系

- [[concepts/java/spring-ai/概念_Spring_AI_RAG检索增强生成]] — VectorStore 是 RAG 检索阶段最常见的 `DocumentRetriever` 数据源。
- [[concepts/java/spring-ai/概念_Spring_AI_Advisor拦截链]] — `RetrievalAugmentationAdvisor` 可把 VectorStore 检索透明嵌入 ChatClient 调用链。
- [[concepts/java/spring-ai/概念_Spring_AI_架构设计]] — VectorStore 延续了 Spring AI 的接口抽象与多厂商适配思路。

## 我的理解

VectorStore 是 Spring AI 把“知识库”工程化的关键位置。ChatModel 解决“怎么问模型”，VectorStore 解决“模型能看到哪些外部知识”。真正的 RAG 质量往往不取决于是否调用了 LLM，而取决于 `SearchRequest` 的 topK、阈值、元数据过滤、chunk 质量和 embedding 一致性。

## 来源

- `sai/spring-ai-vector-store/src/main/java/org/springframework/ai/vectorstore/VectorStore.java`
- `sai/spring-ai-vector-store/src/main/java/org/springframework/ai/vectorstore/SearchRequest.java`
- `sai/spring-ai-vector-store/src/main/java/org/springframework/ai/vectorstore/filter/Filter.java`
- `sai/spring-ai-vector-store/src/main/java/org/springframework/ai/vectorstore/observation/AbstractObservationVectorStore.java`
