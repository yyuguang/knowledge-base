---
type: concept
tags:
  - java
  - jvm
  - compilation
  - jit
  - optimization
summary: "JVM 运行时将热点代码编译为本地机器码，并应用多种优化技术提升性能"
sources:
  - "[[30-sources/books/深入理解Java虚拟机]]"
  - "raw/books/深入理解Java虚拟机：JVM高级特性与最佳实践（第3版）周志明.pdf"
aliases:
  - 即时编译
  - JIT
  - 编译器优化
status: evolving
confidence: 0.9
updated: "2026-04-28 10:00:00"
---

# 概念_JIT编译优化技术

## 定义

JIT（Just-In-Time）编译是 JVM 在运行时将热点代码（Hot Spot）编译为本地机器码的技术，是 Java 实现"一次编写，到处运行"与高性能的关键平衡点。

## 编译器架构

- **解释器与编译器共存**：启动快 + 热点代码编译后高效运行
- **Client Compiler（C1）**：轻量级，更快启动
- **Server Compiler（C2）**：重量级，更激进优化
- **分层编译**（JDK 7+）：C1 → C2，平衡启动与峰值性能

### 分层编译等级（Tiered Compilation Levels）

JDK 7+ 引入分层编译后，编译等级 0-4 的演进路径：

| 等级 | 名称 | 说明 |
|------|------|------|
| **Level 0** | Interpreter | 纯解释执行，收集方法调用次数与回边次数 |
| **Level 1** | C1 (no profiling) | 简单 C1 编译，不带统计信息（不收集 profiling data） |
| **Level 2** | C1 (limited profiling) | C1 编译 + 轻量统计，仅统计调用/回边次数 |
| **Level 3** | C1 (full profiling) | C1 编译 + 完整统计，收集类型、分支等用于 C2 优化决策 |
| **Level 4** | C2 | 充分利用 profiling data 做最激进优化 |

典型路径：0 → 3 → 4（大多数方法），或 0 → 2 → 3 → 1（不常用方法退优化，deoptimization）。

路径中关键 JVM 参数：

```
-XX:+TieredCompilation          # 默认开启（JDK 8+）
-XX:TieredStopAtLevel=1         # 只到 C1，调试用
-XX:TieredStopAtLevel=4         # 只到 C2
-XX:-TieredCompilation          # 关闭分层编译，回退到旧的 C1/C2 二选一模式
```

## 热点检测

- **方法计数器**：检测方法调用次数 → 标准编译
- **回边计数器**：检测循环执行次数 → 栈上替换（OSR）
- 超过阈值（-XX:CompileThreshold）触发编译

### -XX:+PrintCompilation 输出解读

```bash
# 启动一个简单应用观察编译行为
java -XX:+PrintCompilation -XX:+UnlockDiagnosticVMOptions \
     -XX:+PrintInlining HotLoopDemo
```

输出示例（带注解）：

```
---   n   java.lang.System::arraycopy (static)   @ 5   (native)
# n = native 方法，不能用 JIT 编译，只能通过 JNI 调用

     64   1       3       java.lang.String::charAt (29 bytes)
#     ^   ^       ^       ^
#     |   |       |       +-- 方法名和字节码大小
#     |   |       +-- 编译等级 (3 = C1 full profiling)
#     |   +-- 编译任务 ID（自增序号）
#     +-- 编译完成时间（ms since JVM start）

     65   2       3       java.lang.String::hashCode (55 bytes)
     66   3       4       java.lang.String::equals (88 bytes)
# 编译等级 4 = C2，说明 equals 热度足够高直接升级到 C2

     69   1       3       java.lang.String::charAt (29 bytes)   made not entrant
# made not entrant = 旧编译版本被废弃（通常是 C2 编译了更好的版本）
```

```java
/**
 * 热点循环演示：触发 OSR（On-Stack Replacement）
 *
 * 运行时加上参数查看编译行为：
 *   java -XX:+PrintCompilation -XX:+UnlockDiagnosticVMOptions \
 *        -XX:+PrintInlining HotLoopDemo
 */
public class HotLoopDemo {
    public static void main(String[] args) {
        int sum = 0;
        // 循环执行足够多次触发回边计数器，触发 OSR 编译
        for (int i = 0; i < 10_000_000; i++) {
            sum += compute(i);
        }
        System.out.println("Sum: " + sum);
    }

    private static int compute(int x) {
        // 热点方法，会被内联到调用处
        return (x * 37 + 17) % 997;
    }
}
```

