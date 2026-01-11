---
title: 三级缓存一致性方案设计与实战
published: 2024-03-22
description: 结合营销交易平台50W QPS实战经验，深入解析Caffeine+Redis+MySQL三级缓存架构的一致性保障策略
tags: [Java, 缓存, Redis, Caffeine, 数据一致性, 高并发, 架构设计]
category: 系统架构
draft: false
---

## 面试题

你在项目中设计的三级缓存体系（Caffeine + Redis + MySQL）是如何保证数据一致性的？

---

## 一、项目背景回顾

### 1.1 业务场景

```
营销交易平台核心数据：
├── 商品信息（名称、价格、图片）
├── 红包/优惠券（面额、有效期、使用规则）
├── 库存数据（商品库存、券库存）
├── 活动配置（满减规则、限购数量）
└── 用户资产（红包余额、积分）

流量特点：
├── 峰值 50W+ QPS
├── 读写比约 100:1（读多写少）
├── 大促期间脉冲流量（红包/券领取）
└── 热点数据集中（爆款商品、主推活动）
```

### 1.2 为什么需要三级缓存？

```
单纯依赖 Redis 的问题：

┌─────────────────────────────────────────────────────────────────────────────┐
│  50W QPS 全部打到 Redis                                                     │
│                                                                             │
│  问题1：网络开销大，RT 增加 1-3ms                                            │
│  问题2：热 Key 导致单节点 QPS > 10W，集群倾斜                                 │
│  问题3：Redis 抖动时，流量直接穿透到 DB                                       │
└─────────────────────────────────────────────────────────────────────────────┘

三级缓存的价值：

┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│  L1 本地缓存 │ ──→ │  L2 Redis   │ ──→ │  L3 MySQL   │
│  Caffeine   │     │  Cluster    │     │  主从集群    │
└─────────────┘     └─────────────┘     └─────────────┘
    命中率60%           命中率38%            命中率2%
    RT: 0.01ms          RT: 1-3ms           RT: 5-20ms

整体效果：
├── 98%+ 缓存命中率
├── 平均 RT < 1ms
├── Redis 压力降低 60%
└── 抗住 50W+ QPS
```

---

## 二、三级缓存架构设计

### 2.1 整体架构

```
                              请求流量
                                 │
                                 ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                            应用服务器集群                                    │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                     L1: Caffeine 本地缓存                            │   │
│  │                                                                     │   │
│  │   特点：                                                             │   │
│  │   ├── 每个 JVM 实例独立                                              │   │
│  │   ├── 容量小（10W 条）                                               │   │
│  │   ├── 过期时间短（10-60s）                                           │   │
│  │   └── 零网络开销，RT < 0.1ms                                         │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                 │ Miss                                     │
│                                 ▼                                          │
└─────────────────────────────────────────────────────────────────────────────┘
                                 │
                                 ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                         L2: Redis Cluster                                   │
│                                                                             │
│   特点：                                                                    │
│   ├── 分布式共享                                                            │
│   ├── 容量大（百万级）                                                       │
│   ├── 过期时间中等（5-30min）                                                │
│   └── RT: 1-3ms                                                            │
└─────────────────────────────────────────────────────────────────────────────┘
                                 │ Miss
                                 ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                         L3: MySQL 主从集群                                   │
│                                                                             │
│   特点：                                                                    │
│   ├── 数据源，保证数据正确性                                                 │
│   ├── 读写分离                                                              │
│   └── RT: 5-20ms                                                           │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 2.2 代码实现

```java
@Service
public class ProductCacheService {

    // L1: Caffeine 本地缓存
    private final Cache<String, Product> localCache = Caffeine.newBuilder()
            .maximumSize(100_000)                    // 最大 10W 条
            .expireAfterWrite(30, TimeUnit.SECONDS) // 写入后 30s 过期
            .recordStats()                          // 开启统计
            .build();

    @Autowired
    private StringRedisTemplate redisTemplate;
    
    @Autowired
    private ProductMapper productMapper;

    // Redis 过期时间：5分钟 + 随机偏移（防止缓存雪崩）
    private static final long REDIS_TTL_SECONDS = 300;
    private static final long REDIS_TTL_RANDOM = 60;

