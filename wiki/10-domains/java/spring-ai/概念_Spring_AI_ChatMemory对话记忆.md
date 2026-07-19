---
type: concept
tags:
  - java
  - spring-ai
  - chat-memory
  - conversation-history
  - sliding-window
summary: "Spring AI ChatMemory 对话记忆：MessageWindowChatMemory 滑动窗口（默认 20 条，SystemMessage 保护），通过 ChatMemoryRepository 接口支持 6 种持久化后端（JDBC/Redis/MongoDB/Cassandra/Neo4j/CosmosDB），MessageChatMemoryAdvisor 自动注入历史"
sources:
  - "[[30-sources/repositories/Spring_AI_源码]]"
  - "raw/sai/spring-ai-model/src/main/java/org/springframework/ai/chat/memory/ChatMemory.java"
  - "raw/sai/spring-ai-model/src/main/java/org/springframework/ai/chat/memory/MessageWindowChatMemory.java"
  - "raw/sai/spring-ai-client-chat/src/main/java/org/springframework/ai/chat/client/advisor/MessageChatMemoryAdvisor.java"
aliases:
  - ChatMemory
  - 对话记忆
  - Conversation Memory
  - 滑动窗口记忆
status: evolving
confidence: 0.95
updated: "2026-07-14 15:05:00"
---

# 概念_Spring_AI_ChatMemory对话记忆

## 定义

ChatMemory 是 Spring AI 中的**多轮对话历史管理抽象**。核心实现 `MessageWindowChatMemory` 采用**滑动窗口 + SystemMessage 保护**策略：维护最近 N 条消息（默认 20），新人消息加入时淘汰最旧的非 SystemMessage，确保窗口不超出 token 限制。通过 `ChatMemoryRepository` 接口隔离底层存储，支持 6 种持久化后端。

## 为什么重要

LLM 是无状态的——每次 API 调用独立，不记得上一轮说了什么。ChatMemory 解决了三个关键问题：

1. **上下文连续性**：多轮对话中 LLM 需要看到之前的交流才能给出连贯回复
2. **Token 预算控制**：滑动窗口裁剪防止 Prompt 越来越长、成本越来越高
3. **持久化恢复**：对话历史存库，服务重启后对话仍可继续

没有 ChatMemory，每次调用都是 "初次见面"，无法实现真实对话体验。

## 使用场景

- **多轮对话机器人**：客服机器人记住用户之前提到的问题
- **AI 助手**：编程助手记住项目上下文和之前的问答
- **教学陪伴**：记住学生的学习进度和薄弱知识点
- **会话迁移**：用户换设备后继续之前的对话（后端持久化）

## 核心机制

### 1. ChatMemory 接口

```java
// spring-ai-model: ChatMemory.java
public interface ChatMemory {

    String CONVERSATION_ID = "chat_memory_conversation_id";

    default void add(String conversationId, Message message) {
        this.add(conversationId, List.of(message));
    }

    void add(String conversationId, List<Message> messages);

    List<Message> get(String conversationId);

    void clear(String conversationId);
}
```

### 2. MessageWindowChatMemory：滑动窗口 + SystemMessage 保护

```java
// spring-ai-model: MessageWindowChatMemory.java
private List<Message> process(List<Message> memoryMsgs, List<Message> newMsgs) {
    List<Message> processed = new ArrayList<>();

    // ① 如果新消息中有新的 SystemMessage → 删除旧的所有 SystemMessage
    boolean hasNewSystemMsg = newMsgs.stream()
        .filter(SystemMessage.class::isInstance)
        .anyMatch(m -> !memoryMsgsSet.contains(m));
    memoryMsgs.stream()
        .filter(m -> !(hasNewSystemMsg && m instanceof SystemMessage))
        .forEach(processed::add);

    // ② 追加新消息
    processed.addAll(newMsgs);

    // ③ 超出 maxMessages 时裁剪：保留 SystemMessage，淘汰其他
    if (processed.size() <= this.maxMessages) return processed;

    int toRemove = processed.size() - this.maxMessages;
    List<Message> trimmed = new ArrayList<>();
    int removed = 0;
    for (Message msg : processed) {
        if (msg instanceof SystemMessage || removed >= toRemove) {
            trimmed.add(msg);           // SystemMessage 永远保留
        } else {
            removed++;                  // 淘汰最旧的非系统消息
        }
    }
    return trimmed;
}
```

关键设计：
- **SystemMessage 保护**：窗口满时优先淘汰对话消息，SystemMessage 始终保留（它是 AI 的角色定义，不能丢）
- **SystemMessage 替换**：新的 SystemMessage 会替换旧的（如：切换角色）
- **默认窗口**：20 条消息（约 4000-8000 token，取决于消息长度）

### 3. MessageChatMemoryAdvisor：自动注入历史

`MessageChatMemoryAdvisor` 在 Advisor 链中自动完成历史注入和回填：

