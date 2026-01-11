---
title: 热Key检测平台设计与LRU+滑动窗口算法实现
published: 2024-01-15
description: 结合营销交易平台实战经验，详解热Key检测的核心算法设计、架构实现及自动本地缓存方案
tags: [Java, Redis, 热Key, LRU, 滑动窗口, 高并发, 缓存]
category: 系统架构
draft: false
---

## 面试题

你提到自研了热 Key 检测平台，LRU + 滑动窗口算法是怎么实现的？

---

## 一、问题背景

### 1.1 什么是热 Key 问题？

```
热 Key：短时间内被大量访问的缓存 Key

营销交易平台典型热 Key 场景：
├── 爆款商品详情（双11 主推商品）
├── 秒杀活动配置（限时抢购）
├── 热门红包/优惠券（全场通用券）
├── 明星店铺信息（头部商家）
└── 首页推荐数据（所有用户访问）
```

### 1.2 热 Key 的危害

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           热 Key 危害示意图                                  │
└─────────────────────────────────────────────────────────────────────────────┘

正常情况：请求均匀分布到 Redis 集群各节点
┌─────────┐     ┌─────────┐     ┌─────────┐
│ Node 1  │     │ Node 2  │     │ Node 3  │
│  3W QPS │     │  3W QPS │     │  3W QPS │
└─────────┘     └─────────┘     └─────────┘

热 Key 场景：某个 Key 的请求全部打到同一节点
┌─────────┐     ┌─────────┐     ┌─────────┐
│ Node 1  │     │ Node 2  │     │ Node 3  │
│  10W QPS│ ←── │  1W QPS │     │  1W QPS │
│  ⚠️ 过载 │     └─────────┘     └─────────┘
└─────────┘
     │
     ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│ 后果：                                                                       │
│ ├── 单节点 CPU 100%，响应变慢                                                │
│ ├── 连接数打满，新请求被拒绝                                                  │
│ ├── 触发集群故障转移，雪崩风险                                                │
│ └── 大促期间引发 P0 故障                                                     │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 1.3 我们遇到的真实案例

```
双11 预热期间：

事件：某爆款商品上了首页推荐位
现象：商品详情 Key（product:12345）QPS 突破 10W
后果：
├── Redis 单节点 CPU 飙升到 95%
├── 该节点响应时间从 1ms 飙升到 50ms
├── 关联的其他 Key 查询也变慢
└── 触发限流，用户投诉激增

根因：热 Key 未识别，未做本地缓存
```

---

## 二、为什么选择 LRU + 滑动窗口？

### 2.1 方案对比

| 方案 | 原理 | 优点 | 缺点 |
|-----|------|------|------|
| **Redis MONITOR** | 监听所有命令 | 简单直接 | 性能影响 50%+，禁止生产使用 |
| **Redis HOTKEYS** | `--hotkeys` 扫描 | 官方支持 | 需要开启 LFU，扫描阻塞 |
| **客户端采样上报** | 埋点统计 | 准确 | 侵入性强，改造成本高 |
| **Proxy 层统计** | 代理层计数 | 无侵入 | 需要代理层支持 |
| **本地实时计算** ⭐ | LRU + 滑动窗口 | 实时、准确、低开销 | 需要自研 |

### 2.2 设计思路

```
问题一：内存有限，不能统计所有 Key（百万级）
────────────────────────────────────────────
解决：LRU（Least Recently Used）
├── 只保留最近访问的 N 个 Key 的统计信息
├── 冷 Key 自动淘汰，不占用内存
└── 热 Key 访问频繁，天然会留在 LRU 中

问题二：需要统计"单位时间内"的访问量
────────────────────────────────────────────
解决：滑动窗口
├── 统计最近 N 秒内的访问次数
├── 窗口滑动，旧数据自动过期
└── 避免瞬时流量抖动误判

两者结合：
LRU 解决"统计哪些 Key"
滑动窗口解决"怎么统计访问量"
```

---

## 三、整体架构

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        热 Key 检测平台架构                                   │
└─────────────────────────────────────────────────────────────────────────────┘

                            ┌─────────────────┐
                            │   业务请求      │
                            └────────┬────────┘
                                     │
                                     ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                           应用服务器                                         │
