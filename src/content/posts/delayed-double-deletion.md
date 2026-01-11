---
title: 什么是延迟双删？为什么需要它？
published: 2025-01-08
description: 深入解析延迟双删策略的原理、解决的问题、实现方式及其局限性，帮助理解缓存一致性的关键技术
tags: [Java, Redis, 缓存一致性, 延迟双删, 并发编程]
category: Java并发编程
draft: false
---

## 面试题

什么是延迟双删？为什么需要它？

## 一句话回答

延迟双删是一种缓存更新策略：**先删缓存 → 更新数据库 → 延迟一段时间 → 再删一次缓存**，目的是解决高并发场景下"先删缓存再更新数据库"导致的缓存脏数据问题。

## 详细解析

### 一、为什么需要延迟双删？

#### 1.1 常规策略的问题

**策略 A：先更新数据库，再删缓存（Cache Aside）**

```
大多数情况下没问题，但存在极端场景：

T1: 线程A 查询缓存，未命中
T2: 线程A 查询数据库，得到旧值 V1
T3: 线程B 更新数据库为新值 V2
T4: 线程B 删除缓存
T5: 线程A 将旧值 V1 写入缓存  ← 脏数据！

结果：数据库=V2，缓存=V1（不一致）
```

这个场景发生概率很低（需要读操作跨越写操作），但在极高并发下仍可能出现。

**策略 B：先删缓存，再更新数据库**

```
问题更严重：

T1: 线程A 删除缓存
T2: 线程B 查询缓存，未命中
T3: 线程B 查询数据库，得到旧值 V1
T4: 线程B 将旧值 V1 写入缓存
T5: 线程A 更新数据库为新值 V2

结果：数据库=V2，缓存=V1（不一致）
发生概率：很高！（因为更新数据库比较慢）
```

#### 1.2 问题图解

```
先删缓存，再更新数据库的问题：

时间线 ─────────────────────────────────────────────────→

线程A（写）: ──[删缓存]────────────────────[更新DB为V2]──→
                  │                              │
                  │     这个时间窗口内              │
                  │     其他线程会读到旧值          │
                  │                              │
线程B（读）: ─────────[查缓存Miss]─[查DB得V1]─[写缓存V1]──→
                                                    │
                                                    ↓
                                              缓存是 V1（脏数据）
                                              数据库是 V2
```

### 二、延迟双删的解决思路

#### 2.1 核心思想

既然"先删缓存"后，其他线程可能在"更新数据库"之前读取旧值并写入缓存，那就在更新完数据库后，**等一段时间再删一次缓存**，把这个脏数据清掉。

```
延迟双删流程：

1. 删除缓存          ← 第一次删除
2. 更新数据库
3. 休眠 N 毫秒       ← 等待读请求完成
4. 再次删除缓存      ← 第二次删除（清除脏数据）
```

#### 2.2 执行流程图解

```
时间线 ─────────────────────────────────────────────────────────→

线程A（写）: ─[删缓存①]───[更新DB]───[延迟N毫秒]───[删缓存②]───→
                 │            │                        │
                 │            │                        │
线程B（读）: ────────[Miss]─[查DB]─[写缓存V1]          │
                                        │              │
                                        │              │
                                     脏数据          被清除！
                                                       │
                                                       ↓
线程C（读）: ─────────────────────────────────────[查缓存Miss]─[查DB得V2]─→
                                                                    │
                                                                    ↓
                                                              缓存更新为V2（正确）
```

#### 2.3 延迟时间如何确定？

```
延迟时间 > 一次"读请求"的耗时

读请求耗时 = 查询数据库 + 网络延迟 + 写入缓存

通常设置：
- 保守值：500ms ~ 1s
- 根据业务监控的 P99 读耗时来设置
- 建议：读耗时的 1.5 ~ 2 倍
```

### 三、代码实现

#### 3.1 基础实现（同步延迟）

```java
@Service
public class ProductService {
    
    @Autowired
    private StringRedisTemplate redisTemplate;
    
    @Autowired
    private ProductMapper productMapper;
    
    /**
     * 延迟双删 - 同步版本
     * 缺点：延迟会阻塞主线程，影响响应时间
     */
    public void updateProductSync(Product product) {
        String key = "product:" + product.getId();
        
        // 1. 第一次删除缓存
        redisTemplate.delete(key);
        
        // 2. 更新数据库
        productMapper.updateById(product);
        
        // 3. 延迟（阻塞主线程，不推荐）
        try {
            Thread.sleep(500);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
        
        // 4. 第二次删除缓存
        redisTemplate.delete(key);
    }
}
```

