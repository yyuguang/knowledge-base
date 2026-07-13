---
type: concept
tags:
  - java
  - jvm
  - concurrency
  - jmm
  - memory-model
summary: "定义多线程环境下变量访问规则，是 Java 并发编程的底层理论基础"
sources:
  - "raw/books/深入理解Java虚拟机：JVM高级特性与最佳实践（第3版）周志明.pdf"
aliases:
  - JMM
  - Java Memory Model
  - 内存模型
status: evolving
confidence: 0.9
updated: "2026-04-28 10:00:00"
---

# 概念_Java内存模型JMM

## 定义

Java 内存模型（JMM）用于屏蔽不同硬件和操作系统的内存访问差异，定义程序中各种变量的访问规则。它规定了主内存与工作内存的交互协议，以及 volatile、synchronized 等关键字的底层语义。

## 主内存 vs 工作内存

- **主内存**：所有变量存储位置，对应堆中实例数据部分
- **工作内存**：线程私有，保存变量副本，对应虚拟机栈部分
- 线程间变量传递必须经过主内存

### volatile 可见性演示

下面的代码演示了在没有 volatile 修饰时，一个线程对共享变量的修改可能对另一个线程永远不可见（线程可能在寄存器或缓存中读取旧值，形成死循环）：

```java
/**
 * volatile 可见性演示：不加 volatile → 线程可能永远看不到修改
 *
 * JVM 参数（可选）：
 *   -Xint             纯解释模式，降低 JIT 干扰
 *   -XX:+UnlockDiagnosticVMOptions -XX:+PrintAssembly  查看实际内存屏障指令
 */
public class VolatileVisibilityDemo {

    // 对比：加 volatile 后线程-1 能立即看到修改
    private static /* volatile */ boolean flag = false;

    public static void main(String[] args) throws Exception {
        Thread worker = new Thread(() -> {
            int spinCount = 0;
            while (!flag) {
                spinCount++;
                // 注：若循环体内有 synchronized / System.out.println，
                // 会因隐式同步刷新工作内存，使不可见性失效
            }
            System.out.println("[worker] detected flag=true after " + spinCount + " spins");
        }, "worker");

        worker.start();
        Thread.sleep(1000);               // 确保 worker 已开始自旋

        flag = true;                      // main 线程写入
        System.out.println("[main] set flag=true");
        worker.join();
        System.out.println("[main] done — worker exited");
    }
}
```

> **关键点**：不加 volatile 时，worker 线程可能在 CPU cache/寄存器中持有 flag 的旧值，永远看不到 main 的写入。加上 volatile 后，写操作后会执行 `lock addl $0x0,(%rsp)`（x86 下空操作的 lock 前缀指令），该指令会：
> 1. 将当前处理器缓存行写回主内存
> 2. 使其他 CPU 中该缓存行失效（MESI 协议）
> 3. 提供 Store-Load 屏障语义，禁止指令重排序

### DCL 单例：指令重排序导致的问题

```java
/**
 * DCL（Double-Checked Locking）单例 — 错误版本（无 volatile）
 *
 * 问题：instance = new Singleton() 在字节码层面不是原子的：
 *   1. new            # 分配内存
 *   2. dup
 *   3. invokespecial  # 调用构造器 <init>
 *   4. astore         # 将引用赋给 instance
 *
 * JIT/CPU 可能将步骤 3 和 4 重排序：
 *   分配内存 → astore（instance 非 null） → 调用构造器
 * 此时另一个线程在第一个 if(instance==null) 鉴读到非 null，
 * 返回了一个尚未初始化完毕的对象 → 部分构造问题。
 */
public class DclSingletonBroken {
    private static DclSingletonBroken instance; // 缺少 volatile

    private DclSingletonBroken() {
        // 构造器内有耗时逻辑时重排序危害更明显
        System.out.println("DclSingletonBroken init done");
    }

    public static DclSingletonBroken getInstance() {
        if (instance == null) {                     // 第一次检查
            synchronized (DclSingletonBroken.class) {
                if (instance == null) {             // 第二次检查
                    instance = new DclSingletonBroken(); // 危险！
                }
            }
        }
        return instance;
    }
}
```

### DCL 单例：正确版本（volatile）

