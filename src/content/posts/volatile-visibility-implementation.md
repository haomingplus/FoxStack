---
title: Volatile 的可见性是如何实现的？
published: 1970-01-03
description: 深入了解 Firefly 的布局系统，包括侧边栏布局（左侧/双侧）和文章列表布局（列表/网格），以及全新的三列网格模式。
image: ./images/firefly1.webp
tags: [Java, 并发编程]
category: 博客指南
draft: false
---

## 引言

在 Java 面试中，volatile 是一个高频考点。很多人都知道 volatile 能保证可见性，但当面试官追问"可见性是如何实现的"时，往往答不上来。今天我们就从硬件到软件，层层剥开 volatile 可见性的实现原理。

## 从一个问题开始

先看一段经典代码：

```java
public class VisibilityDemo {
    private static boolean flag = false;

    public static void main(String[] args) throws InterruptedException {
        new Thread(() -> {
            while (!flag) {
                // 空循环
            }
            System.out.println("线程结束");
        }).start();

        Thread.sleep(1000);
        flag = true;
        System.out.println("flag已设为true");
    }
}
```

运行这段代码，你会发现程序可能永远不会终止。子线程读取的 flag 一直是 false，即使主线程已经将其修改为 true。这就是**可见性问题**。

给 flag 加上 volatile 关键字后，程序就能正常结束了。那么 volatile 到底做了什么？

## 理解问题的根源：CPU 缓存架构

要理解 volatile，首先要理解现代 CPU 的缓存架构。

### 多级缓存结构

```
┌─────────────────────────────────────────────────────────┐
│                        主内存                            │
└─────────────────────────────────────────────────────────┘
                            ↑↓
         ┌──────────────────┴───────────────────┐
         ↓                                      ↓
┌─────────────────┐                    ┌─────────────────┐
│    L3 Cache     │                    │    L3 Cache     │
│   (CPU共享)      │                    │   (CPU共享)      │
└─────────────────┘                    └─────────────────┘
         ↑↓                                     ↑↓
┌─────────────────┐                    ┌─────────────────┐
│    L2 Cache     │                    │    L2 Cache     │
│  (核心独占)      │                    │  (核心独占)      │
└─────────────────┘                    └─────────────────┘
         ↑↓                                     ↑↓
┌─────────────────┐                    ┌─────────────────┐
│    L1 Cache     │                    │    L1 Cache     │
│  (核心独占)      │                    │  (核心独占)      │
└─────────────────┘                    └─────────────────┘
         ↑↓                                     ↑↓
┌─────────────────┐                    ┌─────────────────┐
│     CPU 0       │                    │     CPU 1       │
└─────────────────┘                    └─────────────────┘
```

CPU 访问主内存需要约 100 个时钟周期，而访问 L1 缓存只需要约 4 个时钟周期。为了提升性能，CPU 会将数据缓存在离自己最近的 Cache 中。

问题来了：当 CPU0 修改了一个变量，这个修改首先发生在 CPU0 的 L1 Cache 中，CPU1 的 Cache 中还是旧值。这就是可见性问题的硬件根源。

## 硬件层面的解决方案：MESI 缓存一致性协议

为了解决多核 CPU 的缓存一致性问题，硬件层面引入了 MESI 协议。

### MESI 的四种状态

每个缓存行都有一个状态标记：

| 状态 | 全称 | 含义 |
|------|------|------|
| M | Modified | 数据被修改，与主内存不一致，只存在于当前缓存 |
| E | Exclusive | 数据与主内存一致，只存在于当前缓存 |
| S | Shared | 数据与主内存一致，可能存在于多个缓存 |
| I | Invalid | 缓存行无效 |

### MESI 的工作流程

假设 CPU0 要修改一个处于 S 状态的变量：

1. CPU0 发出 Invalidate 消息到总线
2. 其他 CPU 收到消息后，将自己缓存中该变量的状态改为 I（无效）
3. 其他 CPU 返回 Invalidate Acknowledge
4. CPU0 收到所有确认后，将缓存行状态改为 M，执行修改

当 CPU1 再次读取这个变量时，发现状态是 I，就会从 CPU0 的缓存或主内存重新加载数据。

### 为什么 MESI 还不够？

你可能会想：有了 MESI 协议，可见性问题不就解决了吗？为什么还需要 volatile？

原因在于：**CPU 为了提升性能，引入了 Store Buffer 和 Invalidate Queue。**

```
┌─────────────────────────────────────────────┐
│                   CPU                        │
│  ┌──────────┐    ┌──────────────────────┐   │
│  │ 执行单元  │───→│    Store Buffer      │───┼──→ 缓存
│  └──────────┘    └──────────────────────┘   │
│       ↑                                      │
│       └──────────┬───────────────────────   │
│            ┌──────────────────────┐         │
│            │   Invalidate Queue   │←────────┼── 其他CPU的Invalidate消息
│            └──────────────────────┘         │
└─────────────────────────────────────────────┘
```

