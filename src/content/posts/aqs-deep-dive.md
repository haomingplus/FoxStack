---
title: AQS核心原理深度解析
published: 2026-01-07
description: 深入剖析AQS的CLH队列、state变量、独占/共享模式，详解ReentrantLock、CountDownLatch、Semaphore的底层实现原理
tags: [Java, 并发编程, AQS, JUC, 锁机制]
category: Java并发编程
draft: false
---


## 一、AQS 概述

### 1.1 什么是 AQS

AQS（AbstractQueuedSynchronizer）是 Java 并发包 `java.util.concurrent.locks` 的核心基础框架，由 Doug Lea 设计。它提供了一套通用的机制来构建锁和同步器。

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        AQS 在并发包中的地位                                   │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│                         java.util.concurrent                                │
│    ┌─────────────────────────────────────────────────────────────────┐     │
│    │                                                                 │     │
│    │  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐ ┌───────────┐ │     │
│    │  │ReentrantLock│ │  Semaphore  │ │CountDownLatch│ │ ReadWrite │ │     │
│    │  │             │ │             │ │             │ │   Lock    │ │     │
│    │  └──────┬──────┘ └──────┬──────┘ └──────┬──────┘ └─────┬─────┘ │     │
│    │         │               │               │               │       │     │
│    │         └───────────────┴───────────────┴───────────────┘       │     │
│    │                                 │                               │     │
│    │                                 ▼                               │     │
│    │         ┌─────────────────────────────────────────────┐        │     │
│    │         │    AbstractQueuedSynchronizer (AQS)         │        │     │
│    │         │                                             │        │     │
│    │         │  • CLH 队列（等待队列）                       │        │     │
│    │         │  • state 状态变量                            │        │     │
│    │         │  • 独占/共享模式                             │        │     │
│    │         └─────────────────────────────────────────────┘        │     │
│    │                                                                 │     │
│    └─────────────────────────────────────────────────────────────────┘     │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 1.2 AQS 的设计思想

AQS 采用**模板方法模式**，将通用的排队、阻塞、唤醒等逻辑封装在抽象类中，子类只需实现特定的状态获取和释放逻辑。

```java
// AQS 的核心模板方法（简化）
public abstract class AbstractQueuedSynchronizer {
    
    // 子类需要实现的方法（模板方法的钩子）
    protected boolean tryAcquire(int arg)      // 独占式获取
    protected boolean tryRelease(int arg)      // 独占式释放
    protected int tryAcquireShared(int arg)    // 共享式获取
    protected boolean tryReleaseShared(int arg) // 共享式释放
    protected boolean isHeldExclusively()      // 是否被当前线程独占
    
    // AQS 提供的模板方法（已实现）
    public final void acquire(int arg)         // 独占式获取（阻塞）
    public final boolean release(int arg)      // 独占式释放
    public final void acquireShared(int arg)   // 共享式获取（阻塞）
    public final boolean releaseShared(int arg) // 共享式释放
}
```

---

## 二、AQS 三大核心组件

### 2.1 State 状态变量

#### 2.1.1 定义与作用

```java
/**
 * 同步状态，使用 volatile 保证可见性
 * 不同的同步器赋予它不同的语义
 */
private volatile int state;

// 三个访问方法
protected final int getState()           // 获取状态
protected final void setState(int s)     // 设置状态
protected final boolean compareAndSetState(int expect, int update)  // CAS 更新
```

#### 2.1.2 State 的不同语义

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        State 在不同同步器中的含义                             │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─────────────────┬────────────────────────────────────────────────────┐  │
│  │    同步器        │                  state 的含义                      │  │
│  ├─────────────────┼────────────────────────────────────────────────────┤  │
│  │ ReentrantLock   │ 0: 未锁定                                          │  │
│  │                 │ 1: 已锁定                                          │  │
│  │                 │ >1: 重入次数                                        │  │
│  ├─────────────────┼────────────────────────────────────────────────────┤  │
│  │ CountDownLatch  │ 计数器值，初始为 N                                   │  │
│  │                 │ 每次 countDown() 减 1                               │  │
│  │                 │ 为 0 时释放所有等待线程                              │  │
│  ├─────────────────┼────────────────────────────────────────────────────┤  │
│  │ Semaphore       │ 可用许可数                                          │  │
│  │                 │ acquire() 减少许可                                  │  │
│  │                 │ release() 增加许可                                  │  │
│  ├─────────────────┼────────────────────────────────────────────────────┤  │
│  │ ReentrantRead   │ 高16位: 读锁持有数量                                 │  │
│  │ WriteLock       │ 低16位: 写锁重入次数                                 │  │
│  └─────────────────┴────────────────────────────────────────────────────┘  │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 2.2 CLH 队列（等待队列）