│                                                                             │
│   ┌───────────────────────────────────────────────────────────────────┐    │
│   │                     热 Key 检测模块                                │    │
│   │                                                                   │    │
│   │   ┌─────────────────┐                                            │    │
│   │   │  LRU Cache      │  Key: "product:123"                        │    │
│   │   │  (10000 条)     │  Value: SlidingWindowCounter               │    │
│   │   │                 │                                            │    │
│   │   │  ┌───────────┐  │   ┌─────────────────────────────────────┐ │    │
│   │   │  │Key1 → 计数器│─┼──→│ 滑动窗口: [50,45,60,55,70,80,90,85] │ │    │
│   │   │  │Key2 → 计数器│  │   │ 10秒窗口，每秒一个槽位              │ │    │
│   │   │  │Key3 → 计数器│  │   │ 总计数 = 535 > 500 → 热Key!        │ │    │
│   │   │  │...         │  │   └─────────────────────────────────────┘ │    │
│   │   │  └───────────┘  │                                            │    │
│   │   └─────────────────┘                                            │    │
│   │            │                                                      │    │
│   │            │ 检测到热 Key                                         │    │
│   │            ▼                                                      │    │
│   │   ┌─────────────────┐                                            │    │
│   │   │ 自动写入本地缓存 │                                            │    │
│   │   │ (Caffeine)      │                                            │    │
│   │   └─────────────────┘                                            │    │
│   └───────────────────────────────────────────────────────────────────┘    │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 四、核心算法实现

### 4.1 滑动窗口计数器

```java
/**
 * 滑动窗口计数器
 * 基于环形数组实现，统计最近 N 秒内的访问次数
 */
public class SlidingWindowCounter {

    // 窗口大小（秒）
    private final int windowSizeInSeconds;
    
    // 槽位数量（每个槽位代表的时间片）
    private final int slotCount;
    
    // 每个槽位的时间跨度（毫秒）
    private final long slotSizeInMs;
    
    // 环形数组：存储每个槽位的计数
    private final AtomicLongArray counts;
    
    // 环形数组：存储每个槽位的时间戳
    private final AtomicLongArray timestamps;

    /**
     * 构造函数
     * @param windowSizeInSeconds 窗口大小，如 10 秒
     * @param slotCount 槽位数量，如 10 个（每秒一个槽）
     */
    public SlidingWindowCounter(int windowSizeInSeconds, int slotCount) {
        this.windowSizeInSeconds = windowSizeInSeconds;
        this.slotCount = slotCount;
        this.slotSizeInMs = (windowSizeInSeconds * 1000L) / slotCount;
        this.counts = new AtomicLongArray(slotCount);
        this.timestamps = new AtomicLongArray(slotCount);
    }

    /**
     * 记录一次访问
     */
    public void increment() {
        long now = System.currentTimeMillis();
        int slotIndex = getSlotIndex(now);
        long slotStartTime = getSlotStartTime(now);
        
        // 检查当前槽位是否过期
        long oldTimestamp = timestamps.get(slotIndex);
        
        if (slotStartTime != oldTimestamp) {
            // 槽位过期，需要重置
            // CAS 更新时间戳，成功者负责重置计数
            if (timestamps.compareAndSet(slotIndex, oldTimestamp, slotStartTime)) {
                counts.set(slotIndex, 1);
            } else {
                // CAS 失败，其他线程已重置，直接累加
                counts.incrementAndGet(slotIndex);
            }
        } else {
            // 槽位未过期，直接累加
            counts.incrementAndGet(slotIndex);
        }
    }

    /**
     * 获取窗口内的总访问次数
     */
    public long getCount() {
        long now = System.currentTimeMillis();
        long windowStart = now - (windowSizeInSeconds * 1000L);
        long total = 0;
        
        for (int i = 0; i < slotCount; i++) {
            // 只统计窗口内的槽位
            if (timestamps.get(i) >= windowStart) {
                total += counts.get(i);
            }
        }
        
        return total;
    }

    /**
     * 计算时间戳对应的槽位索引
     */
    private int getSlotIndex(long timeMs) {
        // 环形数组：时间戳对槽位数量取模
        return (int) ((timeMs / slotSizeInMs) % slotCount);
    }

    /**
     * 计算槽位的起始时间
     */
    private long getSlotStartTime(long timeMs) {
        return (timeMs / slotSizeInMs) * slotSizeInMs;
    }
}
```