## 核心优化技术

- **方法内联**：优化之母，消除调用开销，为其他优化奠定基础。虚方法通过 CHA + 守护内联 + 内联缓存解决
- **逃逸分析**：判断对象作用域，衍生优化（栈上分配 + 标量替换 + 同步消除）
- 公共子表达式消除 / 数组边界检查消除 / 复写传播 / 无用代码消除 / 冗余访问消除

### 逃逸分析演示

```java
/**
 * 逃逸分析 + 标量替换演示
 *
 * JVM 参数：
 *   -XX:+UnlockDiagnosticVMOptions
 *   -XX:+PrintEscapeAnalysis          # 打印逃逸分析结果
 *   -XX:+EliminateAllocations         # 开启标量替换（默认开启）
 *   -XX:+PrintGCDetails               # 观察 GC 次数验证对象是否分配在堆上
 *
 * 对比关闭逃逸分析：
 *   -XX:-DoEscapeAnalysis             # 关闭逃逸分析 → GC 频繁
 */
public class EscapeAnalysisDemo {

    static class Point {
        final int x;
        final int y;
        Point(int x, int y) { this.x = x; this.y = y; }
    }

    /**
     * alloc() 中创建的 Point 对象没有逃逸出方法：
     *   - 不作为返回值
     *   - 不赋值给静态/成员变量
     *   - 不传递给外部方法
     * → JIT 可将 p.x 和 p.y 替换为两个局部 int 变量（标量替换）
     * → Point 对象本身不需要在堆上分配
     */
    private static int alloc() {
        Point p = new Point(1, 2); // Point 不逃逸 → 标量替换 → 等价于 int x=1; int y=2;
        return p.x + p.y;
    }

    public static void main(String[] args) {
        long start = System.nanoTime();
        for (int i = 0; i < 100_000_000; i++) {
            alloc(); // 如果标量替换生效，这里不会产生任何堆分配
        }
        long elapsed = System.nanoTime() - start;
        System.out.println("Elapsed: " + elapsed / 1_000_000 + " ms");

        // 建议分别用以下参数对比运行时间：
        // 1) 默认（逃逸分析开启）: java EscapeAnalysisDemo
        // 2) 关闭逃逸分析:       java -XX:-DoEscapeAnalysis EscapeAnalysisDemo
        // 关闭后 GC 次数大幅增加，执行时间可能翻倍
    }
}
```

### 方法内联演示：-XX:+PrintInlining

```java
/**
 * 方法内联演示
 *
 * JVM 参数：
 *   java -XX:+UnlockDiagnosticVMOptions \
 *        -XX:+PrintInlining \
 *        -XX:+PrintCompilation \
 *        InliningDemo
 *
 * 输出示例解读：
 *   @ 1   java.lang.Math::max (11 bytes)   inline (hot)
 *   # inline (hot) = 方法被内联（热点）
 *
 *   @ 5   AbstractClass::virtualMethod (24 bytes)   virtual call
 *   # virtual call = 调用点太多，内联失败
 *
 *   @ 12   TooBigMethod::hugeMethod (326 bytes)   hot method too big
 *   # 方法字节码过大，超过内联阈值（-XX:MaxInlineSize=35，-XX:FreqInlineSize=325）
 */
public class InliningDemo {

    // 简单方法 → 几乎必定内联
    private static int square(int x) {
        return x * x;
    }

    // 虚方法 → 内联取决于 CHA 能否确认单一实现
    interface Calculator {
        int calc(int x);
    }

    static class FastCalc implements Calculator {
        @Override
        public int calc(int x) { return x * x + x; }
    }

    public static void main(String[] args) {
        int sum = 0;
        Calculator calc = new FastCalc(); // CHA 可能确认只有 FastCalc 一个实现

        // 预热
        for (int i = 0; i < 20_000; i++) {
            sum += square(i);       // 内联后等价于 sum += i * i
            sum += calc.calc(i);   // 单态调用 → 内联缓存易命中
        }
        System.out.println("Warmup sum: " + sum);

        // 正式执行
        long start = System.nanoTime();
        sum = 0;
        for (int i = 0; i < 100_000_000; i++) {
            sum += square(i);
        }
        long elapsed = System.nanoTime() - start;
        System.out.println("Sum: " + sum + ", elapsed: " + elapsed / 1_000_000 + " ms");
    }
}
```

