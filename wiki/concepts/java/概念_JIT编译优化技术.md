---
type: concept
tags:
  - java
  - jvm
  - compilation
  - optimization
summary: "JVM 运行时将热点代码编译为本地机器码，并应用多种优化技术提升性能"
sources:
  - "raw/books/深入理解Java虚拟机：JVM高级特性与最佳实践（第3版）周志明.pdf"
aliases:
  - 即时编译
  - JIT
  - 编译器优化
status: evolving
confidence: 0.9
updated: "2026-04-27 22:30:00"
---

# 概念_JIT编译优化技术

## 定义

JIT（Just-In-Time）编译是 JVM 在运行时将热点代码（Hot Spot）编译为本地机器码的技术，是 Java 实现"一次编写，到处运行"与高性能的关键平衡点。

## 编译器架构

- **解释器与编译器共存**：启动快 + 热点代码编译后高效运行
- **Client Compiler（C1）**：轻量级，更快启动
- **Server Compiler（C2）**：重量级，更激进优化
- **分层编译**（JDK 7+）：C1 → C2，平衡启动与峰值性能

## 热点检测

- **方法计数器**：检测方法调用次数 → 标准编译
- **回边计数器**：检测循环执行次数 → 栈上替换（OSR）
- 超过阈值（-XX:CompileThreshold）触发编译

## 核心优化技术

- **方法内联**：优化之母，消除调用开销，为其他优化奠定基础。虚方法通过 CHA + 守护内联 + 内联缓存解决
- **逃逸分析**：判断对象作用域，衍生优化（栈上分配 + 标量替换 + 同步消除）
- 公共子表达式消除 / 数组边界检查消除 / 复写传播 / 无用代码消除 / 冗余访问消除

## 提前编译（AOT）

- **Jaotc**：JDK 9 引入，预编译 java.base 模块为 .so
- **Substrate VM**：Graal 生态的静态 AOT，可编译为原生可执行文件

## Graal 编译器

用 Java 编写的新一代 JIT，支持 JVM CI 接口和 Graph IR，可脱离 HotSpot 独立运行。

## 在本知识库中的应用

- 理解 JVM 预热（Warm-up）：为何压测需要先跑几轮再统计
- 调优参数：`-XX:CompileThreshold`、`-XX:-TieredCompilation`
- [[concepts/java/概念_字节码执行引擎]] 是 JIT 的输入，JIT 是执行引擎的加速器

## 关联

- [[concepts/java/概念_字节码执行引擎]] — 字节码是 JIT 的输入，JIT 将热点字节码编译为本地机器码以加速执行
- [[concepts/java/概念_Java语法糖]] — 语法糖在前端编译期处理完成，JIT 在后端编译期做更高的优化
- [[concepts/java/概念_Java内存模型JMM]] — volatile 的内存屏障限制了 JIT 的重排序优化空间
