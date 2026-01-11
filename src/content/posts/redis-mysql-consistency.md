---
title: Redis缓存与数据库一致性保证方案
published: 2025-01-08
description: 深入剖析缓存一致性的核心挑战，从Cache Aside到延迟双删、Canal监听，全面解析读写场景下的数据一致性解决方案
tags: [Java, Redis, 缓存一致性, 分布式, MySQL, Canal]
category: Java并发编程
draft: false
---

## 面试题

热点数据既有查询又有更新，如何保证 Redis 缓存与数据库的一致性？

## 题目分析

这是一道经典的分布式系统面试题，考察候选人对**缓存一致性问题**的理解深度。

**核心矛盾**：Redis 和 MySQL 是两个独立的系统，无法在一个事务中同时更新，必然存在时间窗口导致数据不一致。

**面试官期望**：
1. 能分析出不一致产生的原因
2. 能说清楚各种方案的优缺点
3. 能根据业务场景选择合适的方案

## 参考答案

### 一、为什么会产生不一致？

#### 1.1 读写并发场景

无论采用哪种策略，在高并发下都可能出现不一致：

```
场景：库存从 100 更新为 99

时间线（先更新数据库，再删除缓存）：
T1: 线程A 更新数据库 stock = 99
T2: 线程B 查询缓存命中 stock = 100（旧值）
T3: 线程A 删除缓存
T4: 线程C 查询缓存未命中，从数据库读取 stock = 99

问题：T2 时刻线程B 读到了旧值
```

```
更严重的场景（缓存刚好失效时）：
T1: 线程A 查询缓存未命中
T2: 线程A 查询数据库 stock = 100
T3: 线程B 更新数据库 stock = 99
T4: 线程B 删除缓存（此时缓存本来就没有）
T5: 线程A 将 stock = 100 写入缓存  ← 脏数据！

结果：数据库是 99，缓存是 100，严重不一致！
```

#### 1.2 问题本质

```
根本原因：
1. 缓存和数据库是两个系统，操作非原子
2. 并发读写存在时序问题
3. 网络延迟、程序异常导致部分操作失败
```

### 二、常见的缓存更新策略

#### 2.1 策略对比总览

| 策略 | 操作顺序 | 一致性 | 适用场景 |
|-----|---------|--------|---------|
| Cache Aside | 先更新DB，再删缓存 | ⭐⭐⭐ 较好 | **通用首选** |
| Read/Write Through | 缓存代理读写 | ⭐⭐⭐ 较好 | 特定框架 |
| Write Behind | 只写缓存，异步刷DB | ⭐⭐ 一般 | 高性能写入 |
| 延迟双删 | 删缓存→更新DB→延迟再删 | ⭐⭐⭐⭐ 好 | 高并发场景 |
| Canal 监听 | 监听 Binlog 更新缓存 | ⭐⭐⭐⭐⭐ 最好 | 强一致要求 |

---

### 三、方案一：Cache Aside Pattern（旁路缓存）⭐ 推荐

这是最常用的缓存策略，也叫"先更新数据库，再删除缓存"。

#### 3.1 读操作流程

```java
public Product getProduct(Long productId) {
    String key = "product:" + productId;
    
    // 1. 先查缓存
    String json = redisTemplate.opsForValue().get(key);
    if (json != null) {
        return JSON.parseObject(json, Product.class);
    }
    
    // 2. 缓存未命中，查数据库
    Product product = productMapper.selectById(productId);
    if (product == null) {
        // 防止缓存穿透：缓存空值
        redisTemplate.opsForValue().set(key, "", 60, TimeUnit.SECONDS);
        return null;
    }
    
    // 3. 写入缓存
    redisTemplate.opsForValue().set(key, JSON.toJSONString(product), 
                                    30, TimeUnit.MINUTES);
    return product;
}
```

```
读流程：
┌─────────┐    ①查缓存    ┌─────────┐
│  应用   │ ──────────→  │  Redis  │
└─────────┘              └─────────┘
     │                        │
     │ 命中？                  │
     │ ├─ 是 ←────────────────┘ 返回数据
     │ │
     │ └─ 否
     │      │
     ▼      ▼
┌─────────┐    ②查数据库
│  MySQL  │ ←─────────
└─────────┘
     │
     │ ③写入缓存
     ▼
┌─────────┐
│  Redis  │
└─────────┘
```

