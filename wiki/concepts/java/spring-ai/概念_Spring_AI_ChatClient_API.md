---
type: concept
tags:
  - java
  - spring-ai
  - chat-client
  - fluent-api
  - builder-pattern
summary: "Spring AI ChatClient 声明式 Fluent API：Builder 模式、PromptUserSpec/PromptSystemSpec、CallResponseSpec/StreamResponseSpec、entity() 结构化输出、与 RestClient/WebClient 的设计对比"
sources:
  - "sai/spring-ai-client-chat/src/main/java/org/springframework/ai/chat/client/ChatClient.java"
  - "sai/spring-ai-client-chat/src/main/java/org/springframework/ai/chat/client/DefaultChatClient.java"
aliases:
  - ChatClient
  - ChatClient API
  - Spring AI Chat API
status: evolving
confidence: 0.95
updated: "2026-05-12 00:00:00"
---

# 概念_Spring_AI_ChatClient_API

![[Attachments/spring-ai-chatclient-call-flow.svg]]

## 定义

ChatClient 是 Spring AI 1.0 引入的**声明式 Fluent API 客户端**，对标 Spring 生态的 `RestClient` / `WebClient`。它将底层 `ChatModel` 封装为不可变对象，通过 Builder 模式预设系统提示词、工具、Advisor 链，每次调用只需传入用户消息即可获得回复或流式响应。

## 为什么重要

ChatClient 是使用 Spring AI 的**唯一推荐入口**。它解决了直接使用 ChatModel 的三个痛点：
1. 每次调用都需要手动构造完整的 `Prompt`（SystemMessage + UserMessage + ChatOptions）
2. 工具注册和管理需要业务代码手动编排
3. 横切关注点（日志、记忆、安全）无法统一注入

ChatClient 让 AI 调用变得像 `RestClient.get().uri("/api").retrieve().body(MyDto.class)` 一样简洁。

## 使用场景

- **快速原型**：`ChatClient.create(chatModel).prompt().user("hello").call().content()`
- **预设角色**：通过 `defaultSystem()` 预设 AI 助手角色，所有调用自动包含
- **工具集成**：`defaultTools(weatherService)` 自动发现 `@Tool` 方法
- **多轮对话**：`defaultAdvisors(new MessageChatMemoryAdvisor(memory))` 自动管理历史
- **流式输出**：`.stream().content()` 返回 `Flux<String>`，SSE 推送给前端
- **结构化提取**：`.call().entity(PersonDto.class)` 自动 JSON → DTO

## 核心机制

### Builder（构建期）

```java
// spring-ai-client-chat: ChatClient.Builder
public interface Builder {
    // --- 默认系统/用户提示词（每次 prompt() 自动附加） ---
    Builder defaultSystem(String text);
    Builder defaultSystem(Consumer<PromptSystemSpec> spec);

    Builder defaultUser(String text);
    Builder defaultUser(Consumer<PromptUserSpec> spec);

    // --- 默认工具注册 ---
    Builder defaultTools(Object... toolObjects);       // 自动扫描 @Tool 注解
    Builder defaultToolCallbacks(ToolCallback...);
    Builder defaultToolCallbacks(ToolCallbackProvider...);
    Builder defaultToolNames(String... toolNames);     // 工具白名单
    Builder defaultToolContext(Map<String, Object>);   // 工具执行上下文

    // --- 默认参数 + Advisor ---
    Builder defaultOptions(ChatOptions.Builder options);
    Builder defaultAdvisors(Advisor... advisors);
    Builder defaultAdvisors(Consumer<AdvisorSpec> spec);

    // --- 构建不可变 ChatClient ---
    ChatClient build();
}
```

### RequestSpec（每次调用的请求规格）

```java
// ChatClient.ChatClientRequestSpec
ChatClientRequestSpec system(String text);      // 单次覆盖 defaultSystem
ChatClientRequestSpec user(String text);        // 设置用户消息
ChatClientRequestSpec advisors(Advisor...);     // 单次追加 Advisor
ChatClientRequestSpec tools(Object...);         // 单次追加工具
ChatClientRequestSpec options(Consumer<...>);   // 单次覆盖 ChatOptions

CallResponseSpec call();                        // 终止：同步调用
StreamResponseSpec stream();                     // 终止：流式调用
```

### CallResponseSpec（同步结果）

```java
// ChatClient.CallResponseSpec
String content();                                                     // 纯文本
ChatResponse chatResponse();                                          // 完整响应（含 Usage/FinishReason）
ChatClientResponse chatClientResponse();                              // 包装响应

<T> T entity(Class<T> type);                                          // JSON → DTO（自动 Schema 注入）
<T> T entity(ParameterizedTypeReference<T> type);                     // 泛型 DTO
<T> T entity(StructuredOutputConverter<T> converter);                 // 自定义转换器

<T> ResponseEntity<ChatResponse, T> responseEntity(Class<T> type);    // 含元数据响应
```

### StreamResponseSpec（流式结果）

```java
// ChatClient.StreamResponseSpec
Flux<String> content();                    // 纯文本流（适合 SSE 推送）
Flux<ChatResponse> chatResponse();         // 完整响应流（每 chunk 一个 ChatResponse）
Flux<ChatClientResponse> chatClientResponse();
```

### PromptUserSpec / PromptSystemSpec（高级定制）

```java
// 都支持：text / resource / params / media / metadata
interface PromptUserSpec {
    PromptUserSpec text(String text);
    PromptUserSpec text(Resource text, Charset charset);
    PromptUserSpec params(Map<String, Object> p);
    PromptUserSpec param(String k, Object v);
    PromptUserSpec media(Media... media);           // 多模态：图片/音频
    PromptUserSpec media(MimeType mimeType, URL url);
}

interface PromptSystemSpec {
    PromptSystemSpec text(String text);
    PromptSystemSpec text(Resource text, Charset charset);
    PromptSystemSpec params(Map<String, Object> p);
}
```

