---
title: TP99从198ms优化到75ms实战全解析
published: 2023-11-08
description: 结合营销交易平台50W QPS实战经验，从三级缓存、线程池调优、GC优化、链路治理、跨机房优化五大维度详解尾延迟治理方法论
tags: [Java, 性能优化, TP99, JVM调优, 线程池, 缓存, 高并发]
category: 性能优化
draft: false
---

## 面试题

TP99 从 198ms 优化到 75ms，具体做了哪些事？

---

## 一、问题背景

### 1.1 优化前的现状

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           优化前核心指标                                     │
├─────────────────────┬───────────────────────────────────────────────────────┤
│ TP50                │  15ms                                                 │
│ TP95                │  82ms                                                 │
│ TP99                │  198ms  ← 严重超标                                    │
│ TP999               │  500ms+                                               │
│ 峰值 QPS            │  约 15W（目标 50W）                                    │
│ 大促故障            │  频繁触发限流降级                                      │
└─────────────────────┴───────────────────────────────────────────────────────┘

问题分析：
├── TP50 和 TP99 差距巨大（13倍），说明存在严重的尾延迟问题
├── 尾延迟导致用户体验差、超时重试放大流量
└── 无法支撑双11/618大促流量
```

### 1.2 优化目标

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           优化目标                                          │
├─────────────────────┬───────────────────────────────────────────────────────┤
│ TP95                │  82ms → 30ms 以内                                     │
│ TP99                │  198ms → 80ms 以内                                    │
│ 峰值 QPS            │  15W → 50W+                                           │
│ 大促表现            │  零降级触发                                            │
└─────────────────────┴───────────────────────────────────────────────────────┘
```

### 1.3 优化成果总览

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        优化措施与收益汇总                                    │
├──────────────────────┬──────────────────┬───────────────────────────────────┤
│        优化方向       │     TP99 收益    │              关键动作              │
├──────────────────────┼──────────────────┼───────────────────────────────────┤
│ 1. 三级缓存体系       │    -50ms         │ Caffeine+Redis+MySQL，命中率98%+  │
│ 2. 线程池精细化调优   │    -25ms         │ 隔离+参数调优+监控                 │
│ 3. GC 深度调优        │    -20ms         │ G1参数优化，STW减少80%            │
│ 4. 链路治理           │    -18ms         │ 接口合并+异步化+超时治理           │
│ 5. 跨机房优化         │    -10ms         │ 就近访问+同机房优先                │
├──────────────────────┼──────────────────┼───────────────────────────────────┤
│ 合计                  │   -123ms         │ 198ms → 75ms                     │
└──────────────────────┴──────────────────┴───────────────────────────────────┘
```

---

## 二、优化一：三级缓存体系（-50ms）

### 2.1 问题分析

```
优化前：
┌─────────────┐         ┌─────────────┐
│   应用层     │ ──────→ │   Redis     │ ──────→  RT: 2-5ms
└─────────────┘         └─────────────┘
                              │
                              │ Miss (约20%)
                              ▼
                        ┌─────────────┐
                        │   MySQL     │ ──────→  RT: 10-50ms
                        └─────────────┘

问题：
├── Redis 命中率只有 80%，20% 请求穿透到 DB
├── 热点数据每次都要走网络到 Redis，增加 2-5ms
├── 数据库慢查询拖高整体 RT
└── 大促期间 Redis 压力大，响应时间飙升
```

### 2.2 优化方案

```
优化后：三级缓存
┌─────────────┐     ┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│   应用层     │ ──→ │ L1 Caffeine │ ──→ │ L2 Redis    │ ──→ │ L3 MySQL    │
└─────────────┘     └─────────────┘     └─────────────┘     └─────────────┘
                      命中率 60%          命中率 38%          命中率 2%
                      RT: 0.05ms          RT: 1-3ms           RT: 5-20ms

整体效果：
├── 缓存命中率：80% → 98%+
├── 平均 RT：8ms → 1ms
└── TP99 贡献：-50ms
```

### 2.3 关键代码

```java
@Service
public class ProductCacheService {

