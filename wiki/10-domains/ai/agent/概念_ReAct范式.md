---
type: concept
tags:
  - ai
  - agent
  - react
  - llm-pattern
summary: "ReAct 将 Reasoning 与 Acting 在同一个循环中交替进行，核心由 ReActAgent + ToolExecutor + 提示词模板三部分组成"
sources:
  - "[[30-sources/courses/Hello-Agents教程]]"
  - "https://github.com/datawhalechina/hello-agents"
aliases:
  - ReAct
  - Reasoning and Acting
  - 思考-行动范式
status: stable
confidence: 0.95
updated: "2026-04-28 22:30:00"
---

# 概念_ReAct范式

## 定义

ReAct（Reasoning + Acting）是最经典的智能体范式。LLM 以 `Thought → Action → Observation` 的节奏交替运行：每轮模型输出一个 Thought（推理文本）紧跟一个 Action（工具调用），观察返回值后进入下一轮循环，逐步逼近目标。

## 核心架构

ReAct 智能体由三个组件构成：

| 组件 | 职责 |
|------|------|
| **HelloAgentsLLM** | 统一封装 LLM 调用，支持流式输出 |
| **ToolExecutor** | 工具注册、获取、格式化描述 |
| **ReActAgent** | 循环驱动：构建 prompt → 调用 LLM → 解析输出 → 执行工具 |

## 完整实现

### 1. LLM 客户端（带流式输出）

```python
import os
from openai import OpenAI
from dotenv import load_dotenv

load_dotenv()

class HelloAgentsLLM:
    def __init__(self, model=None, apiKey=None, baseUrl=None, timeout=None):
        self.model = model or os.getenv("LLM_MODEL_ID")
        apiKey = apiKey or os.getenv("LLM_API_KEY")
        baseUrl = baseUrl or os.getenv("LLM_BASE_URL")
        timeout = timeout or int(os.getenv("LLM_TIMEOUT", 60))
        if not all([self.model, apiKey, baseUrl]):
            raise ValueError("模型ID、API密钥和服务地址必须被提供或在.env文件中定义。")
        self.client = OpenAI(api_key=apiKey, base_url=baseUrl, timeout=timeout)

    def think(self, messages, temperature=0):
        print(f"🧠 正在调用 {self.model} 模型...")
        try:
            response = self.client.chat.completions.create(
                model=self.model, messages=messages,
                temperature=temperature, stream=True,
            )
            collected_content = []
            for chunk in response:
                content = chunk.choices[0].delta.content or ""
                print(content, end="", flush=True)
                collected_content.append(content)
            print()
            return "".join(collected_content)
        except Exception as e:
            print(f"❌ 调用LLM API时发生错误: {e}")
            return None
```

### 2. 工具系统

```python
class ToolExecutor:
    def __init__(self):
        self.tools = {}

    def registerTool(self, name, description, func):
        self.tools[name] = {"description": description, "func": func}

    def getTool(self, name):
        return self.tools.get(name, {}).get("func")

    def getAvailableTools(self):
        return "\n".join([
            f"- {name}: {info['description']}"
            for name, info in self.tools.items()
        ])


# 实战搜索工具（基于 SerpApi）
from serpapi import SerpApiClient

def search(query: str) -> str:
    """基于 SerpApi 的 Google 搜索工具，智能解析结果优先级"""
    print(f"🔍 正在执行 [SerpApi] 网页搜索: {query}")
    try:
        api_key = os.getenv("SERPAPI_API_KEY")
        if not api_key:
            return "错误: SERPAPI_API_KEY 未配置。"
        params = {
            "engine": "google", "q": query,
            "api_key": api_key, "gl": "cn", "hl": "zh-cn",
        }
        client = SerpApiClient(params)
        results = client.get_dict()

        # 智能解析：优先返回结构化答案
        if "answer_box_list" in results:
            return "\n".join(results["answer_box_list"])
        if "answer_box" in results and "answer" in results["answer_box"]:
            return results["answer_box"]["answer"]
        if "knowledge_graph" in results and "description" in results["knowledge_graph"]:
            return results["knowledge_graph"]["description"]
        if "organic_results" in results and results["organic_results"]:
            snippets = [
                f"[{i+1}] {res.get('title', '')}\n{res.get('snippet', '')}"
                for i, res in enumerate(results["organic_results"][:3])
            ]
            return "\n\n".join(snippets)
        return f"对不起，没有找到关于 '{query}' 的信息。"
    except Exception as e:
        return f"搜索时发生错误: {e}"
```

### 3. ReAct 提示词模板

