---
type: concept
tags:
  - java
  - spring-ai
  - advisor
  - aop
  - interceptor
  - chain-of-responsibility
summary: "Spring AI Advisor 拦截链：BaseAdvisor 模板方法（before/after）实现 AOP 环绕拦截，12 个内置 Advisor 覆盖日志/记忆/安全/工具/RAG 等横切关注点，链式调用机制与 Advisor 排序策略"
sources:
  - "raw/code/spring-ai/spring-ai-client-chat/src/main/java/org/springframework/ai/chat/client/advisor/api/BaseAdvisor.java"
  - "raw/code/spring-ai/spring-ai-client-chat/src/main/java/org/springframework/ai/chat/client/advisor/api/AdvisorChain.java"
  - "raw/code/spring-ai/spring-ai-client-chat/src/main/java/org/springframework/ai/chat/client/advisor/SimpleLoggerAdvisor.java"
aliases:
  - Advisor 链
  - Advisor Chain
  - Spring AI 拦截器
status: evolving
confidence: 0.95
updated: "2026-05-12 00:00:00"
---

# 概念_Spring_AI_Advisor拦截链

![[Attachments/spring-ai-chatclient-call-flow.svg]]

## 定义

Advisor 是 Spring AI 中的 **AOP 环绕拦截器**，对标 Spring MVC 的 `HandlerInterceptor`。每个 Advisor 通过统一的 `before()` / `after()` 模板方法，在每次 LLM 调用前后插入横切逻辑（日志、记忆注入、安全检查、工具调用、RAG 检索）。多个 Advisor 按 `order` 排序组成**责任链**，`before()` 正序执行 → `ChatModel.call()` → `after()` 逆序执行。

## 为什么重要

Advisor 链是 Spring AI 实现**横切关注点统一注入**的核心机制。没有它：
1. 日志记录需要散布在每个 ChatClient 调用点
2. 对话记忆管理需要业务代码手动编排
3. Tool Calling 的循环检测无法透明嵌入
4. RAG 检索增强需要每次手动拼接 pipeline
5. 不同关注点之间无法解耦，代码会爆炸

有了 Advisor 链，用户只需 `defaultAdvisors(new MessageChatMemoryAdvisor(memory), new SimpleLoggerAdvisor())` 即可组合所有能力。

## 使用场景

- **日志审计**：`SimpleLoggerAdvisor` 记录每次请求/响应的完整内容
- **对话记忆**：`MessageChatMemoryAdvisor` 自动注入历史消息、回填助手回复
- **安全检查**：`SafeGuardAdvisor` 拦截敏感词/注入攻击
- **工具调用**：`ToolCallAdvisor` 实现 ReAct 循环（思考→行动→观察）
- **RAG 增强**：`RetrievalAugmentationAdvisor` 自动执行 7 步检索流水线
- **结构化校验**：`StructuredOutputValidationAdvisor` 校验 LLM 输出的 JSON Schema
- **自定义扩展**：实现 `BaseAdvisor` 接口插入任意自定义逻辑

## 核心机制

### 1. BaseAdvisor 模板方法

```java
// spring-ai-client-chat: BaseAdvisor.java
public interface BaseAdvisor extends CallAdvisor, StreamAdvisor {

    Scheduler DEFAULT_SCHEDULER = Schedulers.boundedElastic();

    // 模板方法：子类只需实现 before() 和 after()
    @Override
    default ChatClientResponse adviseCall(ChatClientRequest request, CallAdvisorChain chain) {
        ChatClientRequest processed = before(request, chain);       // ① 前置处理
        ChatClientResponse response = chain.nextCall(processed);   // ② 调用下一节点
        return after(response, chain);                              // ③ 后置处理
    }

    @Override
    default Flux<ChatClientResponse> adviseStream(...) {
        return Mono.just(request)
            .publishOn(getScheduler())
            .map(r -> this.before(r, chain))       // ① 前置（响应式线程池）
            .flatMapMany(chain::nextStream)         // ② 流式转发
            .map(response -> {
                if (onFinishReason().test(response))
                    response = after(response, chain); // ③ 仅最终 chunk 触发 after
                return response;
            });
    }

    // 子类必须实现
    ChatClientRequest before(ChatClientRequest request, AdvisorChain chain);
    ChatClientResponse after(ChatClientResponse response, AdvisorChain chain);
}
```

### 2. Advisor 链执行流程

