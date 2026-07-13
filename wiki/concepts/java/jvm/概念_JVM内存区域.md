---
type: concept
tags:
  - java
  - jvm
  - memory-management
  - runtime-data-area
summary: "JVM 运行时将内存划分为多个区域，各区域有不同用途和生命周期"
sources:
  - "raw/books/深入理解Java虚拟机：JVM高级特性与最佳实践（第3版）周志明.pdf"
aliases:
  - JVM内存区域
  - JVM运行时数据区
status: evolving
confidence: 0.9
updated: "2026-04-28 10:00:00"
---

# 概念_JVM内存区域

## 定义

JVM 在执行 Java 程序过程中将所管理的内存划分为若干运行时数据区域，各区域有各自的用途、创建和销毁时间。部分随虚拟机进程启动而存在，部分依赖用户线程的启动和结束而建立和销毁。

## 内存布局总览

![[Attachments/jvm-memory-layout.excalidraw]]

## 区域划分

### 线程私有的区域

- **程序计数器（PC）**：当前线程执行的字节码行号指示器，唯一无 OOM 的区域
- **Java 虚拟机栈**：每个方法对应一个栈帧，存储局部变量表（Slot）、操作数栈、动态连接、方法出口。异常：StackOverflowError / OOM
- **本地方法栈**：为 Native 方法服务，HotSpot 合二为一

#### 栈溢出实战：StackOverflowError

栈容量由 `-Xss` 控制，默认约 1MB。递归过深或大量局部变量会触发 StackOverflowError。下面通过无限递归来推高栈深度，演示如何复现并获取递归层次：

```java
// -Xss128k  将线程栈容量压缩到 128KB，加速溢出
public class StackOOM {
    private int stackLength = 1;

    public void stackLeak() {
        stackLength++;
        stackLeak();  // 每次递归增加一个栈帧
    }

    public static void main(String[] args) {
        StackOOM oom = new StackOOM();
        try {
            oom.stackLeak();
        } catch (Throwable e) {
            System.out.println("stack length:" + oom.stackLength);
            throw e;
        }
    }
}
```

运行结果类似：
```
stack length:982
Exception in thread "main" java.lang.StackOverflowError
```

栈深度取决于栈帧大小：局部变量表越大（方法参数多、大量本地变量），可递归的层数越少。实际排查时可使用 `jstack <pid>` 看到阻塞在递归调用的线程。

### 线程共享的区域

- **Java 堆**：存放对象实例，GC 管理的主要区域。可细分为新生代（Eden / Survivor）和老年代。支持 TLAB 提升分配效率
- **方法区**：存储类型信息、常量、静态变量、JIT 代码缓存。JDK 8 后以**元空间（Metaspace）**取代永久代，使用本地内存
- **运行时常量池**：Class 文件中常量池的运行时表示，支持动态扩展（如 String.intern()）
- **直接内存**：NIO 通过 DirectByteBuffer 操作，不受堆大小限制，受本机总内存限制

#### 堆溢出实战：Java Heap Space OOM

堆是对象存储的主战场。持续向集合中添加对象且保持 GC Roots 可达，即可触发 `OutOfMemoryError: Java heap space`。添加 JVM 参数 `-Xmx20m -Xms20m` 将堆锁定在 20MB：

```java
// -Xms20m -Xmx20m -XX:+HeapDumpOnOutOfMemoryError
import java.util.ArrayList;
import java.util.List;

public class HeapOOM {
    static class OOMObject {
        // 每个对象约占用部分内存用于演示
    }

    public static void main(String[] args) {
        List<OOMObject> list = new ArrayList<>();
        while (true) {
            list.add(new OOMObject());  // GC Roots 可达，永不回收
        }
    }
}
```

`-XX:+HeapDumpOnOutOfMemoryError` 会在 OOM 时生成 `.hprof` 堆转储，用 Eclipse MAT 或 `jhat` 分析即可定位到持有大量对象的 GC Root 路径。

#### 元空间溢出实战：Metaspace OOM（JDK 8+）

方法区在 JDK 8 之后变为 Metaspace，默认无上限（受本机内存限制）。当动态生成大量类（CGLIB、ASM、反射、Groovy）时会撑爆 Metaspace。通过 `-XX:MaxMetaspaceSize=10m` 限制即可触发：

