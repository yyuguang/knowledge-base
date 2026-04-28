## [2026-04-27] ingest | raw/books/深入理解Java虚拟机 → wiki/sources/深入理解Java虚拟机.md (+ 10 concepts + 2 entities + index + log)

### created
- sources/深入理解Java虚拟机.md
- concepts/概念_JVM内存区域.md
- concepts/概念_垃圾收集算法.md
- concepts/概念_垃圾收集器.md
- concepts/概念_类加载机制.md
- concepts/概念_Class文件结构.md
- concepts/概念_字节码执行引擎.md
- concepts/概念_Java内存模型JMM.md
- concepts/概念_JIT编译优化技术.md
- concepts/概念_Java语法糖.md
- concepts/概念_Java线程安全与锁优化.md
- entities/人物_周志明.md
- entities/项目_HotSpot_VM.md

### updated
- index.md（新增索引条目）
- log.md（本次记录）

---

## [2026-04-27] fix | 规范修正：重命名+补frontmatter+补结构

全部页面按 TheSchema 规范修正：
- 命名加前缀（概念_/人物_/项目_）
- frontmatter 补 aliases / status / confidence / updated(time)
- 概念页补"在本知识库中的应用示例"
- 实体页补"事件/计划/实验链接"
- 内部链接全部更新为新路径

---

## [2026-04-27] fix | 关联加上下文说明

所有页面的关联链接增加 why 说明，解释每条链接的关联原因

---

## [2026-04-27] restructure | 迁移 JVM 内容至 java/ 子目录 + 更新 TheSchema.md

将 13 个 wiki 页面从扁平目录迁移至领域子目录：
- `sources/java/`、`concepts/java/`、`entities/java/`
- 全部 wikilink 更新为新路径
- TheSchema.md 新增「领域子目录」说明
- README.md 目录结构同步更新