Store Buffer 让 CPU 不必等待 Invalidate Acknowledge 就能继续执行。Invalidate Queue 让 CPU 不必立即处理 Invalidate 消息。

这两个优化提升了性能，但破坏了 MESI 的强一致性保证，导致了可见性问题。

## volatile 的实现：内存屏障

volatile 通过**内存屏障（Memory Barrier）**来解决上述问题。

### 内存屏障的类型

JMM 定义了四种内存屏障：

| 屏障类型 | 指令示例 | 说明 |
|---------|---------|------|
| LoadLoad | Load1; LoadLoad; Load2 | 确保 Load1 的读取先于 Load2 |
| StoreStore | Store1; StoreStore; Store2 | 确保 Store1 的写入先于 Store2 |
| LoadStore | Load1; LoadStore; Store2 | 确保 Load1 的读取先于 Store2 的写入 |
| StoreLoad | Store1; StoreLoad; Load2 | 确保 Store1 的写入先于 Load2 的读取 |

### volatile 的内存屏障插入策略

JVM 在 volatile 变量的读写操作前后插入内存屏障：

**volatile 写操作：**
```
StoreStore 屏障
volatile 写操作
StoreLoad 屏障
```

**volatile 读操作：**
```
volatile 读操作
LoadLoad 屏障
LoadStore 屏障
```

### 内存屏障的底层实现

在 x86 架构下，volatile 写操作会被编译成带有 lock 前缀的指令。我们可以通过 JIT 编译器的输出来验证：

```java
// 使用 -XX:+UnlockDiagnosticVMOptions -XX:+PrintAssembly 查看汇编
private volatile int value;

public void setValue(int v) {
    value = v;
}
```

生成的汇编代码会包含类似这样的指令：

```asm
lock addl $0x0,(%rsp)  ; 或者
lock xchg ...
```

lock 前缀指令会：

1. **锁定缓存行**：确保当前 CPU 独占该缓存行
2. **刷新 Store Buffer**：将 Store Buffer 中的数据强制写入缓存
3. **使其他 CPU 缓存失效**：通过总线发送信号，让其他 CPU 的相关缓存行失效

## 用一张图总结 volatile 的工作原理

```
线程A执行 volatile 写                     线程B执行 volatile 读
        │                                        │
        ↓                                        ↓
┌───────────────────┐                  ┌───────────────────┐
│  StoreStore屏障   │                  │  volatile 读     │
│  (禁止上面的写与   │                  │                   │
│   volatile写重排) │                  │                   │
└───────────────────┘                  └───────────────────┘
        │                                        │
        ↓                                        ↓
┌───────────────────┐                  ┌───────────────────┐
│  volatile 写      │                  │  LoadLoad屏障    │
│  lock 前缀指令    │   ─────────────→  │  LoadStore屏障   │
│  1.刷新Store Buffer│  使缓存失效      │  (禁止下面的读写与│
│  2.使其他CPU缓存   │                  │   volatile读重排)│
│    失效           │                  │                   │
└───────────────────┘                  └───────────────────┘
        │                                        │
        ↓                                        ↓
┌───────────────────┐                  
│  StoreLoad屏障    │                  
│  (禁止volatile写  │                  
│   与后面的读重排) │                  
└───────────────────┘                  
```

## 延伸：happens-before 与 volatile

JMM 通过 happens-before 规则来描述操作之间的可见性关系。volatile 相关的规则是：

**volatile 变量规则**：对一个 volatile 变量的写操作 happens-before 于后续对这个变量的读操作。

结合**程序顺序规则**和**传递性规则**，volatile 还能保证：在 volatile 写之前的所有普通变量的修改，对 volatile 读之后的操作都可见。

```java
int a = 0;
volatile boolean flag = false;

// 线程A
a = 1;           // 1
flag = true;     // 2

// 线程B
if (flag) {      // 3
    int b = a;   // 4，能读到 a = 1
}
```

这就是 volatile 常用于双重检查锁定单例模式的原因。

## 面试回答要点

回答这个问题时，可以分层次展开：

1. **问题根源**：多核 CPU 的缓存架构导致各核心看到的数据可能不一致

2. **硬件解决方案**：MESI 协议保证缓存一致性，但 Store Buffer 和 Invalidate Queue 的存在仍可能导致可见性问题

3. **软件解决方案**：JMM 通过内存屏障来解决。volatile 写会插入 StoreStore 和 StoreLoad 屏障，volatile 读会插入 LoadLoad 和 LoadStore 屏障

4. **底层实现**：在 x86 架构下，volatile 写通过 lock 前缀指令实现，该指令会锁定缓存行、刷新 Store Buffer、使其他 CPU 缓存失效

## 总结

volatile 的可见性实现是一个跨越硬件和软件的精妙设计。从 CPU 缓存架构到 MESI 协议，从 Store Buffer 到内存屏障，再到 JMM 的 happens-before 规则，每一层都在为最终的可见性保证贡献力量。

理解这些底层原理，不仅能帮助你在面试中脱颖而出，更能让你在实际开发中正确使用并发工具，写出更高效、更安全的代码。