#### 2.2.1 CLH 队列简介

CLH（Craig, Landin, and Hagersten）队列是一种基于链表的自旋锁队列。AQS 使用了 CLH 队列的变体来管理等待获取锁的线程。

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           CLH 队列结构                                       │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│        head                                                    tail         │
│          │                                                       │          │
│          ▼                                                       ▼          │
│       ┌──────┐    prev    ┌──────┐    prev    ┌──────┐    prev ┌──────┐    │
│       │ Node │ ←───────── │ Node │ ←───────── │ Node │ ←────── │ Node │    │
│       │(哨兵) │ ─────────→ │ (T1) │ ─────────→ │ (T2) │ ──────→ │ (T3) │    │
│       └──────┘    next    └──────┘    next    └──────┘    next └──────┘    │
│                                                                             │
│   说明：                                                                     │
│   • head 指向哨兵节点（已获取锁的线程或空节点）                                │
│   • tail 指向队尾（最后加入等待的线程）                                       │
│   • 双向链表，支持前向和后向遍历                                              │
│   • 新线程从 tail 入队，获取锁从 head 出队                                    │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

#### 2.2.2 Node 节点结构

```java
static final class Node {
    
    // 节点模式
    static final Node SHARED = new Node();    // 共享模式标记
    static final Node EXCLUSIVE = null;       // 独占模式标记
    
    // 等待状态 waitStatus 的取值
    static final int CANCELLED =  1;   // 线程已取消
    static final int SIGNAL    = -1;   // 后继节点需要被唤醒
    static final int CONDITION = -2;   // 在条件队列中等待
    static final int PROPAGATE = -3;   // 共享模式下无条件传播
    //                            0    // 初始状态
    
    volatile int waitStatus;           // 等待状态
    volatile Node prev;                // 前驱节点
    volatile Node next;                // 后继节点
    volatile Thread thread;            // 节点对应的线程
    Node nextWaiter;                   // 条件队列的下一个节点/模式标记
}
```

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         Node 的 waitStatus 状态                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   ┌───────────┬───────┬──────────────────────────────────────────────────┐ │
│   │   状态     │  值   │                     含义                         │ │
│   ├───────────┼───────┼──────────────────────────────────────────────────┤ │
│   │ CANCELLED │   1   │ 线程已取消等待（超时/中断），需要从队列移除        │ │
│   ├───────────┼───────┼──────────────────────────────────────────────────┤ │
│   │ SIGNAL    │  -1   │ 当前节点释放锁时，需要唤醒后继节点                 │ │
│   ├───────────┼───────┼──────────────────────────────────────────────────┤ │
│   │ CONDITION │  -2   │ 节点在 Condition 条件队列中等待                   │ │
│   ├───────────┼───────┼──────────────────────────────────────────────────┤ │
│   │ PROPAGATE │  -3   │ 共享模式下，释放操作需要向后传播                  │ │
│   ├───────────┼───────┼──────────────────────────────────────────────────┤ │
│   │ 初始状态   │   0   │ 新节点的默认状态                                  │ │
│   └───────────┴───────┴──────────────────────────────────────────────────┘ │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

#### 2.2.3 入队操作（addWaiter）

```java
private Node addWaiter(Node mode) {
    // 1. 创建新节点
    Node node = new Node(Thread.currentThread(), mode);
    
    // 2. 快速尝试添加到队尾
    Node pred = tail;
    if (pred != null) {
        node.prev = pred;
        if (compareAndSetTail(pred, node)) {  // CAS 设置 tail
            pred.next = node;
            return node;
        }
    }
    
    // 3. 快速添加失败，进入完整的入队流程
    enq(node);
    return node;
}

private Node enq(final Node node) {
    for (;;) {  // 自旋
        Node t = tail;
        if (t == null) {
            // 队列为空，初始化（创建哨兵节点）
            if (compareAndSetHead(new Node()))
                tail = head;
        } else {
            // 将节点添加到队尾
            node.prev = t;
            if (compareAndSetTail(t, node)) {
                t.next = node;
                return t;
            }
        }
    }
}
```

