# AGENTS.md

This repository is an Obsidian knowledge base maintained in the style of an LLM Wiki. Agents working here should treat `raw/` as source material and `wiki/` as the compiled, reusable knowledge layer.

The primary job is not to answer once, but to turn useful material into durable Markdown knowledge that can be queried, linked, updated, and evolved over time.

## Repository Map

```text
.
├── prompt/          # Custom prompt templates
├── raw/             # Original source materials, read-only
│   ├── papers/
│   ├── books/
│   └── media/
├── wiki/            # Compiled knowledge layer maintained by agents
│   ├── sources/     # Source summary pages
│   ├── concepts/    # Concepts, methods, models, theories
│   ├── entities/    # People, books, projects, tools, organizations
│   ├── comparisons/ # Comparison and decision pages
│   ├── overview/    # Cross-source synthesis and topic overviews
│   ├── insights/    # Higher-level reusable insights
│   ├── index.md     # Wiki index
│   └── log.md       # Operation log
├── Attachments/     # Obsidian attachments
├── README.md        # Repository overview
├── TheSchema.md     # Wiki schema and workflow guide
└── CLAUDE.md        # Prior agent instructions
```

## Hard Boundaries

- `raw/` is read-only. Never modify source materials.
- By default, create or modify only files under `wiki/`.
- Do not delete wiki pages unless the user explicitly asks.
- Do not perform broad restructuring without user confirmation.
- Prefer incremental improvement over destructive rewriting.
- Do not fabricate sources, citations, or confidence.
- Root-level configuration files such as `AGENTS.md`, `CLAUDE.md`, `TheSchema.md`, and `README.md` may be edited only when the user explicitly asks.

## Core Model

`raw/` is the source layer. `wiki/` is the compiled knowledge layer.

When ingesting or updating knowledge, do more than summarize. Extract:

- Concepts and definitions
- Entities and relationships
- Examples and trade-offs
- Reusable frameworks
- Comparisons and decision criteria
- Open questions and uncertainty

A good wiki page should still be useful one month later without rereading the original raw material.

## Operating Modes

Choose the mode from the user's request.

### Query Mode

Use when the user asks a question, explanation, comparison, decision suggestion, or summary based on the existing wiki.

Workflow:

1. Search `wiki/index.md` first.
2. Search relevant frontmatter summaries.
3. Search the relevant area subdirectory.
4. Load only the minimum required pages.
5. Answer based on the wiki.
6. If coverage is insufficient, say so clearly.
7. If the answer creates reusable knowledge, propose or perform a wiki write-back depending on the user's intent.

Output:

- Keep concise but useful, usually 5-10 key points.
- Mention source or concept pages used.
- Do not dump full pages unless requested.
- State knowledge gaps plainly.

### Ingest Mode

Use when the user provides raw notes, an article, transcript, PDF text, copied documentation, long-form material, or a file under `raw/`.

Workflow:

1. Identify the domain area.
2. Check whether related pages already exist.
3. Create or update one source summary page.
4. Create or update up to 3 important concept/entity pages.
5. Optionally create or update one overview/comparison page if the material connects multiple ideas.
6. Update `wiki/index.md`.
7. Append a record to `wiki/log.md`.

Default output:

- 1 source page
- 0-3 concept/entity pages
- 0-1 overview/comparison page
- index update
- log update

Do not create many shallow pages at once. If more pages are needed, add a TODO section or propose the next batch.

### Update Mode

Use when the user asks to improve, extend, refactor, merge, or update an existing wiki page.

Workflow:

1. Locate the target page.
2. Load only that page and 1-2 similar pages for duplication checks.
3. Decide whether to append, restructure, merge, or create a related page.
4. Apply the smallest useful change.
5. Update backlinks when necessary.
6. Update `wiki/index.md` if the summary or page list changes.
7. Append to `wiki/log.md`.

### Synthesis Mode

Use when the user asks for deeper understanding across multiple notes or sources.

Workflow:

1. Load relevant source, concept, comparison, and overview pages.
2. Identify repeated patterns.
3. Identify conflicts or unresolved questions.
4. Produce or update an overview page when useful.
5. Link supporting pages.
6. Record the change in `wiki/log.md`.

Synthesis is not a short summary. It should include current understanding, conceptual framework, recurring patterns, uncertainty, open questions, and next actions.

### Lint Mode

Use only when the user explicitly asks for lint, health check, audit, review, or diagnosis.

Check:

- Missing or incomplete frontmatter
- Weak definitions
- Duplicate or overlapping pages
- Isolated pages and missing links
- Shallow pages
- Outdated information
- Missing overview pages

