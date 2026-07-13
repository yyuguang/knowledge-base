---
type: concept
tags:
  - java
  - spring-ai
  - tool-calling
  - function-calling
  - agent
  - react
summary: "Spring AI Tool Calling：@Tool 注解声明工具 → MethodToolCallback 反射封装 → ToolCallAdvisor 编排 ReAct Agent 循环（思考→行动→观察），支持 parallelToolCalls 和 returnDirect 模式"
sources:
  - "raw/code/spring-ai/spring-ai-model/src/main/java/org/springframework/ai/tool/annotation/Tool.java"
  - "raw/code/spring-ai/spring-ai-client-chat/src/main/java/org/springframework/ai/chat/client/advisor/ToolCallAdvisor.java"
aliases:
  - Tool Calling
  - Function Calling
  - Spring AI 工具调用
  - @Tool
status: evolving
confidence: 0.95
updated: "2026-05-12 00:00:00"
---

# 概念_Spring_AI_ToolCalling工具调用

![[Attachments/spring-ai-tool-calling-loop.svg]]

## 定义

Tool Calling 是 Spring AI 对 LLM Function Calling 能力的封装，实现 **ReAct（Reasoning + Acting）Agent 模式**。通过 `@Tool` 注解声明 Java 方法为工具，框架自动生成 OpenAI/Anthropic 兼容的 Tool 定义 JSON，`ToolCallAdvisor` 在 Advisor 链中检测 LLM 的工具调用请求、反射执行工具方法、回填结果、再次调用 LLM，形成 "思考→行动→观察" 的 Agent 循环。

## 为什么重要

Tool Calling 是让 LLM **突破文本生成边界**的关键机制：
1. **实时数据**：LLM 训练数据截止后的事件需要工具获取（天气、股价、新闻）
2. **精确计算**：LLM 不擅长数学，交给 `@Tool` 方法精确计算
3. **外部系统操作**：发邮件、创建工单、查询数据库等需要工具对接
4. **Agent 基础**：多步推理 + 工具调用 = 真正的 AI Agent，而非单次问答

Spring AI 的设计让工具注册像 `@Service` 一样自然，无需手写 JSON Schema 或 HTTP 回调。

## 使用场景

- **天气查询**：`@Tool(description = "获取指定城市的天气")` → LLM 决定何时调用
- **数学计算**：`@Tool double calculate(String expression)` → 精确计算
- **数据库查询**：`@Tool List<User> searchUsers(String keyword)` → 对接业务数据
- **API 调用**：`@Tool String createTicket(String title, String desc)` → 外部系统操作
- **多步 Agent**：先查天气 → 根据天气推荐衣物 → 给出最终建议（3 次 Tool Calling 循环）

## 核心机制

### 1. @Tool 注解

```java
// spring-ai-model: org/springframework/ai/tool/annotation/Tool.java
@Target({ ElementType.METHOD, ElementType.ANNOTATION_TYPE })
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface Tool {

    String name() default "";        // 工具名称，默认用方法名
    String description() default ""; // 工具描述，默认用方法名（被 LLM 读取以决定何时调用）

    boolean returnDirect() default false;  // true = 结果直接返回用户，不再回传 LLM

    Class<? extends ToolCallResultConverter> resultConverter()
        default DefaultToolCallResultConverter.class; // 结果转 String 的转换器
}
```

### 2. 工具声明（自动发现）

```java
// 方式一：ChatClient.Builder 自动扫描
@Service
public class WeatherService {
    @Tool(description = "获取指定城市的当前天气")
    public String getWeather(
        @ToolParam(description = "城市名称，如北京") String city) {
        return city + "：晴，22°C，湿度 45%";
    }

    @Tool(description = "计算数学表达式")
    public double calculator(
        @ToolParam(description = "数学表达式") String expression) {
        return new ScriptEngineManager().getEngineByName("js").eval(expression);
    }
}

ChatClient client = ChatClient.builder(chatModel)
    .defaultTools(new WeatherService())  // 自动扫描 @Tool 注解，生成 ToolCallback
    .build();
```

框架的行为：
1. `defaultTools(weatherService)` → 反射扫描所有 `@Tool` 方法
2. 为每个 `@Tool` 方法创建 `MethodToolCallback`（含 JSON Schema 参数定义）
3. 将所有 ToolCallback 注入 `ToolCallingChatOptions`
4. LLM 收到请求时，框架自动附加 tools 定义到 API 调用

### 3. ToolCallAdvisor：ReAct Agent 循环

ToolCallAdvisor 是整个 Tool Calling 的编排引擎，实现了 **Recursive Advisor** 模式：