```
入队过程图示：

初始状态（队列为空）:
    head = null, tail = null

步骤1: 创建哨兵节点
    ┌──────┐
    │ Node │ ← head = tail
    │(哨兵) │
    └──────┘

步骤2: T1 入队
    ┌──────┐      ┌──────┐
    │ 哨兵  │ ←──→ │  T1  │ ← tail
    └──────┘      └──────┘
        ↑
      head

步骤3: T2 入队
    ┌──────┐      ┌──────┐      ┌──────┐
    │ 哨兵  │ ←──→ │  T1  │ ←──→ │  T2  │ ← tail
    └──────┘      └──────┘      └──────┘
        ↑
      head
```

### 2.3 独占模式与共享模式

#### 2.3.1 两种模式对比

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        独占模式 vs 共享模式                                   │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   ┌─────────────────────────────────────────────────────────────────────┐  │
│   │                          独占模式 (Exclusive)                        │  │
│   ├─────────────────────────────────────────────────────────────────────┤  │
│   │  • 同一时刻只有一个线程能获取锁                                       │  │
│   │  • 其他线程必须等待                                                   │  │
│   │  • 典型实现：ReentrantLock                                           │  │
│   │                                                                     │  │
│   │       ┌─────┐                                                       │  │
│   │       │Lock │ ←── Thread A (持有)                                   │  │
│   │       └─────┘                                                       │  │
│   │          ↑                                                          │  │
│   │     ┌────┴────┐                                                    │  │
│   │     │ Waiting │ ←── Thread B, C, D (等待)                           │  │
│   │     └─────────┘                                                    │  │
│   └─────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
│   ┌─────────────────────────────────────────────────────────────────────┐  │
│   │                          共享模式 (Shared)                           │  │
│   ├─────────────────────────────────────────────────────────────────────┤  │
│   │  • 多个线程可以同时获取锁/资源                                        │  │
│   │  • 获取成功后会尝试唤醒后续的共享节点                                  │  │
│   │  • 典型实现：Semaphore, CountDownLatch, ReadLock                     │  │
│   │                                                                     │  │
│   │       ┌──────────┐                                                  │  │
│   │       │ Resource │ ←── Thread A (持有)                              │  │
│   │       │ permits=3│ ←── Thread B (持有)                              │  │
│   │       └──────────┘ ←── Thread C (持有)                              │  │
│   │            ↑                                                        │  │
│   │       ┌────┴────┐                                                  │  │
│   │       │ Waiting │ ←── Thread D, E (等待，permits用完)               │  │
│   │       └─────────┘                                                  │  │
│   └─────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

#### 2.3.2 独占模式获取流程

```java
// acquire 方法 - 独占式获取
public final void acquire(int arg) {
    if (!tryAcquire(arg) &&                    // 1. 尝试获取
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))  // 2. 失败则入队等待
        selfInterrupt();                        // 3. 响应中断
}
```

```
                            acquire(1) 独占获取流程
                                     │
                                     ▼
                        ┌─────────────────────────┐
                        │ tryAcquire(1)           │
                        │ 尝试获取锁（子类实现）    │
                        └─────────────────────────┘
                                     │
                        ┌────────────┴────────────┐
                        │                         │
                        ▼                         ▼
                   ┌─────────┐              ┌─────────┐
                   │ 获取成功 │              │ 获取失败 │
                   │ 返回    │              │         │
                   └─────────┘              └─────────┘
                                                  │
                                                  ▼
                                    ┌─────────────────────────┐
                                    │ addWaiter(EXCLUSIVE)    │
                                    │ 将当前线程包装成Node入队  │
                                    └─────────────────────────┘
                                                  │
                                                  ▼
                                    ┌─────────────────────────┐
                                    │ acquireQueued(node,1)   │
                                    │ 在队列中自旋获取         │
                                    └─────────────────────────┘
                                                  │
                                                  ▼
                               ┌──────────────────────────────────┐
                               │ for(;;) {                        │
                               │   if (前驱是head && tryAcquire)  │
                               │     设置自己为head，返回          │
                               │   else                           │
                               │     检查是否需要阻塞              │
                               │     parkAndCheckInterrupt()      │
                               │ }                                │
                               └──────────────────────────────────┘
```