#### 3.2 异步延迟（推荐）

```java
@Service
public class ProductService {
    
    @Autowired
    private StringRedisTemplate redisTemplate;
    
    @Autowired
    private ProductMapper productMapper;
    
    // 专用线程池处理延迟删除
    private final ScheduledExecutorService scheduler = 
        Executors.newScheduledThreadPool(4);
    
    /**
     * 延迟双删 - 异步版本
     * 优点：不阻塞主线程
     */
    public void updateProductAsync(Product product) {
        String key = "product:" + product.getId();
        
        // 1. 第一次删除缓存
        redisTemplate.delete(key);
        
        // 2. 更新数据库
        productMapper.updateById(product);
        
        // 3. 异步延迟删除（不阻塞主线程）
        scheduler.schedule(() -> {
            redisTemplate.delete(key);
            log.info("延迟双删完成: {}", key);
        }, 500, TimeUnit.MILLISECONDS);
    }
    
    @PreDestroy
    public void shutdown() {
        scheduler.shutdown();
    }
}
```

#### 3.3 使用消息队列（更可靠）

```java
@Service
public class ProductService {
    
    @Autowired
    private RocketMQTemplate mqTemplate;
    
    /**
     * 延迟双删 - MQ 版本
     * 优点：可靠性高，支持重试
     */
    public void updateProductWithMQ(Product product) {
        String key = "product:" + product.getId();
        
        // 1. 第一次删除缓存
        redisTemplate.delete(key);
        
        // 2. 更新数据库
        productMapper.updateById(product);
        
        // 3. 发送延迟消息
        // RocketMQ 延迟级别：1=1s, 2=5s, 3=10s...
        mqTemplate.syncSend("cache-delete-topic",
            MessageBuilder.withPayload(key).build(),
            3000,  // 发送超时
            1      // 延迟级别 1 = 1秒
        );
    }
}

/**
 * 消费者：执行第二次删除
 */
@Component
@RocketMQMessageListener(
    topic = "cache-delete-topic", 
    consumerGroup = "cache-delete-group"
)
public class CacheDeleteConsumer implements RocketMQListener<String> {
    
    @Autowired
    private StringRedisTemplate redisTemplate;
    
    @Override
    public void onMessage(String key) {
        try {
            redisTemplate.delete(key);
            log.info("延迟双删-第二次删除完成: {}", key);
        } catch (Exception e) {
            log.error("删除缓存失败: {}", key, e);
            // MQ 会自动重试
            throw e;
        }
    }
}
```

#### 3.4 使用 Redis 过期键（简化方案）

```java
@Service
public class ProductService {
    
    /**
     * 利用 Redis 过期时间实现延迟删除
     * 原理：设置一个很短的过期时间，让 Redis 自动删除
     */
    public void updateProductWithExpire(Product product) {
        String key = "product:" + product.getId();
        
        // 1. 设置缓存很快过期（相当于延迟删除）
        redisTemplate.expire(key, 500, TimeUnit.MILLISECONDS);
        
        // 2. 更新数据库
        productMapper.updateById(product);
        
        // 缓存会在 500ms 后自动过期
        // 下次读取时会从数据库加载最新值
    }
}
```

### 四、延迟双删的完整时序分析

#### 4.1 正常情况

```
T0: 初始状态
    缓存: product:1 = {price: 100}
    数据库: price = 100

T1: 线程A 执行更新操作
    ① 删除缓存 product:1
    缓存: (空)
    
T2: 线程A 更新数据库
    数据库: price = 200
    
T3: 线程A 开始延迟等待 500ms

T4: (延迟期间) 线程B 查询商品
    缓存未命中 → 查数据库 → 得到 price=200 → 写入缓存
    缓存: product:1 = {price: 200}
    
T5: 线程A 延迟结束，执行第二次删除
    删除缓存 product:1（实际上缓存已经是正确值，删不删都可以）

结果：一致 ✓
```

#### 4.2 解决脏数据的情况

