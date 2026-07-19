---
type: map
area: ecommerce
scope: project
project: [mall-swarm]
summary: "mall-swarm 项目工作区入口；承接项目专属的架构、决策、实验和既有历史分析。"
sources:
  - "[[30-sources/repositories/mall-swarm/来源_mall-swarm_项目入口与模块地图]]"
  - "[[30-sources/repositories/mall-swarm/来源_mall-swarm_项目源码]]"
aliases: ["mall-swarm 项目地图"]
status: evolving
confidence: 0.9
created: "2026-07-19 00:00:00"
updated: "2026-07-19 00:00:00"
---

# 项目：mall-swarm

## 项目入口

- [[20-projects/mall-swarm/项目_mall-swarm]] — 既有项目实体与来源入口。
- [[20-projects/mall-swarm/architecture/主题_mall-swarm_架构全景_综述]] — 既有架构全景。
- [[30-sources/repositories/mall-swarm/来源_mall-swarm_项目源码]] — 既有源码范围与证据边界。

## 后续内容如何放置

- 架构事实、模块地图：`architecture/`
- 明确取舍和决定：`decisions/`，采用 `ADR_YYYYMMDD_主题.md`
- 验证、假设、性能或运行记录：`experiments/`
- 可独立复用的结论：写入 [[10-domains/ecommerce/_index]] 并从这里链接。

## 工作边界

本目录可以引用 `raw/` 中的项目材料，但不得修改原始源码或其他原始资料。所有项目性新页面使用 `scope: project` 和 `project: [mall-swarm]`。