### 4.2 滑动窗口工作原理图解

```
滑动窗口示意（10秒窗口，10个槽位，每槽1秒）

时间轴：
  0s    1s    2s    3s    4s    5s    6s    7s    8s    9s    10s   11s   12s
  │     │     │     │     │     │     │     │     │     │     │     │     │

═══════════════════════════════════════════════════════════════════════════════

【T=5s 时刻】窗口范围：0s ~ 5s
┌─────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┐
│  50 │  45 │  60 │  55 │  70 │  80 │  -  │  -  │  -  │  -  │
│ [0] │ [1] │ [2] │ [3] │ [4] │ [5] │ [6] │ [7] │ [8] │ [9] │
└─────┴─────┴─────┴─────┴─────┴─────┴─────┴─────┴─────┴─────┘
  0s    1s    2s    3s    4s    5s
  └──────────── 有效窗口 ────────────┘

总计数 = 50+45+60+55+70+80 = 360

═══════════════════════════════════════════════════════════════════════════════

【T=8s 时刻】窗口范围：-2s ~ 8s（实际 0s ~ 8s）
┌─────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┐
│ 过期│ 过期│  60 │  55 │  70 │  80 │  90 │  85 │  75 │  -  │
│ [0] │ [1] │ [2] │ [3] │ [4] │ [5] │ [6] │ [7] │ [8] │ [9] │
└─────┴─────┴─────┴─────┴─────┴─────┴─────┴─────┴─────┴─────┘
              2s    3s    4s    5s    6s    7s    8s
              └────────────── 有效窗口 ──────────────┘

总计数 = 60+55+70+80+90+85+75 = 515 > 500 → 热 Key！

═══════════════════════════════════════════════════════════════════════════════

【T=12s 时刻】环形复用，槽位 [0][1][2] 存储 10s/11s/12s 数据
┌─────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┐
│  65 │  70 │  60 │ 过期│ 过期│ 过期│  90 │  85 │  75 │  80 │
│ [0] │ [1] │ [2] │ [3] │ [4] │ [5] │ [6] │ [7] │ [8] │ [9] │
│ 10s │ 11s │ 12s │  3s │  4s │  5s │  6s │  7s │  8s │  9s │
└─────┴─────┴─────┴─────┴─────┴─────┴─────┴─────┴─────┴─────┘
  └─────────┘                         └─────────────────────┘
   新数据复用槽位                           有效窗口
```

### 4.3 热 Key 检测器（LRU + 滑动窗口整合）