    // L1: 本地缓存 - 热点数据
    private final Cache<String, Product> localCache = Caffeine.newBuilder()
            .maximumSize(100_000)
            .expireAfterWrite(30, TimeUnit.SECONDS)
            .build();

    public Product getProduct(Long productId) {
        String key = "product:" + productId;
        
        // L1: 本地缓存（0.05ms）
        Product product = localCache.getIfPresent(key);
        if (product != null) {
            return product;
        }
        
        // L2: Redis（1-3ms）
        String json = redis.get(key);
        if (json != null) {
            product = JSON.parseObject(json, Product.class);
            localCache.put(key, product);  // 回填 L1
            return product;
        }
        
        // L3: MySQL（5-20ms）
        product = loadFromDBWithLock(productId, key);
        return product;
    }
}
```

### 2.4 量化收益

| 指标 | 优化前 | 优化后 | 收益 |
|-----|-------|-------|------|
| 缓存命中率 | 80% | 98%+ | +18% |
| 平均 RT | 8ms | 1ms | -7ms |
| TP99 贡献 | - | - | **-50ms** |
| Redis QPS | 50W | 20W | -60% |

---

## 三、优化二：线程池精细化调优（-25ms）

### 3.1 问题分析

```
优化前线程池配置：
┌─────────────────────────────────────────────────────────────────────────────┐
│  所有业务共用一个线程池                                                       │
│  ├── corePoolSize: 50                                                       │
│  ├── maxPoolSize: 200                                                       │
│  ├── queueCapacity: 1000  ← 队列过长，任务排队严重                            │
│  └── keepAliveTime: 60s                                                     │
└─────────────────────────────────────────────────────────────────────────────┘

问题：
├── 慢任务（如发MQ、写日志）阻塞快任务（如查缓存）
├── 队列堆积导致任务排队等待，RT 飙升
├── 线程数不足时，新任务直接进队列等待
└── 无法隔离故障，一个下游慢导致全部慢
```

**火焰图分析发现**：

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         火焰图热点分析                                       │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  █████████████████████████████████████  40% - 业务逻辑                      │
│  ████████████████████                   25% - 线程等待（排队）← 问题！       │
│  ███████████████                        18% - 网络 IO                       │
│  ████████                               10% - JSON 序列化                   │
│  ████                                    7% - GC                            │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 3.2 优化方案

**方案一：线程池隔离**

```java
@Configuration
public class ThreadPoolConfig {

    /**
     * 核心业务线程池：查询商品、活动等高频操作
     * 特点：快任务，RT 要求 < 10ms
     */
    @Bean("coreBusinessPool")
    public ThreadPoolExecutor coreBusinessPool() {
        return new ThreadPoolExecutor(
            100,                              // corePoolSize: 保持足够的核心线程
            200,                              // maxPoolSize
            60, TimeUnit.SECONDS,
            new SynchronousQueue<>(),         // 不排队，直接创建新线程或拒绝
            new ThreadFactoryBuilder().setNameFormat("core-biz-%d").build(),
            new CallerRunsPolicy()            // 拒绝时由调用线程执行
        );
    }

    /**
     * IO密集型线程池：调用下游RPC、发MQ等
     * 特点：慢任务，RT 可能 10-100ms
     */
    @Bean("ioPool")
    public ThreadPoolExecutor ioPool() {
        return new ThreadPoolExecutor(
            200,                              // IO 密集型，线程数可以多一些
            400,
            60, TimeUnit.SECONDS,
            new LinkedBlockingQueue<>(500),   // 允许短暂排队
            new ThreadFactoryBuilder().setNameFormat("io-pool-%d").build(),
            new AbortPolicy()                 // 拒绝时快速失败
        );
    }

    /**
     * 非核心业务线程池：写日志、统计埋点等
     * 特点：可降级，不影响主流程
     */
    @Bean("asyncPool")
    public ThreadPoolExecutor asyncPool() {
        return new ThreadPoolExecutor(
            20,
            50,
            60, TimeUnit.SECONDS,
            new LinkedBlockingQueue<>(10000), // 大队列，允许堆积
            new ThreadFactoryBuilder().setNameFormat("async-%d").build(),
            new DiscardOldestPolicy()         // 满了丢弃最老的
        );
    }
}
```

**方案二：参数精细化调优**

```java
/**
 * 核心线程池参数计算
 */
