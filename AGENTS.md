# 知识库维护规范

本仓库是基于 Obsidian 的长期知识库。`raw/` 是只读来源层，`wiki/` 是经过整理、可链接、可查询和可持续演化的知识层。

## 核心原则

- 不只回答一次；将可复用的理解写入 `wiki/`。
- 目录表达页面的稳定归属与生命周期；`type` frontmatter 表达知识形态。
- 不再使用按 `concepts`、`sources`、`entities`、`comparisons`、`overview` 分类的旧目录；这些目录已删除。
- `raw/` 始终只读。除非用户明确授权，不修改 `wiki/` 以外的文件。
- 不删除页面、不做大规模重构，除非用户明确要求。
- 不伪造来源、引用、运行结果或置信度；不确定时清楚标注。

## 当前目录

```text
wiki/
  00-inbox/       # 尚未确定稳定归属的输入
  10-domains/     # 长期、可复用的领域知识
  20-projects/    # 有目标、上下文和结束条件的项目工作
  30-sources/     # 图书、论文、仓库、课程等来源摘要
  40-maps/        # 跨领域导航、比较、学习路径和问题地图
  90-system/      # 架构、模板、工作流、查询和维护脚本
  _meta/          # 迁移、登记和维护说明
  index.md        # 全库入口
  log.md          # 变更日志
```

详细定义以 [[90-system/architecture/知识库架构骨架]] 为准。

### 归档判断

1. 尚未确定归属 → `00-inbox/`
2. 一份资料本身说了什么 → `30-sources/<媒介>/`
3. 服务于明确项目 → `20-projects/<项目>/`
4. 可脱离项目长期复用 → `10-domains/<领域>/`
5. 用于导航、比较、路线或跨来源综合 → `40-maps/`
6. 用于知识库自身能力 → `90-system/`

活跃领域和项目均应维护 `_index.md`。目录不确定时先写入 `00-inbox/`，不要为了新笔记类型新增顶层目录。

## Frontmatter

新页面使用以下字段；保留已有页面的其他有意义字段。

```yaml
---
type: concept                  # source | concept | entity | comparison | overview | insight | decision | experiment | map
area: java/jvm
scope: durable                 # inbox | durable | project | source | map | system
project: []
summary: "一句话说明页面解决的问题"
sources: []
aliases: []
tags: []
status: draft                  # draft | evolving | stable | archived
confidence: 0.0
created: "YYYY-MM-DD HH:mm:ss"
updated: "YYYY-MM-DD HH:mm:ss"
---
```

### 来源规则

- 来源摘要页放在 `30-sources/`，其 `sources` 记录原始文件路径、仓库地址或外部链接。
- 领域页、项目页、地图页的 `sources` 必须优先链接对应来源摘要页，例如：
  `"[[30-sources/repositories/来源_Spring_Framework_源码]]"`。
- 若需要保留细粒度证据，可在来源摘要链接之后附加 `raw/` 文件路径或外部链接。
- 不使用已删除目录的路径，也不用 `wiki/30-sources/...` 作为来源字段；统一使用 `[[30-sources/...]]`。

## 页面与链接

- 文件名使用稳定、清晰的中文主题名；不再强制以“概念_”“实体_”“综述_”区分类型。
- 项目决策命名为 `ADR_YYYYMMDD_主题.md`；实验记录命名为 `实验_主题.md`。
- 使用完整路径 Wikilink，如 `[[10-domains/java/jvm/概念_JVM内存区域]]`。
- 链接必须说明关系，避免无上下文的裸链接。
- 每个普通知识页尽量连接至少两个相关页面；来源页可例外。
- 新建前先查找同领域、别名和摘要，优先更新已有页面，避免语义重复。

## 工作模式

### 查询

1. 先看 `wiki/index.md`、领域或项目 `_index.md`。
2. 再按 frontmatter `summary` 和相关目录定位最少页面。
3. 基于现有知识回答；覆盖不足时明确说明缺口。
4. 若结论可复用，按用户意图写回或提出写回建议。

### 导入

适用于 `raw/` 文件、文章、转录、文档或用户提供的长材料。

1. 识别领域或项目，检查是否已有相近页面。
2. 在 `30-sources/` 创建或更新一份来源摘要。
3. 在 `10-domains/` 或 `20-projects/` 创建或更新 0–3 个高价值页面。
4. 必要时更新一个 `40-maps/` 页面或领域 / 项目 `_index.md`。
5. 让知识页的 `sources` 链接到新来源摘要。
6. 更新 `wiki/index.md`（仅全库入口变化时）和 `wiki/log.md`。

### 更新与综合

- 先读取目标页和 1–2 个近似页面，确认不存在重复。
- 使用最小必要改动；更新来源、相关链接和对应 `_index.md`。
- 多来源综合应说明当前结论、框架、冲突、不确定性和待验证问题。

### 健康检查

仅在用户明确要求 lint、审计、诊断或评审时执行。检查来源缺失、失效链接、重复、孤立页、浅层页面和过期内容；默认只报告，不直接修改。

## 索引与日志

- `wiki/index.md` 只作为全库入口，不枚举每一页。
- 领域和项目导航优先由各自 `_index.md` 承担。
- 每次创建、更新、迁移、合并或删除 Wiki 页面，都在 `wiki/log.md` 追加：

  ```markdown
  ## [YYYY-MM-DD] action | description
  ```

## 完成前检查

1. 页面是否一个月后仍有用？
2. 是否解释了关系，而不是只罗列事实？
3. `scope` 是否与目录一致？
4. 是否避免了重复，并建立了有意义的链接？
5. 来源摘要链接是否有效，原始证据是否没有被伪造？
6. 不确定性是否通过 `status`、`confidence` 或正文明确说明？
