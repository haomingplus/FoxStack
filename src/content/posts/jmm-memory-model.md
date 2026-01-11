---
title: Java 内存模型（JMM）深度解析
published: 2026-01-03
description: Java 内存模型（Java Memory Model）是 JVM 规范中定义的一套规则，用于屏蔽各种硬件和操作系统的内存访问差异，让 Java 程序在各种平台下都能达到一致的内存访问效果。
tags: [Java, JMM, 并发编程]
category: Java并发编程
draft: false
---

## 一、JMM 的核心内涵

### 1.1 为什么需要 JMM

Java 内存模型（Java Memory Model）是 JVM 规范中定义的一套规则，用于屏蔽各种硬件和操作系统的内存访问差异，让 Java 程序在各种平台下都能达到一致的内存访问效果。

现代计算机体系结构中存在三个关键问题：

```
┌─────────────────────────────────────────────────────────────┐
│                        主内存 (Main Memory)                  │
└─────────────────────────────────────────────────────────────┘
        ↑↓                    ↑↓                    ↑↓
┌───────────────┐    ┌───────────────┐    ┌───────────────┐
│   CPU Cache   │    │   CPU Cache   │    │   CPU Cache   │
│   (L1/L2/L3)  │    │   (L1/L2/L3)  │    │   (L1/L2/L3)  │
└───────────────┘    └───────────────┘    └───────────────┘
        ↑↓                    ↑↓                    ↑↓
┌───────────────┐    ┌───────────────┐    ┌───────────────┐
│     CPU 0     │    │     CPU 1     │    │     CPU 2     │
└───────────────┘    └───────────────┘    └───────────────┘
```

**三大核心问题：**

| 问题 | 描述 | 根源 |
|------|------|------|
| **可见性** | 一个线程修改共享变量后，其他线程能否立即看到 | CPU 缓存导致各核心数据不一致 |
| **原子性** | 一个操作是否不可分割 | 线程切换导致操作被打断 |
| **有序性** | 程序执行顺序是否与代码顺序一致 | 编译器/处理器的指令重排序优化 |

### 1.2 JMM 的抽象结构

JMM 定义了线程与主内存之间的抽象关系：

```
┌─────────────────────────────────────────────────────────────────┐
│                          主内存                                  │
│              (所有共享变量存储的位置)                              │
│         ┌─────────┬─────────┬─────────┬─────────┐               │
│         │ 变量 A  │ 变量 B  │ 变量 C  │   ...   │               │
│         └─────────┴─────────┴─────────┴─────────┘               │
└─────────────────────────────────────────────────────────────────┘
              ↑↓ save/load        ↑↓ save/load
    ┌─────────────────┐    ┌─────────────────┐
    │   工作内存       │    │   工作内存       │
    │  (线程私有)      │    │  (线程私有)      │
    │ ┌─────┬─────┐   │    │ ┌─────┬─────┐   │
    │ │副本A│副本B│   │    │ │副本A│副本C│   │
    │ └─────┴─────┘   │    │ └─────┴─────┘   │
    └─────────────────┘    └─────────────────┘
           ↑↓                      ↑↓
    ┌─────────────┐          ┌─────────────┐
    │   线程 1    │          │   线程 2    │
    └─────────────┘          └─────────────┘
```

### 1.3 JMM 的八种原子操作

JMM 定义了 8 种原子操作来完成主内存与工作内存之间的交互：

```java
// JMM 八种原子操作示意
┌────────────────────────────────────────────────────────────┐
│  主内存操作:                                                │
│    lock   → 将变量标识为线程独占                             │
│    unlock → 释放变量的锁定状态                               │
│    read   → 从主内存读取变量值                               │
│    write  → 将值写入主内存变量                               │
│                                                            │
│  工作内存操作:                                               │
│    load   → 将 read 的值放入工作内存副本                     │
│    use    → 将工作内存变量值传递给执行引擎                    │
│    assign → 将执行引擎的值赋给工作内存变量                    │
│    store  → 将工作内存变量值传送到主内存                      │
└────────────────────────────────────────────────────────────┘
```