Return up to 10 issues grouped by type. Do not modify files unless the user confirms.

## Wiki Structure

Use domain subdirectories under each page type:

```text
wiki/
  index.md
  log.md
  sources/<area>/来源_<title>.md
  concepts/<area>/概念_<name>.md
  entities/<area>/实体_<name>.md
  comparisons/<area>/<A>_vs_<B>.md
  overview/<area>/主题_<name>_综述.md
  insights/<area>/洞察_<name>.md
```

Examples:

```text
wiki/concepts/java/jvm/概念_JVM内存区域.md
wiki/concepts/java/concurrency/概念_AQS.md
wiki/sources/ai/来源_Karpathy_LLM_Wiki.md
wiki/overview/ai/主题_LLM知识库设计_综述.md
wiki/comparisons/java/gc/Serial_vs_ParNew.md
```

## Frontmatter

Every wiki page should use frontmatter.

```yaml
---
type: "source|concept|entity|comparison|overview|insight"
tags: []
summary: ""
sources: []
aliases: []
status: "draft|evolving|stable"
confidence: 0.0
created: "YYYY-MM-DD HH:mm:ss"
updated: "YYYY-MM-DD HH:mm:ss"
---
```

Use `status: draft` and lower `confidence` when uncertain. Do not hide uncertainty.

## Page Templates

### Source Page

Path: `wiki/sources/<area>/来源_<title>.md`

Recommended sections:

- `# 来源：<title>`
- `## 一句话总结`
- `## 来源信息`
- `## 核心观点`
- `## 关键论证`
- `## 重要概念`
- `## 可复用表达`
- `## 我的理解`
- `## 未解决问题`
- `## 相关链接`

### Concept Page

Path: `wiki/concepts/<area>/概念_<name>.md`

Recommended sections:

- `# 概念：<name>`
- `## 定义`
- `## 为什么重要`
- `## 使用场景`
- `## 核心机制`
- `## 例子`
- `## 常见误区`
- `## 和其它概念的关系`
- `## 我的理解`
- `## 来源`

### Entity Page

Path: `wiki/entities/<area>/实体_<name>.md`

Recommended sections:

- `# 实体：<name>`
- `## 基本信息`
- `## 关键特征`
- `## 相关事件 / 行为 / 作品`
- `## 和其它实体或概念的关系`
- `## 我的理解`
- `## 来源`

### Comparison Page

Path: `wiki/comparisons/<area>/<A>_vs_<B>.md`

Recommended sections:

- `# <A> vs <B>`
- `## 一句话结论`
- `## 比较对象`
- `## 相同点`
- `## 不同点`
- `## 选择建议`
- `## 相关链接`

### Overview Page

Path: `wiki/overview/<area>/主题_<name>_综述.md`

Recommended sections:

- `# 主题：<name> 综述`
- `## 当前结论`
- `## 总体框架`
- `## 核心概念`
- `## 关键来源`
- `## 重要关系`
- `## 当前判断`
- `## 未解决问题`
- `## 下一步`

### Insight Page

Path: `wiki/insights/<area>/洞察_<name>.md`

Create insight pages sparingly. Only create one when at least 2 sources support the same pattern and the insight can guide future understanding or action.

Recommended sections:

- `# 洞察：<name>`
- `## 洞察`
- `## 支撑依据`
- `## 推理过程`
- `## 适用边界`
- `## 后续验证`

## Linking Rules

Use Obsidian wikilinks with paths and context.

Good:

```markdown
[[concepts/java/jvm/概念_JVM内存区域]] — 堆的分代结构是理解分代 GC 的前提
```

Avoid naked links:

```markdown
[[概念_JVM内存区域]]
```

Rules:

- Use area paths in links.
- Prefer stable concept links over temporary source links.
- Each non-source page should have at least 2 internal links when possible.
- Each section should have at most 3 highly relevant links.
- Avoid over-linking.
- Avoid orphan pages.
- Every link should explain why it exists.

## Naming Rules

Use Chinese, readable, stable filenames.

- Source: `来源_<资料标题>.md`
- Concept: `概念_<概念名>.md`
- Entity: `实体_<实体名>.md`
- Overview: `主题_<主题名>_综述.md`
- Insight: `洞察_<洞察名>.md`
- Comparison: `<A>_vs_<B>.md`

Use the most specific useful domain area:

