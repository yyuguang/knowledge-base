---
type: concept
tags:
  - java
  - spring-ai
  - architecture
  - design-pattern
  - model
  - chat-client
summary: "Spring AI 的核心架构设计：Generic Model 抽象层、ChatClient 声明式 API 层、Advisor AOP 拦截链、多厂商可移植性实现原理"
sources:
  - "[[30-sources/repositories/Spring_AI_源码]]"
  - "raw/sai/spring-ai-model/src/main/java/org/springframework/ai/"
  - "raw/sai/spring-ai-client-chat/src/main/java/org/springframework/ai/chat/client/"
aliases:
  - Spring AI 架构
  - Spring AI 设计
status: evolving
confidence: 0.95
updated: "2026-07-14 15:05:00"
---

# 概念_Spring_AI_架构设计

![[Attachments/spring-ai-source-architecture.svg]]

## 定义

Spring AI 采用 **3 层架构**：Commons（基础内容/文档/评估）→ Model（模型抽象/消息/工具/转换器）→ Client-Chat（ChatClient/Advisor 链）。核心设计模式是 **Generic Model 抽象**——所有 AI 能力通过统一的 `Model<TReq, TRes>` 接口暴露，具体厂商通过 Spring Boot Auto-Configuration 注入实现。

## 为什么重要

这是 Spring AI 区别于 LangChain/LlamaIndex 的根本设计。它决定了：
- 你的业务代码**只依赖接口，不依赖厂商 SDK**
- 切换 OpenAI → Anthropic → Ollama **只需换 Starter，零代码修改**
- 所有横切关注点（日志/记忆/安全/RAG）通过 Advisor 链**统一注入**

如果不理解这个架构，你会把 Spring AI 用成 "又一个 HTTP 客户端"，失去可移植性和扩展性。

## 使用场景

- **多厂商切换**：开发用 Ollama（免费本地），生产用 OpenAI/Azure（企业级 SLA）
- **渐进式增强**：先跑通 Chat，再逐步加 ChatMemory→Tool Calling→RAG→StructuredOutput
- **模板化 Prompt**：通过 `PromptTemplate` + `ChatClient` 实现可复用的对话模板
- **企业级 Observability**：Micrometer 自动埋点，token 消耗/延迟/错误全链路追踪

## 核心机制

### 1. Generic Model 抽象

```java
// spring-ai-model: org/springframework/ai/model/Model.java
public interface Model<TReq extends ModelRequest<?>, TRes extends ModelResponse<?>> {
    TRes call(TReq request);
}

// spring-ai-model: org/springframework/ai/model/StreamingModel.java
public interface StreamingModel<TReq extends ModelRequest<?>, TResChunk extends ModelResponse<?>> {
    Flux<TResChunk> stream(TReq request);
}
```

所有 AI 能力都继承这两个接口：

| 能力 | 接口 | 继承关系 |
|------|------|----------|
| Chat | `ChatModel` | `Model<Prompt, ChatResponse>` + `StreamingChatModel` |
| Embedding | `EmbeddingModel` | `Model<EmbeddingRequest, EmbeddingResponse>` |
| Image | `ImageModel` | `Model<ImagePrompt, ImageResponse>` |
| Audio Transcription | `TranscriptionModel` | `Model<AudioTranscriptionPrompt, AudioTranscriptionResponse>` |
| Text-to-Speech | `TextToSpeechModel` | `Model<TextToSpeechPrompt, TextToSpeechResponse>` + `StreamingTextToSpeechModel` |
| Moderation | `ModerationModel` | `Model<ModerationPrompt, ModerationResponse>` |

### 2. ChatModel：对话的统一入口

```java
// spring-ai-model: org/springframework/ai/chat/model/ChatModel.java
public interface ChatModel extends Model<Prompt, ChatResponse>, StreamingChatModel {

    // 快捷方法：单条文本
    default String call(String message) {
        Prompt prompt = new Prompt(new UserMessage(message));
        Generation generation = call(prompt).getResult();
        return (generation != null) ? generation.getOutput().getText() : "";
    }

    // 核心方法：Prompt → ChatResponse
    @Override
    ChatResponse call(Prompt prompt);

    // 流式接口（默认抛 UnsupportedOperationException）
    default Flux<ChatResponse> stream(Prompt prompt) {
        throw new UnsupportedOperationException("streaming is not supported");
    }

    // 获取默认 ChatOptions（各厂商实现覆盖此方法提供默认参数）
    default ChatOptions getDefaultOptions() {
        return ChatOptions.builder().build();
    }
}
```