## 例子

### 基础对话

```java
ChatClient chatClient = ChatClient.create(chatModel);
String reply = chatClient.prompt().user("Hello").call().content();
```

### 预设角色与模板

```java
ChatClient client = ChatClient.builder(chatModel)
    .defaultSystem("你是{role}，请用{language}回答。")  // 模板占位符
    .build();

String reply = client.prompt()
    .user("介绍一下Spring AI")
    .system(spec -> spec.param("role", "Java技术专家").param("language", "中文"))
    .call().content();
```

### 流式输出到 SSE

```java
@GetMapping(value = "/chat/stream", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
public Flux<String> streamChat(@RequestParam String q) {
    return chatClient.prompt().user(q).stream().content();
}
```

### 结构化输出

```java
record Person(String name, int age, String role) {}

Person person = chatClient.prompt()
    .user("张三，30岁，Java开发工程师。提取信息。")
    .call()
    .entity(Person.class);
// → Person[name="张三", age=30, role="Java开发工程师"]
```

## 常见误区

1. **"ChatClient 每次 prompt() 都新建对象"**：`prompt()` 返回的 `ChatClientRequestSpec` 是每次调用的可变规格，但 ChatClient 本身是构建后不可变的。Builder 中设置的所有 default* 值在每次调用中自动包含。
2. **"defaultUser 和 user() 是冲突的"**：不会。`defaultUser()` 设置默认用户消息前缀（可选），`.user()` 是每次调用的实际用户消息。
3. **"entity() 和 content() 互斥"**：对。调用 `entity()` 时框架自动注入 JSON Schema 到 System Prompt，LLM 必须返回 JSON，否则转换失败。
4. **"ChatClient 线程不安全"**：ChatClient 是不可变对象，天然线程安全。可以在多个线程中同时调用同一个 ChatClient 实例。

## 源码级执行链

`ChatClient` 的 API 看起来是 fluent builder，但源码里的关键分界是“配置阶段”和“执行阶段”：

| 阶段 | 关键类/方法 | 行为 |
|------|-------------|------|
| 默认配置 | `DefaultChatClientBuilder.default*()` | 写入 `defaultRequest` |
| 构建客户端 | `DefaultChatClientBuilder.build()` | 创建不可变 `DefaultChatClient` |
| 开始请求 | `DefaultChatClient.prompt()` | copy 默认请求规格 |
| 请求追加 | `DefaultChatClientRequestSpec.user/system/tools/advisors/options()` | 改当前请求规格 |
| 触发调用 | `call()` / `stream()` | 构建 Advisor 链和 `ChatClientRequest` |
| 观测包装 | `DefaultCallResponseSpec.doGetObservableChatClientResponse()` | 创建 `AI_CHAT_CLIENT` Observation |
| 链式执行 | `advisorChain.nextCall()` / `nextStream()` | 进入 Advisor 链 |
| 终端调用 | `ChatModelCallAdvisor` / `ChatModelStreamAdvisor` | 调 provider 的 `ChatModel` |

关键源码细节：

- `call()` 和 `stream()` 都会调用 `buildAdvisorChain()`，因此终端模型调用 Advisor 是每次请求临时加入的。
- `DefaultCallResponseSpec.entity()` 不直接调用模型，而是先把 `ChatClientAttributes.OUTPUT_FORMAT` 写入 request context，再走同一条 Advisor 链。
- `stream()` 会通过 `ChatClientMessageAggregator.aggregateChatClientResponse()` 聚合流式响应，用于 Observation 记录完整响应。
- `mutate()` 会从当前 `DefaultChatClientRequestSpec` 反建一个 builder，适合复制已有 ChatClient 的默认配置后小幅变更。

## 源码级扩展点

- **全局默认行为**：实现 `ChatClientCustomizer`，在 `ChatClientBuilderConfigurer` 中被 ordered stream 应用。
- **单次请求增强**：通过 `prompt().advisors(...)` 或 `prompt().toolCallbacks(...)` 只影响本次请求。
- **结构化输出**：通过 `entity(Class)` / `entity(StructuredOutputConverter)` 写入输出格式上下文，而不是新建另一套调用路径。
- **观测定制**：提供 `ChatClientObservationConvention` 或 `AdvisorObservationConvention` Bean，改变 Observation 标签和字段。

## 和其它概念的关系

- [[concepts/java/spring-ai/概念_Spring_AI_架构设计]] — ChatClient 在 3 层架构中的 Client-Chat 层
- [[concepts/java/spring-ai/概念_Spring_AI_Advisor拦截链]] — `defaultAdvisors()` 和 `advisors()` 注册 Advisor
- [[concepts/java/spring-ai/概念_Spring_AI_ToolCalling工具调用]] — `defaultTools()` 和 `tools()` 注册 ToolCallback
- [[concepts/java/spring-ai/概念_Spring_AI_StructuredOutput结构化输出]] — `entity()` 触发 StructuredOutputConverter
- [[overview/java/spring-ai/主题_Spring_AI源码架构_综述]] — ChatClient 在完整源码主线中的位置

## 来源

- `sai/spring-ai-client-chat/src/main/java/org/springframework/ai/chat/client/ChatClient.java` (298 行，完整接口定义)
- `sai/spring-ai-client-chat/src/main/java/org/springframework/ai/chat/client/DefaultChatClient.java` (实现)
- `sai/spring-ai-client-chat/src/main/java/org/springframework/ai/chat/client/DefaultChatClientBuilder.java`
- `sai/spring-ai-client-chat/src/main/java/org/springframework/ai/chat/client/DefaultChatClientUtils.java`
