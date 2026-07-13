---
type: "concept"
tags:
  - java
  - spring-ai
  - mcp
  - tool-calling
  - agent
summary: "Spring AI MCP 集成：通过 MCP Client/Server 自动配置、ToolCallbackProvider、McpToolUtils 和注解扫描，把远程 MCP 工具转换为 Spring AI ToolCallback，也能把本地 Spring AI 工具暴露为 MCP Tool。"
sources:
  - "raw/code/spring-ai/mcp/common/src/main/java/org/springframework/ai/mcp/McpToolUtils.java"
  - "raw/code/spring-ai/auto-configurations/mcp/spring-ai-autoconfigure-mcp-client-common/src/main/java/org/springframework/ai/mcp/client/common/autoconfigure/McpClientAutoConfiguration.java"
  - "raw/code/spring-ai/auto-configurations/mcp/spring-ai-autoconfigure-mcp-client-common/src/main/java/org/springframework/ai/mcp/client/common/autoconfigure/McpToolCallbackAutoConfiguration.java"
  - "raw/code/spring-ai/mcp/mcp-annotations/src/main/java/org/springframework/ai/mcp/annotation/McpTool.java"
aliases:
  - Spring AI MCP
  - Model Context Protocol
  - MCP 集成
status: "evolving"
confidence: 0.88
created: "2026-05-12 00:00:00"
updated: "2026-05-12 00:00:00"
---

# 概念：Spring AI MCP 集成

## 定义

MCP（Model Context Protocol）是让模型应用发现和调用外部上下文能力的协议。Spring AI 的 MCP 集成主要做两件事：

1. 作为 **MCP Client**：连接外部 MCP Server，把远程工具转换成 Spring AI 的 `ToolCallback`，供 ChatClient / Tool Calling 使用。
2. 作为 **MCP Server**：把本地 Spring AI `ToolCallback` 或 MCP 注解方法暴露成 MCP Tool，供其他 MCP Client 调用。

因此 MCP 在 Spring AI 中不是另一套孤立工具系统，而是和已有 Tool Calling 体系互相转换。

## 为什么重要

传统 `@Tool` 更适合应用内工具。MCP 解决的是跨进程、跨语言、跨应用的工具发现与调用：文件系统、数据库、浏览器、企业系统、知识库等都可以用 MCP Server 暴露出来。

Spring AI 的价值在于把 MCP 工具接进原有 `ToolCallback` 抽象。这样 ChatClient 侧仍然通过工具调用机制工作，而不必关心工具来自本地 Bean 还是远程 MCP Server。

## 核心机制

### 1. MCP Client 自动配置

`McpClientAutoConfiguration` 根据配置创建同步或异步 MCP Client：

- `spring.ai.mcp.client.enabled`：是否启用，默认启用。
- `spring.ai.mcp.client.type`：`SYNC` 或 `ASYNC`，默认 `SYNC`。
- `spring.ai.mcp.client.name/version`：客户端身份信息。
- `spring.ai.mcp.client.request-timeout`：请求超时。
- `spring.ai.mcp.client.initialized`：创建后是否立即初始化。

它会收集 `NamedClientMcpTransport`，为每个 transport 构造 `McpSyncClient` 或 `McpAsyncClient`。transport 可以来自 stdio、SSE、Streamable HTTP 等不同自动配置模块。

### 2. MCP ToolCallback 自动转换

`McpToolCallbackAutoConfiguration` 把 MCP Client 转成 Spring AI 工具提供者：

```java
SyncMcpToolCallbackProvider.builder()
    .mcpClients(mcpClients)
    .toolFilter(...)
    .toolNamePrefixGenerator(...)
    .toolContextToMcpMetaConverter(...)
    .build();
```

这一步完成后，远程 MCP Server 暴露的工具就能作为 Spring AI `ToolCallback` 进入 ChatClient 工具调用链。

它还提供 `McpToolNamePrefixGenerator`，用于处理多个 MCP Server 工具重名的问题。默认实现会根据连接信息生成前缀，并把工具名规范化，避免 LLM 工具列表中出现冲突。

### 3. McpToolUtils 双向桥接

`McpToolUtils` 是 MCP 与 Spring AI Tool Calling 之间的转换工具：