```
T0: 初始状态
    缓存: (空，恰好过期)
    数据库: price = 100

T1: 线程B 查询商品（缓存未命中）
    开始查询数据库...
    
T2: 线程A 执行更新
    ① 删除缓存（本来就空，无影响）
    
T3: 线程A 更新数据库
    数据库: price = 200
    
T4: 线程B 查询数据库完成
    得到旧值 price = 100（因为 T1 开始查询时数据库还是旧值）
    
T5: 线程B 将旧值写入缓存
    缓存: product:1 = {price: 100}  ← 脏数据！
    
T6: 线程A 延迟 500ms 后执行第二次删除
    删除缓存 product:1  ← 清除脏数据！
    缓存: (空)
    
T7: 线程C 查询商品
    缓存未命中 → 查数据库 → 得到 price=200 → 写入缓存
    缓存: product:1 = {price: 200}

结果：最终一致 ✓
```

### 五、延迟双删的优缺点

#### 5.1 优点

```
✅ 解决了"先删缓存再更新数据库"的脏数据问题
✅ 实现相对简单
✅ 可以与消息队列结合提高可靠性
```

#### 5.2 缺点

```
❌ 延迟时间难以精确确定
   - 设置太短：可能还没清除脏数据
   - 设置太长：不一致窗口变大

❌ 延迟期间仍有不一致窗口
   - 从第一次删除到第二次删除之间，可能读到脏数据

❌ 增加了系统复杂度
   - 需要异步处理或 MQ 支持

❌ 第二次删除可能失败
   - 需要重试机制保证可靠性
```

### 六、延迟双删 vs 其他策略对比

| 策略 | 操作顺序 | 不一致窗口 | 复杂度 | 适用场景 |
|-----|---------|-----------|--------|---------|
| 先更新DB再删缓存 | DB→删缓存 | 极小（极端情况） | 低 | **通用首选** |
| 先删缓存再更新DB | 删缓存→DB | 大 | 低 | 不推荐 |
| 延迟双删 | 删缓存→DB→延迟→删缓存 | 中等 | 中 | 高并发写 |
| Canal 监听 | DB→Canal→删缓存 | 中等 | 高 | 大规模系统 |

### 七、实际项目中的最佳实践

#### 7.1 组合策略（推荐）

```java
@Service
public class ProductService {
    
    /**
     * 生产环境推荐：先更新DB再删缓存 + 异步延迟双删兜底
     */
    @Transactional(rollbackFor = Exception.class)
    public void updateProduct(Product product) {
        String key = "product:" + product.getId();
        
        // 1. 先更新数据库
        productMapper.updateById(product);
        
        // 2. 删除缓存
        boolean deleted = deleteCache(key);
        
        // 3. 异步延迟再删一次（兜底）
        if (deleted) {
            asyncDelayDelete(key, 500);
        }
    }
    
    /**
     * 删除缓存（带重试）
     */
    private boolean deleteCache(String key) {
        for (int i = 0; i < 3; i++) {
            try {
                redisTemplate.delete(key);
                return true;
            } catch (Exception e) {
                log.warn("删除缓存失败，重试 {}/3: {}", i + 1, key);
            }
        }
        // 重试失败，发送告警
        alertService.send("缓存删除失败: " + key);
        return false;
    }
    
    /**
     * 异步延迟删除
     */
    private void asyncDelayDelete(String key, long delayMs) {
        CompletableFuture.runAsync(() -> {
            try {
                Thread.sleep(delayMs);
                redisTemplate.delete(key);
            } catch (Exception e) {
                log.error("延迟删除失败: {}", key, e);
            }
        });
    }
}
```

#### 7.2 何时使用延迟双删？

```
适合使用延迟双删的场景：
├── 写操作频繁
├── 对一致性要求较高
├── 读写并发压力大
└── 能接受一定的延迟

不建议使用的场景：
├── 读多写少（普通 Cache Aside 足够）
├── 实时性要求极高
└── 系统简单，不想增加复杂度
```

### 八、总结

**面试回答模板**：

> "延迟双删是解决缓存一致性的一种策略，执行流程是：**删缓存 → 更新数据库 → 延迟 N 毫秒 → 再删缓存**。
>
> **为什么需要它**：当采用'先删缓存再更新数据库'策略时，在删除缓存和更新数据库之间的时间窗口内，其他线程可能读取到数据库旧值并写入缓存，造成脏数据。第二次延迟删除就是为了清除这个脏数据。
>
> **延迟时间设置**：应大于一次读请求的耗时，通常 500ms ~ 1s。
>
> **实际应用中**，我更推荐'先更新数据库再删缓存'作为主策略，配合异步延迟双删作为兜底，这样既保证了大多数情况下的一致性，又能处理极端并发场景。"