    /**
     * 三级缓存读取
     */
    public Product getProduct(Long productId) {
        String cacheKey = "product:" + productId;

        // L1: 查本地缓存
        Product product = localCache.getIfPresent(cacheKey);
        if (product != null) {
            return product;
        }

        // L2: 查 Redis
        String json = redisTemplate.opsForValue().get(cacheKey);
        if (StringUtils.isNotBlank(json)) {
            product = JSON.parseObject(json, Product.class);
            // 回填 L1
            localCache.put(cacheKey, product);
            return product;
        }

        // L3: 查数据库（需要加锁防止缓存击穿）
        product = loadFromDBWithLock(productId, cacheKey);
        return product;
    }

    /**
     * 加锁查询 DB，防止缓存击穿
     */
    private Product loadFromDBWithLock(Long productId, String cacheKey) {
        String lockKey = "lock:" + cacheKey;
        RLock lock = redissonClient.getLock(lockKey);
        
        try {
            // 尝试获取锁，最多等待 100ms
            if (lock.tryLock(100, 10000, TimeUnit.MILLISECONDS)) {
                try {
                    // 双重检查：可能其他线程已经加载
                    String json = redisTemplate.opsForValue().get(cacheKey);
                    if (StringUtils.isNotBlank(json)) {
                        Product product = JSON.parseObject(json, Product.class);
                        localCache.put(cacheKey, product);
                        return product;
                    }

                    // 查询数据库
                    Product product = productMapper.selectById(productId);
                    
                    if (product != null) {
                        // 回填 L2 Redis（随机过期时间）
                        long ttl = REDIS_TTL_SECONDS + RandomUtils.nextLong(0, REDIS_TTL_RANDOM);
                        redisTemplate.opsForValue().set(cacheKey, JSON.toJSONString(product), 
                                                        ttl, TimeUnit.SECONDS);
                        // 回填 L1
                        localCache.put(cacheKey, product);
                    } else {
                        // 缓存空值，防止缓存穿透
                        redisTemplate.opsForValue().set(cacheKey, "", 60, TimeUnit.SECONDS);
                    }
                    
                    return product;
                } finally {
                    lock.unlock();
                }
            } else {
                // 获取锁失败，短暂等待后重试读缓存
                Thread.sleep(50);
                String json = redisTemplate.opsForValue().get(cacheKey);
                if (StringUtils.isNotBlank(json)) {
                    return JSON.parseObject(json, Product.class);
                }
                return null;
            }
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            return null;
        }
    }
}
```

---

## 三、一致性问题分析

### 3.1 不一致场景

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        三级缓存不一致场景                                     │
└─────────────────────────────────────────────────────────────────────────────┘

场景1：数据更新后，各级缓存数据不一致
┌─────────┐     ┌─────────┐     ┌─────────┐
│ L1: 旧值 │     │ L2: 旧值 │     │ DB: 新值 │
└─────────┘     └─────────┘     └─────────┘
     问题：用户看到的是过期数据

场景2：多实例本地缓存不一致
┌─────────────┐     ┌─────────────┐
│ Server A    │     │ Server B    │
│ L1: 新值    │     │ L1: 旧值    │  ← 同一数据，不同实例值不同
└─────────────┘     └─────────────┘

场景3：缓存更新顺序错乱
T1: 更新 DB = V2
T2: 删除 Redis
T3: 另一个请求读取 DB = V2，写入 Redis = V2
T4: 删除本地缓存
T5: 又一个请求从 Redis 读取 V2，写入本地缓存 = V2
✓ 这种情况是正确的

但如果：
T1: 请求A 读取 DB = V1（缓存 Miss）
T2: 更新 DB = V2
T3: 删除 Redis
T4: 请求A 将 V1 写入 Redis  ← 脏数据！
```