#### 2.3.3 共享模式获取流程

```java
// acquireShared 方法 - 共享式获取
public final void acquireShared(int arg) {
    if (tryAcquireShared(arg) < 0)             // 1. 尝试获取
        doAcquireShared(arg);                  // 2. 失败则入队等待
}
```

```
                         acquireShared(1) 共享获取流程
                                     │
                                     ▼
                        ┌─────────────────────────┐
                        │ tryAcquireShared(1)     │
                        │ 尝试获取（子类实现）      │
                        │ 返回值: <0失败 ≥0成功    │
                        └─────────────────────────┘
                                     │
                        ┌────────────┴────────────┐
                        │                         │
                        ▼                         ▼
                   ┌─────────┐              ┌─────────┐
                   │  ≥ 0    │              │  < 0    │
                   │ 获取成功 │              │ 获取失败 │
                   └─────────┘              └─────────┘
                                                  │
                                                  ▼
                                    ┌─────────────────────────┐
                                    │ doAcquireShared(1)      │
                                    │ 入队并等待              │
                                    └─────────────────────────┘
                                                  │
                                                  ▼
                              ┌───────────────────────────────────┐
                              │ for(;;) {                         │
                              │   if (前驱是head) {               │
                              │     r = tryAcquireShared()       │
                              │     if (r >= 0) {                │
                              │       setHeadAndPropagate()      │
                              │       // 关键：传播唤醒后继共享节点│
                              │       return                     │
                              │     }                            │
                              │   }                              │
                              │   parkAndCheckInterrupt()        │
                              │ }                                │
                              └───────────────────────────────────┘
```

---

## 三、ReentrantLock 的实现

### 3.1 整体结构

```java
public class ReentrantLock implements Lock {
    
    private final Sync sync;
    
    // 抽象内部类，继承 AQS
    abstract static class Sync extends AbstractQueuedSynchronizer {
        abstract void lock();
        // ... 公共方法
    }
    
    // 非公平锁实现
    static final class NonfairSync extends Sync { ... }
    
    // 公平锁实现
    static final class FairSync extends Sync { ... }
    
    // 默认非公平锁
    public ReentrantLock() {
        sync = new NonfairSync();
    }
    
    public ReentrantLock(boolean fair) {
        sync = fair ? new FairSync() : new NonfairSync();
    }
}
```

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         ReentrantLock 类结构                                 │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│                          ReentrantLock                                      │
│                               │                                             │
│                               │ has-a                                       │
│                               ▼                                             │
│                      ┌───────────────┐                                      │
│                      │     Sync      │ extends AQS                          │
│                      │   (abstract)  │                                      │
│                      └───────────────┘                                      │
│                           ↑       ↑                                         │
│                          ╱         ╲                                        │
│            ┌────────────┘           └────────────┐                          │
│            │                                     │                          │
│   ┌─────────────────┐                 ┌─────────────────┐                   │
│   │   NonfairSync   │                 │    FairSync     │                   │
│   │   (非公平锁)     │                 │    (公平锁)      │                   │
│   └─────────────────┘                 └─────────────────┘                   │
│                                                                             │
│   state 语义：                                                               │
│   • state = 0  → 锁空闲                                                     │
│   • state = 1  → 锁被占用                                                   │
│   • state > 1  → 锁被重入，值为重入次数                                       │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 3.2 非公平锁实现

```java
static final class NonfairSync extends Sync {
    
    // 加锁入口
    final void lock() {
        // 1. 直接尝试 CAS 抢锁（插队）
        if (compareAndSetState(0, 1))
            setExclusiveOwnerThread(Thread.currentThread());
        else
            // 2. 抢锁失败，走 AQS 流程
            acquire(1);
    }
    
    // 尝试获取锁
    protected final boolean tryAcquire(int acquires) {
        return nonfairTryAcquire(acquires);
    }
}

// Sync 类中的 nonfairTryAcquire
final boolean nonfairTryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    int c = getState();
    
    // 状态为0，锁空闲
    if (c == 0) {
        // CAS 尝试获取（不检查队列）
        if (compareAndSetState(0, acquires)) {
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    // 当前线程已持有锁，可重入
    else if (current == getExclusiveOwnerThread()) {
        int nextc = c + acquires;
        if (nextc < 0) // overflow
            throw new Error("Maximum lock count exceeded");
        setState(nextc);  // 不需要 CAS，因为是同一线程
        return true;
    }
    return false;
}
```

