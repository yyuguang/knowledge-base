---
type: concept
tags:
  - java
  - jvm
  - concurrency
  - thread-safety
  - lock-optimization
summary: "Java 并发编程中的线程安全等级、实现方法及 HotSpot 锁优化技术"
sources:
  - "[[30-sources/books/深入理解Java虚拟机]]"
  - "raw/books/深入理解Java虚拟机：JVM高级特性与最佳实践（第3版）周志明.pdf"
aliases:
  - 线程安全
  - 锁优化
  - 偏向锁
status: evolving
confidence: 0.9
updated: "2026-04-28 10:00:00"
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

### synchronized 字节码验证

synchronized 在字节码层面由 `monitorenter`/`monitorexit` 指令对实现，编译器会在异常表中额外插入一条 `monitorexit` 确保锁在异常路径上释放。

```java
public class SyncBytecode {
    public synchronized void syncMethod() {
        int sum = 0;
        for (int i = 0; i < 10; i++) sum += i;
    }

    public void syncBlock() {
        synchronized (this) {
            int sum = 0;
            for (int i = 0; i < 10; i++) sum += i;
        }
    }
}
```

反编译验证：
```bash
javap -c -p SyncBytecode.class
```
```text
// syncMethod(): flags: ACC_PUBLIC, ACC_SYNCHRONIZED  ← 方法级同步，JVM 隐式处理
// 字节码中无 monitorenter/monitorexit，由 ACC_SYNCHRONIZED flag 标示

// syncBlock():
//   aload_0           # 加载 this
//   dup               # 复制引用（monitorenter 会弹出，所以需要备份）
//   monitorenter      # 获取锁（this 的 monitor）
//   ... body ...
//   aload_0
//   monitorexit       # 正常路径释放锁
//   goto 24
//   astore_1          # 异常路径入口
//   aload_0
//   monitorexit       # 异常路径释放锁（避免死锁）
//   aload_1
//   athrow
```

### ReentrantLock 公平锁 vs 非公平锁

```java
import java.util.concurrent.locks.ReentrantLock;

public class FairVsNonfair {
    public static void main(String[] args) throws Exception {
        // 公平锁：线程按排队顺序获取锁，吞吐量低
        ReentrantLock fairLock = new ReentrantLock(true);
        // 非公平锁（默认）：插队模式，吞吐量高但可能饥饿
        ReentrantLock nonfairLock = new ReentrantLock(false);

        // 对比：公平锁 tryAcquire(1) 会先检查 hasQueuedPredecessors()
        // 非公平锁直接 CAS 抢夺，失败再排队
        test(fairLock, "Fair Lock");
        test(nonfairLock, "Non-fair Lock");
    }

    static void test(ReentrantLock lock, String label) throws Exception {
        Runnable task = () -> {
            lock.lock();
            try {
                System.out.println(Thread.currentThread().getName() + " acquired " + label);
            } finally {
                lock.unlock();
            }
        };
        for (int i = 0; i < 5; i++) {
            new Thread(task, "T" + i).start();
        }
        Thread.sleep(500);
        System.out.println("---");
    }
}
/*
 * 公平锁 AQS 内部：tryAcquire 前调用 hasQueuedPredecessors() 判断是否有前驱节点
 * 非公平锁 AQS 内部：nonfairTryAcquire 直接 CAS state，不检查队列
 */
```

### CAS ABA 问题与 AtomicStampedReference

```java
import java.util.concurrent.atomic.AtomicStampedReference;

public class ABADemo {
    public static void main(String[] args) throws Exception {
        // AtomicStampedReference 通过版本号解决 CAS 的 ABA 问题
        AtomicStampedReference<String> ref = new AtomicStampedReference<>("A", 0);

        // 模拟 ABA：A → B → A
        int[] stamp = new int[1];
        String val = ref.get(stamp); // val="A" stamp=0

        // Thread 1: A → B (stamp 0 → 1)
        boolean ok1 = ref.compareAndSet("A", "B", stamp[0], stamp[0] + 1);
        System.out.println("A→B: " + ok1 + ", stamp=" + (stamp[0] + 1)); // true, stamp=1

        // Thread 2: B → A (stamp 1 → 2)
        ref.get(stamp); // 获取当前 stamp=1
        boolean ok2 = ref.compareAndSet("B", "A", stamp[0], stamp[0] + 1);
        System.out.println("B→A: " + ok2 + ", stamp=" + (stamp[0] + 1)); // true, stamp=2

        // 如果用 AtomicReference，此时另一个线程看到值仍是 "A"，无法察觉已经发生过 A→B→A
        // AtomicStampedReference 的 stamp 从 0 变到 2，版本号不一致，CAS 会失败
        System.out.println("Final: " + ref.getReference() + " stamp=" + ref.getStamp());
        // Output: Final: A stamp=2
    }
}
```

