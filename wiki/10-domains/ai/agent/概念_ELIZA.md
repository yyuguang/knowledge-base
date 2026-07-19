---
type: concept
tags:
  - ai
  - agent
  - eliza
  - history
  - nlp
summary: "1966 年首个对话系统，基于正则模式匹配+代词转换，~80 行 Python 可完整重现"
sources:
  - "[[30-sources/courses/Hello-Agents教程]]"
  - "https://github.com/datawhalechina/hello-agents"
aliases:
  - ELIZA
  - 伊莉莎
  - 模式匹配聊天机器人
status: stable
confidence: 0.95
updated: "2026-04-28 22:30:00"
---

# 概念_ELIZA

## 定义

ELIZA 是 Joseph Weizenbaum 于 1966 年在 MIT 开发的早期自然语言处理程序，模拟 Rogerian 心理治疗师的对话风格。它是历史上第一个聊天机器人，底层原理极简：**正则模式匹配 + 代词转换 + 模板回复**。

## 核心算法

```
1. 将用户输入拆分成单词
2. 按优先级寻找匹配的关键词规则
3. 用规则分解用户输入
4. 对分解出的部分进行代词转换（I→you, my→your）
5. 从重组模板中随机选择并格式化回复
```

## 完整实现（~80 行 Python）

```python
import re
import random

# 规则库：正则模式 → 响应模板列表（优先级从高到低）
rules = {
    r'I need (.*)': [
        "Why do you need {0}?",
        "Would it really help you to get {0}?",
        "Are you sure you need {0}?"
    ],
    r'Why don\'t you (.*)\?': [
        "Do you really think I don't {0}?",
        "Perhaps eventually I will {0}.",
    ],
    r'I am (.*)': [
        "Did you come to me because you are {0}?",
        "How long have you been {0}?",
        "How do you feel about being {0}?"
    ],
    r'.* mother .*': [
        "Tell me more about your mother.",
        "What was your relationship with your mother like?",
    ],
    r'.* father .*': [
        "Tell me more about your father.",
        "How did your father make you feel?",
    ],
    r'.*': [  # 兜底规则
        "Please tell me more.",
        "Let's change focus a bit... Tell me about your family.",
        "Can you elaborate on that?"
    ]
}

# 代词转换表（第一人称 ↔ 第二人称）
pronoun_swap = {
    "i": "you", "you": "i", "me": "you", "my": "your",
    "am": "are", "are": "am", "was": "were",
    "i've": "you have", "i'll": "you will",
    "mine": "yours", "yours": "mine"
}

def swap_pronouns(phrase):
    """第一/第二人称代词转换"""
    words = phrase.lower().split()
    return " ".join([pronoun_swap.get(w, w) for w in words])

def respond(user_input):
    """核心：模式匹配 → 代词转换 → 模板填充"""
    for pattern, responses in rules.items():
        match = re.search(pattern, user_input, re.IGNORECASE)
        if match:
            captured = match.group(1) if match.groups() else ''
            swapped = swap_pronouns(captured)
            return random.choice(responses).format(swapped)
    return random.choice(rules[r'.*'])

# 运行
if __name__ == '__main__':
    print("Therapist: Hello! How can I help you today?")
    while True:
        user_input = input("You: ")
        if user_input.lower() in ["quit", "exit", "bye"]:
            print("Therapist: Goodbye. It was nice talking to you.")
            break
        print(f"Therapist: {respond(user_input)}")
```

运行示例：
```
You: I am feeling sad today.
Therapist: How long have you been feeling sad today?
You: I need some help with my project.
Therapist: Are you sure you need some help with your project?
You: My mother is not happy with my work.
Therapist: Tell me more about your mother.
```

## 设计要点

- **规则优先级**：`r'.* mother .*'` 比 `r'I am (.*)'` 更具体，匹配更靠前
- **`{0}` 占位符**：Python `str.format()` 将捕获组注入模板
- **随机选择**：每条规则有多个模板，随机选取避免机械感
- **兜底规则**：`r'.*'` 确保任何输入都有回复

## 历史意义

ELIZA 虽无真正"理解"能力，但通过简单的语法变换制造了理解的幻象（ELIZA 效应）。这种"表面智能"的设计哲学至今仍影响着现代 Agent 的 prompt 工程。

## 关联

- [[10-domains/ai/agent/概念_Agent_Loop]] — ELIZA 是最早的感知-响应循环原型
- [[30-sources/courses/Hello-Agents教程]] — Chapter 2 详述了 ELIZA 的历史与实现