```java
/**
 * 热 Key 检测器
 * 结合 LRU 缓存和滑动窗口计数
 */
public class HotKeyDetector {

    /**
     * LRU 缓存：Key → 滑动窗口计数器
     * 最多保留 maxKeyCount 个 Key 的统计信息
     */
    private final Cache<String, SlidingWindowCounter> keyCounterCache;
    
    // 热 Key 阈值（窗口内访问次数）
    private final int hotKeyThreshold;
    
    // 窗口大小（秒）
    private final int windowSeconds;
    
    // 已识别的热 Key（防止重复回调）
    private final Cache<String, Boolean> identifiedHotKeys;
    
    // 热 Key 回调处理器
    private final Consumer<HotKeyEvent> hotKeyHandler;

    public HotKeyDetector(int maxKeyCount, int hotKeyThreshold, 
                          int windowSeconds, Consumer<HotKeyEvent> hotKeyHandler) {
        this.hotKeyThreshold = hotKeyThreshold;
        this.windowSeconds = windowSeconds;
        this.hotKeyHandler = hotKeyHandler;
        
        // LRU 缓存配置
        this.keyCounterCache = Caffeine.newBuilder()
                .maximumSize(maxKeyCount)                           // LRU 淘汰
                .expireAfterAccess(windowSeconds * 2, TimeUnit.SECONDS)  // 长时间不访问则过期
                .build();
        
        // 已识别热 Key 缓存（60秒内不重复触发）
        this.identifiedHotKeys = Caffeine.newBuilder()
                .maximumSize(1000)
                .expireAfterWrite(60, TimeUnit.SECONDS)
                .build();
    }

    /**
     * 记录 Key 访问
     * @return 是否为热 Key
     */
    public boolean recordAccess(String key) {
        // 1. 从 LRU 缓存获取或创建计数器
        SlidingWindowCounter counter = keyCounterCache.get(key, k -> 
                new SlidingWindowCounter(windowSeconds, windowSeconds));
        
        // 2. 滑动窗口计数 +1
        counter.increment();
        
        // 3. 获取窗口内总访问量
        long count = counter.getCount();
        
        // 4. 判断是否达到热 Key 阈值
        if (count >= hotKeyThreshold) {
            // 检查是否已经识别过（避免重复触发）
            if (identifiedHotKeys.getIfPresent(key) == null) {
                identifiedHotKeys.put(key, true);
                
                // 触发热 Key 事件
                HotKeyEvent event = new HotKeyEvent(key, count, count / windowSeconds);
                hotKeyHandler.accept(event);
                
                return true;
            }
        }
        
        return identifiedHotKeys.getIfPresent(key) != null;
    }

    /**
     * 获取 Key 当前访问量
     */
    public long getKeyCount(String key) {
        SlidingWindowCounter counter = keyCounterCache.getIfPresent(key);
        return counter != null ? counter.getCount() : 0;
    }

    /**
     * 获取 Top N 热 Key
     */
    public List<HotKeyInfo> getTopHotKeys(int n) {
        return keyCounterCache.asMap().entrySet().stream()
                .map(e -> new HotKeyInfo(e.getKey(), e.getValue().getCount()))
                .sorted((a, b) -> Long.compare(b.getCount(), a.getCount()))
                .limit(n)
                .collect(Collectors.toList());
    }

    @Data
    @AllArgsConstructor
    public static class HotKeyEvent {
        private String key;
        private long count;      // 窗口内访问次数
        private long qps;        // 平均 QPS
    }

    @Data
    @AllArgsConstructor
    public static class HotKeyInfo {
        private String key;
        private long count;
    }
}
```

### 4.4 集成到缓存服务

```java
/**
 * 热 Key 感知的缓存服务
 */
@Service
@Slf4j
public class HotKeyAwareCacheService {

    // 本地缓存：存储热 Key 数据
    private final Cache<String, String> localCache = Caffeine.newBuilder()
            .maximumSize(10_000)
            .expireAfterWrite(10, TimeUnit.SECONDS)  // 热 Key 本地缓存 10 秒
            .build();

    // 热 Key 检测器
    private final HotKeyDetector hotKeyDetector;

    @Autowired
    private StringRedisTemplate redisTemplate;

    public HotKeyAwareCacheService() {
        // 初始化检测器：10000 Key 容量，10秒内超500次为热 Key
        this.hotKeyDetector = new HotKeyDetector(
                10000,              // maxKeyCount: LRU 最多统计 1 万个 Key
                500,                // hotKeyThreshold: 10秒内 500 次
                10,                 // windowSeconds: 10 秒窗口
                this::onHotKeyDetected  // 热 Key 回调
        );
    }

    /**
     * 获取缓存（集成热 Key 检测）
     */
    public String get(String key) {
        // 1. 记录访问并检测热 Key
        boolean isHotKey = hotKeyDetector.recordAccess(key);
        
        // 2. 优先查本地缓存（热 Key 会在这里）
        String value = localCache.getIfPresent(key);
        if (value != null) {
            return value;
        }
        
        // 3. 查 Redis
        value = redisTemplate.opsForValue().get(key);
        if (value == null) {
            return null;
        }
        
        // 4. 如果是热 Key，写入本地缓存
        if (isHotKey) {
            localCache.put(key, value);
            log.info("热Key已缓存到本地: {}", key);
        }
        
        return value;
    }

    /**
     * 热 Key 检测回调
     */
    private void onHotKeyDetected(HotKeyDetector.HotKeyEvent event) {
        log.warn("【热Key告警】key={}, count={}, qps={}", 
                event.getKey(), event.getCount(), event.getQps());
        
        // 1. 立即加载到本地缓存
        String value = redisTemplate.opsForValue().get(event.getKey());
        if (value != null) {
            localCache.put(event.getKey(), value);
        }
        
        // 2. 发送监控指标
        Metrics.counter("hotkey.detected", "key", event.getKey()).increment();
        
        // 3. 发送告警（可选）
        if (event.getQps() > 1000) {
            alertService.send(String.format("极热Key告警: %s, QPS: %d", 
                    event.getKey(), event.getQps()));
        }
    }
}
```