### ThreadLocal 内存泄漏与线程池陷阱

```java
// JVM 参数: -Xms20m -Xmx20m
import java.util.concurrent.*;

public class ThreadLocalLeak {
    // 每个线程分配 5MB，线程复用导致 value 永远不会被 GC
    private static ThreadLocal<byte[]> tl = ThreadLocal.withInitial(() -> new byte[5 * 1024 * 1024]);

    public static void main(String[] args) throws Exception {
        ExecutorService pool = Executors.newFixedThreadPool(1);

        // 线程池复用同一线程，threadLocals 强引用一直存在
        // Entry extends WeakReference<ThreadLocal> ← key 是弱引用，但 value 是强引用
        // 如果 key (ThreadLocal 对象) 被 GC，value 仍被 Entry 强引用 → 内存泄漏
        pool.execute(() -> {
            tl.get();                 // 分配 5MB
            tl.remove();              // 必须显式 remove() 清理 Entry
            // 若注释掉 remove()，线程池复用时这 5MB 永远不释放
        });

        pool.shutdown();
    }
}
/*
 * ThreadLocal 内部结构：
 * Thread.threadLocals → ThreadLocalMap
 *   Entry[] table → Entry(key=WeakReference<ThreadLocal>, value=强引用)
 * 关键：key 是弱引用但 value 是强引用，只有显式 remove() 才能清理 value
 */
```

## HotSpot 锁优化

![[Attachments/jvm-lock-inflation.excalidraw]]

锁膨胀路径（不可逆）：**无锁 → 偏向锁 → 轻量级锁 → 重量级锁**

| 锁技术 | 原理 | 场景 |
|--------|------|------|
| **偏向锁** | Mark Word 记录线程 ID，无竞争时省略 CAS | 单线程反复获取同一锁 |
| **轻量级锁** | CAS 替换 Mark Word，避免 OS 互斥 | 少量线程交替执行 |
| **自旋锁** | 忙等待避免线程切换 | 锁持有时间很短 |
| **自适应自旋** | 根据历史动态调整自旋次数 | 同上，更智能 |
| **锁消除** | 逃逸分析 + 同步消除 | 线程私有对象 |
| **锁粗化** | 合并相邻同步块 | 反复加解锁同一对象 |

### 锁膨胀观测（使用 JOL）

依赖 JOL（Java Object Layout）库观察 Mark Word 中锁状态的变化：

```java
// JVM 参数: -XX:+UseBiasedLocking -XX:BiasedLockingStartupDelay=0
// 依赖: org.openjdk.jol:jol-core
import org.openjdk.jol.info.ClassLayout;

public class LockInflationDemo {
    public static void main(String[] args) throws Exception {
        Object lock = new Object();
        System.out.println("=== Initial: 无锁（可偏向） ===");
        // Mark Word 末 3 位: 101 = biased (可偏向但未锁定)
        System.out.println(ClassLayout.parseInstance(lock).toPrintable());

        System.out.println("=== Phase 1: 偏向锁（Thread main） ===");
        synchronized (lock) {
            // Mark Word 记录 main 线程的 thread ID
            System.out.println(ClassLayout.parseInstance(lock).toPrintable());
        }
        // 偏向锁不会在退出同步块时撤销，Mark Word 仍保留线程 ID
        System.out.println("=== After sync exit（偏向锁仍保留） ===");
        System.out.println(ClassLayout.parseInstance(lock).toPrintable());

        // 等待安全点触发偏向撤销
        Thread.sleep(5000);

        System.out.println("=== Phase 2: 轻量级锁（竞争） ===");
        Thread t = new Thread(() -> {
            synchronized (lock) {
                // Mark Word 指向线程栈上的 Lock Record（00 = lightweight locked）
                System.out.println(ClassLayout.parseInstance(lock).toPrintable());
            }
        });
        t.start();
        t.join();

        // 批量重偏向/批量撤销：
        // 当某个类的偏向锁撤销次数达到 BiasedLockingBulkRebiasThreshold(20) 时，
        // JVM 对该类执行批量重偏向；达到 BiasedLockingBulkRevokeThreshold(40) 时，
        // JVM 彻底禁用该类的偏向锁
    }
}
/*
 * 锁状态在 Mark Word 末 3 位编码：
 *   001 = 无锁
 *   101 = 偏向锁 (biased)
 *   00  = 轻量级锁 (lightweight) ← 末 2 位
 *   10  = 重量级锁 (heavyweight) ← 末 2 位
 *   11  = GC 标记
 */
```