### 3.2 一致性要求分级

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                     数据一致性要求分级                                        │
├─────────────────────┬───────────────────────────────────────────────────────┤
│       强一致性       │  库存、余额、券数量                                     │
│                     │  → 不使用本地缓存，直接 Redis + DB                       │
│                     │  → 使用 Redis Lua 脚本保证原子性                        │
├─────────────────────┼───────────────────────────────────────────────────────┤
│       最终一致性     │  商品信息、活动配置、价格                                │
│                     │  → 三级缓存 + 延迟双删 + 消息广播                        │
│                     │  → 容忍秒级延迟                                         │
├─────────────────────┼───────────────────────────────────────────────────────┤
│       弱一致性       │  用户行为数据、浏览记录                                  │
│                     │  → 只用本地缓存，定期刷新                                │
│                     │  → 容忍分钟级延迟                                       │
└─────────────────────┴───────────────────────────────────────────────────────┘
```

---

## 四、一致性保障方案

### 4.1 方案一：延迟双删 + MQ 广播（我们的主方案）

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                      延迟双删 + MQ 广播方案                                   │
└─────────────────────────────────────────────────────────────────────────────┘

数据更新流程：

    ┌─────────────┐
    │  1. 删除     │
    │  本地缓存    │
    └──────┬──────┘
           │
           ▼
    ┌─────────────┐
    │  2. 删除     │
    │  Redis 缓存  │
    └──────┬──────┘
           │
           ▼
    ┌─────────────┐
    │  3. 更新     │
    │  数据库      │
    └──────┬──────┘
           │
           ├──────────────────────────────┐
           │                              │
           ▼                              ▼
    ┌─────────────┐               ┌─────────────────┐
    │  4. 发送MQ   │               │  5. 延迟500ms   │
    │  广播消息    │               │  再删Redis      │
    └──────┬──────┘               └─────────────────┘
           │
           ▼
    ┌─────────────────────────────────────────────┐
    │  6. 所有实例消费MQ，删除各自的本地缓存         │
    │     Server A: localCache.invalidate(key)   │
    │     Server B: localCache.invalidate(key)   │
    │     Server C: localCache.invalidate(key)   │
    └─────────────────────────────────────────────┘
```

**代码实现**：

```java
@Service
public class ProductCacheService {

    @Autowired
    private RocketMQTemplate mqTemplate;
    
    @Autowired
    private ScheduledExecutorService scheduler;

    /**
     * 更新商品（保证缓存一致性）
     */
    @Transactional(rollbackFor = Exception.class)
    public void updateProduct(Product product) {
        String cacheKey = "product:" + product.getId();

        // 1. 删除本地缓存
        localCache.invalidate(cacheKey);

        // 2. 删除 Redis 缓存
        redisTemplate.delete(cacheKey);

        // 3. 更新数据库
        productMapper.updateById(product);

        // 4. 发送 MQ 广播，通知所有实例删除本地缓存
        CacheInvalidateMessage message = new CacheInvalidateMessage();
        message.setCacheKey(cacheKey);
        message.setTimestamp(System.currentTimeMillis());
        mqTemplate.syncSend("cache-invalidate-topic", message);

        // 5. 延迟双删：500ms 后再删一次 Redis
        scheduler.schedule(() -> {
            redisTemplate.delete(cacheKey);
            log.info("延迟双删完成: {}", cacheKey);
        }, 500, TimeUnit.MILLISECONDS);
    }
}

/**
 * 缓存失效消息消费者（每个实例都订阅）
 */
@Component
@RocketMQMessageListener(
    topic = "cache-invalidate-topic",
    consumerGroup = "${spring.application.name}-cache-group",
    messageModel = MessageModel.BROADCASTING  // 广播模式，所有实例都收到
)
public class CacheInvalidateConsumer implements RocketMQListener<CacheInvalidateMessage> {

    @Autowired
    private Cache<String, Product> localCache;

    @Override
    public void onMessage(CacheInvalidateMessage message) {
        // 删除本地缓存
        localCache.invalidate(message.getCacheKey());
        log.info("收到缓存失效广播，删除本地缓存: {}", message.getCacheKey());
    }
}
```

### 4.2 方案二：版本号机制

