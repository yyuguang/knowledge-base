---
type: entity
tags:
  - java
  - jvm
  - hotspot
summary: "OracleJDK/OpenJDK 默认的 Java 虚拟机，市场占有率最高"
sources:
  - "raw/books/深入理解Java虚拟机：JVM高级特性与最佳实践（第3版）周志明.pdf"
aliases:
  - HotSpot VM
  - HotSpot
  - OpenJDK
status: stable
confidence: 0.9
updated: "2026-04-27 22:30:00"
---

# 项目_HotSpot_VM

## 基本信息

- **所属**：Oracle（原 Sun Microsystems）
- **首次发布**：1999 年（JDK 1.2），源自 Longview Technologies
- **地位**：OracleJDK / OpenJDK 默认虚拟机
- **语言**：C/C++ 实现
- **名称来源**：热点代码检测技术（Hot Spot Detection）

## 核心特性

- **JIT 编译器**：C1（Client）/ C2（Server）/ Graal，支持分层编译
- **垃圾收集器**：最全收集器家族（Serial → Parallel → CMS → G1 → ZGC / Shenandoah）
- **OopMap**：通过栈映射加速 GC Roots 枚举
- **服务性代理（SA）**：JHSDB 基于此进行进程外调试
- **模板解释器**：运行期生成字节码执行代码

## 关键事件与技术路线

- **1999**：HotSpot 成为 Sun JDK 默认虚拟机
- **2006**：Sun 开源 OpenJDK，HotSpot 成为核心
- **2010**：Oracle 收购 Sun，同时拥有 HotSpot 和 JRockit
- **2012–2017**：HotRockit 项目将 JRockit 特性（JFR/JMC/NMT）合并至 HotSpot
- **2018**：JDK 11 发布，ZGC 作为实验特性加入
- **2019**：JDK 12，Shenandoah 正式发布

## 关键参数

- `-Xms` / `-Xmx`：堆初始/最大大小
- `-XX:+UseG1GC` / `-XX:+UseZGC`：选择收集器
- `-XX:MaxGCPauseMillis`：停顿目标
- `-Xlog:gc`：GC 日志（JDK 9+）
- `-XX:+HeapDumpOnOutOfMemoryError`：自动 dump
- `-XX:+PrintFlagsFinal`：查看所有 JVM 参数

## 关联

- [[sources/java/深入理解Java虚拟机]] — 该书以 HotSpot 为主要分析对象，各章机制均以其实现为准
- [[concepts/java/概念_垃圾收集器]] — HotSpot 拥有最全的垃圾收集器家族（Serial→ZGC），每款均为 HotSpot 独有实现
- [[concepts/java/概念_JIT编译优化技术]] — HotSpot 名称源自其热点检测技术，C1/C2/Graal 编译器均为其核心组件
- [[concepts/java/概念_类加载机制]] — HotSpot 的双亲委派模型和 OopMap 加速验证是类加载的具体实现方式