```java
// JDK 8+: -XX:MaxMetaspaceSize=10m
import org.springframework.cglib.proxy.Enhancer;
import org.springframework.cglib.proxy.MethodInterceptor;
import org.springframework.cglib.proxy.MethodProxy;
import java.lang.reflect.Method;

public class MetaspaceOOM {
    static class OOMObject {}

    public static void main(String[] args) {
        while (true) {
            Enhancer enhancer = new Enhancer();
            enhancer.setSuperclass(OOMObject.class);
            enhancer.setUseCache(false);  // 禁用缓存，每次都生成新类
            enhancer.setCallback(new MethodInterceptor() {
                @Override
                public Object intercept(Object obj, Method m, Object[] args,
                                        MethodProxy proxy) throws Throwable {
                    return proxy.invokeSuper(obj, args);
                }
            });
            enhancer.create();  // 不停生成新类元数据
        }
    }
}
```

> CGLIB 通过字节码技术生成 `OOMObject` 的子类，每个子类的 Class 元数据进入 Metaspace。禁缓存后每次生成新类，最终 `OutOfMemoryError: Metaspace`。

运行时可配合 `jstat -gc <pid> 1000` 观察 MC（Metaspace Capacity）和 MU（Metaspace Used）持续增长直至 OOM。

#### 直接内存溢出实战：Direct Memory OOM

DirectByteBuffer 分配的是堆外内存，不受 `-Xmx` 限制。可通过 `-XX:MaxDirectMemorySize` 设限后触发：

```java
// -XX:MaxDirectMemorySize=10m -Xmx20m
import java.nio.ByteBuffer;
import java.util.ArrayList;
import java.util.List;

public class DirectMemoryOOM {
    private static final int _1MB = 1024 * 1024;

    public static void main(String[] args) {
        List<ByteBuffer> list = new ArrayList<>();
        int count = 0;
        try {
            while (true) {
                ByteBuffer buf = ByteBuffer.allocateDirect(_1MB);
                list.add(buf);  // GC Roots 保持引用
                count++;
            }
        } catch (Throwable e) {
            System.out.println("allocated count: " + count);
            throw e;
        }
    }
}
```

**关键点**：直接内存超出 `MaxDirectMemorySize` 时抛 `OutOfMemoryError`（无 "Direct buffer memory" 字样时注意与堆 OOM 区分）。实际排查可以用 `jcmd <pid> VM.native_memory summary` 查看 Native Memory Tracking 数据（需启动时加 `-XX:NativeMemoryTracking=summary`）。

#### 对象内存布局实战：JOL 解析对象头

HotSpot 中每个对象在堆中的布局为：**Mark Word（锁状态/GC年龄/hash）→ Klass Pointer（类元数据）→ 实例数据 → 对齐填充**。使用 JOL（Java Object Layout）可以直接打印对象的内存布局：

```java
// 需引入 org.openjdk.jol:jol-core
// VM options: -XX:-UseCompressedOops  可选：关闭指针压缩看 8 字节指针
import org.openjdk.jol.info.ClassLayout;
import org.openjdk.jol.info.GraphLayout;

public class ObjectLayoutDemo {
    public static void main(String[] args) {
        // 1. 普通 Object
        System.out.println(ClassLayout.parseInstance(new Object()).toPrintable());

        // 2. 含字段的对象
        System.out.println(ClassLayout.parseInstance(new TestObj()).toPrintable());

        // 3. 对象图总深度（含引用的对象大小）
        // System.out.println(GraphLayout.parseInstance(obj).toPrintable());
    }

    static class TestObj {
        int id = 1;          // 4 bytes
        String name = "a";   // 4 bytes (compressed oop) / 8 bytes
        boolean flag = true; // 1 byte → 对齐后可能占 8 bytes
        long timestamp = System.currentTimeMillis(); // 8 bytes
    }
}
```

**典型输出（开启压缩指针 `-XX:+UseCompressedOops`，64位 JDK 8）**：
```
java.lang.Object object internals:
 OFFSET  SIZE   TYPE DESCRIPTION               VALUE
      0     4         (object header)           01 00 00 00
      4     4         (object header)           00 00 00 00
      8     4         (object header)           ...
```

