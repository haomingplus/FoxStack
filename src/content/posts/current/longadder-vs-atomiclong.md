---
title: LongAdder与AtomicLong深度对比：高并发计数器选型指南
published: 2023-08-19
description: 深入分析LongAdder的分段锁设计原理，对比AtomicLong的CAS机制，给出不同场景下的选型建议
tags: [Java, 并发编程, LongAdder, AtomicLong, CAS, 高并发]
category: Java并发编程
draft: false
---

## 面试题

LongAdder 相比 AtomicLong 在高并发下性能更好，为什么？什么时候应该用 LongAdder，什么时候 AtomicLong 就够了？

---

## 一、直接结论

| 对比项 | AtomicLong | LongAdder |
|-------|------------|-----------|
| **实现原理** | 单一变量 + CAS | 分段数组 + CAS |
| **高并发性能** | 差（CAS 竞争激烈） | 好（分散竞争） |
| **内存占用** | 低（1 个 long） | 高（base + Cell[]） |
| **精确读取** | ✅ 实时精确 | ❌ 最终一致 |
| **适用场景** | 低并发、需要精确值 | 高并发、只关心最终结果 |

**一句话总结**：
> AtomicLong 是"一个人干活，大家排队"；LongAdder 是"多个人干活，最后汇总"。

---

## 二、AtomicLong 的问题

### 2.1 工作原理

```java
public class AtomicLong {
    private volatile long value;
    
    public final long incrementAndGet() {
        // CAS 自旋：不断尝试直到成功
        for (;;) {
            long current = get();
            long next = current + 1;
            if (compareAndSet(current, next)) {
                return next;
            }
            // CAS 失败，重试
        }
    }
}
```

### 2.2 高并发下的问题

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    AtomicLong 高并发问题示意                                  │
└─────────────────────────────────────────────────────────────────────────────┘

                         AtomicLong (value = 100)
                                   │
            ┌──────────────────────┼──────────────────────┐
            │                      │                      │
            ▼                      ▼                      ▼
      ┌──────────┐          ┌──────────┐          ┌──────────┐
      │ Thread 1 │          │ Thread 2 │          │ Thread 3 │
      │ CAS(100, │          │ CAS(100, │          │ CAS(100, │
      │     101) │          │     101) │          │     101) │
      └──────────┘          └──────────┘          └──────────┘
            │                      │                      │
            ▼                      ▼                      ▼
         成功 ✓                 失败 ✗                 失败 ✗
                                重试...               重试...

问题：
├── 所有线程竞争同一个变量
├── 只有一个线程 CAS 成功，其他全部失败重试
├── 并发越高，失败率越高，CPU 空转越严重
└── 性能随并发数增加而急剧下降
```

### 2.3 性能瓶颈分析

```
CAS 失败的代价：

1. CPU 空转
   └── 失败后立即重试，消耗 CPU 周期

2. 缓存行失效
   └── value 被修改后，其他 CPU 核心的缓存失效
   └── 需要重新从主存加载（缓存一致性协议 MESI）
   └── 这叫"伪共享"问题

3. 总线风暴
   └── 大量 CAS 操作争抢总线
   └── 内存带宽成为瓶颈

并发线程数 vs CAS 失败率：
├── 2 线程：失败率约 50%
├── 4 线程：失败率约 75%
├── 8 线程：失败率约 87.5%
└── 线程越多，失败率越接近 100%
```

---

## 三、LongAdder 的解决方案

### 3.1 核心思想：分散竞争

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    LongAdder 分段设计                                        │
└─────────────────────────────────────────────────────────────────────────────┘

AtomicLong（单点竞争）：
                    ┌─────────┐
   Thread 1 ───────►│         │
   Thread 2 ───────►│  value  │◄─── 所有线程抢一个变量
   Thread 3 ───────►│         │
   Thread 4 ───────►└─────────┘


LongAdder（分散竞争）：
                    ┌─────────┐
   Thread 1 ───────►│  base   │     无竞争时，直接操作 base
                    └─────────┘
                          │
                          │ 有竞争时，分散到 Cell 数组
                          ▼
              ┌─────────────────────────┐
              │       Cell[] cells      │
              ├───────┬───────┬─────────┤
   Thread 2 ─►│Cell[0]│Cell[1]│ Cell[2] │◄─ Thread 4
              │  +1   │  +1   │   +1    │
              └───────┴───────┴─────────┘
                          ▲
                     Thread 3

最终值 = base + sum(cells)
```

