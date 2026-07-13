---
type: concept
tags:
  - ai
  - agent
  - peas
  - task-analysis
summary: "PEAS 模型从 Performance / Environment / Actuators / Sensors 四个维度精确描述智能体的任务环境，来自《人工智能：一种现代方法》"
sources:
  - "https://github.com/datawhalechina/hello-agents"
  - "Russell & Norvig, Artificial Intelligence: A Modern Approach"
aliases:
  - PEAS
  - PEAS模型
  - 任务环境描述
status: stable
confidence: 0.95
updated: "2026-04-28 22:30:00"
---

# 概念_PEAS模型

## 定义

PEAS 是描述智能体任务环境的标准框架，源自《Artificial Intelligence: A Modern Approach》。四个维度：

- **P (Performance)** — 如何度量智能体的成功？
- **E (Environment)** — 智能体所处的环境是什么？
- **A (Actuators)** — 智能体有哪些可执行操作？
- **S (Sensors)** — 智能体如何感知环境状态？

## 环境特性维度

| 特性 | 含义 | 示例（旅行助手） |
|------|------|------------------|
| **部分可观察 vs 完全可观察** | 能否获取全部状态？ | 部分可观察：只知用户当前输入 |
| **确定性 vs 随机性** | 行动后果是否可预测？ | 随机：天气/搜索结果是动态的 |
| **静态 vs 动态** | 环境在决策时会变化？ | 动态：天气实时变化 |
| **单智能体 vs 多智能体** | 是否有其他行动者？ | 多智能体：用户也在做决策 |
| **序贯性** | 当前行动是否影响未来？ | 是：每步结果影响下一步 |

## 实例分析

### 智能旅行助手 PEAS

```
P (Performance):  用户满意度、推荐准确率、回复时效性
E (Environment):  互联网、天气API、搜索API、用户偏好
A (Actuators):    get_weather(city), get_attraction(city, weather), Finish[answer]
S (Sensors):      用户输入文本、API 返回值
```

对应代码中的工具注册：

```python
def get_weather(city: str) -> str:
    """查询城市实时天气 → Actuator"""
    url = f"https://wttr.in/{city}?format=j1"
    response = requests.get(url)
    data = response.json()
    return f"{city}当前天气:{data['current_condition'][0]['weatherDesc'][0]['value']}"

def get_attraction(city: str, weather: str) -> str:
    """搜索景点推荐 → Actuator"""
    response = tavily.search(query=f"{city} {weather} 旅游景点", include_answer=True)
    return response.get("answer", "未找到推荐")
```

### 自动驾驶 PEAS（对比参考）

| 维度 | 自动驾驶 |
|------|----------|
| P | 安全、速度、舒适度、遵守交通规则 |
| E | 道路、车辆、行人、天气、交通信号 |
| A | 方向盘、油门、刹车、转向灯 |
| S | 摄像头、激光雷达、GPS、速度传感器 |

## 设计工作流

使用 PEAS 设计新智能体的标准流程：

1. **P 先行** — 确定评估指标，这是智能体的"成功定义"
2. **E 范围界定** — 明确环境边界和特性组合
3. **A 工具选择** — 列出所有可用操作，一个 Actuator = 一个工具函数
4. **S 输入定义** — 明确传感器获取的数据格式和来源

## 关联

- [[concepts/agent/概念_Agent_Loop]] — PEAS 定义了 Agent Loop 运行的外部环境约束，A 即工具集、S 即输入源