#### 3.2 写操作流程

```java
@Transactional(rollbackFor = Exception.class)
public void updateProduct(Product product) {
    String key = "product:" + product.getId();
    
    // 1. 先更新数据库
    productMapper.updateById(product);
    
    // 2. 再删除缓存（而不是更新缓存）
    redisTemplate.delete(key);
}
```

```
写流程：
┌─────────┐    ①更新    ┌─────────┐
│  应用   │ ─────────→  │  MySQL  │
└─────────┘              └─────────┘
     │
     │ ②删除缓存
     ▼
┌─────────┐
│  Redis  │
└─────────┘
```

#### 3.3 为什么是"删除"而不是"更新"缓存？

```
场景：商品价格更新

如果是"更新缓存"：
T1: 线程A 更新数据库 price = 100
T2: 线程B 更新数据库 price = 200
T3: 线程B 更新缓存 price = 200
T4: 线程A 更新缓存 price = 100  ← 旧值覆盖新值！

结果：数据库是 200，缓存是 100

如果是"删除缓存"：
T1: 线程A 更新数据库 price = 100
T2: 线程B 更新数据库 price = 200
T3: 线程B 删除缓存
T4: 线程A 删除缓存
T5: 下次查询从数据库读取 price = 200

结果：正确！
```

**删除缓存的优点**：
1. 避免并发更新导致的数据错乱
2. 懒加载思想，下次访问时才重建缓存
3. 避免频繁更新但很少读取的数据占用缓存

#### 3.4 为什么是"先更新数据库"而不是"先删缓存"？

```
如果"先删缓存，再更新数据库"：

T1: 线程A 删除缓存
T2: 线程B 查询缓存未命中
T3: 线程B 查询数据库（旧值）
T4: 线程B 将旧值写入缓存
T5: 线程A 更新数据库（新值）

结果：数据库是新值，缓存是旧值，不一致！
发生概率：很高（因为更新数据库比较慢）
```

```
如果"先更新数据库，再删缓存"：

T1: 线程A 查询缓存未命中
T2: 线程A 查询数据库（旧值）
T3: 线程B 更新数据库（新值）
T4: 线程B 删除缓存
T5: 线程A 将旧值写入缓存

结果：同样不一致！
但发生概率：极低（需要 T2~T5 的时间窗口内完成）
```

**为什么"先更新数据库"更好？**

```
概率分析：
- 先删缓存：不一致发生在"写操作期间有读操作"，概率高
- 先更新DB：不一致发生在"缓存恰好失效 + 读操作慢于写操作"，概率极低

因为：
- 读操作（查 DB + 写缓存）通常是毫秒级
- 写操作（更新 DB + 删缓存）通常是几十毫秒
- 读操作在写操作期间完成的概率极低
```

---

### 四、方案二：延迟双删

针对 Cache Aside 的极端情况，增加一次延迟删除。

#### 4.1 实现原理

```
延迟双删流程：
1. 删除缓存
2. 更新数据库
3. 延迟 N 毫秒
4. 再次删除缓存
```

```java
@Service
public class ProductService {
    
    @Autowired
    private StringRedisTemplate redisTemplate;
    
    @Autowired
    private ThreadPoolExecutor executor;
    
    public void updateProduct(Product product) {
        String key = "product:" + product.getId();
        
        // 1. 先删除缓存
        redisTemplate.delete(key);
        
        // 2. 更新数据库
        productMapper.updateById(product);
        
        // 3. 延迟再删一次（异步执行，不阻塞主流程）
        executor.execute(() -> {
            try {
                // 延迟时间 > 一次读操作的时间（通常 500ms ~ 1s）
                Thread.sleep(500);
                redisTemplate.delete(key);
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        });
    }
}
```

#### 4.2 为什么需要延迟？