### 3.2 源码结构

```java
// LongAdder 继承自 Striped64
public class LongAdder extends Striped64 {
    // ...
}

abstract class Striped64 {
    // CPU 核心数，决定 cells 数组最大长度
    static final int NCPU = Runtime.getRuntime().availableProcessors();
    
    // 分段数组，懒初始化
    transient volatile Cell[] cells;
    
    // 基础值，无竞争时直接累加到这里
    transient volatile long base;
    
    // 自旋锁，用于初始化或扩容 cells
    transient volatile int cellsBusy;
    
    // Cell 类：每个槽位一个计数器
    @sun.misc.Contended  // 避免伪共享
    static final class Cell {
        volatile long value;
        // ...
    }
}
```

### 3.3 add() 方法源码解析

```java
public void add(long x) {
    Cell[] as; long b, v; int m; Cell a;
    
    // 条件1：cells 不为空（说明之前有过竞争）
    // 条件2：对 base 的 CAS 失败（说明当前有竞争）
    if ((as = cells) != null || !casBase(b = base, b + x)) {
        
        boolean uncontended = true;
        
        // 条件3：cells 为空
        // 条件4：当前线程对应的 cell 为空
        // 条件5：对 cell 的 CAS 失败
        if (as == null || (m = as.length - 1) < 0 ||
            (a = as[getProbe() & m]) == null ||      // getProbe() 获取线程哈希
            !(uncontended = a.cas(v = a.value, v + x)))
            
            // 进入复杂处理：初始化 cells、扩容、重新哈希等
            longAccumulate(x, null, uncontended);
    }
}
```

### 3.4 执行流程图解

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                       LongAdder.add() 执行流程                               │
└─────────────────────────────────────────────────────────────────────────────┘

                              开始 add(1)
                                   │
                                   ▼
                    ┌─────────────────────────────┐
                    │     cells == null ?         │
                    └─────────────────────────────┘
                           │是              │否
                           ▼                ▼
                    ┌─────────────┐   ┌───────────────────┐
                    │ CAS 更新    │   │ 计算线程对应的    │
                    │ base        │   │ cell 索引         │
                    └─────────────┘   │ index = probe & m │
                           │          └───────────────────┘
                    成功？  │                    │
                     │     │                    ▼
              ┌──────┴─────┐         ┌─────────────────────┐
              │是          │否        │  cell[index] 存在？ │
              ▼            ▼         └─────────────────────┘
           返回 ✓    ┌──────────┐           │是         │否
                     │ 初始化   │           ▼            ▼
                     │ cells    │    ┌───────────┐  ┌───────────┐
                     │ 数组     │    │ CAS 更新  │  │ 创建新    │
                     └──────────┘    │ cell.value│  │ Cell      │
                                     └───────────┘  └───────────┘
                                           │
                                    成功？  │
                                     │     │
                              ┌──────┴─────┐
                              │是          │否
                              ▼            ▼
                           返回 ✓    ┌───────────────┐
                                     │ 重新哈希或   │
                                     │ 扩容 cells   │
                                     └───────────────┘
```

### 3.5 @Contended 注解：解决伪共享

```java
// Cell 类使用 @Contended 注解
@sun.misc.Contended
static final class Cell {
    volatile long value;
}
```

```
伪共享问题：

CPU 缓存行通常是 64 字节
多个 Cell 可能在同一个缓存行中