### 3.3 公平锁实现

```java
static final class FairSync extends Sync {
    
    final void lock() {
        // 不插队，直接走 AQS 流程
        acquire(1);
    }
    
    protected final boolean tryAcquire(int acquires) {
        final Thread current = Thread.currentThread();
        int c = getState();
        
        if (c == 0) {
            // 关键区别：先检查是否有前驱节点
            if (!hasQueuedPredecessors() &&
                compareAndSetState(0, acquires)) {
                setExclusiveOwnerThread(current);
                return true;
            }
        }
        else if (current == getExclusiveOwnerThread()) {
            int nextc = c + acquires;
            if (nextc < 0)
                throw new Error("Maximum lock count exceeded");
            setState(nextc);
            return true;
        }
        return false;
    }
}
```

### 3.4 公平锁 vs 非公平锁

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        公平锁 vs 非公平锁 对比                                │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   非公平锁（默认）:                                                          │
│   ┌─────────────────────────────────────────────────────────────────────┐  │
│   │                                                                     │  │
│   │     新线程 T4 到来                                                   │  │
│   │         │                                                          │  │
│   │         ▼                                                          │  │
│   │    直接尝试 CAS ──→ 成功则获得锁（插队）                              │  │
│   │         │                                                          │  │
│   │         ▼ 失败                                                     │  │
│   │   ┌──────┐   ┌──────┐   ┌──────┐                                  │  │
│   │   │  T1  │ → │  T2  │ → │  T3  │ → [T4入队]                        │  │
│   │   └──────┘   └──────┘   └──────┘                                  │  │
│   │                                                                     │  │
│   │   优点：吞吐量高（减少线程切换）                                      │  │
│   │   缺点：可能导致线程饥饿                                             │  │
│   └─────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
│   公平锁:                                                                   │
│   ┌─────────────────────────────────────────────────────────────────────┐  │
│   │                                                                     │  │
│   │     新线程 T4 到来                                                   │  │
│   │         │                                                          │  │
│   │         ▼                                                          │  │
│   │    检查队列是否有等待者 ──→ 有则直接入队（不插队）                     │  │
│   │         │                                                          │  │
│   │         ▼                                                          │  │
│   │   ┌──────┐   ┌──────┐   ┌──────┐   ┌──────┐                       │  │
│   │   │  T1  │ → │  T2  │ → │  T3  │ → │  T4  │                       │  │
│   │   └──────┘   └──────┘   └──────┘   └──────┘                       │  │
│   │                                                                     │  │
│   │   优点：FIFO，不会饥饿                                               │  │
│   │   缺点：吞吐量较低                                                   │  │
│   └─────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 3.5 释放锁

```java
// ReentrantLock.unlock()
public void unlock() {
    sync.release(1);
}

// AQS.release()
public final boolean release(int arg) {
    if (tryRelease(arg)) {          // 尝试释放
        Node h = head;
        if (h != null && h.waitStatus != 0)
            unparkSuccessor(h);     // 唤醒后继节点
        return true;
    }
    return false;
}

// Sync.tryRelease()
protected final boolean tryRelease(int releases) {
    int c = getState() - releases;
    
    // 只有持有锁的线程才能释放
    if (Thread.currentThread() != getExclusiveOwnerThread())
        throw new IllegalMonitorStateException();
    
    boolean free = false;
    if (c == 0) {                   // 完全释放
        free = true;
        setExclusiveOwnerThread(null);
    }
    setState(c);                    // 更新 state
    return free;
}
```

---

## 四、CountDownLatch 的实现

### 4.1 设计思想