```java
/**
 * 带版本号的缓存数据
 */
@Data
public class VersionedProduct {
    private Product product;
    private Long version;      // 数据版本号
    private Long updateTime;   // 更新时间戳
}

@Service
public class VersionedCacheService {

    /**
     * 读取时校验版本号
     */
    public Product getProduct(Long productId) {
        String cacheKey = "product:" + productId;
        String versionKey = "product:version:" + productId;

        // 获取最新版本号
        String latestVersion = redisTemplate.opsForValue().get(versionKey);

        // 查本地缓存
        VersionedProduct cached = localCache.getIfPresent(cacheKey);
        if (cached != null) {
            // 版本号一致才返回
            if (latestVersion != null && latestVersion.equals(String.valueOf(cached.getVersion()))) {
                return cached.getProduct();
            }
            // 版本号不一致，本地缓存失效
            localCache.invalidate(cacheKey);
        }

        // 继续查询 Redis 和 DB...
        return loadFromRedisOrDB(productId, cacheKey);
    }

    /**
     * 更新时递增版本号
     */
    @Transactional
    public void updateProduct(Product product) {
        String cacheKey = "product:" + product.getId();
        String versionKey = "product:version:" + product.getId();

        // 更新数据库
        productMapper.updateById(product);

        // 递增版本号（原子操作）
        Long newVersion = redisTemplate.opsForValue().increment(versionKey);

        // 删除缓存
        redisTemplate.delete(cacheKey);
        localCache.invalidate(cacheKey);

        // 广播通知
        broadcastInvalidate(cacheKey, newVersion);
    }
}
```

### 4.3 方案三：Canal 监听 Binlog（强一致场景）

```java
/**
 * Canal 监听数据库变更，自动更新缓存
 */
@Component
public class CanalCacheSync {

    @Autowired
    private StringRedisTemplate redisTemplate;

    @Autowired
    private RocketMQTemplate mqTemplate;

    /**
     * 监听 product 表变更
     */
    @CanalEventListener
    public void onProductChange(CanalEntry.Entry entry) {
        if (entry.getEntryType() != CanalEntry.EntryType.ROWDATA) {
            return;
        }

        CanalEntry.RowChange rowChange = CanalEntry.RowChange.parseFrom(entry.getStoreValue());
        for (CanalEntry.RowData rowData : rowChange.getRowDatasList()) {
            // 获取主键
            String productId = getColumnValue(rowData, "id");
            String cacheKey = "product:" + productId;

            // 删除 Redis 缓存
            redisTemplate.delete(cacheKey);

            // 广播删除本地缓存
            mqTemplate.syncSend("cache-invalidate-topic", 
                new CacheInvalidateMessage(cacheKey));

            log.info("Canal 监听到数据变更，删除缓存: {}", cacheKey);
        }
    }
}
```

---

## 五、不同数据的一致性策略

### 5.1 策略矩阵

| 数据类型 | 一致性要求 | 缓存层级 | 更新策略 | 过期时间 |
|---------|-----------|---------|---------|---------|
| **库存** | 强一致 | Redis only | Lua 原子操作 | 不过期 |
| **券/余额** | 强一致 | Redis only | 分布式锁 | 不过期 |
| **商品基础信息** | 最终一致 | L1 + L2 + L3 | 延迟双删 + MQ | L1:30s, L2:5min |
| **活动配置** | 最终一致 | L1 + L2 + L3 | MQ 广播 | L1:60s, L2:10min |
| **价格** | 秒级一致 | L2 + L3 | Canal 监听 | L2:1min |
| **用户浏览记录** | 弱一致 | L1 only | 定时刷新 | L1:5min |

### 5.2 库存场景：强一致实现

```java
/**
 * 库存服务：不使用本地缓存，Redis + DB 强一致
 */
@Service
public class StockService {

    // Lua 脚本：原子扣减库存
    private static final String DEDUCT_STOCK_SCRIPT =
        "local stock = tonumber(redis.call('GET', KEYS[1])) " +
        "if stock == nil then return -1 end " +
        "if stock < tonumber(ARGV[1]) then return -2 end " +
        "return redis.call('DECRBY', KEYS[1], ARGV[1])";

    /**
     * 扣减库存（强一致）
     */
    public StockResult deductStock(Long productId, int quantity) {
        String stockKey = "stock:" + productId;

        // 1. Redis Lua 原子扣减
        Long result = redisTemplate.execute(
            new DefaultRedisScript<>(DEDUCT_STOCK_SCRIPT, Long.class),
            Collections.singletonList(stockKey),
            String.valueOf(quantity)
        );

        if (result == null || result == -1) {
            return StockResult.fail("商品不存在");
        }
        if (result == -2) {
            return StockResult.fail("库存不足");
        }

        // 2. 异步落库（最终一致）
        mqTemplate.asyncSend("stock-sync-topic", 
            new StockSyncMessage(productId, quantity));

        return StockResult.success(result);
    }
}
```