```
场景分析：

T1: 线程A 删除缓存
T2: 线程B 查询缓存未命中
T3: 线程B 查询数据库（旧值）
T4: 线程A 更新数据库（新值）
T5: 线程B 将旧值写入缓存  ← 此时缓存是旧值
T6: 线程A 延迟后再次删除缓存  ← 清除旧值！
T7: 后续查询会从数据库读取新值

延迟时间 > T5 - T4，确保能清除掉脏数据
```

#### 4.3 使用消息队列实现延迟双删

更可靠的方式是使用 MQ 的延迟消息：

```java
@Service
public class ProductService {
    
    @Autowired
    private RocketMQTemplate mqTemplate;
    
    public void updateProduct(Product product) {
        String key = "product:" + product.getId();
        
        // 1. 删除缓存
        redisTemplate.delete(key);
        
        // 2. 更新数据库
        productMapper.updateById(product);
        
        // 3. 发送延迟消息，500ms 后再删缓存
        mqTemplate.syncSend("cache-delete-topic", 
            MessageBuilder.withPayload(key).build(),
            3000,  // 发送超时
            1      // 延迟级别 1 = 1s
        );
    }
}

@RocketMQMessageListener(topic = "cache-delete-topic", consumerGroup = "cache-delete-group")
public class CacheDeleteConsumer implements RocketMQListener<String> {
    
    @Autowired
    private StringRedisTemplate redisTemplate;
    
    @Override
    public void onMessage(String key) {
        redisTemplate.delete(key);
        log.info("延迟删除缓存: {}", key);
    }
}
```

---

### 五、方案三：基于 Canal 监听 Binlog

最终一致性的最佳方案，完全解耦应用与缓存更新。

#### 5.1 架构原理

```
┌─────────────┐        ┌─────────────┐
│    应用     │ ─────→ │    MySQL    │
└─────────────┘  写入   └─────────────┘
                              │
                              │ Binlog
                              ▼
                       ┌─────────────┐
                       │    Canal    │  监听 Binlog
                       └─────────────┘
                              │
                              │ 解析变更
                              ▼
                       ┌─────────────┐
                       │  MQ / 直连   │
                       └─────────────┘
                              │
                              │ 更新/删除
                              ▼
                       ┌─────────────┐
                       │    Redis    │
                       └─────────────┘

优点：
- 应用只管写数据库，无需关心缓存
- 数据库和缓存更新完全解耦
- 保证最终一致性
```

#### 5.2 Canal 客户端实现

```java
@Component
@Slf4j
public class CanalClient {
    
    @Autowired
    private StringRedisTemplate redisTemplate;
    
    private CanalConnector connector;
    
    @PostConstruct
    public void init() {
        // 连接 Canal Server
        connector = CanalConnectors.newSingleConnector(
            new InetSocketAddress("127.0.0.1", 11111),
            "example",  // destination
            "",         // username
            ""          // password
        );
        
        // 启动监听线程
        new Thread(this::process).start();
    }
    
    private void process() {
        connector.connect();
        connector.subscribe("mydb.product");  // 订阅表
        
        while (true) {
            try {
                Message message = connector.getWithoutAck(100);
                long batchId = message.getId();
                
                if (batchId != -1 && !message.getEntries().isEmpty()) {
                    for (Entry entry : message.getEntries()) {
                        if (entry.getEntryType() == EntryType.ROWDATA) {
                            handleRowChange(entry);
                        }
                    }
                }
                
                connector.ack(batchId);
            } catch (Exception e) {
                log.error("Canal 处理异常", e);
            }
        }
    }
    
    private void handleRowChange(Entry entry) throws Exception {
        RowChange rowChange = RowChange.parseFrom(entry.getStoreValue());
        EventType eventType = rowChange.getEventType();
        
        for (RowData rowData : rowChange.getRowDatasList()) {
            if (eventType == EventType.UPDATE || eventType == EventType.DELETE) {
                // 获取主键（假设第一列是 id）
                String id = rowData.getBeforeColumnsList().get(0).getValue();
                String key = "product:" + id;
                
                // 删除缓存
                redisTemplate.delete(key);
                log.info("Canal 触发缓存删除: {}, 事件类型: {}", key, eventType);
            }
        }
    }
}
```

#### 5.3 结合 MQ 的可靠方案