CountDownLatch 是一个同步辅助类，允许一个或多个线程等待，直到其他线程中执行的一组操作完成。

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        CountDownLatch 工作原理                               │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   初始状态：state = 3 (计数器)                                               │
│                                                                             │
│           Main Thread                                                       │
│               │                                                             │
│               ▼                                                             │
│         ┌───────────┐                                                       │
│         │  await()  │ ─── 阻塞等待，直到 state = 0                          │
│         └───────────┘                                                       │
│                                                                             │
│                              state = 3                                      │
│                                  │                                          │
│    ┌─────────────────────────────┼─────────────────────────────┐           │
│    │                             │                             │           │
│    ▼                             ▼                             ▼           │
│ ┌──────┐                     ┌──────┐                     ┌──────┐        │
│ │Task 1│                     │Task 2│                     │Task 3│        │
│ └──────┘                     └──────┘                     └──────┘        │
│    │                             │                             │           │
│    ▼                             ▼                             ▼           │
│ countDown()                  countDown()                  countDown()      │
│ state: 3→2                   state: 2→1                   state: 1→0      │
│                                                               │           │
│                                                               ▼           │
│                                                    state = 0，唤醒主线程   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 4.2 源码实现

```java
public class CountDownLatch {
    
    private final Sync sync;
    
    // 内部类，继承 AQS
    private static final class Sync extends AbstractQueuedSynchronizer {
        
        Sync(int count) {
            setState(count);  // 初始化计数器
        }
        
        int getCount() {
            return getState();
        }
        
        // 共享模式尝试获取
        // await() 调用此方法
        protected int tryAcquireShared(int acquires) {
            // state == 0 返回1（获取成功）
            // state != 0 返回-1（获取失败，需要等待）
            return (getState() == 0) ? 1 : -1;
        }
        
        // 共享模式尝试释放
        // countDown() 调用此方法
        protected boolean tryReleaseShared(int releases) {
            for (;;) {
                int c = getState();
                if (c == 0)
                    return false;  // 已经是0了
                int nextc = c - 1;
                if (compareAndSetState(c, nextc))
                    return nextc == 0;  // 减到0时返回true，触发唤醒
            }
        }
    }
    
    public CountDownLatch(int count) {
        if (count < 0) throw new IllegalArgumentException("count < 0");
        this.sync = new Sync(count);
    }
    
    // 等待计数器归零
    public void await() throws InterruptedException {
        sync.acquireSharedInterruptibly(1);
    }
    
    // 计数器减1
    public void countDown() {
        sync.releaseShared(1);
    }
}
```

### 4.3 执行流程图

```
                          CountDownLatch 执行流程
                                    
    Thread A (await)                              Thread B, C, D (countDown)
         │                                                │
         ▼                                                │
  ┌─────────────────┐                                     │
  │ await()         │                                     │
  │ ↓               │                                     │
  │ acquireShared() │                                     │
  └────────┬────────┘                                     │
           │                                              │
           ▼                                              │
  ┌─────────────────────┐                                 │
  │ tryAcquireShared()  │                                 │
  │ state=3, return -1  │ ← 获取失败                       │
  └────────┬────────────┘                                 │
           │                                              │
           ▼                                              │
  ┌─────────────────────┐                                 │
  │ doAcquireShared()   │                                 │
  │ 入队并 park 阻塞    │                                  │
  └────────┬────────────┘                                 │
           │                                              │
           │ (阻塞中...)                                   │
           │                                    ┌─────────┴─────────┐
           │                                    │   countDown()     │
           │                                    │   state: 3→2→1→0  │
           │                                    └─────────┬─────────┘
           │                                              │
           │                                              ▼
           │                               ┌────────────────────────────┐
           │                               │ tryReleaseShared()        │
           │                               │ state=0, return true      │
           │                               │ 触发 doReleaseShared()    │
           │                               └─────────────┬──────────────┘
           │                                             │
           │ ◄────────────────── unpark ─────────────────┘
           │
           ▼
  ┌─────────────────────┐
  │ 被唤醒，重新检查      │
  │ tryAcquireShared()  │
  │ state=0, return 1   │ ← 获取成功
  └────────┬────────────┘
           │
           ▼
  ┌─────────────────┐
  │ await() 返回    │
  │ 继续执行        │
  └─────────────────┘
```

---

## 五、Semaphore 的实现

### 5.1 设计思想

