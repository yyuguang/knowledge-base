---
type: concept
tags:
  - ai
  - nlp
  - tokenization
  - bpe
  - llm
summary: "BPE（Byte Pair Encoding）从字符级开始迭代合并高频相邻对，构建子词词表，是现代 LLM 分词的基础"
sources:
  - "https://github.com/datawhalechina/hello-agents"
aliases:
  - BPE
  - Byte Pair Encoding
  - 字节对编码
  - 子词分词
status: stable
confidence: 0.95
updated: "2026-04-28 22:30:00"
---

# 概念_BPE分词算法

## 定义

BPE（Byte Pair Encoding）是一种自底向上的子词分词算法。从字符级开始，统计语料中所有相邻符号对的频率，每次合并最高频的一对为新符号，迭代直到达到预设词表大小。它是 GPT 系列、LLaMA 等现代 LLM 分词器的核心基础。

## 核心思想

解决"词表爆炸"与"UNK 问题"：纯词级分词词表过大且无法处理新词；纯字符级分词序列过长丢失语义。BPE 取折中——高频词作为整体，低频词拆为子词。

## 完整实现

```python
import re, collections

def get_stats(vocab):
    """统计所有相邻符号对的频率"""
    pairs = collections.defaultdict(int)
    for word, freq in vocab.items():
        symbols = word.split()
        for i in range(len(symbols) - 1):
            pairs[symbols[i], symbols[i+1]] += freq
    return pairs

def merge_vocab(pair, v_in):
    """将指定符号对合并为一个新符号"""
    v_out = {}
    bigram = re.escape(' '.join(pair))
    p = re.compile(r'(?<!\S)' + bigram + r'(?!\S)')
    for word in v_in:
        w_out = p.sub(''.join(pair), word)
        v_out[w_out] = v_in[word]
    return v_out

# 语料：hug, pug, pun, bun（带</w>结束标记）
vocab = {
    'h u g </w>': 1, 'p u g </w>': 1,
    'p u n </w>': 1, 'b u n </w>': 1
}
num_merges = 4

for i in range(num_merges):
    pairs = get_stats(vocab)
    if not pairs:
        break
    best = max(pairs, key=pairs.get)
    vocab = merge_vocab(best, vocab)
    print(f"合并 {i+1}: {best} -> {''.join(best)}")
    print(f"词表: {list(vocab.keys())}")
```

输出：
```
合并 1: ('u', 'g') -> ug
词表: ['h ug </w>', 'p ug </w>', 'p u n </w>', 'b u n </w>']
合并 2: ('ug', '</w>') -> ug</w>
词表: ['h ug</w>', 'p ug</w>', 'p u n </w>', 'b u n </w>']
合并 3: ('u', 'n') -> un
词表: ['h ug</w>', 'p ug</w>', 'p un </w>', 'b un </w>']
合并 4: ('un', '</w>') -> un</w>
词表: ['h ug</w>', 'p ug</w>', 'p un</w>', 'b un</w>']
```

## 关键设计点

| 设计 | 说明 |
|------|------|
| **`</w>` 结束标记** | 标记词边界，确保合并不会跨词 |
| **贪婪合并** | 每轮合并频率最高的对，逐轮构建更大子词 |
| **字符级起点** | 所有词首先被拆分为单个字符+空格 |
| **`num_merges` 超参数** | 控制词表大小，40000-50000 是 LLM 常见设置 |

## 在现代 LLM 中的应用

- **GPT 系列**：基于 BPE，使用 tiktoken 库
- **LLaMA**：基于 BPE 变体 SentencePiece（BPE + Unigram）
- **BERT**：使用 WordPiece（BPE 变体，基于概率而非频率）

## 关联

- [[concepts/agent/概念_Transformer架构]] — BPE 的输出（token IDs）是 Transformer 的输入
- [[sources/agent/Hello-Agents教程]] — Chapter 3.2.2 详解了 BPE 算法原理与实现