**操作执行流程：**

```
主内存变量 ──read──→ 传输 ──load──→ 工作内存副本 ──use──→ 线程使用
                                        ↑
主内存变量 ←─write── 传输 ←─store── 工作内存副本 ←─assign─ 线程赋值
```

---

## 二、Happens-Before 规则详解

### 2.1 什么是 Happens-Before

Happens-Before 是 JMM 中最核心的概念，它定义了两个操作之间的偏序关系。如果操作 A happens-before 操作 B，则：

1. A 的执行结果对 B 可见
2. A 的执行顺序排在 B 之前（从程序员视角）

**重要理解：** Happens-Before 并不意味着 A 必须在 B 之前执行，而是说 A 的结果必须对 B 可见。如果重排序后的执行结果与按 happens-before 顺序执行的结果一致，那么这种重排序是允许的。

### 2.2 八大 Happens-Before 规则

#### 规则一：程序顺序规则 (Program Order Rule)

```java
// 在单线程内，按照代码顺序，前面的操作 happens-before 后面的操作
int a = 1;      // 操作1
int b = 2;      // 操作2
int c = a + b;  // 操作3

// 1 happens-before 2 happens-before 3
// 但编译器可能重排序 1 和 2（因为它们没有依赖关系）
```

#### 规则二：监视器锁规则 (Monitor Lock Rule)

```java
// 对一个锁的解锁 happens-before 于后续对这个锁的加锁
synchronized (this) {  // 加锁
    // 临界区
}  // 解锁 happens-before 下一个线程的加锁

// 示例：
class Counter {
    private int count = 0;
    
    public synchronized void increment() {
        count++;  // 线程A的写操作
    }
    
    public synchronized int getCount() {
        return count;  // 线程B能看到线程A的修改
    }
}
```

**图示：**
```
线程A                          线程B
  │                              │
  ├─ 获取锁                       │
  │                              │
  ├─ count = 1                   │
  │                              │
  ├─ 释放锁 ─────happens-before───→ 获取锁
  │                              │
  │                              ├─ 读取 count (值为1)
  │                              │
  │                              └─ 释放锁
```

#### 规则三：volatile 变量规则 (Volatile Variable Rule)

```java
// 对 volatile 变量的写操作 happens-before 于后续对该变量的读操作
volatile boolean flag = false;

// 线程A
flag = true;  // volatile 写

// 线程B
if (flag) {   // volatile 读，能看到线程A的写入
    // ...
}
```

#### 规则四：线程启动规则 (Thread Start Rule)

```java
// Thread.start() happens-before 于该线程内的任何操作
int x = 10;
Thread t = new Thread(() -> {
    System.out.println(x);  // 保证能看到 x = 10
});
t.start();  // start() happens-before 线程t内的所有操作
```

#### 规则五：线程终止规则 (Thread Termination Rule)

```java
// 线程中的所有操作 happens-before 于对该线程的终止检测
Thread t = new Thread(() -> {
    x = 42;  // 线程t中的操作
});
t.start();
t.join();  // 线程t的所有操作 happens-before join() 返回
System.out.println(x);  // 保证能看到 x = 42
```

#### 规则六：线程中断规则 (Thread Interruption Rule)

```java
// 对线程 interrupt() 的调用 happens-before 
// 被中断线程检测到中断事件的发生
Thread t = new Thread(() -> {
    while (!Thread.currentThread().isInterrupted()) {
        // isInterrupted() 能检测到 interrupt() 的调用
    }
});
t.start();
t.interrupt();  // happens-before isInterrupted() 返回 true
```

#### 规则七：对象终结规则 (Finalizer Rule)

```java
// 对象的构造函数执行完成 happens-before 于 finalize() 方法的开始
class MyObject {
    private int value;
    
    public MyObject() {
        value = 42;  // 构造完成
    }
    
    @Override
    protected void finalize() {
        // 保证能看到 value = 42
        System.out.println(value);
    }
}
```