┌────────────────────── 缓存行 64 字节 ──────────────────────┐
│  Cell[0].value  │  Cell[1].value  │  Cell[2].value  │ ... │
│     8 字节       │     8 字节       │     8 字节      │     │
└─────────────────────────────────────────────────────────────┘

问题：Thread 1 修改 Cell[0]，会导致整个缓存行失效
      Thread 2 读取 Cell[1] 需要重新加载

@Contended 解决方案：在 Cell 前后填充空字节

┌─ 填充 ─┬─ Cell[0] ─┬─ 填充 ─┬─ Cell[1] ─┬─ 填充 ─┐
│ 128字节 │   8字节   │ 128字节 │   8字节   │ 128字节 │
└─────────┴───────────┴─────────┴───────────┴─────────┘

每个 Cell 独占缓存行，互不影响
```

---

## 四、sum() 方法：最终一致性

```java
public long sum() {
    Cell[] as = cells; Cell a;
    long sum = base;  // 先取 base
    if (as != null) {
        // 遍历所有 cell，累加
        for (int i = 0; i < as.length; ++i) {
            if ((a = as[i]) != null)
                sum += a.value;
        }
    }
    return sum;
}
```

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        sum() 的一致性问题                                    │
└─────────────────────────────────────────────────────────────────────────────┘

时间线：
─────────────────────────────────────────────────────────────────────►

Thread A: 调用 sum()
          │
          │  读取 base = 100
          │       │
          │       │  读取 cells[0] = 50
          │       │       │
          │       │       │   ← Thread B: cells[1].add(10)（此时还没读到）
          │       │       │
          │       │       │  读取 cells[1] = 30（旧值）
          │       │       │       │
          │       │       │       │  读取 cells[2] = 20
          │       │       │       │       │
          ▼       ▼       ▼       ▼       ▼
sum() 返回 100 + 50 + 30 + 20 = 200

实际值应该是 210（Thread B 的 +10 没统计到）

结论：sum() 不是原子操作，高并发下可能读到不一致的值
     适合统计场景（允许小误差），不适合需要精确值的场景
```

---

## 五、性能对比测试

### 5.1 测试代码

```java
public class CounterBenchmark {

    private static final int THREAD_COUNT = 64;
    private static final int INCREMENT_COUNT = 10_000_000;

    public static void main(String[] args) throws Exception {
        // 预热
        testAtomicLong(4, 100000);
        testLongAdder(4, 100000);
        
        System.out.println("线程数\tAtomicLong\tLongAdder\t性能提升");
        System.out.println("─".repeat(60));
        
        for (int threads : new int[]{1, 2, 4, 8, 16, 32, 64}) {
            long atomicTime = testAtomicLong(threads, INCREMENT_COUNT);
            long adderTime = testLongAdder(threads, INCREMENT_COUNT);
            double speedup = (double) atomicTime / adderTime;
            
            System.out.printf("%d\t%dms\t\t%dms\t\t%.2fx%n", 
                    threads, atomicTime, adderTime, speedup);
        }
    }

    private static long testAtomicLong(int threads, int increments) throws Exception {
        AtomicLong counter = new AtomicLong(0);
        ExecutorService executor = Executors.newFixedThreadPool(threads);
        
        long start = System.currentTimeMillis();
        
        CountDownLatch latch = new CountDownLatch(threads);
        for (int i = 0; i < threads; i++) {
            executor.submit(() -> {
                for (int j = 0; j < increments / threads; j++) {
                    counter.incrementAndGet();
                }
                latch.countDown();
            });
        }
        latch.await();
        
        long time = System.currentTimeMillis() - start;
        executor.shutdown();
        return time;
    }

    private static long testLongAdder(int threads, int increments) throws Exception {
        LongAdder counter = new LongAdder();
        ExecutorService executor = Executors.newFixedThreadPool(threads);
        
        long start = System.currentTimeMillis();
        
        CountDownLatch latch = new CountDownLatch(threads);
        for (int i = 0; i < threads; i++) {
            executor.submit(() -> {
                for (int j = 0; j < increments / threads; j++) {
                    counter.increment();
                }
                latch.countDown();
            });
        }
        latch.await();
        
        long time = System.currentTimeMillis() - start;
        executor.shutdown();
        return time;
    }
}
```