```java
/**
 * DCL 单例 — 正确版本（JDK 5+ volatile 语义）
 *
 * volatile 在这里的作用：
 *   1. 保证 instance 写入的可见性
 *   2. 禁止 instance = new Xxx() 的指令重排序
 *      （JDK 5 增强 volatile 内存语义，提供 StoreLoad 屏障）
 */
public class DclSingletonFixed {
    private static volatile DclSingletonFixed instance;

    private DclSingletonFixed() {}

    public static DclSingletonFixed getInstance() {
        if (instance == null) {
            synchronized (DclSingletonFixed.class) {
                if (instance == null) {
                    instance = new DclSingletonFixed(); // volatile 保证安全
                }
            }
        }
        return instance;
    }
}
```

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

### volatile 自增竞态演示

```java
import java.util.concurrent.CountDownLatch;

/**
 * volatile 不保证原子性：race++ 实际包含 4 条字节码指令
 *
 * javap -c -p 查看：
 *   getstatic race   → 读取
 *   iconst_1         → 入栈常量 1
 *   iadd             → 相加
 *   putstatic race   → 写回
 * 多线程交错执行 → 结果 < 保证数
 */
public class VolatileAtomicityDemo {
    private static volatile int race = 0;
    private static final int THREADS = 20;
    private static final int ITERATIONS = 10000;

    public static void main(String[] args) throws Exception {
        CountDownLatch latch = new CountDownLatch(THREADS);
        for (int i = 0; i < THREADS; i++) {
            new Thread(() -> {
                for (int j = 0; j < ITERATIONS; j++) {
                    race++; // 非原子，预期结果 < THREADS * ITERATIONS
                }
                latch.countDown();
            }).start();
        }
        latch.await();
        System.out.println("Expected: " + (THREADS * ITERATIONS) + ", Actual: " + race);
    }
}
```

## 三个特性

- **原子性**：lock/unlock（底层）+ CAS + synchronized 保证
- **可见性**：volatile + synchronized + final 保证
- **有序性**：volatile（禁止重排）+ synchronized（同一锁串行）保证

### CAS 原子操作：AtomicInteger

```java
import java.util.concurrent.atomic.AtomicInteger;

/**
 * CAS (Compare-And-Set) 是 CPU 级的原子指令（x86: cmpxchg）
 * CAS 是 java.util.concurrent 原子类的基石
 *
 * CAS 的 ABA 问题：
 *   线程-1 读 V=A → 线程-2 V=A→B→A → 线程-1 CAS 成功但中间状态已变
 *   解决方案：AtomicStampedReference / AtomicMarkableReference
 */
public class CasAtomicDemo {
    private static final AtomicInteger counter = new AtomicInteger(0);

    public static void main(String[] args) throws Exception {
        // 基本原子操作
        System.out.println("incrementAndGet: " + counter.incrementAndGet()); // +1 返回新值
        System.out.println("getAndIncrement: " + counter.getAndIncrement()); // 返回旧值后 +1
        System.out.println("addAndGet(5): " + counter.addAndGet(5));         // +5 返回新值

        // CAS 自旋循环模式（compareAndSet 循环）
        int oldValue, newValue;
        do {
            oldValue = counter.get();
            newValue = oldValue * 2;
        } while (!counter.compareAndSet(oldValue, newValue));
        System.out.println("After CAS-spin (doubled): " + counter.get());

        // 函数式更新（JDK 8+）
        counter.updateAndGet(v -> v + 10);
        System.out.println("After updateAndGet: " + counter.get());
    }
}
```

### Cache Line Padding：避免伪共享

