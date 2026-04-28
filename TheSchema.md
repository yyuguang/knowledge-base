---
created: 2026-04-27T21:00:00
updated: 2026-04-27T21:49:00
type: guide
tags:
  - schema
  - wiki
  - knowledge-management
---

## 0. 目标与边界

> [!info] 核心目标
> 维护 `wiki/` 目录，将其打造为**可复利、可进化的知识层**

- **目标**：
	你负责维护本仓库中的 `wiki/` 目录，使其成为：
	- 可复用（Reusable）
	- 可推理（Queryable）
	- 可进化（Evolving）
	的长期知识系统。
- **边界**：
  - 原始资料（`raw/`）是唯一事实来源，**只读不改**
  - 只在 `wiki/` 里创建、修改 Markdown 页面
  - 不要动其它目录

---

## 1. 目录结构约定

### 原始资料层
- `raw/`：原始资料（PDF、网页 Markdown、图片等），**只读**

### Wiki 维护层
`wiki/` 由你维护，建议子目录：

| 目录                  | 用途           |
| ------------------- | ------------ |
| `wiki/sources/`     | 单个来源的摘要页     |
| `wiki/entities/`    | 人物、书籍、项目等实体页 |
| `wiki/concepts/`    | 方法、理论、模型等概念页 |
| `wiki/comparisons/` | 比较分析页        |
| `wiki/overview/`    | 总览、综合页       |

> **领域子目录**：上述目录可按领域划分子目录组织，例如 `wiki/concepts/java/`、`wiki/concepts/redis/`、`wiki/concepts/mq/`。每种领域目录遵循相同的页面类型规范。所有 `index.md` 中的索引条目也对应使用子目录路径。

### 根目录文件
- `wiki/index.md`：内容索引（可选，用 Obsidian 视图替代也行）
- `wiki/log.md`：操作日志


---

## 2. 页面类型与基本格式

所有 wiki 页面使用 Markdown，顶部 frontmatter 示例：

```yaml
---
type: "source|entity|concept|comparison|overview|insight"
tags: ["tag1", "tag2"]

summary: "一句话核心内容"
sources: ["raw/xxx.pdf"]

aliases: []
status: "draft|evolving|stable"
confidence: 0.0

updated: "YYYY-MM-DD HH:mm:ss"
---
```

### 2.1 Source Summary（来源摘要页）

路径：`wiki/sources/xxx.md`

- 来源信息（标题、作者、时间、链接）
- 核心要点（3–7 条 bullet）
- 关键引文（可选）
- 关联实体/概念链接（`[[entities/xxx]]` / `[[concepts/yyy]]`）

### 2.2 Entity Page（实体页）

路径：`wiki/entities/人物_马斯克.md` 等

- 基本信息
- 行为 / 特征 / 状态
- 相关事件 / 计划 / 实验链接
- 来自哪些来源（列出 `sources`）

### 2.3 Concept Page（概念页）

路径：`wiki/concepts/概念_费曼学习法.md` 等

- 定义
- 使用场景 / 步骤
- 在本知识库中的应用示例
- 关联实体 / 其它概念

### 2.4 Comparison Page（比较页）

路径：`wiki/comparisons/xxx_vs_yyy.md`

- 比较对象简介
- 相同点
- 不同点（目标、成本、适用场景…）
- 结论 / 选择建议

### 2.5 Overview / Synthesis（总览 / 综合）

路径：`wiki/overview/主题_儿童自驱学习_综述.md` 等

- 一句话结论（Summary）
- 当前理解 / 总体框架
- 支撑它的主要来源和页面链接
- 未决问题 / 待验证假设


---

## 3. 工作流：Ingest / Query / Lint

### 3.1 Ingest（导入新资料）

当我说”请基于 `raw/xxx` 进行 Ingest”时：

1. 阅读 `raw/xxx`，提炼要点，与我简短确认重点
2. 在 `wiki/sources/` 新建或更新摘要页
3. 根据内容更新或创建：
   - 相关实体页（`wiki/entities/`）
   - 相关概念页（`wiki/concepts/`）
4. 维护索引 / Log：
   - 在 `wiki/index.md` 补上新页面条目（标题、链接、一句话 summary）
   - 在 `wiki/log.md` 追加记录：
     ```
     ## [2026-04-08] ingest | raw/xxx → wiki/sources/xxx.md (+ affected pages)
     ```

### 3.2 Query（基于 wiki 回答问题）

当我提问时：

1. 通过 `wiki/index.md`、frontmatter 的 `summary` 找到候选页面
2. 读取页面内容，综合回答
3. 如回答有价值（比较 / 分析 / 计划），可建议：
   - 写回为新 wiki 页面（`wiki/comparisons/` 或 `wiki/overview/`）
   - 在 `wiki/log.md` 记录：
     ```
     ## [2026-04-08] query | 新建 wiki/comparisons/xxx_vs_yyy.md
     ```

### 3.3 Lint（健康检查）
当用户说：“对 wiki 做 Lint”

执行：

---

#### 扫描问题

1. 结构问题：
    - 页面缺少标准结构
    - frontmatter 不完整
2. 知识问题：
    - 明显冲突
    - 过时信息
    - 定义质量低
3. 图结构问题：
    - 孤立页面（无入链）
    - 链接过少
4. 覆盖问题：
    - 高频概念无独立页面
    - 缺少 overview

---
#### 输出

- 问题清单
- 修改建议（merge / split / create）

👉 不直接修改，需用户确认

---
### 3.4 Knowledge Evolution（知识进化）
wiki 是持续进化系统，而非静态记录。

---
#### 触发条件

##### 1. 定义升级
旧定义过于简单 → 更新
##### 2. 知识合并
重复或碎片化 → 合并
##### 3. 抽象提升
多个来源 → 提炼 concept / insight
##### 4. 结构优化
页面结构混乱 → 重构

---
所有变更记录到 `wiki/log.md`
## 4. 约定与风格
### 4.1 强制写回

禁止只回答而不更新 wiki

---

### 4.2 去重机制（Critical）

创建页面前必须：

1. 搜索语义相似页面
2. 若存在 → 更新而不是创建
3. 若冲突 → 提议合并

---

### 4.3 Wiki 优先

wiki 是主知识源

---

### 4.4 链接密度

每个页面必须：

- 至少 2 个内部链接
- 避免孤立

---

### 4.5 命名规范

中文 + 下划线 ,稳定可读（如 `概念_费曼学习法`），同时包含领域路径（如 `concepts/java/概念_费曼学习法.md`）
比较页：`xxx_vs_yyy`（如 `comparisons/java/Serial_vs_ParNew.md`）

---

### 4.6 不确定策略

不确定时，先提议再执行，不做大规模自动改动

### 4.7 工具选择

当需要查找笔记间的关系时，**优先使用 obsidian-cli skill**：

- Wiki links（双向链接）查询
- Tags 或带特定 tags 的笔记搜索
- Frontmatter 信息提取
- 笔记之间的相互关系梳理与分析


---
## 5. Log规范
### 写入格式
```
## [YYYY-MM-DD] action | description
```

### 示例
```
## [2026-04-08] ingest | raw/xxx → wiki/sources/xxx.md
## [2026-04-08] update | 概念_费曼学习法.md（补充步骤）
## [2026-04-08] merge | 概念_A + 概念_B → 概念_A
```
