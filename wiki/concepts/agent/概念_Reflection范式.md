---
type: concept
tags:
  - ai
  - agent
  - reflection
  - self-correction
summary: "Execute → Reflect → Refine 三阶段迭代循环：Memory 记录全轨迹，ReflectionAgent 驱动自我批判和代码优化"
sources:
  - "https://github.com/datawhalechina/hello-agents"
aliases:
  - Reflection
  - 自我反思
  - 自修正范式
status: stable
confidence: 0.90
updated: "2026-04-28 22:30:00"
---

# 概念_Reflection范式

## 定义

Reflection（反思范式）在标准 Agent 执行后增加**自我批判**环节：智能体完成一轮或多轮行动后，对自己的输出进行审视，检查错误、遗漏或不合理之处，并据此修正后续行为。核心流程为 **Execute → Reflect → Refine** 三阶段迭代。

## 三组件架构

| 组件 | 职责 |
|------|------|
| **Memory** | 存储执行记录和反思记录，序列化为提示词 |
| **三个提示词模板** | 初始执行 / 反思审查 / 代码优化 |
| **ReflectionAgent** | 驱动迭代循环，直到反馈含"无需改进" |

## 完整实现

### 1. Memory 记忆模块

```python
from typing import List, Dict, Any, Optional

class Memory:
    """短期记忆模块，存储智能体的行动与反思轨迹"""

    def __init__(self):
        self.records: List[Dict[str, Any]] = []

    def add_record(self, record_type: str, content: str):
        """添加记录：record_type = 'execution' 或 'reflection'"""
        self.records.append({"type": record_type, "content": content})
        print(f"📝 记忆已更新，新增一条 '{record_type}' 记录。")

    def get_trajectory(self) -> str:
        """将所有记录格式化为连贯字符串，用于构建提示词"""
        parts = []
        for record in self.records:
            if record['type'] == 'execution':
                parts.append(f"--- 上一轮尝试 (代码) ---\n{record['content']}")
            elif record['type'] == 'reflection':
                parts.append(f"--- 评审员反馈 ---\n{record['content']}")
        return "\n\n".join(parts)

    def get_last_execution(self) -> Optional[str]:
        """获取最近一次的执行结果（最新的生成代码）"""
        for record in reversed(self.records):
            if record['type'] == 'execution':
                return record['content']
        return None
```

### 2. 三个关键提示词

```python
# --- 初始执行提示词 ---
INITIAL_PROMPT_TEMPLATE = """
你是一位资深的Python程序员。请根据以下要求，编写一个Python函数。
你的代码必须包含完整的函数签名、文档字符串，并遵循PEP 8编码规范。

要求: {task}

请直接输出代码，不要包含任何额外的解释。
"""

# --- 反思审查提示词 ---
REFLECT_PROMPT_TEMPLATE = """
你是一位极其严格的代码评审专家和资深算法工程师，对代码的性能有极致的要求。
你的任务是审查以下Python代码，并专注于找出其在**算法效率**上的主要瓶颈。

# 原始任务:
{task}

# 待审查的代码:
```python
{code}
```

请分析该代码的时间复杂度，并思考是否存在一种**算法上更优**的解决方案来显著提升性能。
如果存在，请清晰地指出当前算法的不足，并提出具体的、可行的改进算法建议。
如果代码在算法层面已经达到最优，才能回答"无需改进"。

请直接输出你的反馈，不要包含任何额外的解释。
"""

# --- 代码优化提示词 ---
REFINE_PROMPT_TEMPLATE = """
你是一位资深的Python程序员。你正在根据一位代码评审专家的反馈来优化你的代码。

# 原始任务:
{task}

# 你上一轮尝试的代码:
{last_code_attempt}

# 评审员的反馈：
{feedback}

请根据评审员的反馈，生成一个优化后的新版本代码。
你的代码必须包含完整的函数签名、文档字符串，并遵循PEP 8编码规范。
请直接输出优化后的代码，不要包含任何额外的解释。
"""
```

### 3. ReflectionAgent 主类