public class ThreadPoolCalculator {

    /**
     * 计算最优线程数
     * 
     * CPU 密集型：N + 1（N = CPU核心数）
     * IO 密集型：N * (1 + WT/ST)
     *   - WT: 等待时间（网络IO、磁盘IO）
     *   - ST: 计算时间
     */
    public static int calculateOptimalThreads(double waitTime, double computeTime) {
        int cpuCores = Runtime.getRuntime().availableProcessors();
        double ratio = waitTime / computeTime;
        return (int) (cpuCores * (1 + ratio));
    }
    
    // 示例：假设 IO 等待 80ms，计算 20ms
    // 最优线程数 = 8 * (1 + 80/20) = 8 * 5 = 40
}
```

**方案三：动态监控与调整**

```java
@Component
public class ThreadPoolMonitor {

    @Scheduled(fixedRate = 10000)
    public void monitor() {
        monitorPool("coreBusinessPool", coreBusinessPool);
        monitorPool("ioPool", ioPool);
    }

    private void monitorPool(String name, ThreadPoolExecutor pool) {
        int activeCount = pool.getActiveCount();
        int poolSize = pool.getPoolSize();
        int queueSize = pool.getQueue().size();
        long completedTasks = pool.getCompletedTaskCount();
        
        // 关键指标
        double utilization = (double) activeCount / poolSize;
        
        log.info("ThreadPool[{}] active={}, poolSize={}, queue={}, completed={}, utilization={}%",
                name, activeCount, poolSize, queueSize, completedTasks, 
                String.format("%.2f", utilization * 100));
        
        // 告警：队列堆积
        if (queueSize > 100) {
            alertService.send("线程池队列堆积告警: " + name + ", queueSize=" + queueSize);
        }
        
        // 告警：利用率过高
        if (utilization > 0.8) {
            alertService.send("线程池利用率过高: " + name + ", utilization=" + utilization);
        }
    }
}
```

### 3.3 量化收益

| 指标 | 优化前 | 优化后 | 收益 |
|-----|-------|-------|------|
| 线程等待时间占比 | 25% | 5% | -20% |
| 队列平均长度 | 200+ | <10 | -95% |
| TP99 贡献 | - | - | **-25ms** |

---

## 四、优化三：GC 深度调优（-20ms）

### 4.1 问题分析

```
优化前 GC 状况：
┌─────────────────────────────────────────────────────────────────────────────┐
│  GC 收集器：G1                                                               │
│  堆大小：8G                                                                  │
│  Young GC：每秒 3-5 次，单次 15-30ms                                         │
│  Mixed GC：每分钟 1-2 次，单次 50-100ms                                      │
│  Full GC：偶发，单次 500ms-2s  ← 导致 TP999 飙升                             │
└─────────────────────────────────────────────────────────────────────────────┘

GC 日志分析：
[GC pause (G1 Evacuation Pause) (young), 0.0256789 secs]
   [Parallel Time: 22.5 ms, GC Workers: 8]
   [Eden: 2048.0M(2048.0M)->0.0B(1856.0M) Survivors: 128.0M->320.0M Heap: 5120.0M(8192.0M)->3200.0M(8192.0M)]
   
问题发现：
├── Eden 区过大（2G），单次回收对象多，STW 时间长
├── Survivor 区频繁溢出，对象过早晋升到 Old 区
├── Mixed GC 触发频繁，回收效率低
└── 大对象直接进入 Old 区，增加 Full GC 风险
```

### 4.2 优化方案

**核心 JVM 参数调优**：

```bash
# 优化前
-Xms8g -Xmx8g -XX:+UseG1GC

