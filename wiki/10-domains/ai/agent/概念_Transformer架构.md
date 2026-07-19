---
type: concept
tags:
  - ai
  - llm
  - transformer
  - attention
  - deep-learning
summary: "Transformer 是现代 LLM 的核心架构，由 MultiHeadAttention + PositionWiseFeedForward + PositionalEncoding 组合堆叠而成"
sources:
  - "[[30-sources/courses/Hello-Agents教程]]"
  - "https://github.com/datawhalechina/hello-agents"
  - "Vaswani et al., Attention Is All You Need (2017)"
aliases:
  - Transformer
  - 注意力机制
  - 多头注意力
status: stable
confidence: 0.95
updated: "2026-04-28 22:30:00"
---

# 概念_Transformer架构

## 定义

Transformer 是 2017 年提出的纯注意力机制序列建模架构，完全摒弃了 RNN/LSTM。核心组件：**MultiHeadAttention**（多头自注意力）、**PositionWiseFeedForward**（位置前馈网络）、**PositionalEncoding**（位置编码），通过残差连接+层归一化堆叠。

## 完整实现

### 1. 多头注意力（MultiHeadAttention）

```python
import torch
import torch.nn as nn
import math

class MultiHeadAttention(nn.Module):
    def __init__(self, d_model, num_heads):
        super().__init__()
        assert d_model % num_heads == 0
        self.d_model = d_model
        self.num_heads = num_heads
        self.d_k = d_model // num_heads

        # Q, K, V 和输出投影矩阵
        self.W_q = nn.Linear(d_model, d_model)
        self.W_k = nn.Linear(d_model, d_model)
        self.W_v = nn.Linear(d_model, d_model)
        self.W_o = nn.Linear(d_model, d_model)

    def scaled_dot_product_attention(self, Q, K, V, mask=None):
        # Attention(Q,K,V) = softmax(QK^T / √d_k) V
        attn_scores = torch.matmul(Q, K.transpose(-2, -1)) / math.sqrt(self.d_k)

        if mask is not None:
            attn_scores = attn_scores.masked_fill(mask == 0, -1e9)

        attn_probs = torch.softmax(attn_scores, dim=-1)
        return torch.matmul(attn_probs, V)

    def split_heads(self, x):
        # (batch, seq_len, d_model) → (batch, num_heads, seq_len, d_k)
        batch, seq_len, _ = x.size()
        return x.view(batch, seq_len, self.num_heads, self.d_k).transpose(1, 2)

    def combine_heads(self, x):
        # 反向操作：合并多头
        batch, _, seq_len, _ = x.size()
        return x.transpose(1, 2).contiguous().view(batch, seq_len, self.d_model)

    def forward(self, Q, K, V, mask=None):
        Q = self.split_heads(self.W_q(Q))
        K = self.split_heads(self.W_k(K))
        V = self.split_heads(self.W_v(V))

        attn_output = self.scaled_dot_product_attention(Q, K, V, mask)
        return self.W_o(self.combine_heads(attn_output))
```

### 2. 位置前馈网络（PositionWiseFeedForward）

```python
class PositionWiseFeedForward(nn.Module):
    def __init__(self, d_model, d_ff, dropout=0.1):
        super().__init__()
        self.linear1 = nn.Linear(d_model, d_ff)   # 扩展
        self.linear2 = nn.Linear(d_ff, d_model)    # 压缩回
        self.relu = nn.ReLU()
        self.dropout = nn.Dropout(dropout)

    def forward(self, x):
        return self.linear2(self.dropout(self.relu(self.linear1(x))))
```

### 3. 位置编码（PositionalEncoding）

```python
class PositionalEncoding(nn.Module):
    def __init__(self, d_model, dropout=0.1, max_len=5000):
        super().__init__()
        self.dropout = nn.Dropout(dropout)

        position = torch.arange(max_len).unsqueeze(1)
        div_term = torch.exp(
            torch.arange(0, d_model, 2) * (-math.log(10000.0) / d_model)
        )

        pe = torch.zeros(max_len, d_model)
        pe[:, 0::2] = torch.sin(position * div_term)  # 偶数维度: sin
        pe[:, 1::2] = torch.cos(position * div_term)  # 奇数维度: cos
        self.register_buffer('pe', pe.unsqueeze(0))

    def forward(self, x):
        return self.dropout(x + self.pe[:, :x.size(1)])
```

### 4. Encoder / Decoder 层

```python
class EncoderLayer(nn.Module):
    def __init__(self, d_model, num_heads, d_ff, dropout):
        super().__init__()
        self.self_attn = MultiHeadAttention(d_model, num_heads)
        self.feed_forward = PositionWiseFeedForward(d_model, d_ff, dropout)
        self.norm1 = nn.LayerNorm(d_model)
        self.norm2 = nn.LayerNorm(d_model)
        self.dropout = nn.Dropout(dropout)

    def forward(self, x, mask):
        # 残差连接 + 层归一化
        attn_out = self.self_attn(x, x, x, mask)
        x = self.norm1(x + self.dropout(attn_out))
        ff_out = self.feed_forward(x)
        x = self.norm2(x + self.dropout(ff_out))
        return x

class DecoderLayer(nn.Module):
    def __init__(self, d_model, num_heads, d_ff, dropout):
        super().__init__()
        self.self_attn = MultiHeadAttention(d_model, num_heads)   # 掩码自注意力
        self.cross_attn = MultiHeadAttention(d_model, num_heads)  # 交叉注意力
        self.feed_forward = PositionWiseFeedForward(d_model, d_ff, dropout)
        self.norm1 = nn.LayerNorm(d_model)
        self.norm2 = nn.LayerNorm(d_model)
        self.norm3 = nn.LayerNorm(d_model)
        self.dropout = nn.Dropout(dropout)

    def forward(self, x, encoder_output, src_mask, tgt_mask):
        x = self.norm1(x + self.dropout(self.self_attn(x, x, x, tgt_mask)))
        x = self.norm2(x + self.dropout(self.cross_attn(x, encoder_output, encoder_output, src_mask)))
        x = self.norm3(x + self.dropout(self.feed_forward(x)))
        return x
```

## 关键公式

| 组件 | 公式 |
|------|------|
| 缩放点积注意力 | `Attention(Q,K,V) = softmax(QK^T / √d_k) V` |
| 位置编码（偶数维） | `PE(pos, 2i) = sin(pos / 10000^(2i/d_model))` |
| 位置编码（奇数维） | `PE(pos, 2i+1) = cos(pos / 10000^(2i/d_model))` |
| 残差连接 | `Output = LayerNorm(x + Sublayer(x))` |

## 关键设计

- **√d_k 缩放**：防止点积过大导致 softmax 进入饱和区
- **Mask 机制**：Encoder 用 padding mask；Decoder 用 causal mask（下三角矩阵）防止看到未来 token
- **PositionalEncoding 为何用 sin/cos**：允许模型容易学习相对位置关系（`PE(pos+k)` 可表示为 `PE(pos)` 的线性函数）

## 关联

- [[10-domains/ai/agent/概念_BPE分词算法]] — BPE 产生 token IDs，是 Transformer 的输入层
- [[10-domains/ai/agent/概念_提示工程]] — Transformer 是提示工程的底层模型
- [[30-sources/courses/Hello-Agents教程]] — Chapter 3.1 详解 Transformer 的逐步构建