### 锁粗化示例

JIT 编译器会将相邻的 synchronized 块合并以减少加锁/解锁开销：

```java
// JVM 参数: -XX:-EliminateLocks -XX:+EliminateLocks（默认开启）
public class LockCoarseningDemo {
    public static void main(String[] args) {
        Object lock = new Object();
        StringBuilder sb = new StringBuilder();

        // 连续对同一对象加锁 → JIT 锁粗化合并为一个同步块
        for (int i = 0; i < 100; i++) {
            synchronized (lock) {
                sb.append('a');
            }
            // 原本每次循环都 monitorenter/monitorexit
            // JIT 会将整个循环包裹在单个 synchronized 块中
        }
    }
}
/*
 * 锁粗化前提：
 * 1. 相邻的 synchronized 块锁定同一个对象
 * 2. 中间没有其他加锁操作
 * 3. 由 C2 编译器（Server Compiler）在 JIT 编译时完成
 *
 * 反编译验证（需要 PrintAssembly）：
 *   java -XX:+UnlockDiagnosticVMOptions -XX:+PrintAssembly LockCoarseningDemo
 * 可见循环体外只有一个 monitorenter/monitorexit
 */
```

### 死锁演示与 jstack 诊断

```java
public class DeadlockDemo {
    private static final Object A = new Object();
    private static final Object B = new Object();

    public static void main(String[] args) {
        // Thread 1: 持有 A，等待 B
        new Thread(() -> {
            synchronized (A) {
                System.out.println("T1: locked A");
                try { Thread.sleep(50); } catch (InterruptedException e) {}
                synchronized (B) {
                    System.out.println("T1: locked B");
                }
            }
        }, "T1").start();

        // Thread 2: 持有 B，等待 A
        new Thread(() -> {
            synchronized (B) {
                System.out.println("T2: locked B");
                try { Thread.sleep(50); } catch (InterruptedException e) {}
                synchronized (A) {
                    System.out.println("T2: locked A");
                }
            }
        }, "T2").start();
    }
}
/*
 * 诊断步骤：
 *   1. jps -l                      # 找到 DeadlockDemo 的 PID
 *   2. jstack <PID>                # 导出线程堆栈
 *   3. 检查输出末尾：
 *
 *   Found one Java-level deadlock:
 *   =============================
 *   "T2":
 *     waiting to lock monitor 0x00007f8a1c004e28 (object 0x...a, a java.lang.Object),
 *     which is held by "T1"
 *   "T1":
 *     waiting to lock monitor 0x00007f8a1c006158 (object 0x...b, a java.lang.Object),
 *     which is held by "T2"
 *
 *   Java stack information for the threads listed above:
 *   ===================================================
 *   "T2": at DeadlockDemo.lambda$main$1(DeadlockDemo.java:24)
 *         - waiting to lock <0x...a> (a java.lang.Object)
 *         - locked <0x...b> (a java.lang.Object)
 *   "T1": at DeadlockDemo.lambda$main$0(DeadlockDemo.java:15)
 *         - waiting to lock <0x...b> (a java.lang.Object)
 *         - locked <0x...a> (a java.lang.Object)
 *
 *   4. jconsole / VisualVM: 点击 "检测死锁" 按钮可视化
 */
```

## 在本知识库中的应用

- 选择同步策略：读多写少用 volatile + CAS，竞争激烈用 synchronized
- 性能调优：观察锁膨胀情况，避免不必要的 synchronized
- [[10-domains/java/jvm/概念_Java内存模型JMM]] 提供理论基础，本页是实践手段

## 关联

- [[10-domains/java/jvm/概念_Java内存模型JMM]] — JMM 定义原子性/可见性/有序性规则，本页的 synchronized 和 CAS 是实现这些规则的具体手段
- [[10-domains/java/jvm/概念_JIT编译优化技术]] — 锁消除依赖逃逸分析判断对象是否线程私有，偏向锁的批量撤销涉及 JIT 编译
- [[10-domains/java/jvm/项目_HotSpot_VM]] — 锁膨胀路径（偏向锁→轻量级锁→重量级锁）是 HotSpot 独有的实现细节
