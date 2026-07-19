---
type: map
area: ai/llm-applications
scope: map
project: []
summary: "面向具有五年 Java 经验的开发者，将课程主题收敛为生产级大模型应用开发能力、学习顺序与三阶段作品集。"
sources:
  - "[[30-sources/courses/来源_Java大模型应用开发课程大纲]]"
  - "[[30-sources/repositories/Spring_AI_源码]]"
aliases:
  - Java 转型 AI 应用开发路线
  - Java LLM 应用开发学习地图
tags:
  - java
  - llm
  - rag
  - agent
  - mcp
  - learning-path
status: evolving
confidence: 0.85
created: "2026-07-19 00:00:00"
updated: "2026-07-19 00:00:00"
---

# Java 开发者转型大模型应用开发学习地图

## 定位

目标不是成为模型训练研究员，而是成为能把大模型做成**可靠、可测、可观测、可控的业务系统**的应用工程师。五年 Java 经验应转化为优势：服务化、数据边界、鉴权、事务、缓存、异步、测试、发布和运维，而不是从零重复学习 Web 开发。

课程目录的详情见 [[30-sources/courses/来源_Java大模型应用开发课程大纲]]。其中多种 Agent 框架和“手写某产品”内容属于实现样本；它们不应取代 RAG、工具安全和评测这些通用能力。

## 能力优先级

| 优先级 | 先获得的能力 | 对应课程主题 | 达标证据 |
|---|---|---|---|
| P0 | 模型调用与结构化输出：提示、流式响应、JSON schema、重试、限流、成本记录 | Spring AI 基础、DeepSeek | 一个 Spring Boot API 能稳定流式返回结构化结果，并处理超时、限流和供应商错误 |
| P0 | 生产 RAG：文档解析、切分、Embedding、检索、元数据过滤、引用、评测 | Spring AI RAG、AI Data | 能用一组标注问题证明检索/回答改进，而非仅展示“能问答” |
| P0 | 受控 Tool Calling 与 Agent：工具 schema、权限、幂等、确认、审计、失败恢复 | React Agent、Tool Calling、Agent Framework | 能让 Agent 查数据和提交“待确认”操作，但不能越权或重复执行 |
| P0 | MCP：既会接入外部 MCP，也会将已有 Java 领域服务包装为 MCP Server | MCP Client/Server、MCP Streamable HTTP | 一个带鉴权、输入校验和审计的订单/库存查询 MCP 服务 |
| P1 | 可观测与评测：Trace、token/延迟/成本、检索命中、工具成功率、离线回归集 | AI Agent 监控、日志监控 | 仪表板+告警阈值+可重复的评测报告 |
| P1 | 上下文工程与多步骤编排：记忆、压缩、工作流状态、人工审批 | Chat Memory、Advisor、Graph、Plan | 复杂任务可恢复、可中断、可回放；知道何时用固定工作流而非自主 Agent |
| P2 | 多 Agent、Manus/DeepResearch 复刻、框架源码、OpenClaw 手写 | Manus、Graph、AgentScope、手写 OpenClaw | 能解释适用边界，并将其用于明确业务问题，而非为“多 Agent”而多 Agent |
| P2 | AI 前端生成、网页抓取等 | 前端、爬虫 | 仅作为作品完整度补充；不占据主学习时间 |

## 课程取舍

### 必学且要深入

1. **Spring AI 1 条主线**：`ChatClient → Advisor → Structured Output → Tool Calling → RAG → MCP`。先理解调用路径，再使用 starter。见 [[10-domains/java/spring-ai/主题_Spring_AI源码架构_综述]]。
2. **RAG 的数据与评测闭环**：解析质量、chunk 策略、embedding、混合/过滤检索、重排、引用与失败案例。向量库只是组件，不能把“用了 pgvector/ES”当成 RAG 成功。
3. **工具与 Agent 工程化**：工具应小而确定；写操作必须鉴权、参数校验、幂等键、人工确认和审计。ReAct 是基础范式，见 [[10-domains/ai/agent/概念_ReAct范式]]；上下文质量决定上限，见 [[10-domains/ai/agent/概念_上下文工程]]。
4. **MCP 与已有 Java 服务集成**：它是工具生态的接口层，不是 Agent 的替代品。学习 client、server、HTTP/stdio transport、OAuth/服务凭证、资源范围与审计。
5. **评测、观测与安全**：至少记录请求、版本、上下文、检索片段、工具调用、token、时延、成本和结果；敏感提示词/工具参数默认不外发。把 prompt injection、数据越权、幻觉、重复写入作为测试对象。

### 学到“能选型和实现”的程度即可

- Spring AI Alibaba、Graph、Agent Framework：选一套作为工作流/多 Agent 实现工具；不要同时深挖两套同类 API。
- React/Manus/DeepResearch：理解 ReAct、规划、反思、浏览/检索的组成和失败模式即可；先完成可控单 Agent。
- DeepSeek/其他模型厂商：做统一模型接口与能力/成本基准，避免业务代码绑定一个 SDK。
- AgentScope Java：当目标团队明确采用或你需要其分布式 Agent 能力时再投入。

### 可以后置或跳过

