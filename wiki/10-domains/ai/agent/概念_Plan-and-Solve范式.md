---
type: concept
tags:
  - ai
  - agent
  - plan-and-solve
  - llm-pattern
summary: "先规划后执行：Planner 用 LLM 将任务分解为 Python 列表步骤，Executor 逐步执行并传递历史状态"
sources:
  - "[[30-sources/courses/Hello-Agents教程]]"
  - "https://github.com/datawhalechina/hello-agents"
aliases:
  - Plan-and-Solve
  - 先规划后执行
status: stable
confidence: 0.90
updated: "2026-04-28 22:30:00"
---

# 概念_Plan-and-Solve范式

## 定义

Plan-and-Solve 是一种"三思而后行"的智能体范式：LLM 先对整个任务生成一份完整的、分步骤的计划(Plan)，随后按计划顺序执行每一步(Solve)，一步执行完毕再进入下一步。与 ReAct 的"边想边做"形成鲜明对比。

## 两阶段架构

| 阶段 | 组件 | 职责 |
|------|------|------|
| **Plan** | `Planner` | 调用 LLM 将问题分解为结构化步骤列表 |
| **Solve** | `Executor` | 循环遍历计划，每步调用 LLM 执行，历史状态链式传递 |
| **Orchestrate** | `PlanAndSolveAgent` | 整合 Planner + Executor，提供统一入口 |

## 完整实现

### 1. 规划器提示词

```python
PLANNER_PROMPT_TEMPLATE = """
你是一个顶级的AI规划专家。你的任务是将用户提出的复杂问题分解成一个由多个简单步骤组成的行动计划。
请确保计划中的每个步骤都是一个独立的、可执行的子任务，并且严格按照逻辑顺序排列。
你的输出必须是一个Python列表，其中每个元素都是一个描述子任务的字符串。

问题: {question}

请严格按照以下格式输出你的计划, ```python与```作为前后缀是必要的:
```python
["步骤1", "步骤2", "步骤3", ...]
```
"""
```

### 2. Planner 类

```python
import ast

class Planner:
    def __init__(self, llm_client):
        self.llm_client = llm_client

    def plan(self, question: str) -> list[str]:
        """根据用户问题生成行动计划"""
        prompt = PLANNER_PROMPT_TEMPLATE.format(question=question)
        messages = [{"role": "user", "content": prompt}]

        print("--- 正在生成计划 ---")
        response_text = self.llm_client.think(messages=messages) or ""

        try:
            # 从 LLM 输出中提取 Python 列表
            plan_str = response_text.split("```python")[1].split("```")[0].strip()
            plan = ast.literal_eval(plan_str)  # 安全解析
            return plan if isinstance(plan, list) else []
        except (ValueError, SyntaxError, IndexError) as e:
            print(f"❌ 解析计划时出错: {e}")
            return []
```

### 3. 执行器提示词

```python
EXECUTOR_PROMPT_TEMPLATE = """
你是一位顶级的AI执行专家。你的任务是严格按照给定的计划，一步步地解决问题。
你将收到原始问题、完整的计划、以及到目前为止已经完成的步骤和结果。
请你专注于解决"当前步骤"，并仅输出该步骤的最终答案，不要输出任何额外的解释或对话。

# 原始问题:
{question}

# 完整计划:
{plan}

# 历史步骤与结果:
{history}

# 当前步骤:
{current_step}

请仅输出针对"当前步骤"的回答:
"""
```

### 4. Executor 类

```python
class Executor:
    def __init__(self, llm_client):
        self.llm_client = llm_client

    def execute(self, question: str, plan: list[str]) -> str:
        """根据计划逐步执行并解决问题"""
        history = ""  # 历史步骤与结果的累积字符串

        print("\n--- 正在执行计划 ---")
        for i, step in enumerate(plan):
            print(f"\n-> 正在执行步骤 {i+1}/{len(plan)}: {step}")

            prompt = EXECUTOR_PROMPT_TEMPLATE.format(
                question=question, plan=plan,
                history=history if history else "无",
                current_step=step
            )
            messages = [{"role": "user", "content": prompt}]
            response_text = self.llm_client.think(messages=messages) or ""

            # 更新历史：累积已完成步骤及结果
            history += f"步骤 {i+1}: {step}\n结果: {response_text}\n\n"
            print(f"✅ 步骤 {i+1} 已完成，结果: {response_text}")

        return response_text  # 最后一步的结果即为最终答案
```

### 5. PlanAndSolveAgent 整合

```python
class PlanAndSolveAgent:
    def __init__(self, llm_client):
        self.llm_client = llm_client
        self.planner = Planner(self.llm_client)
        self.executor = Executor(self.llm_client)

    def run(self, question: str):
        print(f"\n--- 开始处理问题 ---\n问题: {question}")

        # 阶段一：规划
        plan = self.planner.plan(question)
        if not plan:
            print("\n--- 任务终止 ---\n无法生成有效的行动计划。")
            return

        # 阶段二：执行
        final_answer = self.executor.execute(question, plan)
        print(f"\n--- 任务完成 ---\n最终答案: {final_answer}")
```

## 运行轨迹示例

```
--- 开始处理问题 ---
问题: 一个水果店周一卖出了15个苹果。周二卖出的苹果数量是周一的两倍。
      周三卖出的数量比周二少了5个。请问这三天总共卖出了多少个苹果？

--- 正在生成计划 ---
["计算周一卖出的苹果数量：15个",
 "计算周二卖出的苹果数量：周一数量 × 2 = 15 × 2 = 30个",
 "计算周三卖出的苹果数量：周二数量 - 5 = 30 - 5 = 25个",
 "计算三天总销量：周一 + 周二 + 周三 = 15 + 30 + 25 = 70个"]

--- 正在执行计划 ---
-> 步骤 1/4: 计算周一卖出的苹果数量: 15个 → 结果: 15
-> 步骤 2/4: 计算周二卖出的苹果数量 → 结果: 30
-> 步骤 3/4: 计算周三卖出的苹果数量 → 结果: 25
-> 步骤 4/4: 计算三天总销量 → 结果: 70

--- 任务完成 ---
最终答案: 70
```

## 与 ReAct 对比

| 维度 | Plan-and-Solve | ReAct |
|------|---------------|-------|
| 规划时机 | 执行前一次性完成 | 每步动态调整 |
| 适用场景 | 路径明确、可枚举步骤 | 信息不完全、需要探索 |
| 状态传递 | 纯文本历史累积 | Thought-Action-Observation 三元组 |
| 容错性 | 低（计划错误难修正） | 高（每步可根据观察调整） |
| Token 消耗 | 较低（无需每轮注入工具描述） | 较高（每轮都注入完整工具列表） |

## 关键设计

- **`ast.literal_eval` 安全解析**：避免 `eval()` 的安全风险
- **格式约束 ` ```python ... ``` `**：强制 LLM 输出可解析的 Python 列表
- **历史链式累积**：Executor 用纯文本累积已完成步骤，后续步骤自动获得上下文
- **无工具调用**：Plan-and-Solve 通常不涉及外部工具，适合纯推理任务

## 关联

- [[10-domains/ai/agent/概念_ReAct范式]] — 两种范式形成"边想边做 vs 先规划后执行"的策略对比
- [[10-domains/ai/agent/概念_Agent_Loop]] — Plan-and-Solve 将规划前置，是对标准 Agent Loop 的变体