- Java JVM: `java/jvm`
- Java concurrency: `java/concurrency`
- Spring AI: `java/spring-ai`
- LangChain4j: `java/langchain4j`
- Redis: `redis`
- Message queue: `mq`
- AI Agent: `ai/agent`
- Prompt Engineering: `ai/prompt`
- Knowledge management: `knowledge-management`

If unsure, choose a broader area and mark uncertainty.

## Anti-Duplication Rules

Before creating a new page:

1. Search within the same area subdirectory.
2. Search aliases and summaries.
3. Search the type root if the area is uncertain.
4. If a similar page exists, update it instead of creating a new one.
5. If two pages overlap heavily, propose a merge.
6. Do not create semantic duplicates.

Prefer updating existing pages. When uncertain, create a TODO proposal instead of many new pages.

## Context Loading Rules

Use the smallest context that can complete the task correctly.

Search order:

1. `wiki/index.md`
2. Frontmatter `summary`
3. Relevant area subdirectory
4. Type root directory
5. `raw/` only when needed

Default context budget:

- Up to 3 directly relevant wiki pages
- Up to 1 source page
- Up to 1 overview page

Do not scan the whole wiki unless the user explicitly requests it.

Use `raw/` only when:

- Ingesting a specified raw file
- Wiki coverage is missing
- Verification against the original source is required
- The user explicitly asks to inspect raw material

Otherwise, prefer `wiki/`.

## Index Rules

Maintain `wiki/index.md`.

When creating or significantly updating a page:

- Add it if missing.
- Update the summary if stale.
- Keep entries organized by area.
- Keep index entries short but useful.

Example:

```markdown
## Java / JVM

- [[concepts/java/jvm/概念_JVM内存区域]] — JVM 运行时内存结构，是理解 GC、对象分配和性能调优的基础
- [[concepts/java/jvm/概念_类加载机制]] — 描述 class 文件如何被加载、验证、准备、解析和初始化
```

## Log Rules

Record all wiki changes in `wiki/log.md`.

Format:

```markdown
## [YYYY-MM-DD] action | description
```

Examples:

```markdown
## [2026-04-08] ingest | raw/karpathy-llm-wiki.md → wiki/sources/ai/来源_Karpathy_LLM_Wiki.md
## [2026-04-08] update | wiki/concepts/ai/agent/概念_LLM_Wiki.md（补充编译型知识库定义）
## [2026-04-08] create | wiki/overview/ai/主题_LLM知识库设计_综述.md
## [2026-04-08] merge | 概念_A + 概念_B → 概念_A
```

Use the current local date.

## Quality Gate

Before finishing any Ingest, Update, or Synthesis task, check:

1. Would this note still be useful one month later?
2. Can future questions be answered from this page without rereading raw material?
3. Does the page explain relationships, not just list facts?
4. Are important concepts defined, not merely named?
5. Are links meaningful and contextual?
6. Is the page placed in the right domain area?
7. Is the page connected to at least two relevant internal pages when possible?
8. Is uncertainty clearly marked?

If the answer is no, improve the page before finishing.

## Style

- Write wiki content primarily in Chinese unless the source terminology is better preserved in English.
- Use clear Markdown headings and short sections.
- Prefer stable structure over decorative formatting.
- Use tags sparingly. Prefer directory structure for domain classification.
- Good tags: `java`, `jvm`, `gc`, `ai-agent`, `knowledge-management`.
- Avoid vague tags such as `important`, `todo`, `note`, `misc`.
- Do not create huge unstructured pages.
- Do not create many tiny shallow pages.

## Completion Reports

After making changes, return a concise report.

For Query:

```text
回答：
- ...

写回：
- 已更新：wiki/concepts/...
- 原因：这个解释可复用
```

For Ingest:

```text
完成：
- 新建：wiki/sources/...
- 新建/更新：wiki/concepts/...
- 更新：wiki/index.md
- 更新：wiki/log.md

后续建议：
- ...
```

For Update:

```text
完成：
- 更新：wiki/concepts/...
- 变更点：
  - ...
```

For Lint:

```text
发现问题：
1. ...
2. ...

建议：
- ...
```

Keep chat reports short. The useful long-form content should live in wiki files, not only in the conversation.

## Forbidden Behavior

Do not:

- Modify `raw/`.
- Touch files outside `wiki/` unless explicitly asked.
- Create duplicate pages.
- Create many shallow pages.
- Leave important concepts only as names.
- Generate pages without checking existing pages.
- Use naked wikilinks.
- Rewrite unrelated pages.
- Fabricate sources.
- Hide uncertainty.
- Create overview pages from one weak source unless explicitly marked as draft.
- Perform broad restructuring without user confirmation.