#### 规则八：传递性规则 (Transitivity)

```java
// 如果 A happens-before B，B happens-before C，那么 A happens-before C
class Example {
    int a = 0;
    volatile boolean flag = false;
    
    // 线程A
    public void writer() {
        a = 1;           // 操作1
        flag = true;     // 操作2 (volatile写)
    }
    
    // 线程B  
    public void reader() {
        if (flag) {      // 操作3 (volatile读)
            int i = a;   // 操作4，保证读到 a = 1
        }
    }
}
// 1 happens-before 2（程序顺序规则）
// 2 happens-before 3（volatile规则）
// 由传递性：1 happens-before 3，因此 1 happens-before 4
```

### 2.3 Happens-Before 规则汇总表

| 规则 | 描述 | 应用场景 |
|------|------|----------|
| 程序顺序规则 | 单线程内代码顺序 | 所有单线程程序 |
| 监视器锁规则 | unlock → lock | synchronized 块 |
| volatile规则 | volatile写 → volatile读 | volatile 变量 |
| 线程启动规则 | start() → 线程体 | 线程启动 |
| 线程终止规则 | 线程体 → join()/isAlive() | 线程等待 |
| 中断规则 | interrupt() → 中断检测 | 线程中断 |
| 终结器规则 | 构造函数 → finalize() | 对象回收 |
| 传递性 | A→B, B→C ⇒ A→C | 规则组合 |

---

## 三、volatile 的实现原理

### 3.1 volatile 的两大特性

```java
volatile int sharedVar = 0;
```

| 特性 | 描述 | 保证 |
|------|------|------|
| **可见性** | 写操作立即刷新到主内存，读操作从主内存读取 | 一个线程的修改对其他线程立即可见 |
| **禁止重排序** | 编译器和处理器不能对 volatile 变量进行重排序优化 | 有序性 |

**注意：volatile 不保证原子性！**

```java
volatile int count = 0;

// 以下操作不是原子的，volatile 无法保证线程安全
count++;  // 实际上是: read-modify-write 三个操作
```

### 3.2 可见性的实现机制

#### 3.2.1 MESI 缓存一致性协议

```
┌─────────────────────────────────────────────────────────────┐
│                    总线 (Bus)                                │
│    ┌──────────────────────────────────────────────────┐     │
│    │              总线嗅探机制 (Bus Snooping)           │     │
│    └──────────────────────────────────────────────────┘     │
└─────────────────────────────────────────────────────────────┘
        ↑↓                    ↑↓                    ↑↓
┌───────────────┐    ┌───────────────┐    ┌───────────────┐
│    Cache      │    │    Cache      │    │    Cache      │
│   状态: M/E/S/I│    │   状态: M/E/S/I│    │   状态: M/E/S/I│
└───────────────┘    └───────────────┘    └───────────────┘
```

**MESI 四种状态：**

| 状态 | 全称 | 描述 |
|------|------|------|
| **M** | Modified | 被修改，与主内存不一致，是唯一有效副本 |
| **E** | Exclusive | 独占，与主内存一致，是唯一副本 |
| **S** | Shared | 共享，与主内存一致，可能存在其他副本 |
| **I** | Invalid | 无效，缓存行失效 |

#### 3.2.2 volatile 写操作流程

```java
// volatile 写操作
volatile int flag = 0;
flag = 1;  // volatile 写
```

```
┌─────────────────────────────────────────────────────────────┐
│ 1. CPU执行 volatile 写操作                                   │
│    ↓                                                        │
│ 2. 将数据写入 Store Buffer                                   │
│    ↓                                                        │
│ 3. 发出 LOCK# 信号（或使用 MESI 协议）                        │
│    ↓                                                        │
│ 4. 将 Store Buffer 中的数据刷新到缓存/主内存                  │
│    ↓                                                        │
│ 5. 通过总线嗅探，使其他 CPU 缓存中该变量的副本失效 (Invalid)   │
└─────────────────────────────────────────────────────────────┘
```