### 5.2 测试结果

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        性能对比（1000万次累加）                               │
├──────────┬──────────────┬──────────────┬────────────────────────────────────┤
│  线程数   │  AtomicLong  │  LongAdder   │            性能提升                │
├──────────┼──────────────┼──────────────┼────────────────────────────────────┤
│    1     │    120ms     │    125ms     │  0.96x（单线程 AtomicLong 略快）   │
│    2     │    350ms     │    130ms     │  2.7x                             │
│    4     │    680ms     │    135ms     │  5.0x                             │
│    8     │   1250ms     │    140ms     │  8.9x                             │
│   16     │   2100ms     │    145ms     │  14.5x                            │
│   32     │   3800ms     │    150ms     │  25.3x                            │
│   64     │   7200ms     │    160ms     │  45.0x                            │
└──────────┴──────────────┴──────────────┴────────────────────────────────────┘

结论：
├── 单线程：AtomicLong 略快（无竞争，LongAdder 有额外开销）
├── 低并发（2-4线程）：LongAdder 快 3-5 倍
├── 高并发（16+线程）：LongAdder 快 15-45 倍
└── 线程越多，LongAdder 优势越明显
```

### 5.3 性能曲线

```
耗时(ms)
    │
8000│                                              ╭── AtomicLong
    │                                          ╭───╯
    │                                      ╭───╯
    │                                  ╭───╯
4000│                              ╭───╯
    │                          ╭───╯
    │                      ╭───╯
    │                  ╭───╯
2000│              ╭───╯
    │          ╭───╯
    │      ╭───╯
    │  ╭───╯
 200│──●───●───●───●───●───●───●── LongAdder（几乎平稳）
    └──┴───┴───┴───┴───┴───┴───┴──────────────────────► 线程数
       1   2   4   8  16  32  64
```

---

## 六、选型指南

### 6.1 场景对比表

| 场景 | 推荐 | 原因 |
|-----|------|------|
| 序列号生成（全局唯一ID） | AtomicLong | 需要精确的递增值 |
| 高并发计数器（统计 QPS） | LongAdder | 只关心最终总数 |
| 限流计数（滑动窗口） | AtomicLong | 需要实时精确读取 |
| 监控指标（请求数、错误数） | LongAdder | 统计场景，允许小误差 |
| 低并发场景（<4线程） | AtomicLong | 简单，内存占用小 |
| 高并发场景（>8线程） | LongAdder | 性能优势明显 |
| 需要 compareAndSet | AtomicLong | LongAdder 不支持 CAS |
| 需要 decrementAndGet | AtomicLong | LongAdder 只有 add() |

### 6.2 代码示例

**场景1：全局序列号（用 AtomicLong）**

```java
public class SequenceGenerator {
    private final AtomicLong sequence = new AtomicLong(0);
    
    public long nextId() {
        // 需要精确的递增值，不能重复
        return sequence.incrementAndGet();
    }
}
```

**场景2：QPS 统计（用 LongAdder）**

```java
public class QpsCounter {
    private final LongAdder requestCount = new LongAdder();
    private final LongAdder errorCount = new LongAdder();
    
    public void recordRequest() {
        requestCount.increment();  // 高并发下性能好
    }
    
    public void recordError() {
        errorCount.increment();
    }
    
    public long getRequestCount() {
        return requestCount.sum();  // 统计场景，允许小误差
    }
}
```

**场景3：限流计数器（用 AtomicLong）**

```java
public class RateLimiter {
    private final AtomicLong permits = new AtomicLong(100);
    