**输出解读**：
- offset 0–8 为 Mark Word（8 bytes）
- offset 8–12 为 Klass Pointer（4 bytes，compressed）
- 普通 Object 共 12 bytes → 对齐到 16 bytes（8 倍数对齐）
- 可通过 `-XX:-UseCompressedOops` 关闭压缩对比观察

#### TLAB 分配实战

TLAB（Thread Local Allocation Buffer）是每个线程在 Eden 区独占的一小块分配缓冲区，避免并发分配时的同步开销。线程对象分配优先在 TLAB 内完成，TLAB 耗尽或大对象时才走 Eden 共享区域。

```bash
# 开启 TLAB 观测（生产建议仅在排查时启用）
-XX:+PrintTLAB -XX:+UseTLAB
```

```java
/*
 * -XX:+PrintTLAB 会在每次 GC 时输出每个线程的 TLAB 分配统计：
 * TLAB: gc thread: 0x... [id: 1234] desired_size: 256KB slow allocs: 3
 *
 * 关键指标：
 * - desired_size: JVM 为该线程计算的 TLAB 期望大小
 * - slow allocs:   线程在 TLAB 外分配的慢速分配次数（值大说明 TLAB 偏小）
 * - refills:       TLAB 被垃圾回收前被重新填充的次数
 *
 * 调优参数：-XX:TLABSize=256k  -XX:-ResizeTLAB（关闭自适应调整）
 */
public class TLABDemo {
    private static final int _1MB = 1024 * 1024;

    public static void main(String[] args) {
        // 分配大量小对象，TLAB 可以高效消化
        for (int i = 0; i < 100_000; i++) {
            byte[] data = new byte[1024]; // 1KB，TLAB 加速分配
        }
        System.out.println("done");
    }
}
```

现实意义：高并发下单线程分配大量临时对象时，TLAB 避免 CAS 竞争；若 TLAB 过小导致大量 slow allocs，可增大 `-XX:TLABSize` 或打开 `-XX:+AlwaysPreTouch` 减少首次访问缺页开销。

## 使用场景

- **内存溢出排查**：根据 OOM 类型判断是堆溢出（-Xmx）、栈溢出（-Xss）还是直接内存溢出（-XX:MaxDirectMemorySize）

```bash
# 工具链速查
jmap -heap <pid>           # 堆使用概览
jstat -gc <pid> 1000 10    # 每秒采样一次，连续 10 次
jcmd <pid> GC.heap_info    # 堆详细信息含 Metaspace
jcmd <pid> VM.native_memory summary  # 本地内存（含直接内存）
```

- **性能调优**：调整堆比例（新生代/老年代）、选择 GC 收集器、设置 TLAB 大小

## 在本知识库中的应用

- 排查 Java 应用 OOM 时，可参考此区域划分定位是堆/栈/方法区的问题
- 与 [[concepts/java/jvm/概念_垃圾收集算法]] 联动：理解堆的划分（新生代/老年代）是分代 GC 的前提
- 与 [[concepts/java/jvm/概念_Java内存模型JMM]] 联动：主内存对应堆中实例数据，工作内存对应虚拟机栈

## 对象内存布局

![[Attachments/jvm-object-layout.excalidraw]]

## 对象创建与访问

- **创建**：类加载检查 → 分配内存（指针碰撞/空闲列表）→ 零值初始化 → 设对象头 → 执行 `<init>`
- **内存布局**：对象头（Mark Word + Klass Pointer）→ 实例数据 → 对齐填充
- **访问定位**：句柄池（稳定）vs 直接指针（HotSpot 默认，更快）

```bash
# javap 查看构造代码如何调用 <init>
javap -c -p HeapOOM\$OOMObject.class
# 输出中的 invokespecial #1  <init>:()V  就是构造器调用
```

## 关联

- [[concepts/java/jvm/概念_垃圾收集算法]] — 堆内存的分代结构（新生代/老年代）是标记-复制和分代收集理论能够高效运作的前提
- [[concepts/java/jvm/概念_Java内存模型JMM]] — 主内存对应堆中的实例数据，工作内存对应虚拟机栈，两者共享同一物理内存区域
- [[entities/java/jvm/项目_HotSpot_VM]] — 本页描述的是 HotSpot 的实现（合一的本地方法栈、直接指针访问）