---

## 五、LRU 为什么能筛选出热 Key？

### 5.1 LRU 的天然过滤特性

```
LRU (Least Recently Used) 淘汰策略：

访问越频繁的 Key → 越容易留在缓存中
访问越稀疏的 Key → 越容易被淘汰

┌─────────────────────────────────────────────────────────────────────────────┐
│                          LRU 工作示意                                        │
└─────────────────────────────────────────────────────────────────────────────┘

假设 LRU 容量 = 5

初始状态：[K1, K2, K3, K4, K5]
                │
                ▼
访问 K3：     [K3, K1, K2, K4, K5]     ← K3 移到头部
                │
                ▼
访问 K6：     [K6, K3, K1, K2, K4]     ← K6 进入，K5 被淘汰
                │
                ▼
连续访问K6：  [K6, K3, K1, K2, K4]     ← K6 一直在头部，不会被淘汰

结论：
├── 热 Key（频繁访问）→ 一直在 LRU 中，被统计到
└── 冷 Key（偶尔访问）→ 很快被淘汰，不占用内存
```

### 5.2 为什么不用 HashMap？

```
问题：系统有 100 万个 Key，全部统计内存爆炸

HashMap 方案：
├── 每个 Key 一个计数器
├── 100 万 Key × 250 字节 = 250 MB
└── 而且大部分是冷 Key，统计了也没用

LRU 方案：
├── 只保留最近访问的 1 万个 Key
├── 1 万 Key × 250 字节 = 2.5 MB
├── 冷 Key 自动淘汰
└── 热 Key 一定在 LRU 中
```

---

## 六、参数如何确定？

### 6.1 热 Key 阈值（500）

```
确定方法：

1. 基于 Redis 节点容量
   └── 单节点安全 QPS ≈ 5-8 万
   └── 单 Key QPS > 5000 就可能有风险
   └── 10 秒内 > 500 次 → QPS > 50 → 开始关注

2. 基于历史数据
   └── 日常 Top Key QPS：200-300
   └── 大促 Top Key QPS：2000-5000
   └── 超过 500/10秒 = 50 QPS 属于偏高

3. 实践调整
   └── 初始设 500，观察误报率
   └── 误报多（正常Key被判为热Key）→ 调高
   └── 漏报多（热Key没检测到）→ 调低
```

### 6.2 LRU 容量（10000）

```
确定方法：

1. 内存估算
   └── 单个 Key 平均 50 字节
   └── 滑动窗口计数器约 200 字节
   └── 单条 ≈ 250 字节
   └── 10000 条 ≈ 2.5 MB ✓ 可接受

2. 业务估算
   └── 系统活跃 Key（每分钟访问过）约 5 万
   └── 热 Key 候选（高频访问）约 5000-10000
   └── 设置 10000 可覆盖所有候选

3. 监控验证
   └── 观察 LRU 淘汰率
   └── 淘汰率高 → 容量不足 → 调大
   └── 实际淘汰率 < 5%，说明 10000 足够
```

### 6.3 窗口大小（10秒）

```
确定方法：

窗口太小（如 1 秒）：
└── 易受瞬时抖动影响，误报率高
└── 某个 Key 瞬间被访问 100 次就报热 Key

窗口太大（如 60 秒）：
└── 检测延迟大，热 Key 识别不及时
└── 突发热点无法快速响应

10 秒窗口：
└── 平衡实时性和稳定性
└── 10 秒内持续高频才判定为热 Key
└── 足够快速响应突发热点
```

