---
type: concept
tags:
  - ai
  - agent
  - agent-loop
  - core-mechanism
summary: "智能体通过 感知→思考→行动→观察 的持续循环与环境交互，核心代码仅需 ~70 行即可实现"
sources:
  - "[[30-sources/courses/Hello-Agents教程]]"
  - "https://github.com/datawhalechina/hello-agents"
aliases:
  - Agent Loop
  - 智能体循环
  - 感知-思考-行动-观察
status: stable
confidence: 0.95
updated: "2026-04-28 22:30:00"
---

# 概念_Agent_Loop

## 定义

Agent Loop（智能体循环）是智能体与环境交互的核心运行机制：智能体通过传感器**感知**环境状态，经内部推理**思考**决策，调用执行器采取**行动**，再接收环境反馈的**观察**结果，形成持续闭环。

## 四阶段拆解

| 阶段 | 职责 | 对应组件 |
|------|------|----------|
| **Thought** | 分析问题、拆解任务、规划下一步 | LLM 推理 |
| **Action** | 调用工具或返回最终答案 | 工具函数 / `Finish[...]` |
| **Observation** | 环境返回操作结果，作为下一轮输入 | 工具返回值 |
| **Loop** | 迭代上述三步直到完成或达到上限 | `while` 循环 |

## 核心实现

### 1. LLM 客户端封装

```python
from openai import OpenAI

class OpenAICompatibleClient:
    def __init__(self, model, api_key, base_url):
        self.model = model
        self.client = OpenAI(api_key=api_key, base_url=base_url)

    def generate(self, prompt, system_prompt):
        messages = [
            {'role': 'system', 'content': system_prompt},
            {'role': 'user', 'content': prompt}
        ]
        response = self.client.chat.completions.create(
            model=self.model, messages=messages, stream=False
        )
        return response.choices[0].message.content
```

### 2. 系统提示词（定义 Thought-Action 协议）

```python
AGENT_SYSTEM_PROMPT = """
你是一个智能旅行助手。

# 可用工具:
- `get_weather(city: str)`: 查询指定城市的实时天气。
- `get_attraction(city: str, weather: str)`: 根据城市和天气搜索推荐的旅游景点。

# 输出格式要求:
你的每次回复必须严格遵循以下格式:

Thought: [你的思考过程和下一步计划]
Action: [你要执行的具体行动]

Action的格式必须是以下之一：
1. 调用工具：function_name(arg_name="arg_value")
2. 结束任务：Finish[最终答案]

# 重要提示:
- 每次只输出一对 Thought-Action
- Action 必须在同一行，不要换行
- 当收集到足够信息可以回答用户问题时，必须使用 Action: Finish[最终答案]
"""
```

### 3. 工具定义

```python
import requests

def get_weather(city: str) -> str:
    """通过 wttr.in API 查询真实天气"""
    url = f"https://wttr.in/{city}?format=j1"
    try:
        response = requests.get(url)
        response.raise_for_status()
        data = response.json()
        current = data['current_condition'][0]
        weather_desc = current['weatherDesc'][0]['value']
        temp_c = current['temp_C']
        return f"{city}当前天气:{weather_desc}，气温{temp_c}摄氏度"
    except Exception as e:
        return f"错误:查询天气失败 - {e}"

def get_attraction(city: str, weather: str) -> str:
    """使用 Tavily Search API 搜索景点推荐"""
    from tavily import TavilyClient
    import os
    api_key = os.environ.get("TAVILY_API_KEY")
    tavily = TavilyClient(api_key=api_key)
    query = f"'{city}' 在'{weather}'天气下最值得去的旅游景点推荐及理由"
    response = tavily.search(query=query, search_depth="basic", include_answer=True)
    if response.get("answer"):
        return response["answer"]
    results = response.get("results", [])
    return "\n".join([f"- {r['title']}: {r['content']}" for r in results])

available_tools = {
    "get_weather": get_weather,
    "get_attraction": get_attraction,
}
```

### 4. 主循环（完整 Agent Loop）

```python
import re

# 初始化
user_prompt = "你好，请帮我查询一下今天北京的天气，然后根据天气推荐一个合适的旅游景点。"
prompt_history = [f"用户请求: {user_prompt}"]

for i in range(5):  # 最大循环次数
    # 1. 构建 Prompt 并调用 LLM
    full_prompt = "\n".join(prompt_history)
    llm_output = llm.generate(full_prompt, system_prompt=AGENT_SYSTEM_PROMPT)
    prompt_history.append(llm_output)

    # 2. 解析 Action
    action_match = re.search(r"Action: (.*)", llm_output, re.DOTALL)
    if not action_match:
        continue
    action_str = action_match.group(1).strip()

    # 3. 检查是否结束
    if action_str.startswith("Finish"):
        final_answer = re.match(r"Finish\[(.*)\]", action_str).group(1)
        print(f"任务完成: {final_answer}")
        break

    # 4. 执行工具调用
    tool_name = re.search(r"(\w+)\(", action_str).group(1)
    args_str = re.search(r"\((.*)\)", action_str).group(1)
    kwargs = dict(re.findall(r'(\w+)="([^"]*)"', args_str))

    observation = available_tools[tool_name](**kwargs) if tool_name in available_tools \
        else f"错误:未定义的工具 '{tool_name}'"

    # 5. 记录 Observation 到历史
    observation_str = f"Observation: {observation}"
    prompt_history.append(observation_str)
```

### 5. 运行轨迹

```
用户输入: 帮我查询北京天气并推荐景点
--- 循环 1 ---
Thought: 首先需要获取北京今天的天气。
Action: get_weather(city="北京")
Observation: 北京当前天气:Sunny，气温26摄氏度
--- 循环 2 ---
Thought: 晴天适合户外，搜索推荐景点。
Action: get_attraction(city="北京", weather="Sunny")
Observation: 推荐颐和园和长城
--- 循环 3 ---
Thought: 信息充足，可以给出最终建议。
Action: Finish[今天北京晴26°C，推荐颐和园和长城]
```

## 关键设计决策

- **单次只输出一对 Thought-Action**：防止模型一次性输出多步导致解析失败
- **历史全量传递**：每轮将完整 Thought-Action-Observation 链附加到 prompt，确保上下文连贯
- **最大步数限制**（通常 5-10）：防止死循环耗尽 token
- **Finish 终止符**：显式终止机制，避免模型持续调用工具

## 关联

- [[10-domains/ai/agent/概念_ReAct范式]] — ReAct 是 Agent Loop 最经典的具体实现，显式定义 Thought-Action 解析规范
- [[10-domains/ai/agent/概念_PEAS模型]] — PEAS 定义了 Agent Loop 运行的外部环境约束和交互边界
- [[30-sources/courses/Hello-Agents教程]] — Chapter 1 的完整旅行助手案例演示了 Agent Loop 的端到端实现