### JMH 预热基准测试

```java
import org.openjdk.jmh.annotations.*;
import org.openjdk.jmh.runner.Runner;
import org.openjdk.jmh.runner.options.Options;
import org.openjdk.jmh.runner.options.OptionsBuilder;

import java.util.concurrent.TimeUnit;

/**
 * JMH 基准测试：展示 JIT 预热效应
 *
 * 运行方式：
 *   mvn clean package
 *   java -jar target/benchmarks.jar JmhWarmupDemo
 *
 * 关键观察：
 *   前几个 fork 的吞吐量远低于稳态 → JIT 编译 + profiling 过程中
 *   稳态后吞吐量提高数倍 → C2 编译完毕 + 所有优化生效
 */
@BenchmarkMode(Mode.Throughput)
@OutputTimeUnit(TimeUnit.MILLISECONDS)
@State(Scope.Thread)
@Warmup(iterations = 5, time = 1, timeUnit = TimeUnit.SECONDS)   // 5 轮预热
@Measurement(iterations = 10, time = 1, timeUnit = TimeUnit.SECONDS) // 10 轮测量
@Fork(1)
public class JmhWarmupDemo {

    private int[] data = new int[8192];

    @Setup
    public void setup() {
        for (int i = 0; i < data.length; i++) {
            data[i] = i;
        }
    }

    /**
     * 简单循环累加 → 经过预热后：
     *   1. compute() 被内联
     *   2. 循环展开
     *   3. 边界检查消除
     *   4. 数据预取
     */
    @Benchmark
    public int sumCompute() {
        int sum = 0;
        for (int value : data) {
            sum += compute(value);
        }
        return sum;
    }

    private int compute(int x) {
        return (x * 37 + 17) & 0xFF;
    }

    // JMH main
    public static void main(String[] args) throws Exception {
        Options opt = new OptionsBuilder()
                .include(JmhWarmupDemo.class.getSimpleName())
                .build();
        new Runner(opt).run();
    }
}
```

### 关闭逃逸分析对比

```java
/**
 * 关闭优化选项的性能对比脚本
 *
 * 依次运行以下命令，观察耗时与 GC 行为差异：
 *
 * # 1) 默认（所有优化开启）
 * java -Xms128m -Xmx128m EscapeAnalysisDemo
 *
 * # 2) 关闭逃逸分析
 * java -Xms128m -Xmx128m -XX:-DoEscapeAnalysis EscapeAnalysisDemo
 *
 * # 3) 关闭标量替换（保留逃逸分析但不做替换）
 * java -Xms128m -Xmx128m -XX:-EliminateAllocations EscapeAnalysisDemo
 *
 * # 4) 关闭分层编译
 * java -Xms128m -Xmx128m -XX:-TieredCompilation EscapeAnalysisDemo
 *
 * # 5) 纯解释模式（基线，极慢）
 * java -Xms128m -Xmx128m -Xint EscapeAnalysisDemo
 * ```
 */
```

## 提前编译（AOT）

- **Jaotc**：JDK 9 引入，预编译 java.base 模块为 .so
- **Substrate VM**：Graal 生态的静态 AOT，可编译为原生可执行文件

## Graal 编译器

用 Java 编写的新一代 JIT，支持 JVM CI 接口和 Graph IR，可脱离 HotSpot 独立运行。

## 在本知识库中的应用

- 理解 JVM 预热（Warm-up）：为何压测需要先跑几轮再统计
- 调优参数：`-XX:CompileThreshold`、`-XX:-TieredCompilation`
- [[10-domains/java/jvm/概念_字节码执行引擎]] 是 JIT 的输入，JIT 是执行引擎的加速器

## 关联

- [[10-domains/java/jvm/概念_字节码执行引擎]] — 字节码是 JIT 的输入，JIT 将热点字节码编译为本地机器码以加速执行
- [[10-domains/java/jvm/概念_Java语法糖]] — 语法糖在前端编译期处理完成，JIT 在后端编译期做更高的优化
- [[10-domains/java/jvm/概念_Java内存模型JMM]] — volatile 的内存屏障限制了 JIT 的重排序优化空间