| 方向 | 方法 | 作用 |
|------|------|------|
| Spring AI → MCP Server | `toSyncToolSpecification(ToolCallback)` | 把本地 ToolCallback 暴露成 MCP Tool |
| MCP Client → Spring AI | `createToolDefinition(prefixedToolName, McpSchema.Tool)` | 把 MCP Tool 变成 Spring AI ToolDefinition |
| 调用上下文 | `TOOL_CONTEXT_MCP_EXCHANGE_KEY` | 把 MCP exchange 放入 `ToolContext` |
| 命名 | `prefixedToolName(...)` | 为远程工具生成不冲突名称 |

这说明 MCP 与 `@Tool` 并不是竞争关系，而是两种工具来源：本地 Java 方法适合低延迟内部能力，MCP 适合远程上下文和跨应用扩展。

### 4. MCP 注解模型

`mcp-annotations` 提供了一组注解，把 Java 方法声明成 MCP 能力：

| 注解 | 能力 |
|------|------|
| `@McpTool` | 暴露可执行工具 |
| `@McpResource` | 暴露资源 |
| `@McpPrompt` | 暴露 prompt 模板 |
| `@McpComplete` | 暴露补全能力 |
| `@McpSampling` | 处理采样请求 |
| `@McpProgress` / `@McpLogging` | 处理进度与日志 |

这些注解背后会被扫描成 sync/async、stateful/stateless 的 provider 或 callback。它让 Spring 应用可以像声明 `@Controller` 或 `@Tool` 一样声明 MCP 能力。

## 使用场景

- 把企业内部工具平台作为 MCP Server 接入 Spring AI Agent。
- 把 Spring Boot 应用里的业务方法暴露成 MCP Tool，供 IDE、桌面助手或其他 Agent 使用。
- 多 MCP Server 聚合：文件、数据库、浏览器、知识库工具统一进入 ChatClient 工具列表。
- 对工具做统一过滤：用 `McpToolFilter` 控制哪些远程工具允许进入模型上下文。

## 常见误区

1. **“MCP 替代 @Tool”**：不准确。`@Tool` 是本地工具声明，MCP 是协议化远程工具生态；Spring AI 把二者接到同一 ToolCallback 体系。
2. **“MCP 工具名天然唯一”**：多个 server 很容易暴露同名工具，必须关注 prefix 生成和工具名长度限制。
3. **“远程工具都应该暴露给模型”**：工具越多，选择难度、prompt token 和安全风险越高，应使用 `McpToolFilter` 做白名单或上下文过滤。
4. **“MCP 调用没有上下文”**：`ToolContextToMcpMetaConverter` 和 MCP exchange 能传递元信息，适合做租户、权限、请求追踪等上下文注入。

## 和其它概念的关系

- [[concepts/java/spring-ai/概念_Spring_AI_ToolCalling工具调用]] — MCP 工具最终会进入 Spring AI ToolCallback / ToolCallingManager 体系。
- [[concepts/java/spring-ai/概念_Spring_AI_AutoConfiguration自动配置]] — MCP Client、ToolCallbackProvider、transport 都依赖 Boot 自动配置装配。
- [[concepts/java/spring-ai/概念_Spring_AI_Advisor拦截链]] — ChatClient 使用 MCP 工具时，仍可经由 ToolCallAdvisor 形成 Agent 循环。

## 我的理解

MCP 是 Spring AI 从“应用内 AI 框架”走向“工具生态连接层”的关键。它把工具从 Java Bean 扩展到进程外协议资源，同时又不破坏 Spring AI 原有的 Tool Calling 抽象。真正落地时，最重要的不是“能连多少 MCP Server”，而是工具筛选、命名、权限和观测，否则模型上下文会迅速变成不可控的工具清单。

## 来源

- `raw/code/spring-ai/mcp/common/src/main/java/org/springframework/ai/mcp/McpToolUtils.java`
- `raw/code/spring-ai/auto-configurations/mcp/spring-ai-autoconfigure-mcp-client-common/src/main/java/org/springframework/ai/mcp/client/common/autoconfigure/McpClientAutoConfiguration.java`
- `raw/code/spring-ai/auto-configurations/mcp/spring-ai-autoconfigure-mcp-client-common/src/main/java/org/springframework/ai/mcp/client/common/autoconfigure/McpToolCallbackAutoConfiguration.java`
- `raw/code/spring-ai/mcp/mcp-annotations/src/main/java/org/springframework/ai/mcp/annotation/McpTool.java`