Semaphore（信号量）控制同时访问特定资源的线程数量，通过协调各个线程以保证合理使用公共资源。

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          Semaphore 工作原理                                  │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   初始状态：permits = 3 (可用许可数)                                         │
│                                                                             │
│   ┌─────────────────────────────────────────────────────────────────────┐  │
│   │                         资源池 (permits=3)                           │  │
│   │   ┌─────────┐     ┌─────────┐     ┌─────────┐                      │  │
│   │   │ permit1 │     │ permit2 │     │ permit3 │                      │  │
│   │   └─────────┘     └─────────┘     └─────────┘                      │  │
│   └─────────────────────────────────────────────────────────────────────┘  │
│         ↑ acquire         ↑ acquire         ↑ acquire                      │
│         │                 │                 │                              │
│    ┌─────────┐       ┌─────────┐       ┌─────────┐                        │
│    │ Thread1 │       │ Thread2 │       │ Thread3 │                        │
│    └─────────┘       └─────────┘       └─────────┘                        │
│                                                                             │
│                          permits = 0 (已用完)                               │
│                                  │                                          │
│                                  ▼                                          │
│                         ┌─────────────────┐                                │
│                         │    Waiting      │                                │
│                         │  Thread4, T5... │ ← 等待许可释放                  │
│                         └─────────────────┘                                │
│                                                                             │
│   release() 操作：permits++ ，唤醒等待线程                                   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 5.2 源码实现

```java
public class Semaphore {
    
    private final Sync sync;
    
    abstract static class Sync extends AbstractQueuedSynchronizer {
        
        Sync(int permits) {
            setState(permits);  // 初始化许可数
        }
        
        final int getPermits() {
            return getState();
        }
        
        // 非公平尝试获取
        final int nonfairTryAcquireShared(int acquires) {
            for (;;) {
                int available = getState();
                int remaining = available - acquires;
                // 许可不足返回负数，足够则 CAS 扣减
                if (remaining < 0 ||
                    compareAndSetState(available, remaining))
                    return remaining;
            }
        }
        
        // 释放许可
        protected final boolean tryReleaseShared(int releases) {
            for (;;) {
                int current = getState();
                int next = current + releases;
                if (next < current) // overflow
                    throw new Error("Maximum permit count exceeded");
                if (compareAndSetState(current, next))
                    return true;
            }
        }
    }
    
    // 非公平同步器
    static final class NonfairSync extends Sync {
        NonfairSync(int permits) {
            super(permits);
        }
        
        protected int tryAcquireShared(int acquires) {
            return nonfairTryAcquireShared(acquires);
        }
    }
    
    // 公平同步器
    static final class FairSync extends Sync {
        FairSync(int permits) {
            super(permits);
        }
        
        protected int tryAcquireShared(int acquires) {
            for (;;) {
                // 公平：先检查队列
                if (hasQueuedPredecessors())
                    return -1;
                int available = getState();
                int remaining = available - acquires;
                if (remaining < 0 ||
                    compareAndSetState(available, remaining))
                    return remaining;
            }
        }
    }
    
    // 获取许可
    public void acquire() throws InterruptedException {
        sync.acquireSharedInterruptibly(1);
    }
    
    // 释放许可
    public void release() {
        sync.releaseShared(1);
    }
}
```

### 5.3 Semaphore vs CountDownLatch

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                   Semaphore vs CountDownLatch 对比                           │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   ┌─────────────────────┬──────────────────────┬──────────────────────┐    │
│   │       特性          │    CountDownLatch    │      Semaphore       │    │
│   ├─────────────────────┼──────────────────────┼──────────────────────┤    │
│   │ state 初始值        │ 计数值 N             │ 许可数 permits       │    │
│   ├─────────────────────┼──────────────────────┼──────────────────────┤    │
│   │ state 变化方向      │ 只能减少 (N→0)       │ 可增可减             │    │
│   ├─────────────────────┼──────────────────────┼──────────────────────┤    │
│   │ 是否可重用          │ ❌ 一次性            │ ✅ 可重复使用        │    │
│   ├─────────────────────┼──────────────────────┼──────────────────────┤    │
│   │ 主要用途            │ 等待事件完成         │ 控制并发数量         │    │
│   ├─────────────────────┼──────────────────────┼──────────────────────┤    │
│   │ 典型场景            │ 主线程等待子任务     │ 数据库连接池         │    │
│   │                     │ 完成后继续           │ 限流控制             │    │
│   └─────────────────────┴──────────────────────┴──────────────────────┘    │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 六、三种同步器对比总结