```
prompt().user("hello").call()
    │
    ▼
DefaultAroundAdvisorChain.next(request)        ← 正序执行 before()
    │
    ├── Advisor[0].before()  order=0   SimpleLoggerAdvisor（记录请求日志）
    │       └── chain.nextCall(processedRequest)
    │               │
    ├── Advisor[1].before()  order=100 MessageChatMemoryAdvisor（注入历史消息）
    │       └── chain.nextCall(processedRequest)
    │               │
    ├── Advisor[2].before()  order=200 SafeGuardAdvisor（安全检查）
    │       └── chain.nextCall(processedRequest)
    │               │
    ├── Advisor[3].before()  order=250 RetrievalAugmentationAdvisor（RAG）
    │       └── chain.nextCall(processedRequest)
    │               │
    ├── Advisor[4].before()  order=300 ToolCallAdvisor（工具调用循环）
    │       │
    │       └── ChatModelCallAdvisor（终端 Advisor，实际调用 ChatModel.call()）
    │               │
    │               ▼
    │           ChatModel.call(prompt)
    │               │
    │               ▼
    │           OpenAI / Anthropic / Ollama / DashScope ...
    │               │
    │           ← ChatResponse
    │       │
    │       ← (ToolCallAdvisor 在此检测 toolCalls，循环调用)
    │
    ├── Advisor[4].after()  ← 逆序执行 after()
    ├── Advisor[3].after()  ← 将检索到的 Documents 注入 Response Metadata
    ├── Advisor[2].after()
    ├── Advisor[1].after()  ← 将 AssistantMessage 写回 ChatMemory
    └── Advisor[0].after()  ← 记录响应日志

    ▼
返回 ChatClientResponse
```

### 3. 流式调用中的特殊处理

流式场景中 `after()` **只在最终 chunk（finishReason != null）时触发**：

- 调用期：`before()` 正常执行
- 流式期：每个 chunk 透传，不触发 `after()`
- 结束期：检测 `finishReason`，仅最终 chunk 触发 `after()`
- 工具调用：`ToolCallAdvisor` 通过 `ChatClientMessageAggregator` 聚合所有 chunk 后判断是否 toolCalls

### 4. 12 个内置 Advisor

| Advisor | Order | 职责 |
|---------|-------|------|
| `ChatModelCallAdvisor` | `LOWEST_PRECEDENCE` | 终端 Advisor，实际调用 ChatModel（内部使用） |
| `ChatModelStreamAdvisor` | `LOWEST_PRECEDENCE` | 终端 Advisor，流式调用（内部使用） |
| `ToolCallAdvisor` | `HIGHEST_PRECEDENCE + 300` | 工具调用循环（ReAct 模式） |
| `MessageChatMemoryAdvisor` | `DEFAULT_CHAT_MEMORY` | 历史消息注入与回填 |
| `PromptChatMemoryAdvisor` | — | 以 Prompt 形式注入历史 |
| `VectorStoreChatMemoryAdvisor` | — | 基于 VectorStore 的长期记忆 |
| `RetrievalAugmentationAdvisor` | 0 | 7 步 Modular RAG 流水线 |
| `SafeGuardAdvisor` | — | 输入/输出安全检查 |
| `StructuredOutputValidationAdvisor` | — | JSON Schema 输出校验 |
| `SimpleLoggerAdvisor` | 0 | 请求/响应日志 |
| `QuestionAnswerAdvisor` | — | 基于文档的 QA |
| `LastMaxTokenSizeContentPurger` | — | 超出 token 限制时裁减历史 |

### 5. Advisor 排序策略

```
最高优先级 (first to execute before, last to execute after)
  │  ToolCallAdvisor          (-300)   ← 工具调用循环，最先拦截
  │  SafeGuardAdvisor          (-200)   ← 安全检查
  │  RetrievalAugmentation...  (0)      ← RAG 检索增强
  │  SimpleLoggerAdvisor       (0)      ← 日志记录
  │  MessageChatMemoryAdvisor  (+100)   ← 记忆管理，靠近终端
  ▼
最低优先级 (last to execute before, first to execute after)
```

排序原则：
1. **横切面优先**：日志、安全等横切关注点放在外层
2. **功能增强居中**：RAG、记忆注入放在中间
3. **工具调靠近终端**：ToolCallAdvisor 需要最接近 ChatModel，因为它会发起多次循环调用

## 例子

### 完整 Advisor 链配置

```java
ChatClient chatClient = ChatClient.builder(chatModel)
    .defaultAdvisors(
        new SimpleLoggerAdvisor(),                              // ① 最外层：记录日志
        new SafeGuardAdvisor(),                                 // ② 安全检查
        RetrievalAugmentationAdvisor.builder()                  // ③ RAG 增强
            .documentRetriever(vectorStore)
            .build(),
        new MessageChatMemoryAdvisor(chatMemory),               // ④ 对话记忆
        ToolCallAdvisor.builder()                               // ⑤ 工具调用（最内层）
            .toolCallingManager(manager)
            .build()
    )
    .defaultTools(weatherService, calculator)                   // 注册 @Tool 方法
    .build();

String reply = chatClient.prompt()
    .user("北京今天天气如何？")
    .call().content();
// 依次经过：日志→安全→RAG→记忆→工具调用→ChatModel→逆序返回
```

### 自定义 Advisor