### 3. Prompt / Message / ChatResponse 数据模型

```
Prompt
├── List<Message> instructions        ← 消息列表
│   ├── SystemMessage("你是专业助手")   ← messageType = SYSTEM
│   ├── UserMessage("今天天气？")       ← messageType = USER
│   ├── AssistantMessage("天气晴")      ← messageType = ASSISTANT
│   │   └── List<ToolCall>             ← 工具调用请求
│   └── ToolResponseMessage("22°C")    ← messageType = TOOL
└── ChatOptions options                ← 模型参数

ChatResponse
├── List<Generation> results           ← 回复列表（n > 1 时多个候选）
│   └── AssistantMessage output        ← 助手回复
└── ChatResponseMetadata metadata      ← Usage / RateLimit / PromptMetadata
```

### 4. Prompt Template 体系

```java
// 模板接口（spring-ai-commons）
@FunctionalInterface
public interface TemplateRenderer extends BiFunction<String, Map<String, Object>, String> {}

// PromptTemplate 支持 {placeholder} 风格占位符
PromptTemplate template = new PromptTemplate("翻译成{language}：{text}");
Prompt prompt = template.create(Map.of("language", "英文", "text", "你好世界"));
// → Prompt{ messages=[UserMessage("翻译成英文：你好世界")] }
```

### 5. ChatClient → Advisor 链架构

```
ChatClient.builder(chatModel)
    .defaultSystem("...")         ← 预设 SystemMessage
    .defaultTools(tools)          ← 注册 ToolCallback
    .defaultAdvisors(advisors)    ← 注册 Advisor 拦截链
    .build()
        │
        ▼
prompt().user("hello")
    .call()
        │
        ▼
DefaultAroundAdvisorChain.next(request)
    ├── Advisor[0].before()  ← SimpleLoggerAdvisor
    ├── Advisor[1].before()  ← MessageChatMemoryAdvisor（注入历史）
    ├── Advisor[2].before()  ← SafeGuardAdvisor（安全检查）
    ├── Advisor[3].before()  ← RetrievalAugmentationAdvisor（RAG）
    ├── Advisor[4].before()  ← ToolCallAdvisor
    │       │
    │       └── ChatModelCallAdvisor（终端 Advisor，实际调用 ChatModel.call()）
    │       │       ↓
    │       │   ChatModel.call(prompt)
    │       │       ↓
    │       │   OpenAI / Anthropic / Ollama / DashScope ...
    │       │
    │       ← ChatResponse
    ├── Advisor[4].after()
    ├── Advisor[3].after()
    ... (逆序执行 after)
    └── 返回 ChatClientResponse
```

### 6. 多厂商可移植性原理

```
                      业务代码
                         │
                    ChatModel (接口)
                    ┌────┼────────────────────┐
                    │    │                    │
            OpenAiChatModel  AnthropicChatModel  OllamaChatModel  ...
                    │    │                    │
            OpenAiApi      AnthropicApi        OllamaApi
              (WebClient)    (WebClient)        (RestClient)
                    │    │                    │
            api.openai.com  api.anthropic.com  localhost:11434
```

Auto-Configuration 根据 classpath 中的 Starter 自动创建对应的 ChatModel Bean：

```
spring-ai-openai-spring-boot-starter    → OpenAiChatModel (Bean)
spring-ai-ollama-spring-boot-starter    → OllamaChatModel (Bean)
spring-ai-bedrock-converse-spring-boot-starter → BedrockConverseProxyChatModel (Bean)
```

## 例子

在实际的 `LLMStudy` 项目中（基于 Spring AI Alibaba DashScope）：

```java
// 1. 引入依赖：spring-ai-alibaba-starter-dashscope
// 2. 配置 yml：spring.ai.dashscope.api-key + chat.options
// 3. 注入使用：
@Service
public class LlmChatService {
    private final ChatClient chatClient;

    public LlmChatService(ChatModel chatModel) {
        // ChatModel 由 Spring Boot 自动注入 DashScopeChatModel
        this.chatClient = ChatClient.builder(chatModel)
            .defaultSystem("你是一个简洁专业的 AI 助手。")
            .build();
    }

    public String chat(String prompt) {
        return chatClient.prompt().user(prompt).call().content();
    }
}
```