---

## 七、常见追问

### Q1：为什么用环形数组而不是 TreeMap？

```java
// TreeMap 方案
TreeMap<Long, AtomicLong> slots;  // 时间戳 → 计数

// 问题：
// 1. 内存不固定，可能膨胀
// 2. 需要定期清理过期数据
// 3. TreeMap 操作有锁开销

// 环形数组方案
AtomicLongArray counts;      // 固定大小
AtomicLongArray timestamps;  // 固定大小

// 优势：
// 1. 内存固定，不会膨胀
// 2. 自动覆盖旧数据，无需清理
// 3. CAS 无锁操作
// 4. 连续内存，缓存友好
```

### Q2：多线程并发安全怎么保证？

```java
// 关键点：使用 AtomicLongArray + CAS

public void increment() {
    long now = System.currentTimeMillis();
    int slotIndex = getSlotIndex(now);
    long slotStartTime = getSlotStartTime(now);
    
    long oldTimestamp = timestamps.get(slotIndex);
    
    if (slotStartTime != oldTimestamp) {
        // CAS 更新时间戳
        if (timestamps.compareAndSet(slotIndex, oldTimestamp, slotStartTime)) {
            // CAS 成功，负责重置计数
            counts.set(slotIndex, 1);
        } else {
            // CAS 失败，其他线程已重置，直接累加
            counts.incrementAndGet(slotIndex);
        }
    } else {
        // 时间戳未变，直接累加（原子操作）
        counts.incrementAndGet(slotIndex);
    }
}

// 无锁设计，高并发下性能优秀
```

### Q3：热 Key 误判怎么办？

```
误判场景：
└── 某 Key 瞬时被访问 500 次，但只持续 1 秒
└── 被误判为热 Key，浪费本地缓存空间

解决方案：

1. 连续 N 个窗口超阈值才判定
   if (currentCount > threshold && lastWindowCount > threshold) {
       markAsHotKey(key);
   }

2. 本地缓存时间短（10秒），误判影响有限

3. 监控误报率，动态调整阈值
```

### Q4：集群场景下各实例统计不一致怎么办？

```
问题：10 台服务器，每台只统计到 1/10 的流量

方案一：各实例独立检测
└── 阈值设为总阈值的 1/N
└── 如集群 10 台，阈值从 500 改为 50
└── 简单有效

方案二：集群汇总
└── 各实例定期上报到 Redis
└── 汇总后统一判定
└── 更准确但实现复杂

我们采用方案一，简单且效果好
```

---

## 八、总结

**面试回答模板**：

> "我们自研的热 Key 检测平台，核心是 **LRU + 滑动窗口** 算法：
>
> **LRU 解决"统计哪些 Key"**：
> - 内存有限，不能统计全部 Key
> - 使用 Caffeine LRU 缓存，只保留最近访问的 1 万个 Key
> - 冷 Key 自动淘汰，热 Key 天然会留在 LRU 中
>
> **滑动窗口解决"怎么统计访问量"**：
> - 用环形数组实现 10 秒滑动窗口
> - 分 10 个槽位，每槽 1 秒
> - 窗口滑动时旧数据自动失效
> - CAS 无锁操作，高并发安全
>
> **检测流程**：
> 1. 每次 Redis 访问，从 LRU 获取该 Key 的计数器
> 2. 滑动窗口当前槽位 +1
> 3. 计算窗口内总访问量
> 4. 超过阈值（10 秒内 500 次）则判定为热 Key
> 5. 自动将热 Key 数据加载到本地缓存
>
> **参数确定**：
> - 阈值 500：基于 Redis 容量和历史数据，逐步调优
> - LRU 容量 1 万：覆盖热 Key 候选，内存约 2.5MB
> - 窗口 10 秒：平衡实时性和稳定性
>
> **效果**：秒级识别热 Key，Redis 单节点压力下降 60%，大促零热 Key 故障。"