# 优化后
-Xms8g -Xmx8g
-XX:+UseG1GC
-XX:MaxGCPauseMillis=50            # 目标停顿时间 50ms（默认200ms）
-XX:G1HeapRegionSize=16m           # Region 大小 16M（大对象阈值 = 8M）
-XX:G1NewSizePercent=30            # 新生代最小占比 30%
-XX:G1MaxNewSizePercent=50         # 新生代最大占比 50%
-XX:InitiatingHeapOccupancyPercent=40  # 老年代占用 40% 时触发并发标记
-XX:G1MixedGCLiveThresholdPercent=85   # Region 存活对象超过 85% 不回收
-XX:G1ReservePercent=15            # 预留 15% 空间防止晋升失败
-XX:ParallelGCThreads=8            # 并行 GC 线程数
-XX:ConcGCThreads=4                # 并发标记线程数
-XX:+ParallelRefProcEnabled        # 并行处理 Reference
```

**参数调优思路**：

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        G1 调优核心思路                                       │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  1. 降低单次停顿时间                                                         │
│     └── MaxGCPauseMillis=50（让 G1 自动调整 Eden 大小）                       │
│                                                                             │
│  2. 减少大对象直接进入 Old 区                                                 │
│     └── G1HeapRegionSize=16m（大对象阈值 = 8M）                              │
│                                                                             │
│  3. 提前触发并发标记，避免 Full GC                                            │
│     └── InitiatingHeapOccupancyPercent=40                                   │
│                                                                             │
│  4. 优化新生代比例                                                           │
│     └── 30%-50%，避免频繁 Young GC 或对象过早晋升                             │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 4.3 代码层面优化

```java
/**
 * 减少对象创建，降低 GC 压力
 */
public class GCOptimization {

    // 1. 复用对象：使用对象池
    private static final ObjectPool<StringBuilder> SB_POOL = 
        new GenericObjectPool<>(new StringBuilderFactory());

    public String buildResponse(List<Product> products) {
        StringBuilder sb = SB_POOL.borrowObject();
        try {
            for (Product p : products) {
                sb.append(p.getName()).append(",");
            }
            return sb.toString();
        } finally {
            sb.setLength(0);  // 重置
            SB_POOL.returnObject(sb);
        }
    }

    // 2. 避免装箱拆箱：使用原始类型集合
    // 使用 Eclipse Collections 或 Trove
    private TLongObjectHashMap<Product> productCache = new TLongObjectHashMap<>();

    // 3. 预分配集合容量
    public List<Product> queryProducts(List<Long> ids) {
        List<Product> result = new ArrayList<>(ids.size());  // 预分配
        // ...
        return result;
    }

    // 4. 使用 StringBuilder 而非 String 拼接
    public String concat(String a, String b, String c) {
        return new StringBuilder(a.length() + b.length() + c.length())
                .append(a).append(b).append(c)
                .toString();
    }
}
```

### 4.4 量化收益

| 指标 | 优化前 | 优化后 | 收益 |
|-----|-------|-------|------|
| Young GC 频率 | 3-5次/秒 | 1-2次/秒 | -60% |
| Young GC 耗时 | 15-30ms | 8-15ms | -50% |
| Mixed GC 频率 | 1-2次/分 | 0.5次/分 | -60% |
| Full GC | 偶发 | 基本消除 | -99% |
| TP99 贡献 | - | - | **-20ms** |

---

## 五、优化四：链路治理（-18ms）

### 5.1 问题分析

```
优化前调用链路：

┌─────────┐    ┌─────────┐    ┌─────────┐    ┌─────────┐    ┌─────────┐
│ 网关    │ →  │ 商品服务 │ →  │ 价格服务 │ →  │ 库存服务 │ →  │ 活动服务 │
└─────────┘    └─────────┘    └─────────┘    └─────────┘    └─────────┘
                    │              │              │              │
                    │              │              │              │
                    ▼              ▼              ▼              ▼
               ┌─────────┐   ┌─────────┐   ┌─────────┐   ┌─────────┐
               │ 评价服务 │   │ 优惠券  │   │ 红包服务 │   │ 营销服务 │
               └─────────┘   └─────────┘   └─────────┘   └─────────┘

问题：
├── 调用深度 5-7 层，RT 逐层放大
├── 串行调用过多，无法并行
├── 非核心调用（如评价）阻塞主流程
├── 超时配置不合理，下游慢拖垮上游
└── 单个接口返回数据过多，网络开销大
```

### 5.2 优化方案

**方案一：接口合并，减少网络调用**

```java
/**
 * 优化前：7次 RPC 调用
 */