```python
class ReflectionAgent:
    def __init__(self, llm_client, max_iterations=3):
        self.llm_client = llm_client
        self.memory = Memory()
        self.max_iterations = max_iterations

    def run(self, task: str):
        print(f"\n--- 开始处理任务 ---\n任务: {task}")

        # Step 1: 初始执行
        print("\n--- 正在进行初始尝试 ---")
        initial_prompt = INITIAL_PROMPT_TEMPLATE.format(task=task)
        initial_code = self._get_llm_response(initial_prompt)
        self.memory.add_record("execution", initial_code)

        # Step 2: 迭代循环 — 反思 + 优化
        for i in range(self.max_iterations):
            print(f"\n--- 第 {i+1}/{self.max_iterations} 轮迭代 ---")

            # 2a. 反思
            print("\n-> 正在进行反思...")
            last_code = self.memory.get_last_execution()
            reflect_prompt = REFLECT_PROMPT_TEMPLATE.format(
                task=task, code=last_code
            )
            feedback = self._get_llm_response(reflect_prompt)
            self.memory.add_record("reflection", feedback)

            # 2b. 终止条件检查
            if "无需改进" in feedback:
                print("\n✅ 反思认为代码已无需改进，任务完成。")
                break

            # 2c. 优化
            print("\n-> 正在进行优化...")
            refine_prompt = REFINE_PROMPT_TEMPLATE.format(
                task=task, last_code_attempt=last_code, feedback=feedback
            )
            refined_code = self._get_llm_response(refine_prompt)
            self.memory.add_record("execution", refined_code)

        final_code = self.memory.get_last_execution()
        print(f"\n--- 任务完成 ---\n最终生成的代码:\n```python\n{final_code}\n```")
        return final_code

    def _get_llm_response(self, prompt: str) -> str:
        messages = [{"role": "user", "content": prompt}]
        return self.llm_client.think(messages=messages) or ""
```

## 运行轨迹示例

```
任务: 编写一个Python函数，找出1到n之间所有的素数。

--- 正在进行初始尝试 ---
📝 记忆已更新: 'execution'
# 初始输出：试除法 (O(n*sqrt(n)))

--- 第 1/2 轮迭代 ---
-> 反思: 当前代码时间复杂度 O(n*sqrt(n))，n 较大时性能差。
    建议使用埃拉托斯特尼筛法，时间复杂度 O(n log log n)。
📝 记忆已更新: 'reflection'

-> 优化: 重写为筛法实现
📝 记忆已更新: 'execution'

--- 第 2/2 轮迭代 ---
-> 反思: 当前代码使用筛法，时间复杂度 O(n log log n)，空间 O(n)。
    此算法已经非常高效，"无需改进"。
📝 记忆已更新: 'reflection'
✅ 反思认为代码已无需改进，任务完成。

--- 任务完成 ---
最终生成的代码:
def find_primes(n):
    """Finds all prime numbers between 1 and n using Sieve of Eratosthenes."""
    if n < 2:
        return []
    is_prime = [True] * (n + 1)
    is_prime[0] = is_prime[1] = False
    p = 2
    while p * p <= n:
        if is_prime[p]:
            for i in range(p * p, n + 1, p):
                is_prime[i] = False
        p += 1
    return [num for num in range(2, n + 1) if is_prime[num]]
```

## 关键设计

- **三个角色分离**：程序员（初始+优化） vs 评审专家（反思），防止自我欺骗
- **终止条件**：`"无需改进" in feedback` — 字符串匹配而非复杂逻辑，简洁有效
- **Memory 序列化**：`get_trajectory()` 按 `type` 区分格式，但本例中三个提示词各自独立没使用它
- **`max_iterations`**：默认 3 轮防止无限优化

## 与其他范式的关系

- **非独立范式**：Reflection 是增强层，可叠加到 ReAct 或 Plan-and-Solve 之上
- **与 ReAct 的区别**：ReAct 侧重"工具使用"的思维链，Reflection 侧重"输出质量"的自我审查

## 关联

- [[concepts/agent/概念_ReAct范式]] — Reflection 常叠加在 ReAct 之上，作为每 N 轮的后置反思步骤
- [[concepts/agent/概念_Agent_Loop]] — Reflection 在标准循环后增加 Critique 环节，形成扩展 Agent Loop