```
prompt().user("北京今天天气如何？").call()
    │
    ▼
ToolCallAdvisor.adviseCall()
    │
    ▼
┌─ ToolCallAdvisor 循环 ──────────────────────────────┐
│                                                       │
│  ① 禁用框架内部 Tool Execution                          │
│     options.internalToolExecutionEnabled(false)       │
│                                                       │
│  ② do {                                               │
│       before() → chain.nextCall(request)             │
│           │                                           │
│           ▼                                           │
│       ChatModel.call(prompt + tools)                  │
│           │                                           │
│           ▼                                           │
│       ChatResponse (含 ToolCalls?)                    │
│           │                                           │
│       after()                                         │
│           │                                           │
│       ▼                                               │
│     isToolCall = response.hasToolCalls()             │
│                                                       │
│     if (isToolCall) {                                 │
│       toolExecutionResult = toolCallingManager        │
│           .executeToolCalls(prompt, response)        │
│           │                                           │
│       if (returnDirect) {                             │
│         break;  ← 结果直接返回用户                      │
│       }                                               │
│                                                       │
│       instructions = toolExecutionResult              │
│           .conversationHistory()  ← 更新对话历史        │
│     }                                                 │
│   } while (isToolCall);  ← 循环直到无工具调用           │
│                                                       │
│  ③ doFinalizeLoop() → 返回最终 ChatClientResponse      │
│                                                       │
└───────────────────────────────────────────────────────┘
```

### 4. 流式调用中的 ToolCallAdvisor

流式模式更复杂，因为 LLM 的 tool_calls 信号分散在多个 chunk 中：

```
stream().content()
    │
    ▼
ToolCallAdvisor.adviseStream()
    │
    ▼
responseFlux.publish(shared → {
    // Branch 1: 实时推送所有 chunk（用户体验优先）
    streamingBranch = aggregator.aggregate(shared, callback)

    // Branch 2: 聚合后判断是否需要工具调用
    recursionBranch = handleToolCallRecursion(aggregatedResponse)
        if (hasToolCalls)
            → executeToolCalls()
            → internalStream()  // 递归调用
        else
            → Flux.empty()
})
.filter(ccr -> streamToolCallResponses || !hasToolCalls)  // 可选过滤中间工具调用
```

关键点：
- 使用 `Flux.publish()` 实现热流多播，让 streaming 和 aggregation 并行
- `ChatClientMessageAggregator` 聚合所有 chunk 后才判断 finishReason
- 工具执行在 `Schedulers.boundedElastic()` 上（阻塞 I/O 安全）
- `streamToolCallResponses=true` 时推送中间工具调用响应，false 时仅推送最终答案

### 5. ToolCallingManager 工具编排

```
ToolCallingManager
├── 批量执行多个 ToolCall（支持并行 parallelToolCalls）
├── 通过 MethodToolCallback 反射调用 @Tool 方法
├── 管理工具执行上下文（ToolContext）
└── 返回 ToolExecutionResult{ conversationHistory, returnDirect }
```

### 6. 对话历史管理

每次工具调用后，框架自动管理历史消息：

```
[UserMessage: "北京天气？"]
[AssistantMessage: { toolCalls: [getWeather("北京")] }]
[ToolResponseMessage: "北京：晴，22°C"]      ← 自动插入
[AssistantMessage: "北京今天晴天，22°C..."]    ← LLM 基于工具结果生成
```

`conversationHistoryEnabled=true`（默认）：完整历史传给下次调用
`conversationHistoryEnabled=false`：只传 SystemMessage + 最后一次工具结果

## 例子

### 基础工具调用

```java
@Service
class WeatherTools {
    @Tool(description = "获取天气")
    String weather(@ToolParam(description = "城市") String city) {
        return city + "：晴天 22°C";
    }
}

ChatClient client = ChatClient.builder(chatModel)
    .defaultTools(new WeatherTools())
    .build();

String reply = client.prompt()
    .user("北京今天天气怎么样？")
    .call().content();
// LLM → 决定调用 weather("北京")
// ToolCallAdvisor → 执行方法 → 回填 "北京：晴天 22°C"
// LLM → "北京今天天气晴朗，气温 22°C，适合出行。"
```

### 并行工具调用

```java
@Tool(description = "查航班") Flight searchFlights(String from, String to);
@Tool(description = "查酒店") Hotel searchHotels(String city);

// LLM 同时决定调用两个工具 → parallelToolCalls 并行执行
String reply = client.prompt()
    .user("帮我查北京到上海的航班，还有上海的酒店")
    .options(ToolCallingChatOptions.builder()
        .internalToolExecutionEnabled(false)  // 让 ToolCallAdvisor 处理
        .build())
    .call().content();
```

### returnDirect 模式

```java
@Tool(description = "获取隐私数据", returnDirect = true)
String getSensitiveData(@ToolParam(description = "用户ID") String userId) {
    return "敏感数据内容...";
}
// returnDirect=true → 工具执行结果直接返回给用户，不再回传 LLM 总结
```

## 常见误区