```java
/**
 * Canal 将变更发送到 MQ，消费者处理缓存更新
 * 好处：解耦、可重试、可扩展
 */
@Component
public class CanalMQProducer {
    
    @Autowired
    private RocketMQTemplate mqTemplate;
    
    public void handleRowChange(Entry entry) throws Exception {
        RowChange rowChange = RowChange.parseFrom(entry.getStoreValue());
        
        for (RowData rowData : rowChange.getRowDatasList()) {
            CacheInvalidateMessage msg = new CacheInvalidateMessage();
            msg.setTable(entry.getHeader().getTableName());
            msg.setEventType(rowChange.getEventType().name());
            msg.setId(extractId(rowData));
            
            // 发送到 MQ
            mqTemplate.syncSend("cache-invalidate-topic", msg);
        }
    }
}

@RocketMQMessageListener(topic = "cache-invalidate-topic", consumerGroup = "cache-group")
public class CacheInvalidateConsumer implements RocketMQListener<CacheInvalidateMessage> {
    
    @Autowired
    private StringRedisTemplate redisTemplate;
    
    @Override
    public void onMessage(CacheInvalidateMessage msg) {
        String key = msg.getTable() + ":" + msg.getId();
        redisTemplate.delete(key);
        log.info("缓存失效: {}", key);
    }
}
```

---

### 六、方案四：分布式锁保证强一致

对于要求强一致的场景，可以用分布式锁串行化读写操作。

```java
@Service
public class ProductService {
    
    @Autowired
    private RedissonClient redissonClient;
    
    /**
     * 强一致性读
     */
    public Product getProductConsistent(Long productId) {
        String key = "product:" + productId;
        String lockKey = "lock:product:" + productId;
        
        RLock lock = redissonClient.getLock(lockKey);
        try {
            // 读操作也加锁（读写互斥）
            lock.lock();
            
            // 先查缓存
            String json = redisTemplate.opsForValue().get(key);
            if (json != null) {
                return JSON.parseObject(json, Product.class);
            }
            
            // 查数据库并写缓存
            Product product = productMapper.selectById(productId);
            if (product != null) {
                redisTemplate.opsForValue().set(key, JSON.toJSONString(product), 
                                                30, TimeUnit.MINUTES);
            }
            return product;
            
        } finally {
            lock.unlock();
        }
    }
    
    /**
     * 强一致性写
     */
    public void updateProductConsistent(Product product) {
        String key = "product:" + product.getId();
        String lockKey = "lock:product:" + product.getId();
        
        RLock lock = redissonClient.getLock(lockKey);
        try {
            lock.lock();
            
            // 更新数据库
            productMapper.updateById(product);
            
            // 删除缓存
            redisTemplate.delete(key);
            
        } finally {
            lock.unlock();
        }
    }
}
```

**缺点**：性能较差，读写都需要获取锁，不适合高并发场景。

**适用场景**：金融、库存等对一致性要求极高的业务。

---

### 七、方案五：设置合理的过期时间（兜底）

无论采用哪种策略，都应该设置缓存过期时间作为**最终兜底**：

```java
// 设置过期时间，确保即使出现不一致，也会在一定时间后自动修复
redisTemplate.opsForValue().set(key, value, 30, TimeUnit.MINUTES);
```

```
过期时间的作用：
1. 兜底机制：即使其他策略失效，数据最终也会一致
2. 内存管理：避免冷数据长期占用缓存
3. 容错：删除缓存失败时，过期后自动失效
```

---

### 八、删除缓存失败的补偿机制

#### 8.1 重试机制

```java
@Service
public class CacheService {
    
    @Autowired
    private StringRedisTemplate redisTemplate;
    
    @Autowired
    private RocketMQTemplate mqTemplate;
    
    /**
     * 删除缓存，失败则重试
     */
    public void deleteWithRetry(String key, int maxRetries) {
        for (int i = 0; i < maxRetries; i++) {
            try {
                Boolean deleted = redisTemplate.delete(key);
                if (Boolean.TRUE.equals(deleted)) {
                    return;
                }
            } catch (Exception e) {
                log.warn("删除缓存失败，第 {} 次重试: {}", i + 1, key);
            }
            
            // 短暂等待后重试
            try {
                Thread.sleep(100 * (i + 1));
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        }
        
        // 重试失败，发送到 MQ 异步处理
        mqTemplate.syncSend("cache-delete-retry-topic", key);
        log.error("删除缓存多次失败，已发送到 MQ: {}", key);
    }
}
```