```python
REACT_PROMPT_TEMPLATE = """
请注意，你是一个有能力调用外部工具的智能助手。

可用工具如下:
{tools}

请严格按照以下格式进行回应:

Thought: 你的思考过程，用于分析问题、拆解任务和规划下一步行动。
Action: 你决定采取的行动，必须是以下格式之一:
- `{{tool_name}}[{{tool_input}}]`:调用一个可用工具。
- `Finish[最终答案]`:当你认为已经获得最终答案时。
- 当你收集到足够的信息，能够回答用户的最终问题时，你必须在Action:字段后使用 Finish[最终答案] 来输出最终答案。

现在，请开始解决以下问题:
Question: {question}
History: {history}
"""
```

### 4. ReActAgent 主体

```python
import re

class ReActAgent:
    def __init__(self, llm_client, tool_executor, max_steps=5):
        self.llm_client = llm_client
        self.tool_executor = tool_executor
        self.max_steps = max_steps
        self.history = []

    def run(self, question):
        self.history = []
        current_step = 0

        while current_step < self.max_steps:
            current_step += 1
            print(f"--- 第 {current_step} 步 ---")

            # 1. 格式化提示词
            tools_desc = self.tool_executor.getAvailableTools()
            history_str = "\n".join(self.history)
            prompt = REACT_PROMPT_TEMPLATE.format(
                tools=tools_desc, question=question, history=history_str
            )

            # 2. 调用 LLM
            messages = [{"role": "user", "content": prompt}]
            response_text = self.llm_client.think(messages=messages)
            if not response_text:
                print("错误: LLM未能返回有效响应。")
                break

            # 3. 解析输出
            thought, action = self._parse_output(response_text)
            if thought:
                print(f"🤔 思考: {thought}")

            if not action:
                print("警告: 未能解析出有效的Action，流程终止。")
                break

            # 4. 执行 Action
            if action.startswith("Finish"):
                final_answer = re.match(r"Finish\[(.*)\]", action).group(1)
                print(f"🎉 最终答案: {final_answer}")
                return final_answer

            tool_name, tool_input = self._parse_action(action)
            if not tool_name:
                continue

            print(f"🎬 行动: {tool_name}[{tool_input}]")
            tool_function = self.tool_executor.getTool(tool_name)
            observation = tool_function(tool_input) if tool_function \
                else f"错误: 未找到名为 '{tool_name}' 的工具。"

            print(f"👀 观察: {observation}")

            # 5. 追加历史
            self.history.append(f"Action: {action}")
            self.history.append(f"Observation: {observation}")

        print("已达到最大步数，流程终止。")
        return None

    def _parse_output(self, text):
        thought_match = re.search(r"Thought:\s*(.*?)(?=\nAction:|$)", text, re.DOTALL)
        action_match = re.search(r"Action:\s*(.*?)$", text, re.DOTALL)
        thought = thought_match.group(1).strip() if thought_match else None
        action = action_match.group(1).strip() if action_match else None
        return thought, action

    def _parse_action(self, action_text):
        match = re.match(r"(\w+)\[(.*)\]", action_text, re.DOTALL)
        if match:
            return match.group(1), match.group(2)
        return None, None
```

## 运行轨迹示例

```
工具 'Search' 已注册。

--- 第 1 步 ---
Thought: 需要查找华为最新发布的手机型号及其主要特点。
Action: Search[华为最新手机型号及主要卖点]
👀 观察: [1] 华为手机- 华为官网 Mate 系列 · Pura 系列 · Pocket 系列...

--- 第 2 步 ---
Thought: 搜索结果指向 Mate 70 和 Pura 80 Pro+。可以给出最终答案。
Action: Finish[华为最新旗舰为 HUAWEI Mate 70 和 Pura 80 Pro+。Mate 70 主打顶级拍照配置，Pura 80 Pro+ 强调先锋影像技术。]
```

## 关键设计

- **`Action[input]` 而非 JSON**：简化解析，正则提取工具名和参数
- **History 全量追加**：每轮的 Action 和 Observation 作为纯文本追加到 prompt 中
- **单步限制 max_steps**：默认 5 步，防止无限循环
- **智能结果解析**：SerpApi 优先返回 answer_box/knowledge_graph，回退到 organic_results

## 关联

- [[10-domains/ai/agent/概念_Agent_Loop]] — ReAct 是 Agent Loop 的最经典实现，定义了明确的 Thought-Action 解析规范
- [[10-domains/ai/agent/概念_Plan-and-Solve范式]] — 与 ReAct 形成策略对比：边想边做 vs 先规划再执行
- [[10-domains/ai/agent/概念_Reflection范式]] — 可在 ReAct 循环后附加反思环节，形成更完整的自我修正链条