```java
public class TimingAdvisor implements BaseAdvisor {
    @Override
    public ChatClientRequest before(ChatClientRequest req, AdvisorChain chain) {
        req.context().put("startTime", System.currentTimeMillis());
        return req;
    }

    @Override
    public ChatClientResponse after(ChatClientResponse res, AdvisorChain chain) {
        long start = (long) res.context().get("startTime");
        log.info("LLM call took {}ms", System.currentTimeMillis() - start);
        return res;
    }

    @Override
    public int getOrder() { return 10; }
}
```

## 常见误区

1. **"Advisor 的 before/after 是成对调用的"**：同步调用中是的。但流式调用中 `after()` 只在最终 chunk 触发，中间 chunk 直接透传。
2. **"ToolCallAdvisor 放在链的首位（最低 order）"**：错。ToolCallAdvisor 应该 **order 最低（数值最小）**，因为它在 `before()` 中发起工具调用循环，需要最接近 ChatModel。但实际它排在首位只是因为 `HIGHEST_PRECEDENCE + 300` 的默认值。
3. **"BaseAdvisor 只能实现 CallAdvisor"**：`BaseAdvisor` 同时实现 `CallAdvisor` 和 `StreamAdvisor`，一个类即可同时支持同步和流式。
4. **"Advisor 链可以无限制嵌套"**：理论上是。但 ToolCallAdvisor 的循环机制可能导致多次进入链——需注意幂等性和性能。

## 源码级执行细节

`DefaultAroundAdvisorChain` 是 Advisor 链的真实执行器，几个源码细节会影响你如何写自定义 Advisor：

| 机制 | 源码位置 | 含义 |
|------|----------|------|
| 拆分两条链 | `pushAll()` | 同一个 Advisor 如果实现 `CallAdvisor` 和 `StreamAdvisor` 会分别进入同步/流式队列 |
| 排序 | `reOrder()` + `OrderComparator.sort()` | 每次 push 后按 Spring `Ordered` 规则重排 |
| 消费式执行 | `nextCall()` / `nextStream()` 中 `Deque.pop()` | 每次链执行会弹出当前 Advisor，因此链对象不是可无限复用的游标 |
| 观测包装 | `AdvisorObservationDocumentation.AI_ADVISOR` | 每个 Advisor 都有单独 Observation，记录 advisorName、order、request/response |
| 递归复制 | `copy(after)` | 从原始 Advisor 列表中截取 `after` 后面的剩余链，供 ToolCallAdvisor 递归调用 |
| 流式聚合 | `ChatClientMessageAggregator.aggregateChatClientResponse()` | 流式响应会聚合后写入 observation context |

`copy(after)` 是理解 Tool Calling 的关键。`ToolCallAdvisor` 如果在工具调用后直接调原链，会再次进入自己导致递归错误；它必须复制“自己之后”的链，让后续 Advisor 和终端 `ChatModel*Advisor` 执行。

自定义 Advisor 的源码级注意点：

- `before()` 修改 request 时要用 `chatClientRequest.mutate()`，避免丢失 context。
- `after()` 修改 response 时也要保留原 context，RAG 的 `DOCUMENT_CONTEXT` 依赖 context 传递。
- 流式 `after()` 只在最终聚合 chunk 上有意义，不能假设每个 chunk 都有完整 `ChatResponse`。
- 如果 Advisor 内部会重新进入链，需要明确使用 `chain.copy(this)` 还是完整链，否则可能重复执行上游 Advisor。

## 和其它概念的关系

- [[concepts/java/spring-ai/概念_Spring_AI_架构设计]] — Advisor 链是架构中 Client-Chat 层的 AOP 机制，对标 Spring MVC Interceptor
- [[concepts/java/spring-ai/概念_Spring_AI_ChatClient_API]] — `defaultAdvisors()` 和 `advisors()` 注册 Advisor 到 ChatClient
- [[concepts/java/spring-ai/概念_Spring_AI_ToolCalling工具调用]] — ToolCallAdvisor 是 Advisor 链中最复杂的拦截器，实现 ReAct Agent 循环
- [[concepts/java/spring-ai/概念_Spring_AI_RAG检索增强生成]] — RetrievalAugmentationAdvisor 将 7 步 RAG 流水线嵌入 Advisor 链
- [[overview/java/spring-ai/主题_Spring_AI源码架构_综述]] — Advisor 链在完整源码主线中的位置

## 来源

- `raw/code/spring-ai/spring-ai-client-chat/src/main/java/org/springframework/ai/chat/client/advisor/api/BaseAdvisor.java`
- `raw/code/spring-ai/spring-ai-client-chat/src/main/java/org/springframework/ai/chat/client/advisor/DefaultAroundAdvisorChain.java`
- `raw/code/spring-ai/spring-ai-client-chat/src/main/java/org/springframework/ai/chat/client/advisor/SimpleLoggerAdvisor.java`
- `raw/code/spring-ai/spring-ai-client-chat/src/main/java/org/springframework/ai/chat/client/advisor/MessageChatMemoryAdvisor.java`
- `raw/code/spring-ai/spring-ai-client-chat/src/main/java/org/springframework/ai/chat/client/advisor/ToolCallAdvisor.java`