- “从零手写 OpenClaw/Manus”中的 UI 和产品复刻细节；保留其中的循环、记忆、工具注册和沙箱思想。
- 大量前端页面生成、网页爬虫、Chart/Graph 展示；用现有前端能力完成演示即可。
- 过早研究框架源码。先交付完整应用，再沿 `ChatClient`、Advisor、Tool Calling 三处读源码定位问题。

## 12 周实施节奏（每周约 8–12 小时）

| 周期 | 学习与交付 | 退出标准 |
|---|---|---|
| 第 1–2 周 | Spring AI 接模型；SSE 流式对话；Prompt 模板、结构化输出、会话隔离、异常/重试/限流 | API 有测试，模型切换不改业务层，响应含 requestId 与 token/成本记录 |
| 第 3–5 周 | 企业文档 RAG：导入、异步索引、权限元数据、检索、引用、反馈；建立 30–50 个问题评测集 | 有 baseline 与改进数据；错误回答能定位为解析、检索、上下文或模型问题 |
| 第 6–7 周 | Tool Calling：只读工具→受控写工具；审批、幂等、审计、超时和补偿 | 注入提示不能越权调用；写操作需模拟用户确认 |
| 第 8 周 | MCP client/server：包装一个现有 Java 领域服务，接入一个受信 MCP | 服务有身份、scope、输入 schema、审计；不会把数据库或内部管理 API 直接暴露给模型 |
| 第 9–10 周 | 工作流和 Agent：把固定步骤做成可恢复工作流，仅在工具选择不确定处引入 Agent | 每步状态可追踪；失败可重放；有最大步数、预算与超时 |
| 第 11–12 周 | 观测、评测、压测、部署、README 与录屏；复盘失败案例 | 作品可以由陌生人本地运行、理解架构并验证质量/安全声明 |

## 三个递进作品集项目

### 1. 团队知识库助手（第 1–5 周）

以制度、接口文档、故障手册为语料，提供带来源引用的问答。

- 技术：Spring Boot + Spring AI + PostgreSQL/pgvector（或团队已有检索设施）+ 对象存储；前端可简单。
- 关键难题：文档增量同步、chunk/metadata 设计、租户或部门权限过滤、无法回答时拒答、评测集和结果回归。
- 面试价值：展示的不是 Demo，而是 RAG 为什么可靠及如何衡量。

### 2. 电商/业务运营 Copilot（第 6–10 周，推荐主作品）

基于一个熟悉的订单、库存、售后或工单领域，做“查数据 + 生成建议 + 经确认执行”的助手。若复用现有商城项目，应先从只读查询开始。

- 只读工具：订单状态、库存、商品规则、工单检索；写工具：创建草稿、发起审批、生成待执行指令，绝不让模型直接提交高风险动作。
- 工程点：RBAC/scope、幂等、审批状态机、审计事件、工具超时/重试、会话与用户隔离。
- 升级：将 2–3 个稳定领域能力发布成 MCP Server，再让 Copilot 作为 MCP Client 调用。

### 3. 研发交付助手（第 9–12 周）

将需求拆解、代码库检索、测试建议、变更摘要组织成工作流；“改代码、发消息、创建工单”只生成草稿或进入审批。

- 不要追求完整 IDE Agent；重点是任务状态、上下文选择、代码库 RAG、工具审批、评测和可观测。
- 若需要多 Agent：先让一个 planner 产生结构化计划，再由确定性 workflow 分派；不要让多个自由 Agent 无界对话。

## 作品必须展示的工程证据

- `README`：问题、边界、架构图、运行方式、数据脱敏说明与成本估算。
- `eval/`：问题集、期望证据或判定规则、指标、每次改动前后的对比。
- `security/`：提示注入、越权查询、危险工具调用、重复提交和敏感信息泄露的测试用例。
- `observability/`：一次请求从用户输入、检索、模型、工具到结果的 trace；以及延迟、成功率、token/成本指标。
- `adr/`：说明为何选择 RAG/工具/工作流/Agent，及未选择其他方案的原因。

## 技术判断准则

1. 需要确定步骤、事务和审批时，用**普通 Java 工作流**；模型只负责抽取、分类、生成或在有限工具中选择。
2. 需要从有限工具中决定下一步，且允许失败重试时，用**单 Agent**。
3. 只有角色、上下文和目标真正可分解，并且有评测证明优于单 Agent 时，再用**多 Agent**。
4. 一切外部写操作均在模型之外建立授权与业务校验；模型输出不是可信指令。

## 当前可用资料的连接

- [[10-domains/java/spring-ai/概念_Spring_AI_RAG检索增强生成]] — Spring AI Modular RAG 流水线。
- [[10-domains/java/spring-ai/概念_Spring_AI_ToolCalling工具调用]] — 工具调用和循环实现。
- [[10-domains/java/spring-ai/概念_Spring_AI_MCP集成]] — MCP 与 Spring AI 的桥接。
- [[10-domains/ai/agent/概念_记忆系统]] — 记忆与检索边界。
- [[40-maps/比较/java/ai-frameworks/Spring_AI_vs_LangChain4j]] — Java 框架的选择比较。