```java
import java.util.concurrent.atomic.AtomicLong;

/**
 * 伪共享（False Sharing）：
 *   两个独立变量 A、B 落在同一缓存行（通常 64 字节），
 *   线程-1 修改 A，线程-2 修改 B → 互相使对方缓存行无效 → 性能暴跌
 *
 * JDK 8+ 方案1：@Contended（需要 JVM 参数 -XX:-RestrictContended）
 * JDK 6-7 方案2：手动填充 64 字节
 */
public class FalseSharingDemo {

    // 方案1: @Contended（JDK 8，需要在启动时加 -XX:-RestrictContended）
    // sun.misc.Contended 或 jdk.internal.vm.annotation.Contended（JDK 9+）
    @sun.misc.Contended
    private static final class PaddedCell {
        volatile long value;
    }

    // 方案2: 手动填充 64 字节（JDK 7 及之前）
    private static final class ManualPaddedCell {
        volatile long value;
        long p1, p2, p3, p4, p5, p6, p7; // 7 * 8 = 56 字节
        // value(8) + p1-p7(56) = 64 字节，占满一个缓存行
    }

    // 无填充的普通版本（对比基准）
    private static final class PlainCell {
        volatile long value;
    }

    private static final int THREAD_COUNT = 4;
    private static final long ITERATIONS = 100_000_000L;

    public static void main(String[] args) throws Exception {
        // 运行方式：
        //   java -XX:-RestrictContended FalseSharingDemo
        System.out.println("Running false sharing benchmark...");

        PlainCell[] plain = { new PlainCell(), new PlainCell() };
        long plainTime = runBenchmark(plain);
        System.out.println("Plain cells (no padding):      " + plainTime + " ms");

        ManualPaddedCell[] padded = { new ManualPaddedCell(), new ManualPaddedCell() };
        long paddedTime = runBenchmarkPadded(padded);
        System.out.println("Padded cells (manual 64 bytes): " + paddedTime + " ms");
    }

    private static long runBenchmark(PlainCell[] cells) throws Exception {
        Thread[] threads = new Thread[THREAD_COUNT];
        long start = System.currentTimeMillis();
        for (int i = 0; i < THREAD_COUNT; i++) {
            final int idx = i % 2; // 0 or 1, 均匀分布到两个槽位
            threads[i] = new Thread(() -> {
                for (long l = 0; l < ITERATIONS; l++) {
                    cells[idx].value = l;
                }
            });
            threads[i].start();
        }
        for (Thread t : threads) t.join();
        return System.currentTimeMillis() - start;
    }

    private static long runBenchmarkPadded(ManualPaddedCell[] cells) throws Exception {
        Thread[] threads = new Thread[THREAD_COUNT];
        long start = System.currentTimeMillis();
        for (int i = 0; i < THREAD_COUNT; i++) {
            final int idx = i % 2;
            threads[i] = new Thread(() -> {
                for (long l = 0; l < ITERATIONS; l++) {
                    cells[idx].value = l;
                }
            });
            threads[i].start();
        }
        for (Thread t : threads) t.join();
        return System.currentTimeMillis() - start;
    }
}
```

## 先行发生原则（Happens-Before，8 条规则）

程序次序 → 管程锁定 → volatile → 线程启动 → 线程终止 → 线程中断 → 对象终结 → 传递性

### Happens-Before 传递性演示

```java
/**
 * Happens-Before 传递性演示：
 *
 * 规则：
 *   ① 程序次序：A happens-before B（同一线程内按代码顺序）
 *   ② volatile：对 volatile 变量的写 happens-before 后续对同一变量的读
 *   ③ 传递性：A hb B 且 B hb C → A hb C
 *
 * 本示例：
 *   T1: x=1 → volatileFlag=true (①)
 *   T2: volatileFlag==true → y=x (①)
 *   T1 写 volatileFlag hb T2 读 volatileFlag (②)
 *   → T1 x=1 hb T2 y=x (③传递性) → T2 读取到 x 的正确值
 */
public class HappensBeforeTransitivity {

    private static int x = 0;
    private static volatile boolean volatileFlag = false;

    public static void main(String[] args) throws Exception {
        Thread writer = new Thread(() -> {
            x = 1;                    // A
            volatileFlag = true;      // B: volatile 写
        }, "writer");

        Thread reader = new Thread(() -> {
            while (!volatileFlag) {}  // C: volatile 读（自旋等待）
            int y = x;                // D: 一定能读到 x=1（传递性保证）
            System.out.println("y = " + y + " (expected: 1)");
        }, "reader");

        reader.start();
        Thread.sleep(100);
        writer.start();

        writer.join();
        reader.join();
    }
}
```

## 在本知识库中的应用

- 诊断并发 Bug：用 Happens-Before 规则判断是否存在数据竞争
- 正确使用 volatile：状态标志位（`shutdownRequested`）和 DCL 单例
- [[concepts/java/jvm/概念_Java线程安全与锁优化]] 是 JMM 规则的具体实现

## 关联

- [[concepts/java/jvm/概念_Java线程安全与锁优化]] — JMM 定义可见性/原子性/有序性规则，本页是实现这些规则的具体手段（synchronized/CAS/锁优化）
- [[concepts/java/jvm/概念_JVM内存区域]] — JMM 的主内存对应堆中实例数据，工作内存对应虚拟机栈，两者共享同一物理内存
- [[concepts/java/jvm/概念_JIT编译优化技术]] — volatile 的内存屏障禁止指令重排，限制了 JIT 编译器的优化空间
