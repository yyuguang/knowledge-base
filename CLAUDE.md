# 知识库协作说明

本仓库是 Obsidian 知识库：`raw/` 保存原始材料且只读，`wiki/` 保存经整理、可复用和可查询的知识。

## 当前架构

所有 Wiki 内容使用“稳定归属优先”的目录结构：

```text
wiki/
  00-inbox/       # 临时输入与待分流内容
  10-domains/     # 长期领域知识
  20-projects/    # 项目专属知识、决策和实验
  30-sources/     # 图书、论文、仓库、课程等来源摘要
  40-maps/        # 导航、比较、学习路线与跨来源综合
  90-system/      # 模板、工作流、查询和维护能力
  _meta/          # 迁移与维护说明
  index.md
  log.md
```

不要重新创建或引用旧的按类型目录。目录表达稳定归属，页面形态通过 frontmatter 的 `type` 表达。完整规则见 [[90-system/architecture/知识库架构骨架]]。

## 工作规则

- `raw/` 只读；默认只修改 `wiki/`。
- 新材料先在 `30-sources/` 建立或更新来源摘要，再将可复用结论写入领域或项目。
- 所有领域页和项目页的 `sources` 应优先链接来源摘要，如 `[[30-sources/books/深入理解Java虚拟机]]`；来源页本身记录原始路径或外部链接。
- 归属不明时使用 `00-inbox/`；可长期复用的知识进入 `10-domains/`；带明确目标的内容进入 `20-projects/`。
- 地图、比较、学习路线放入 `40-maps/`；模板、查询、自动化放入 `90-system/`。
- 每个活跃领域或项目维护 `_index.md`，用于导航而不是复制正文。
- 写入前先查重；使用完整路径的 Obsidian 链接，并在必要处说明链接关系。
- 不伪造来源或结论；不确定时使用 `status`、`confidence` 和正文显式说明。
- 每次 Wiki 变更追加到 `wiki/log.md`。

## Frontmatter 最小格式

```yaml
---
type: concept
area: java/jvm
scope: durable
project: []
summary: ""
sources: []
aliases: []
tags: []
status: draft
confidence: 0.0
created: "YYYY-MM-DD HH:mm:ss"
updated: "YYYY-MM-DD HH:mm:ss"
---
```

## 质量要求

完成导入、更新或综合前，确认页面能长期复用、来源可追溯、链接有效、归属正确、没有语义重复，并且清楚写出关系与不确定性。