#### 8.2 基于消息队列的可靠删除

```java
@Service
public class ReliableCacheService {
    
    @Transactional(rollbackFor = Exception.class)
    public void updateProduct(Product product) {
        String key = "product:" + product.getId();
        
        // 1. 更新数据库
        productMapper.updateById(product);
        
        // 2. 记录缓存删除任务到本地消息表（与业务在同一事务）
        CacheDeleteTask task = new CacheDeleteTask();
        task.setCacheKey(key);
        task.setStatus(0);  // 待处理
        task.setCreateTime(LocalDateTime.now());
        cacheDeleteTaskMapper.insert(task);
    }
}

/**
 * 定时任务扫描本地消息表，执行缓存删除
 */
@Scheduled(fixedRate = 1000)
public void processCacheDeleteTasks() {
    List<CacheDeleteTask> tasks = cacheDeleteTaskMapper.selectPending();
    
    for (CacheDeleteTask task : tasks) {
        try {
            redisTemplate.delete(task.getCacheKey());
            task.setStatus(1);  // 处理成功
        } catch (Exception e) {
            task.setRetryCount(task.getRetryCount() + 1);
            if (task.getRetryCount() >= 3) {
                task.setStatus(2);  // 处理失败
            }
        }
        cacheDeleteTaskMapper.updateById(task);
    }
}
```

---

### 九、方案选型指南

```
┌─────────────────────────────────────────────────────────────────┐
│                        如何选择方案？                            │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
                    ┌─────────────────┐
                    │ 业务能否容忍    │
                    │ 短暂不一致？    │
                    └─────────────────┘
                         │
            ┌────────────┴────────────┐
            │ 能                       │ 不能
            ▼                         ▼
    ┌───────────────┐         ┌───────────────┐
    │ Cache Aside   │         │ 分布式锁      │
    │ + 过期时间    │         │ 串行化读写    │
    └───────────────┘         └───────────────┘
            │
            ▼
    ┌─────────────────┐
    │ 并发是否很高？  │
    └─────────────────┘
         │
    ┌────┴────┐
    │ 是      │ 否
    ▼         ▼
┌─────────┐  ┌─────────────┐
│ 延迟双删 │  │ Cache Aside │
│ + Canal │  │ 足够        │
└─────────┘  └─────────────┘
```

| 场景 | 推荐方案 | 一致性级别 |
|-----|---------|-----------|
| 普通业务 | Cache Aside + 过期时间 | 最终一致 |
| 高并发热点 | 延迟双删 | 准最终一致 |
| 强一致要求 | 分布式锁 | 强一致 |
| 大规模系统 | Canal + MQ | 最终一致 |
| 金融级 | 不用缓存 / 读写锁 | 强一致 |

---

### 十、总结

**面试回答模板**：

> "缓存和数据库的一致性问题，本质是**分布式系统下两个数据源无法原子更新**的问题。我通常采用以下策略：
>
> **基础方案**：Cache Aside 模式
> - 读：先缓存，未命中再查库并回填
> - 写：先更新数据库，再删除缓存（不是更新）
> - 配合过期时间作为兜底
>
> **高并发优化**：延迟双删
> - 删缓存 → 更新数据库 → 延迟 N 毫秒 → 再删缓存
> - 解决极端情况下的并发不一致
>
> **最终一致性**：Canal 监听 Binlog
> - 应用只写数据库，完全解耦
> - Canal 监听变更，异步更新缓存
>
> **强一致性**：分布式锁
> - 读写操作都加锁，串行化执行
> - 性能牺牲较大，仅用于核心场景
>
> 实际项目中，我们针对不同数据采用不同策略：普通数据用 Cache Aside + 过期时间，热点数据用延迟双删，核心数据用 Canal 保证最终一致。"