public ProductDetailVO getProductDetail_Before(Long productId) {
    Product product = productService.getById(productId);        // RPC 1
    Price price = priceService.getPrice(productId);             // RPC 2
    Stock stock = stockService.getStock(productId);             // RPC 3
    List<Coupon> coupons = couponService.list(productId);       // RPC 4
    Activity activity = activityService.get(productId);         // RPC 5
    List<Review> reviews = reviewService.list(productId);       // RPC 6
    Promotion promotion = promotionService.get(productId);      // RPC 7
    
    return assemble(product, price, stock, coupons, activity, reviews, promotion);
}

/**
 * 优化后：3次 RPC 调用（批量 + 并行）
 */
public ProductDetailVO getProductDetail_After(Long productId) {
    // 批量接口：一次获取商品+价格+库存
    ProductAggregateDTO aggregate = productAggregateService
        .getAggregate(productId);  // RPC 1（合并了3个接口）
    
    // 并行调用非核心数据
    CompletableFuture<List<Coupon>> couponsFuture = 
        CompletableFuture.supplyAsync(() -> couponService.list(productId), ioPool);
    CompletableFuture<Activity> activityFuture = 
        CompletableFuture.supplyAsync(() -> activityService.get(productId), ioPool);
    
    // 等待并行结果（设置超时）
    List<Coupon> coupons = couponsFuture.get(50, TimeUnit.MILLISECONDS);
    Activity activity = activityFuture.get(50, TimeUnit.MILLISECONDS);
    
    // 非核心数据异步获取，不阻塞主流程
    asyncPool.execute(() -> {
        reviewService.list(productId);  // 异步加载评价
    });
    
    return assemble(aggregate, coupons, activity);
}
```

**方案二：异步化改造**

```java
/**
 * 核心流程同步，非核心流程异步
 */
@Service
public class OrderService {

    public OrderResult createOrder(OrderRequest request) {
        // 1. 核心流程：同步执行
        Order order = doCreateOrder(request);           // 创建订单
        deductStock(order);                             // 扣减库存
        deductCoupon(order);                            // 使用优惠券
        
        // 2. 非核心流程：异步执行（不影响主流程 RT）
        asyncExecute(order);
        
        return OrderResult.success(order);
    }
    
    @Async("asyncPool")
    public void asyncExecute(Order order) {
        sendOrderMessage(order);      // 发送 MQ 消息
        updateUserPoints(order);      // 更新积分
        sendNotification(order);      // 发送通知
        recordBehavior(order);        // 记录行为日志
    }
}
```

**方案三：超时治理**

```java
/**
 * 分层超时配置
 */
@Configuration
public class TimeoutConfig {

    /**
     * 超时配置原则：
     * 1. 上游超时 > 下游超时（避免上游超时但下游还在执行）
     * 2. 核心依赖超时短，非核心依赖超时可以长
     * 3. 总超时 < 用户体验阈值（如 500ms）
     */
    
    // 核心服务：超时短
    @Bean
    public FeignClient productClient() {
        return FeignClient.builder()
            .connectTimeout(100)   // 连接超时 100ms
            .readTimeout(200)      // 读取超时 200ms
            .build();
    }
    
    // 非核心服务：超时可以略长，但要有降级
    @Bean
    public FeignClient reviewClient() {
        return FeignClient.builder()
            .connectTimeout(100)
            .readTimeout(500)
            .fallback(ReviewClientFallback.class)  // 降级
            .build();
    }
}

/**
 * 超时分层设计
 */