## 常见误区

1. **"ChatModel 就是 OpenAI 的 ChatCompletion"**：错。ChatModel 是厂商无关的抽象，OpenAiChatModel 只是 14+ 实现之一。
2. **"ChatClient 每次 prompt() 都新建"**：不完全对。`prompt()` 返回的是请求规格对象，但 ChatClient 本身是不可变的，配置（defaultSystem/defaultTools/defaultAdvisors）在构建时固定。
3. **"StreamingChatModel 的 stream() 一定支持"**：错。`ChatModel` 接口虽然继承了 `StreamingChatModel`，但 `stream()` 的默认实现是抛 `UnsupportedOperationException`，具体厂商不一定支持流式。
4. **"Prompt Template 可以替代所有 Prompt 构造"**：错。PromptTemplate 适合简单的文本模板，复杂场景（StructuredOutput 的 JSON Schema 注入、多消息编排）需要直接构造 `Prompt`。

## 源码级理解

Spring AI 的源码主线可以从 `DefaultChatClientRequestSpec.call()` 往下追：

1. `ChatClientAutoConfiguration` 创建 prototype 作用域的 `ChatClient.Builder`。
2. `DefaultChatClientBuilder` 持有一个 `DefaultChatClientRequestSpec defaultRequest`，所有 `defaultSystem/defaultTools/defaultAdvisors/defaultOptions` 都是在改这个默认请求规格。
3. `DefaultChatClient.prompt()` 复制默认请求规格，得到一次调用专属的可变 `RequestSpec`。
4. `call()` 或 `stream()` 调用 `buildAdvisorChain()`，临时追加 `ChatModelCallAdvisor` / `ChatModelStreamAdvisor` 作为终端 Advisor。
5. `DefaultAroundAdvisorChain.nextCall()` 逐个弹出 Advisor，最后由终端 Advisor 调用 `ChatModel.call(prompt)`。

这个设计让 ChatClient 本体保持不可变，让每次请求可以安全地叠加本次 user/system/tools/advisors。真正可变的是 request spec，不是 client。

源码入口：

| 入口 | 文件 | 看点 |
|------|------|------|
| Builder 默认配置 | `DefaultChatClientBuilder` | 默认配置全部写入 `defaultRequest` |
| 每次请求复制 | `DefaultChatClient.prompt()` | copy constructor 复制默认请求规格 |
| 终端 Advisor 注入 | `DefaultChatClientRequestSpec.buildAdvisorChain()` | 自动追加 `ChatModelCallAdvisor` / `ChatModelStreamAdvisor` |
| 模型抽象 | `ChatModel` | `call(Prompt)` 和 `stream(Prompt)` 是 provider 边界 |
| 自动装配 | `ChatClientAutoConfiguration` | `ChatClient.Builder` 是 prototype |

## 和其它概念的关系

- [[10-domains/java/spring-ai/概念_Spring_AI_ChatClient_API]] — ChatClient 是架构中 Client-Chat 层的核心 API
- [[10-domains/java/spring-ai/概念_Spring_AI_Advisor拦截链]] — Advisor 是架构中实现横切关注点的 AOP 机制
- [[10-domains/java/spring-ai/概念_Spring_AI_ToolCalling工具调用]] — Tool Calling 通过 ToolCallAdvisor 在 Advisor 链中实现
- [[10-domains/java/spring-ai/概念_Spring_AI_RAG检索增强生成]] — RAG 通过 RetrievalAugmentationAdvisor 在 Advisor 链中实现
- [[10-domains/java/spring-ai/主题_Spring_AI源码架构_综述]] — 汇总源码主线、执行链路和架构图

## 来源

- `raw/sai/spring-ai-model/src/main/java/org/springframework/ai/model/Model.java`
- `raw/sai/spring-ai-model/src/main/java/org/springframework/ai/chat/model/ChatModel.java`
- `raw/sai/spring-ai-client-chat/src/main/java/org/springframework/ai/chat/client/ChatClient.java`
- `raw/sai/spring-ai-client-chat/src/main/java/org/springframework/ai/chat/client/DefaultChatClient.java`
- `raw/sai/spring-ai-client-chat/src/main/java/org/springframework/ai/chat/client/DefaultChatClientBuilder.java`
- `raw/sai/spring-ai-model/src/main/java/org/springframework/ai/chat/prompt/Prompt.java`