#### 3.2.3 volatile 读操作流程

```java
// volatile 读操作
int result = flag;  // volatile 读
```

```
┌─────────────────────────────────────────────────────────────┐
│ 1. CPU 发出读取指令                                          │
│    ↓                                                        │
│ 2. 检查本地缓存状态                                          │
│    ↓                                                        │
│ 3. 如果状态为 Invalid，从主内存/其他CPU缓存重新加载           │
│    ↓                                                        │
│ 4. 清空 Invalidate Queue（使之前的失效立即生效）             │
│    ↓                                                        │
│ 5. 读取最新值                                                │
└─────────────────────────────────────────────────────────────┘
```

### 3.3 禁止指令重排序的实现

#### 3.3.1 内存屏障 (Memory Barrier)

JMM 通过插入内存屏障来禁止特定类型的指令重排序：

| 屏障类型 | 指令示例 | 作用 |
|----------|----------|------|
| **LoadLoad** | Load1; LoadLoad; Load2 | 确保 Load1 先于 Load2 及后续读操作 |
| **StoreStore** | Store1; StoreStore; Store2 | 确保 Store1 先于 Store2 及后续写操作 |
| **LoadStore** | Load1; LoadStore; Store2 | 确保 Load1 先于 Store2 及后续写操作 |
| **StoreLoad** | Store1; StoreLoad; Load2 | 确保 Store1 先于 Load2 及后续读操作（全能屏障） |

#### 3.3.2 volatile 的内存屏障策略

**volatile 写操作：**

```
┌────────────────────────────────────────┐
│     普通读/写操作                       │
│              ↓                         │
│    ═══ StoreStore 屏障 ═══            │
│              ↓                         │
│     volatile 写操作                    │
│              ↓                         │
│    ═══ StoreLoad 屏障 ═══             │
│              ↓                         │
│     后续读/写操作                       │
└────────────────────────────────────────┘
```

**volatile 读操作：**

```
┌────────────────────────────────────────┐
│     volatile 读操作                    │
│              ↓                         │
│    ═══ LoadLoad 屏障 ═══              │
│              ↓                         │
│    ═══ LoadStore 屏障 ═══             │
│              ↓                         │
│     后续读/写操作                       │
└────────────────────────────────────────┘
```

#### 3.3.3 具体场景分析

```java
class VolatileExample {
    int a = 0;
    volatile boolean flag = false;
    
    public void writer() {
        a = 1;                  // 1: 普通写
        // ─── StoreStore屏障 ───  禁止 1 和 2 重排序
        flag = true;            // 2: volatile 写
        // ─── StoreLoad屏障 ───   禁止 2 和后续读重排序
    }
    
    public void reader() {
        if (flag) {             // 3: volatile 读
            // ─── LoadLoad屏障 ───
            // ─── LoadStore屏障 ───
            int i = a;          // 4: 普通读，保证读到 a = 1
        }
    }
}
```

**重排序规则表：**

| 第一个操作 | 第二个操作: 普通读写 | 第二个操作: volatile读 | 第二个操作: volatile写 |
|------------|---------------------|----------------------|----------------------|
| 普通读写 | 可以重排 | 可以重排 | **不能重排** |
| volatile读 | **不能重排** | **不能重排** | **不能重排** |
| volatile写 | 可以重排 | **不能重排** | **不能重排** |

### 3.4 JVM 层面的实现

#### 3.4.1 字节码层面

```java
volatile int value;
```

在字节码中，volatile 变量的访问会使用特殊的指令标记：

```
// 访问 volatile 字段时，字节码中会有 ACC_VOLATILE 标志
getfield #2  // Field value:I (volatile)
putfield #2  // Field value:I (volatile)
```

#### 3.4.2 HotSpot 实现

在 x86 架构上，JVM 对 volatile 的实现：