### 6.1 核心机制对比

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                      AQS 三大实现类对比                                       │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌───────────────┬─────────────────┬─────────────────┬─────────────────┐   │
│  │               │  ReentrantLock  │ CountDownLatch  │    Semaphore    │   │
│  ├───────────────┼─────────────────┼─────────────────┼─────────────────┤   │
│  │ 模式          │ 独占模式        │ 共享模式        │ 共享模式        │   │
│  ├───────────────┼─────────────────┼─────────────────┼─────────────────┤   │
│  │ state 含义    │ 锁状态/重入次数 │ 剩余计数        │ 可用许可数      │   │
│  ├───────────────┼─────────────────┼─────────────────┼─────────────────┤   │
│  │ state 初始值  │ 0               │ N (用户指定)    │ permits         │   │
│  ├───────────────┼─────────────────┼─────────────────┼─────────────────┤   │
│  │ 获取条件      │ state=0 或重入  │ state=0         │ state>0         │   │
│  ├───────────────┼─────────────────┼─────────────────┼─────────────────┤   │
│  │ 获取操作      │ state+1         │ 不改变 state    │ state-1         │   │
│  ├───────────────┼─────────────────┼─────────────────┼─────────────────┤   │
│  │ 释放操作      │ state-1         │ state-1         │ state+1         │   │
│  ├───────────────┼─────────────────┼─────────────────┼─────────────────┤   │
│  │ 公平/非公平   │ 支持            │ 不支持          │ 支持            │   │
│  ├───────────────┼─────────────────┼─────────────────┼─────────────────┤   │
│  │ 可重入        │ ✅              │ N/A             │ N/A             │   │
│  ├───────────────┼─────────────────┼─────────────────┼─────────────────┤   │
│  │ 可重用        │ ✅              │ ❌              │ ✅              │   │
│  └───────────────┴─────────────────┴─────────────────┴─────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 6.2 使用场景总结

```java
// 1. ReentrantLock - 互斥访问共享资源
ReentrantLock lock = new ReentrantLock();
lock.lock();
try {
    // 临界区代码
} finally {
    lock.unlock();
}

// 2. CountDownLatch - 等待一组操作完成
CountDownLatch latch = new CountDownLatch(3);

// 工作线程
for (int i = 0; i < 3; i++) {
    executor.submit(() -> {
        doWork();
        latch.countDown();  // 完成后计数减1
    });
}

latch.await();  // 主线程等待
System.out.println("所有任务完成");

// 3. Semaphore - 限制并发访问数量
Semaphore semaphore = new Semaphore(10);  // 最多10个并发

semaphore.acquire();  // 获取许可
try {
    accessDatabase();  // 访问数据库
} finally {
    semaphore.release();  // 释放许可
}
```

---

## 七、面试高频问题

### Q1: AQS 为什么使用 CLH 队列的变体而不是原始 CLH？

> **原始 CLH**：自旋锁，线程在前驱节点上自旋
> **AQS 变体**：
> - 改为双向链表，支持取消操作
> - 不完全自旋，会阻塞线程（park）
> - 支持独占和共享两种模式

### Q2: 为什么 AQS 使用 state 而不是 boolean？

> - 支持可重入（记录重入次数）
> - 支持共享模式（记录资源数量）
> - 支持读写锁（高16位读锁，低16位写锁）

### Q3: ReentrantLock 和 synchronized 的区别？

| 特性 | ReentrantLock | synchronized |
|------|---------------|--------------|
| 实现 | JDK/AQS | JVM/字节码 |
| 公平性 | 可选 | 非公平 |
| 可中断 | ✅ | ❌ |
| 超时获取 | ✅ | ❌ |
| 多条件 | ✅ (多个Condition) | ❌ (单个wait/notify) |
| 性能 | 相当 | 相当 |

### Q4: CountDownLatch 能否重置？

> 不能。CountDownLatch 是一次性的，计数到0后不能重置。如需重复使用，考虑 CyclicBarrier。