```
before():
  ① 从 context 获取 conversationId
  ② chatMemory.get(conversationId) → 历史消息列表
  ③ 历史消息 + 当前 Prompt 消息 → 合并
  ④ SystemMessage 确保在列表首位
  ⑤ 将用户消息存入 chatMemory

after():
  ① 提取 AssistantMessage
  ② chatMemory.add(conversationId, assistantMessages)
  ③ 返回 ChatClientResponse
```

### 4. 多后端持久化

通过 `ChatMemoryRepository` 接口隔离存储：

| 后端 | 实现 | 适用场景 |
|------|------|----------|
| **内存（默认）** | `InMemoryChatMemoryRepository` | 开发/测试，重启丢失 |
| **JDBC** | `JdbcChatMemoryRepository` | 关系型数据库持久化 |
| **Redis** | `RedisChatMemoryRepository` | 高性能缓存，分布式 |
| **MongoDB** | `MongoDbChatMemoryRepository` | 文档存储，灵活 Schema |
| **Cassandra** | `CassandraChatMemoryRepository` | 大规模，高写入吞吐 |
| **Neo4j** | `Neo4jChatMemoryRepository` | 图结构对话关系 |
| **CosmosDB** | `CosmosDbChatMemoryRepository` | Azure 云原生 |

## 例子

### 基础多轮对话

```java
// 1. 创建记忆
ChatMemory memory = MessageWindowChatMemory.builder()
    .maxMessages(10)                        // 保留最近 10 条
    .build();

// 2. 注册 Advisor
ChatClient client = ChatClient.builder(chatModel)
    .defaultAdvisors(
        new MessageChatMemoryAdvisor(memory)  // 自动注入历史
    )
    .build();

// 3. 多轮对话（同一 conversationId）
String id = "user-123";
client.prompt().user("我叫张三").advisors(a -> a.param("chat_memory_conversation_id", id)).call().content();
client.prompt().user("我叫什么名字？").advisors(a -> a.param("chat_memory_conversation_id", id)).call().content();
// → "你叫张三。"（基于历史记忆）
```

### Redis 持久化（分布式部署）

```java
ChatMemory memory = MessageWindowChatMemory.builder()
    .chatMemoryRepository(new RedisChatMemoryRepository(redisTemplate))
    .maxMessages(20)
    .build();
```

### 与 Tool Calling 协作

Tool Calling 也会产生对话历史（UserMessage → AssistantMessage+ToolCall → ToolResponseMessage → AssistantMessage）。ChatMemory 与 ToolCallAdvisor 配合时，两者需要确保对话历史的一致性：

```java
ChatClient client = ChatClient.builder(chatModel)
    .defaultAdvisors(
        new MessageChatMemoryAdvisor(chatMemory),    // 持久化历史
        ToolCallAdvisor.builder()                    // 工具调用循环
            .build()
    )
    .defaultTools(new WeatherService())
    .build();
```

## 常见误区

1. **"ChatMemory 会自动管理 conversationId"**：不会。`conversationId` 需要业务代码显式传入（通过 `advisors(a -> a.param("chat_memory_conversation_id", id))`），框架只负责检索和更新。
2. **"SystemMessage 会被滑动窗口淘汰"**：不会。`MessageWindowChatMemory` 的裁剪逻辑明确保护 SystemMessage，只有非系统消息会被淘汰。
3. **"MessageChatMemoryAdvisor 和 ChatMemory 是一对一的"**：错。一个 `ChatMemory` 实例可以管理多个 conversationId，Advisor 是连接两者的桥梁。
4. **"窗口大小 = token 数"**：错。窗口大小是消息条数（默认 20），不是 token 数。不同消息长度差异大，20 条可能 4000 token，也可能 20000 token。需要 `LastMaxTokenSizeContentPurger` Advisor 做 token 级裁剪。

## 和其它概念的关系

- [[10-domains/java/spring-ai/概念_Spring_AI_Advisor拦截链]] — MessageChatMemoryAdvisor 在 Advisor 链中自动注入/回填历史
- [[10-domains/java/spring-ai/概念_Spring_AI_ToolCalling工具调用]] — ToolCallAdvisor 也需要管理对话历史，与 ChatMemory 协作
- [[10-domains/java/spring-ai/概念_Spring_AI_RAG检索增强生成]] — RAG + ChatMemory 实现 "基于历史的检索增强"
- [[10-domains/java/spring-ai/概念_Spring_AI_架构设计]] — ChatMemory 是架构中 Model 层的记忆抽象

## 来源

- `raw/sai/spring-ai-model/src/main/java/org/springframework/ai/chat/memory/ChatMemory.java`
- `raw/sai/spring-ai-model/src/main/java/org/springframework/ai/chat/memory/MessageWindowChatMemory.java`
- `raw/sai/spring-ai-client-chat/src/main/java/org/springframework/ai/chat/client/advisor/MessageChatMemoryAdvisor.java`