┌─────────────────────────────────────────────────────────────────────────────┐
│                           超时分层配置                                       │
├──────────────────┬─────────────────┬────────────────────────────────────────┤
│      层级         │     超时时间     │                 说明                  │
├──────────────────┼─────────────────┼────────────────────────────────────────┤
│ 网关层           │     800ms       │ 最外层，兜底超时                        │
│ BFF 层           │     500ms       │ 聚合层超时                             │
│ 核心服务         │     200ms       │ 商品/库存/价格                          │
│ 非核心服务       │     300ms       │ 评价/推荐（带降级）                      │
│ 缓存访问         │      50ms       │ Redis/本地缓存                         │
│ 数据库访问       │     100ms       │ MySQL 查询                             │
└──────────────────┴─────────────────┴────────────────────────────────────────┘
```

### 5.3 量化收益

| 指标 | 优化前 | 优化后 | 收益 |
|-----|-------|-------|------|
| RPC 调用次数 | 7次 | 3次 | -57% |
| 链路深度 | 5-7层 | 3-4层 | -40% |
| 串行等待时间 | 50ms | 20ms | -30ms |
| TP99 贡献 | - | - | **-18ms** |

---

## 六、优化五：跨机房优化（-10ms）

### 6.1 问题分析

```
优化前网络拓扑：

        ┌─────────────────────────────────────────────────────────────────┐
        │                      阿里云 - 华东                               │
        │  ┌─────────────┐                        ┌─────────────┐        │
        │  │  机房 A     │                        │  机房 B     │        │
        │  │  ─────────  │                        │  ─────────  │        │
        │  │  应用服务器  │ ◄───── 跨机房调用 ────► │  Redis 主   │        │
        │  │             │       RT: 2-5ms        │             │        │
        │  └─────────────┘                        └─────────────┘        │
        └─────────────────────────────────────────────────────────────────┘

问题：
├── 应用服务器和 Redis 不在同一机房，每次访问增加 2-5ms
├── 跨机房网络抖动时，RT 飙升到 10-20ms
├── 流量调度不合理，请求随机路由到各机房
└── 没有就近访问策略
```

### 6.2 优化方案

**方案一：同机房优先访问**

```java
/**
 * 机房感知的 Redis 客户端
 */
@Configuration
public class ZoneAwareRedisConfig {

    @Value("${deploy.zone}")
    private String currentZone;  // 当前部署机房：zone-a / zone-b

    @Bean
    public LettuceClientConfiguration lettuceClientConfiguration() {
        // 读写分离 + 同机房优先
        return LettuceClientConfiguration.builder()
            .readFrom(ReadFrom.REPLICA_PREFERRED)  // 优先从副本读
            .build();
    }

    /**
     * 自定义读策略：同机房副本优先
     */
    @Bean
    public ReadFrom zoneAwareReadFrom() {
        return new ReadFrom() {
            @Override
            public List<RedisNodeDescription> select(Nodes nodes) {
                // 1. 优先选择同机房的副本
                List<RedisNodeDescription> sameZoneReplicas = nodes.getNodes().stream()
                    .filter(node -> node.getRole() == RedisInstance.Role.SLAVE)
                    .filter(node -> currentZone.equals(getNodeZone(node)))
                    .collect(Collectors.toList());
                
                if (!sameZoneReplicas.isEmpty()) {
                    return sameZoneReplicas;
                }
                
                // 2. 没有同机房副本，选择任意副本
                return nodes.getNodes().stream()
                    .filter(node -> node.getRole() == RedisInstance.Role.SLAVE)
                    .collect(Collectors.toList());
            }
        };
    }
}
```

**方案二：服务调用同机房优先**

```java
/**
 * 同机房优先的负载均衡策略
 */
public class ZoneAwareLoadBalancer implements ReactorServiceInstanceLoadBalancer {

    private final String currentZone;