1. **"ToolCallAdvisor 在 Advisor 链中自动注册"**：不完全对。框架使用 `ToolCallingManager` 机制，默认 behavior 是框架内部处理工具调用。当 Advisor 链中存在 ToolCallAdvisor 时，需显式设置 `internalToolExecutionEnabled(false)` 将控制权交给 ToolCallAdvisor。
2. **"@Tool 方法的返回值必须是 String"**：错。可以是任意类型，`ToolCallResultConverter` 负责转换为 String。默认使用 `DefaultToolCallResultConverter`（Jackson 序列化）。
3. **"流式模式不支持 ToolCalling"**：支持但更复杂。ToolCallAdvisor 需要聚合流式 chunk 才能判断 finishReason，所以流式 + 工具调用会有额外的聚合开销。
4. **"returnDirect=true 时完全不调用 LLM"**：错。LLM 仍然会被调用一次以决定工具调用，只是工具执行结果直接返回，不再进行第二次 LLM 调用。

## 源码级执行链

Tool Calling 有两条可能路径：provider 内部执行，以及 `ToolCallAdvisor` 接管执行。理解源码时要把两者分清。

`ToolCallAdvisor` 接管路径：

1. `adviseCall()` 先校验 `Prompt.options instanceof ToolCallingChatOptions`。
2. 复制 options 并设置 `internalToolExecutionEnabled(false)`，避免模型 provider 自己执行工具。
3. 用当前 instructions 构造新的 `Prompt(instructions, optionsCopy)`。
4. 调 `callAdvisorChain.copy(this).nextCall(processedRequest)`，复制“当前 Advisor 之后”的链，避免递归时再次进入自己。
5. 检查 `ChatResponse.hasToolCalls()`。
6. 有工具调用时，`DefaultToolCallingManager.executeToolCalls(prompt, chatResponse)` 找到第一条带 tool calls 的 generation。
7. `executeToolCall()` 按 tool name 在 request tool callbacks 或 resolver 中查找 `ToolCallback`。
8. 每个工具调用都用 `ToolCallingObservationDocumentation.TOOL_CALL` 包裹。
9. 工具结果被封装成 `ToolResponseMessage`，再和原始 messages、`AssistantMessage(toolCalls)` 组成下一轮 conversation history。
10. 循环直到没有 tool call，或 `returnDirect=true` 直接返回工具结果。

## MethodToolCallback 反射细节

`@Tool` 方法最终会变成 `MethodToolCallback`。它的执行过程不是简单反射调用：

| 步骤 | 源码动作 | 风险点 |
|------|----------|--------|
| 参数解析 | `JsonParser.fromJson(toolInput, TypeReference<Map<String,Object>>)` | LLM 生成非法 JSON 会变成 `ToolExecutionException` |
| ToolContext 校验 | 如果方法参数需要 `ToolContext`，但上下文为空则报错 | 工具方法签名要和调用场景一致 |
| 类型转换 | `JsonParser.toTypedObject()` / `fromJson(json, type)` | 泛型参数也会二次 JSON 转换 |
| 反射调用 | 非 public 类或方法会 `setAccessible(true)` | Native image / AOT 场景需要 runtime hints |
| 结果转换 | `ToolCallResultConverter.convert(result, returnType)` | 默认结果会转成字符串给模型 |

这说明工具调用的稳定性主要取决于三件事：工具 schema 是否准确、LLM 参数 JSON 是否可解析、Java 方法签名是否能被类型转换正确填充。

## 和其它概念的关系

- [[concepts/java/spring-ai/概念_Spring_AI_Advisor拦截链]] — ToolCallAdvisor 是 Advisor 链中最复杂的拦截器，实现递归 Agent 循环
- [[concepts/java/spring-ai/概念_Spring_AI_ChatClient_API]] — `defaultTools()` 和 `tools()` 注册 @Tool 方法到 ChatClient
- [[concepts/java/spring-ai/概念_Spring_AI_架构设计]] — Tool Calling 通过 ToolCallAdvisor 在架构的 Client-Chat 层实现
- [[concepts/java/spring-ai/概念_Spring_AI_ChatMemory对话记忆]] — ToolCallAdvisor 管理对话历史，ChatMemory 持久化历史
- [[concepts/java/spring-ai/概念_Spring_AI_MCP集成]] — MCP 远程工具最终也会桥接为 Spring AI ToolCallback
- [[overview/java/spring-ai/主题_Spring_AI源码架构_综述]] — Tool Calling 在完整源码主线中的位置

## 来源

- `raw/code/spring-ai/spring-ai-model/src/main/java/org/springframework/ai/tool/annotation/Tool.java`
- `raw/code/spring-ai/spring-ai-client-chat/src/main/java/org/springframework/ai/chat/client/advisor/ToolCallAdvisor.java`
- `raw/code/spring-ai/spring-ai-model/src/main/java/org/springframework/ai/model/tool/DefaultToolCallingManager.java`
- `raw/code/spring-ai/spring-ai-model/src/main/java/org/springframework/ai/tool/method/MethodToolCallback.java`