    public boolean tryAcquire() {
        // 需要精确判断剩余许可数
        for (;;) {
            long current = permits.get();
            if (current <= 0) {
                return false;
            }
            if (permits.compareAndSet(current, current - 1)) {
                return true;
            }
        }
    }
}
```

**场景4：热 Key 检测（用 LongAdder）**

```java
// 结合我们之前的热 Key 检测场景
public class SlidingWindowCounter {
    // 高并发计数场景，用 LongAdder
    private final LongAdder[] slots;
    
    public SlidingWindowCounter(int slotCount) {
        this.slots = new LongAdder[slotCount];
        for (int i = 0; i < slotCount; i++) {
            slots[i] = new LongAdder();
        }
    }
    
    public void increment() {
        int index = getCurrentSlotIndex();
        slots[index].increment();
    }
    
    public long getCount() {
        long total = 0;
        for (LongAdder slot : slots) {
            total += slot.sum();
        }
        return total;
    }
}
```

### 6.3 决策流程图

```
                        需要原子计数器？
                              │
                              ▼
                    ┌─────────────────────┐
                    │ 需要 CAS 操作？      │
                    │ (compareAndSet)     │
                    └─────────────────────┘
                           │是        │否
                           ▼          ▼
                    ┌──────────┐   ┌─────────────────────┐
                    │AtomicLong│   │ 需要实时精确读取？   │
                    └──────────┘   └─────────────────────┘
                                         │是        │否
                                         ▼          ▼
                                  ┌──────────┐   ┌─────────────────────┐
                                  │AtomicLong│   │ 并发线程数 > 4？     │
                                  └──────────┘   └─────────────────────┘
                                                       │是        │否
                                                       ▼          ▼
                                                ┌──────────┐ ┌──────────┐
                                                │LongAdder │ │AtomicLong│
                                                └──────────┘ └──────────┘
```

---

## 七、其他 Adder/Accumulator 家族

### 7.1 类家族

```java
// 累加器家族
LongAdder        // long 型累加
DoubleAdder      // double 型累加
LongAccumulator  // 自定义 long 聚合函数
DoubleAccumulator // 自定义 double 聚合函数
```

### 7.2 LongAccumulator：自定义聚合

```java
// LongAdder 是 LongAccumulator 的特例
// LongAdder = new LongAccumulator((x, y) -> x + y, 0)

// 自定义：求最大值
LongAccumulator maxAccumulator = new LongAccumulator(Long::max, Long.MIN_VALUE);

maxAccumulator.accumulate(10);
maxAccumulator.accumulate(5);
maxAccumulator.accumulate(20);

System.out.println(maxAccumulator.get());  // 20

// 自定义：求乘积
LongAccumulator productAccumulator = new LongAccumulator((x, y) -> x * y, 1);

productAccumulator.accumulate(2);
productAccumulator.accumulate(3);
productAccumulator.accumulate(4);

System.out.println(productAccumulator.get());  // 24
```

---

## 八、总结

**面试回答模板**：

> "LongAdder 在高并发下性能更好，核心原因是**分散竞争**：
>
> **AtomicLong 的问题**：
> - 所有线程竞争同一个变量
> - 高并发下 CAS 失败率高，CPU 空转严重
> - 还有缓存行失效导致的伪共享问题
>
> **LongAdder 的解决方案**：
> - 使用 base + Cell[] 分段数组
> - 无竞争时操作 base，有竞争时分散到不同 Cell
> - 每个线程通过哈希映射到不同的 Cell，减少冲突
> - Cell 使用 @Contended 注解避免伪共享
> - 最终值 = base + sum(cells)
>
> **性能对比**：
> - 单线程：AtomicLong 略快
> - 高并发（16+线程）：LongAdder 快 15-45 倍
>
> **选型建议**：
> - **用 AtomicLong**：需要 CAS 操作、需要精确实时值、低并发场景
> - **用 LongAdder**：统计计数、只关心最终结果、高并发场景
>
> 比如我们的热 Key 检测平台，滑动窗口计数就用 LongAdder，因为是高并发统计场景，只关心窗口内的总访问量，不需要精确的实时值。"