    @Override
    public Mono<Response<ServiceInstance>> choose(Request request) {
        return serviceInstanceListSupplier.get()
            .next()
            .map(instances -> {
                // 1. 筛选同机房实例
                List<ServiceInstance> sameZoneInstances = instances.stream()
                    .filter(i -> currentZone.equals(i.getMetadata().get("zone")))
                    .collect(Collectors.toList());
                
                // 2. 同机房有实例则优先使用
                if (!sameZoneInstances.isEmpty()) {
                    return selectInstance(sameZoneInstances);
                }
                
                // 3. 否则使用任意实例（跨机房）
                return selectInstance(instances);
            });
    }
}
```

**方案三：部署架构优化**

```
优化后网络拓扑：

        ┌─────────────────────────────────────────────────────────────────┐
        │                      阿里云 - 华东                               │
        │                                                                 │
        │  ┌─────────────────────────┐   ┌─────────────────────────┐     │
        │  │       机房 A            │   │       机房 B            │     │
        │  │  ┌─────────────────┐   │   │  ┌─────────────────┐   │     │
        │  │  │   应用服务器     │   │   │  │   应用服务器     │   │     │
        │  │  └────────┬────────┘   │   │  └────────┬────────┘   │     │
        │  │           │            │   │           │            │     │
        │  │           ▼ 同机房访问  │   │           ▼ 同机房访问  │     │
        │  │  ┌─────────────────┐   │   │  ┌─────────────────┐   │     │
        │  │  │   Redis 从节点  │◄──┼───┼──│   Redis 主节点  │   │     │
        │  │  │   (只读副本)    │ 主从 │   │                 │   │     │
        │  │  └─────────────────┘ 同步 │   └─────────────────┘   │     │
        │  └─────────────────────────┘   └─────────────────────────┘     │
        │                                                                 │
        └─────────────────────────────────────────────────────────────────┘

优化效果：
├── 读请求走同机房 Redis 从节点，RT: 0.5-1ms
├── 写请求走 Redis 主节点（可接受跨机房延迟）
└── 服务间调用同机房优先
```

### 6.3 量化收益

| 指标 | 优化前 | 优化后 | 收益 |
|-----|-------|-------|------|
| Redis 访问 RT | 2-5ms | 0.5-1ms | -70% |
| 跨机房调用占比 | 50% | 10% | -80% |
| TP99 贡献 | - | - | **-10ms** |

---

## 七、优化效果总结

### 7.1 最终成果

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           优化成果对比                                       │
├─────────────────────┬───────────────────┬───────────────────────────────────┤
│        指标          │      优化前       │              优化后               │
├─────────────────────┼───────────────────┼───────────────────────────────────┤
│ TP50                │      15ms         │              8ms                 │
│ TP95                │      82ms         │              29ms ✓              │
│ TP99                │      198ms        │              75ms ✓              │
│ TP999               │      500ms+       │              150ms               │
│ 峰值 QPS            │      15W          │              50W+ ✓              │
│ 缓存命中率          │      80%          │              98%+                │
│ 大促表现            │   频繁降级触发     │           零降级触发 ✓            │
└─────────────────────┴───────────────────┴───────────────────────────────────┘
```

### 7.2 优化收益拆解

```
TP99: 198ms → 75ms（降低 123ms）

├── 三级缓存体系      -50ms (41%)  ████████████████████
├── 线程池精细化调优   -25ms (20%)  ██████████
├── GC 深度调优       -20ms (16%)  ████████
├── 链路治理          -18ms (15%)  ███████
└── 跨机房优化        -10ms (8%)   ████
```

---

## 八、面试回答模板

> "TP99 从 198ms 优化到 75ms，我们从五个维度入手：
>
> **1. 三级缓存体系（-50ms）**
> 构建 Caffeine + Redis + MySQL 三级缓存，命中率从 80% 提升到 98%+。本地缓存解决热点问题，减少网络开销。
>
> **2. 线程池精细化调优（-25ms）**
> 将共用线程池拆分为核心业务池、IO 密集池、异步任务池，实现故障隔离。核心业务池使用 SynchronousQueue 避免排队。
>
> **3. GC 深度调优（-20ms）**
> 通过 GC 日志和火焰图分析，调优 G1 参数：降低 MaxGCPauseMillis 到 50ms，调整新生代比例，Young GC 耗时降低 50%。
>
> **4. 链路治理（-18ms）**
> 合并 7 个接口为 3 个批量接口，非核心调用异步化，分层设置超时时间，链路深度从 5-7 层降到 3-4 层。
>
> **5. 跨机房优化（-10ms）**
> 实现同机房优先访问策略，Redis 读请求走本机房从节点，服务调用优先路由到同机房实例，跨机房调用占比从 50% 降到 10%。
>
> 最终 TP99 从 198ms 降到 75ms，TP95 从 82ms 降到 29ms，系统稳定支撑 50W+ QPS，双11 大促零降级触发。"
