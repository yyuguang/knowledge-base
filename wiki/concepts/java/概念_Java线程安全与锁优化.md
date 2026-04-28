---
type: concept
tags:
  - java
  - jvm
  - concurrency
  - lock
summary: "Java 并发编程中的线程安全等级、实现方法及 HotSpot 锁优化技术"
sources:
  - "raw/books/深入理解Java虚拟机：JVM高级特性与最佳实践（第3版）周志明.pdf"
aliases:
  - 线程安全
  - 锁优化
  - 偏向锁
status: evolving
confidence: 0.9
updated: "2026-04-27 22:30:00"
---

# 概念_Java线程安全与锁优化

## 定义

线程安全指当多个线程同时访问一个对象时，如果不考虑这些线程在运行时环境下的调度和交替执行，也不需要进行额外的同步，调用这个对象的行为都能获得正确的结果。

## 线程安全等级

| 等级 | 说明 | 例 |
|------|------|-----|
| 不可变 | final 修饰，始终安全 | String / Integer |
| 绝对线程安全 | 不管运行时环境如何，调用者无需额外措施 | Vector（实际上未完全达到） |
| 相对线程安全 | 单独操作安全，复合操作需外部同步 | Vector / synchronizedList |
| 线程兼容 | 本身不安全，通过同步手段安全使用 | ArrayList / HashMap |
| 线程对立 | 无论是否同步都无法安全使用 | Thread.suspend() / resume() |

## 实现方法

1. **互斥同步**：synchronized（monitorenter/monitorexit）+ ReentrantLock（Condition/AQS）
2. **非阻塞同步**：CAS（Compare-And-Swap）+ Atomic 类 + Unsafe
3. **无同步方案**：ThreadLocal + 可重入代码

## HotSpot 锁优化

锁膨胀路径（不可逆）：**无锁 → 偏向锁 → 轻量级锁 → 重量级锁**

| 锁技术 | 原理 | 场景 |
|--------|------|------|
| **偏向锁** | Mark Word 记录线程 ID，无竞争时省略 CAS | 单线程反复获取同一锁 |
| **轻量级锁** | CAS 替换 Mark Word，避免 OS 互斥 | 少量线程交替执行 |
| **自旋锁** | 忙等待避免线程切换 | 锁持有时间很短 |
| **自适应自旋** | 根据历史动态调整自旋次数 | 同上，更智能 |
| **锁消除** | 逃逸分析 + 同步消除 | 线程私有对象 |
| **锁粗化** | 合并相邻同步块 | 反复加解锁同一对象 |

## 在本知识库中的应用

- 选择同步策略：读多写少用 volatile + CAS，竞争激烈用 synchronized
- 性能调优：观察锁膨胀情况，避免不必要的 synchronized
- [[concepts/java/概念_Java内存模型JMM]] 提供理论基础，本页是实践手段

## 关联

- [[concepts/java/概念_Java内存模型JMM]] — JMM 定义原子性/可见性/有序性规则，本页的 synchronized 和 CAS 是实现这些规则的具体手段
- [[concepts/java/概念_JIT编译优化技术]] — 锁消除依赖逃逸分析判断对象是否线程私有，偏向锁的批量撤销涉及 JIT 编译
- [[entities/java/项目_HotSpot_VM]] — 锁膨胀路径（偏向锁→轻量级锁→重量级锁）是 HotSpot 独有的实现细节