```cpp
// volatile 写操作（简化的伪代码）
inline void OrderAccess::storeload() {
    // x86 上使用 lock 前缀指令或 mfence
    __asm__ volatile ("lock; addl $0,0(%%rsp)" : : : "cc", "memory");
}

// 或者使用 mfence 指令
inline void OrderAccess::fence() {
    __asm__ volatile ("mfence" : : : "memory");
}
```

**x86 架构的 lock 前缀作用：**

1. 锁定总线（或缓存行）
2. 将处理器缓存写回主内存
3. 使其他处理器缓存失效
4. 禁止 lock 前后的指令重排序

---

## 四、实战应用与最佳实践

### 4.1 双重检查锁定 (DCL) 单例模式

```java
public class Singleton {
    // 必须使用 volatile，否则可能出现问题
    private static volatile Singleton instance;
    
    private Singleton() {}
    
    public static Singleton getInstance() {
        if (instance == null) {                    // 第一次检查
            synchronized (Singleton.class) {
                if (instance == null) {            // 第二次检查
                    instance = new Singleton();    // 关键点
                }
            }
        }
        return instance;
    }
}
```

**为什么需要 volatile？**

`instance = new Singleton()` 实际上包含三个步骤：
1. 分配内存空间
2. 初始化对象
3. 将 instance 指向分配的内存

如果没有 volatile，步骤 2 和 3 可能被重排序：

```
线程A                              线程B
  │                                  │
  ├─ 分配内存                         │
  │                                  │
  ├─ instance = 内存地址 (步骤3)      │
  │        │                         │
  │        │                    ├─ if(instance != null)  ← 非null！
  │        │                    │
  │        │                    └─ return instance  ← 返回未初始化对象！
  │        │
  └─ 初始化对象 (步骤2)
```

### 4.2 volatile 的正确使用场景

```java
// ✅ 场景1：状态标志
volatile boolean shutdownRequested = false;

public void shutdown() {
    shutdownRequested = true;
}

public void doWork() {
    while (!shutdownRequested) {
        // 执行任务
    }
}

// ✅ 场景2：一次性安全发布
volatile Configuration config;

public void initialize() {
    config = loadConfiguration();  // 一次性发布
}

// ✅ 场景3：独立观察
volatile long lastUpdateTime;

public void update() {
    lastUpdateTime = System.currentTimeMillis();
}

// ❌ 错误使用：复合操作
volatile int counter = 0;

public void increment() {
    counter++;  // 不是原子操作，线程不安全！
}
```

### 4.3 常见面试问题

**Q1: volatile 能保证原子性吗？**

不能。volatile 只保证可见性和有序性。对于复合操作如 `i++`，需要使用 `synchronized` 或 `AtomicInteger`。

**Q2: synchronized 和 volatile 的区别？**

| 特性 | volatile | synchronized |
|------|----------|--------------|
| 可见性 | ✅ | ✅ |
| 原子性 | ❌ | ✅ |
| 有序性 | ✅ | ✅ |
| 阻塞 | ❌ | ✅ |
| 适用范围 | 变量 | 方法/代码块 |
| 性能 | 较高 | 较低 |

**Q3: happens-before 是否意味着时间上的先后？**

不一定。happens-before 只保证可见性和顺序性的语义，编译器和处理器可以在不违反这种语义的情况下进行重排序优化。

---

## 五、总结

### JMM 的核心要点

1. **JMM 是规范**：定义了多线程程序中变量的访问规则
2. **抽象模型**：工作内存 ↔ 主内存
3. **三大特性**：可见性、原子性、有序性
4. **happens-before**：JMM 的核心，定义操作间的偏序关系

### volatile 实现原理

1. **可见性**：通过 MESI 协议 + 总线嗅探机制实现
2. **有序性**：通过内存屏障（Memory Barrier）禁止重排序
3. **底层实现**：x86 上使用 lock 前缀指令或 mfence 指令

### 实践建议

- 优先使用更高层次的并发工具（如 `java.util.concurrent` 包）
- volatile 适用于简单的状态标志和一次性安全发布
- 复合操作使用 `synchronized` 或原子类
- 理解 happens-before 规则，正确分析并发程序的正确性