### 5.3 商品信息：最终一致实现

```java
/**
 * 商品服务：三级缓存 + 延迟双删
 */
@Service
public class ProductService {

    /**
     * 更新商品（最终一致）
     * 容忍 1-2 秒的不一致窗口
     */
    public void updateProduct(Product product) {
        String cacheKey = "product:" + product.getId();

        // 1. 先删缓存
        localCache.invalidate(cacheKey);
        redisTemplate.delete(cacheKey);

        // 2. 更新数据库
        productMapper.updateById(product);

        // 3. 广播删除所有实例的本地缓存
        mqTemplate.syncSend("cache-invalidate-topic", 
            new CacheInvalidateMessage(cacheKey));

        // 4. 延迟双删
        scheduler.schedule(() -> {
            redisTemplate.delete(cacheKey);
        }, 500, TimeUnit.MILLISECONDS);
    }
}
```

---

## 六、面试常见追问

### Q1：为什么不用 Redis Pub/Sub 而用 RocketMQ？

```
Redis Pub/Sub 的问题：
├── 消息不持久化，服务重启会丢失
├── 没有 ACK 机制，无法保证送达
├── 网络抖动时消息丢失
└── 不支持消息堆积

RocketMQ 的优势：
├── 消息持久化，可靠投递
├── 支持广播模式
├── 有完善的监控和告警
└── 和我们的技术栈统一
```

### Q2：本地缓存过期时间怎么定的？

```
考虑因素：
├── 数据变更频率
├── 业务容忍的不一致时间
├── 本地缓存内存压力
└── 热点数据的访问模式

我们的配置：
├── 商品信息：30s（变更较少，容忍短暂不一致）
├── 活动配置：60s（变更极少）
├── 价格信息：10s（敏感数据，短过期）
└── 热点数据：动态调整（热 Key 检测后自动缩短）
```

### Q3：MQ 消息丢失了怎么办？

```
兜底机制：
├── 本地缓存有过期时间（最多 30-60s 不一致）
├── Redis 缓存也有过期时间（最多 5-10min 不一致）
├── 关键数据使用 Canal 监听作为二次保障
└── 监控告警：缓存命中率异常时触发告警
```

### Q4：缓存命中率 98% 怎么达到的？

```
优化手段：
├── 热点数据预热（大促前主动加载）
├── 合理的过期时间（不要太短）
├── 空值缓存（防止穿透）
├── 热 Key 自动识别并本地缓存
└── 缓存 Key 设计规范（避免无效 Key）

监控指标：
├── L1 命中率：约 60%
├── L2 命中率：约 38%
├── L3 命中率：约 2%
└── 整体命中率：98%+
```

---

## 七、总结

**面试回答模板**：

> "我们的三级缓存一致性方案是**分层治理**的思路：
>
> **架构设计**：
> - L1 Caffeine 本地缓存：10W 条，30s 过期，解决热点问题
> - L2 Redis Cluster：百万级，5 分钟过期，分布式共享
> - L3 MySQL：数据源，保证最终正确性
>
> **一致性策略**：
> - **强一致数据**（库存/余额）：不用本地缓存，Redis Lua 原子操作
> - **最终一致数据**（商品/活动）：延迟双删 + MQ 广播删除本地缓存
>
> **更新流程**：
> 1. 删本地缓存 → 2. 删 Redis → 3. 更新 DB → 4. MQ 广播 → 5. 延迟 500ms 再删 Redis
>
> **兜底机制**：
> - 所有缓存都有过期时间
> - Canal 监听 Binlog 作为二次保障
> - 监控缓存命中率异常告警
>
> 这套方案在 50W+ QPS 下稳定运行，缓存命中率 98%+，数据不一致窗口控制在 1 秒以内。"
