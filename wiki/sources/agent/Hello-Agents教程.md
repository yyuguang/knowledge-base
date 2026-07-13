---
type: source
tags:
  - ai
  - agent
  - book
  - tutorial
summary: "Datawhale 开源智能体教程（4W+ Stars），16 章从零构建 AI Native Agent，含完整 Python 代码"
sources:
  - "https://github.com/datawhalechina/hello-agents"
aliases:
  - hello-agents
  - 从零开始构建智能体
status: stable
confidence: 0.95
updated: "2026-04-28 22:30:00"
---

# Hello-Agents

- **标题**：《从零开始构建智能体》（Hello-Agents）
- **作者**：陈思州（项目负责人）、孙韬、姜舒凡 等 Datawhale 社区
- **时间**：2025
- **链接**：https://github.com/datawhalechina/hello-agents
- **章节**：16 章 + 附加章节

## 核心要点

- **Agent Loop 完整实现**：Chapter 1 以旅行助手为例，演示 `Thought → Action → Observation` 循环的完整 Python 实现（~70 行）
- **LLM 基础**：Chapter 3 用 PyTorch 复现 Transformer（MultiHeadAttention / PositionalEncoding / Encoder / Decoder）、BPE 分词算法
- **三大经典范式**：Chapter 4 完整实现 ReAct（ReActAgent + ToolExecutor）、Plan-and-Solve（Planner + Executor）、Reflection（Memory + ReflectionAgent）
- **框架开发**：Chapter 6-7 讲解 Agent 框架的完整架构设计（SimpleAgent / HelloAgentsLLM / ToolRegistry / MemoryManager）
- **记忆与 RAG**：Chapter 8 实现四层记忆系统 + RAGTool（MQE + HyDE 增强检索）
- **多智能体实践**：Chapter 15 赛博小镇——独立 Agent per NPC + 批量生成 + 好感度系统
- **工程深度**：Chapter 10 智能体通信协议（MCP/A2A/ANP），Chapter 11 Agentic RL（GRPO 训练），Chapter 12 性能评估

## 提取的概念

- [[concepts/agent/概念_Agent_Loop]]
- [[concepts/agent/概念_ReAct范式]]
- [[concepts/agent/概念_Plan-and-Solve范式]]
- [[concepts/agent/概念_Reflection范式]]
- [[concepts/agent/概念_PEAS模型]]
- [[concepts/agent/概念_多智能体协作]]
- [[concepts/agent/概念_ELIZA]]
- [[concepts/agent/概念_Transformer架构]]
- [[concepts/agent/概念_BPE分词算法]]
- [[concepts/agent/概念_提示工程]]
- [[concepts/agent/概念_记忆系统]]
- [[concepts/agent/概念_上下文工程]]

## 关联

- [[entities/agent/项目_HelloAgents]] — 本教程作为开源项目的实体信息
- [[entities/agent/人物_陈思州]] — 项目负责人与作者
