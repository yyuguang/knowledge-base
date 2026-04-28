---
type: concept
tags:
  - java
  - jvm
  - concurrency
summary: "定义多线程环境下变量访问规则，是 Java 并发编程的底层理论基础"
sources:
  - "raw/books/深入理解Java虚拟机：JVM高级特性与最佳实践（第3版）周志明.pdf"
aliases:
  - JMM
  - Java Memory Model
  - 内存模型
status: evolving
confidence: 0.9
updated: "2026-04-27 22:30:00"
---

# 概念_Java内存模型JMM

## 定义

Java 内存模型（JMM）用于屏蔽不同硬件和操作系统的内存访问差异，定义程序中各种变量的访问规则。它规定了主内存与工作内存的交互协议，以及 volatile、synchronized 等关键字的底层语义。

## 主内存 vs 工作内存

- **主内存**：所有变量存储位置，对应堆中实例数据部分
- **工作内存**：线程私有，保存变量副本，对应虚拟机栈部分
- 线程间变量传递必须经过主内存

## 8 种原子操作

| 操作 | 范围 | 说明 |
|------|------|------|
| lock / unlock | 主内存 | 标识线程独占 / 释放 |
| read / load | 主→工作 | 从主内存传输到工作内存 |
| use / assign | 工作内存 | 执行引擎使用 / 赋值 |
| store / write | 工作→主 | 回写到主内存 |

## volatile 语义

1. **可见性**：对 volatile 写立即对其他线程可见
2. **禁止指令重排序**：通过内存屏障（`lock addl $0x0,(%rsp)`）实现
3. **不保证原子性**：`race++` 非原子（getstatic + iconst + iadd + putstatic 四条指令）

## 三个特性

- **原子性**：lock/unlock（底层）+ CAS + synchronized 保证
- **可见性**：volatile + synchronized + final 保证
- **有序性**：volatile（禁止重排）+ synchronized（同一锁串行）保证

## 先行发生原则（Happens-Before，8 条规则）

程序次序 → 管程锁定 → volatile → 线程启动 → 线程终止 → 线程中断 → 对象终结 → 传递性

## 在本知识库中的应用

- 诊断并发 Bug：用 Happens-Before 规则判断是否存在数据竞争
- 正确使用 volatile：状态标志位（`shutdownRequested`）和 DCL 单例
- [[concepts/java/概念_Java线程安全与锁优化]] 是 JMM 规则的具体实现

## 关联

- [[concepts/java/概念_Java线程安全与锁优化]] — JMM 定义可见性/原子性/有序性规则，本页是实现这些规则的具体手段（synchronized/CAS/锁优化）
- [[concepts/java/概念_JVM内存区域]] — JMM 的主内存对应堆中实例数据，工作内存对应虚拟机栈，两者共享同一物理内存
- [[concepts/java/概念_JIT编译优化技术]] — volatile 的内存屏障禁止指令重排，限制了 JIT 编译器的优化空间
