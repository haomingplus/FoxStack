---
title: 高并发秒杀场景，如何避免超卖或少卖的问题
published: 2025-01-08
description: 深入剖析高并发秒杀场景下库存一致性的核心挑战，从数据库锁到Redis原子操作再到分布式事务的完整解决方案
tags: [Java, 并发编程, 秒杀系统, Redis, 分布式锁, 库存扣减]
category: Java并发编程
draft: false
---


## 题目分析

这道题考察候选人对**并发控制**和**数据一致性**的理解深度。

**超卖**：库存 100，却卖出了 120 件 → 商家亏损、用户投诉
**少卖**：库存 100，只卖出 80 件，剩余 20 件被"锁死" → 商家损失、库存浪费

两者本质上都是**并发场景下库存扣减的原子性问题**。

## 参考答案

### 一、问题根源分析

#### 1.1 超卖是怎么发生的？

经典的"先查后改"问题：

```java
// 危险代码 - 存在竞态条件
public void unsafeDeductStock(Long itemId) {
    // Step 1: 查询库存
    Item item = itemMapper.selectById(itemId);
    
    // 问题：多个线程同时执行到这里，都看到 stock = 1
    if (item.getStock() > 0) {
        // Step 2: 扣减库存
        item.setStock(item.getStock() - 1);
        itemMapper.updateById(item);  // 多个线程都执行成功 → 超卖！
    }
}
```

```
时间线：
T1: 线程A 查询库存 = 1
T2: 线程B 查询库存 = 1
T3: 线程A 判断 1 > 0，通过
T4: 线程B 判断 1 > 0，通过
T5: 线程A 更新库存 = 0
T6: 线程B 更新库存 = 0  ← 超卖！实际卖了2件
```

#### 1.2 少卖是怎么发生的？

通常发生在**扣减成功但后续流程失败**的场景：

```java
public void createOrder(Long itemId, Long userId) {
    // Step 1: 扣减库存成功
    redisTemplate.decrement("stock:" + itemId);
    
    // Step 2: 创建订单失败（异常、超时等）
    orderMapper.insert(order);  // 抛异常！
    
    // 库存已扣但订单未创建 → 少卖！
}
```

### 二、防止超卖的解决方案

#### 2.1 方案一：数据库悲观锁

使用 `SELECT ... FOR UPDATE` 锁定行：

```java
@Transactional
public boolean deductStockPessimistic(Long itemId) {
    // 1. 加排他锁查询
    Item item = itemMapper.selectForUpdate(itemId);
    
    // 2. 判断库存
    if (item.getStock() <= 0) {
        return false;
    }
    
    // 3. 扣减库存
    item.setStock(item.getStock() - 1);
    itemMapper.updateById(item);
    return true;
}
```

```xml
<select id="selectForUpdate" resultType="Item">
    SELECT * FROM item WHERE id = #{itemId} FOR UPDATE
</select>
```

**优点**：实现简单，强一致性
**缺点**：性能差，高并发下大量线程阻塞等待锁

**适用场景**：并发量不高（< 100 QPS）的普通业务

#### 2.2 方案二：数据库乐观锁（推荐）

利用 `WHERE` 条件保证原子性：

```java
public boolean deductStockOptimistic(Long itemId, int quantity) {
    // 直接用 SQL 条件保证不超卖
    int affected = itemMapper.deductStock(itemId, quantity);
    return affected > 0;
}
```

```xml
<!-- 方式一：库存条件判断（推荐） -->
<update id="deductStock">
    UPDATE item 
    SET stock = stock - #{quantity}
    WHERE id = #{itemId} 
      AND stock >= #{quantity}
</update>

<!-- 方式二：版本号控制 -->
<update id="deductStockWithVersion">
    UPDATE item 
    SET stock = stock - #{quantity},
        version = version + 1
    WHERE id = #{itemId} 
      AND stock >= #{quantity}
      AND version = #{version}
</update>
```

**核心原理**：

```sql
-- 假设当前 stock = 1，两个线程同时执行

-- 线程A 执行：
UPDATE item SET stock = stock - 1 WHERE id = 1 AND stock >= 1;
-- 执行成功，affected = 1，stock 变为 0

-- 线程B 执行（此时 stock 已经是 0）：
UPDATE item SET stock = stock - 1 WHERE id = 1 AND stock >= 1;
-- 条件不满足，affected = 0，扣减失败
```

**优点**：无锁设计，性能好
**缺点**：高并发下失败率高，需要配合重试或其他策略

#### 2.3 方案三：Redis 原子操作（高并发首选）

使用 Redis 的原子命令预扣库存：

```java
@Service
public class StockService {
    
    @Autowired
    private StringRedisTemplate redisTemplate;
    
    /**
     * 方式一：DECR 原子扣减
     */
    public boolean deductStockByDecr(Long itemId) {
        String key = "stock:" + itemId;
        Long remain = redisTemplate.opsForValue().decrement(key);
        
        if (remain != null && remain >= 0) {
            return true;  // 扣减成功
        }
        
        // 库存不足，回滚
        if (remain != null && remain < 0) {
            redisTemplate.opsForValue().increment(key);
        }
        return false;
    }
}
```

**问题**：`DECR` 后判断再 `INCR` 不是原子的，极端情况仍可能出问题。

#### 2.4 方案四：Redis Lua 脚本（最佳实践）

使用 Lua 脚本保证**查询+判断+扣减**的原子性：

```lua
-- deduct_stock.lua
-- KEYS[1]: 库存 key
-- ARGV[1]: 扣减数量

local stock = tonumber(redis.call('GET', KEYS[1]))

-- 库存不存在
if stock == nil then
    return -1
end

-- 库存不足
if stock < tonumber(ARGV[1]) then
    return -2
end

-- 扣减库存，返回剩余数量
return redis.call('DECRBY', KEYS[1], ARGV[1])
```

```java
@Service
public class StockService {
    
    private static final String DEDUCT_STOCK_SCRIPT = 
        "local stock = tonumber(redis.call('GET', KEYS[1])) " +
        "if stock == nil then return -1 end " +
        "if stock < tonumber(ARGV[1]) then return -2 end " +
        "return redis.call('DECRBY', KEYS[1], ARGV[1])";
    
    private final DefaultRedisScript<Long> deductScript;
    
    @PostConstruct
    public void init() {
        deductScript = new DefaultRedisScript<>();
        deductScript.setScriptText(DEDUCT_STOCK_SCRIPT);
        deductScript.setResultType(Long.class);
    }
    
    /**
     * Lua 脚本原子扣减库存
     * @return 剩余库存，-1 表示商品不存在，-2 表示库存不足
     */
    public Long deductStock(Long itemId, int quantity) {
        String key = "stock:" + itemId;
        return redisTemplate.execute(
            deductScript,
            Collections.singletonList(key),
            String.valueOf(quantity)
        );
    }
    
    /**
     * 业务调用示例
     */
    public Result doFlashSale(Long itemId, Long userId) {
        Long result = deductStock(itemId, 1);
        
        if (result == -1) {
            return Result.fail("商品不存在");
        }
        if (result == -2) {
            return Result.fail("商品已售罄");
        }
        
        // 发送 MQ 异步创建订单
        sendOrderMessage(itemId, userId);
        return Result.success("抢购成功，订单处理中");
    }
}
```

#### 2.5 方案五：分布式锁

使用 Redis 分布式锁控制并发：

```java
@Service
public class StockServiceWithLock {
    
    @Autowired
    private RedissonClient redissonClient;
    
    public boolean deductStockWithLock(Long itemId, int quantity) {
        String lockKey = "lock:stock:" + itemId;
        RLock lock = redissonClient.getLock(lockKey);
        
        try {
            // 尝试获取锁，最多等待 3 秒，锁自动过期 10 秒
            boolean acquired = lock.tryLock(3, 10, TimeUnit.SECONDS);
            if (!acquired) {
                return false;  // 获取锁失败
            }
            
            // 获取锁成功，执行库存扣减
            String key = "stock:" + itemId;
            Long stock = Long.parseLong(redisTemplate.opsForValue().get(key));
            
            if (stock < quantity) {
                return false;
            }
            
            redisTemplate.opsForValue().decrement(key, quantity);
            return true;
            
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            return false;
        } finally {
            // 只有持有锁的线程才能释放
            if (lock.isHeldByCurrentThread()) {
                lock.unlock();
            }
        }
    }
}
```

**适用场景**：需要复杂业务逻辑的场景（如一次购买多种商品）

### 三、防止少卖的解决方案

少卖的核心是**扣减库存后，后续流程失败但库存未回滚**。

#### 3.1 方案一：事务补偿（最终一致性）

```java
@Service
public class FlashSaleService {
    
    public Result flashSale(Long itemId, Long userId) {
        String stockKey = "stock:" + itemId;
        String orderKey = "order:" + itemId + ":" visitorId;
        
        try {
            // 1. Redis 预扣库存
            Long remain = deductStock(itemId, 1);
            if (remain < 0) {
                return Result.fail("库存不足");
            }
            
            // 2. 发送订单消息到 MQ
            FlashSaleMessage message = new FlashSaleMessage(itemId, userId);
            SendResult sendResult = mqTemplate.syncSend("order-topic", message);
            
            if (sendResult.getSendStatus() != SendStatus.SEND_OK) {
                // MQ 发送失败，回滚库存
                rollbackStock(itemId, 1);
                return Result.fail("系统繁忙，请重试");
            }
            
            return Result.success("下单成功");
            
        } catch (Exception e) {
            // 异常情况，回滚库存
            rollbackStock(itemId, 1);
            throw e;
        }
    }
    
    /**
     * 回滚库存
     */
    private void rollbackStock(Long itemId, int quantity) {
        String key = "stock:" + itemId;
        redisTemplate.opsForValue().increment(key, quantity);
    }
}
```

#### 3.2 方案二：库存扣减与订单创建的事务消息

使用 RocketMQ 事务消息保证一致性：

```java
@Service
public class TransactionalFlashSaleService {
    
    @Autowired
    private RocketMQTemplate mqTemplate;
    
    public Result flashSale(Long itemId, Long userId) {
        // 发送事务消息
        FlashSaleMessage payload = new FlashSaleMessage(itemId, userId);
        
        TransactionSendResult result = mqTemplate.sendMessageInTransaction(
            "order-topic",
            MessageBuilder.withPayload(payload).build(),
            payload  // 作为 arg 传递给本地事务
        );
        
        if (result.getLocalTransactionState() == LocalTransactionState.COMMIT_MESSAGE) {
            return Result.success("下单成功");
        } else {
            return Result.fail("下单失败");
        }
    }
}

/**
 * 事务消息监听器
 */
@RocketMQTransactionListener
public class FlashSaleTransactionListener implements RocketMQLocalTransactionListener {
    
    @Autowired
    private StockService stockService;
    
    /**
     * 执行本地事务（扣减库存）
     */
    @Override
    public RocketMQLocalTransactionState executeLocalTransaction(Message msg, Object arg) {
        FlashSaleMessage payload = (FlashSaleMessage) arg;
        
        try {
            // 扣减库存
            Long remain = stockService.deductStock(payload.getItemId(), 1);
            
            if (remain >= 0) {
                // 本地事务成功，提交消息
                return RocketMQLocalTransactionState.COMMIT;
            } else {
                // 库存不足，回滚消息
                return RocketMQLocalTransactionState.ROLLBACK;
            }
        } catch (Exception e) {
            // 异常，等待回查
            return RocketMQLocalTransactionState.UNKNOWN;
        }
    }
    
    /**
     * 事务回查（处理 UNKNOWN 状态）
     */
    @Override
    public RocketMQLocalTransactionState checkLocalTransaction(Message msg) {
        FlashSaleMessage payload = parsePayload(msg);
        
        // 检查库存是否已扣减成功（可以通过流水表判断）
        boolean deducted = stockService.checkDeducted(
            payload.getItemId(), 
            payload.getUserId()
        );
        
        return deducted 
            ? RocketMQLocalTransactionState.COMMIT 
            : RocketMQLocalTransactionState.ROLLBACK;
    }
}
```

#### 3.3 方案三：订单超时回滚库存

对于未支付的订单，定时回滚库存：

```java
/**
 * 延迟消息检查订单支付状态
 */
@Service
public class OrderTimeoutService {
    
    public void sendDelayMessage(Long orderId) {
        // 发送延迟消息，30分钟后检查
        mqTemplate.syncSend(
            "order-timeout-topic",
            MessageBuilder.withPayload(orderId).build(),
            3000,  // 发送超时
            16     // 延迟级别，约30分钟
        );
    }
}

@RocketMQMessageListener(topic = "order-timeout-topic", consumerGroup = "order-timeout-group")
public class OrderTimeoutConsumer implements RocketMQListener<Long> {
    
    @Override
    public void onMessage(Long orderId) {
        Order order = orderMapper.selectById(orderId);
        
        // 订单未支付，取消订单并回滚库存
        if (order != null && order.getStatus() == OrderStatus.UNPAID) {
            // 1. 更新订单状态为已取消
            orderMapper.updateStatus(orderId, OrderStatus.CANCELLED);
            
            // 2. 回滚 Redis 库存
            redisTemplate.opsForValue().increment("stock:" + order.getItemId());
            
            // 3. 回滚数据库库存
            itemMapper.incrementStock(order.getItemId(), order.getQuantity());
        }
    }
}
```

### 四、完整架构设计

将防超卖和防少卖整合到完整的秒杀流程中：

```
┌─────────────────────────────────────────────────────────────────┐
│                        秒杀请求流程                              │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│  1. 参数校验 & 用户去重                                          │
│     - 校验活动是否开始                                           │
│     - Redis Set 判断用户是否已购买                                │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│  2. Redis Lua 原子扣减库存 ← 防超卖核心                          │
│     - 成功：继续流程                                             │
│     - 失败：返回售罄                                             │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│  3. 记录用户已抢购（Redis Set）                                  │
│     - SADD flash:buyers:{itemId} {userId}                       │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│  4. 发送 MQ 消息 ← 防少卖核心                                    │
│     - 成功：返回"排队中"                                         │
│     - 失败：回滚库存 + 移除购买记录                               │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│  5. MQ 消费者异步处理                                            │
│     - 数据库乐观锁扣减（二次校验）                                │
│     - 创建订单                                                   │
│     - 发送支付超时延迟消息                                        │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│  6. 超时未支付 → 取消订单 + 回滚库存                              │
└─────────────────────────────────────────────────────────────────┘
```

### 五、完整代码实现

```java
@Service
@Slf4j
public class FlashSaleServiceImpl implements FlashSaleService {
    
    @Autowired
    private StringRedisTemplate redisTemplate;
    
    @Autowired
    private RocketMQTemplate mqTemplate;
    
    private static final String STOCK_KEY = "flash:stock:";
    private static final String BUYER_KEY = "flash:buyers:";
    
    // Lua 脚本：原子扣减库存
    private static final String DEDUCT_SCRIPT = 
        "local stock = tonumber(redis.call('GET', KEYS[1])) " +
        "if stock == nil then return -1 end " +
        "if stock < tonumber(ARGV[1]) then return -2 end " +
        "return redis.call('DECRBY', KEYS[1], ARGV[1])";
    
    @Override
    public Result doFlashSale(Long itemId, Long userId) {
        
        // 1. 用户去重校验
        String buyerKey = BUYER_KEY + itemId;
        Boolean isBuyer = redisTemplate.opsForSet().isMember(buyerKey, userId.toString());
        if (Boolean.TRUE.equals(isBuyer)) {
            return Result.fail("您已参与过此活动");
        }
        
        // 2. Lua 脚本原子扣减库存（防超卖）
        Long result = redisTemplate.execute(
            new DefaultRedisScript<>(DEDUCT_SCRIPT, Long.class),
            Collections.singletonList(STOCK_KEY + itemId),
            "1"
        );
        
        if (result == null || result == -1) {
            return Result.fail("活动不存在");
        }
        if (result == -2) {
            return Result.fail("商品已售罄");
        }
        
        // 3. 标记用户已购买
        redisTemplate.opsForSet().add(buyerKey, userId.toString());
        
        // 4. 发送 MQ 消息（防少卖 - 失败回滚）
        try {
            FlashSaleMessage message = new FlashSaleMessage(itemId, userId);
            SendResult sendResult = mqTemplate.syncSend("flash-sale-topic", message);
            
            if (sendResult.getSendStatus() != SendStatus.SEND_OK) {
                rollback(itemId, userId);
                return Result.fail("系统繁忙，请重试");
            }
            
        } catch (Exception e) {
            log.error("发送消息失败", e);
            rollback(itemId, userId);
            return Result.fail("系统繁忙，请重试");
        }
        
        return Result.success("抢购成功，请等待订单创建");
    }
    
    /**
     * 回滚操作：恢复库存 + 移除购买记录
     */
    private void rollback(Long itemId, Long userId) {
        redisTemplate.opsForValue().increment(STOCK_KEY + itemId);
        redisTemplate.opsForSet().remove(BUYER_KEY + itemId, userId.toString());
    }
}

/**
 * 订单消费者 - 数据库层面防超卖兜底
 */
@Component
@RocketMQMessageListener(topic = "flash-sale-topic", consumerGroup = "flash-sale-group")
@Slf4j
public class FlashSaleConsumer implements RocketMQListener<FlashSaleMessage> {
    
    @Autowired
    private ItemMapper itemMapper;
    
    @Autowired
    private OrderMapper orderMapper;
    
    @Override
    @Transactional(rollbackFor = Exception.class)
    public void onMessage(FlashSaleMessage message) {
        Long itemId = message.getItemId();
        Long userId = message.getUserId();
        
        // 1. 数据库乐观锁扣减（兜底防超卖）
        int affected = itemMapper.deductStock(itemId, 1);
        if (affected == 0) {
            log.warn("数据库库存不足: itemId={}", itemId);
            // 可以发送通知告诉用户抢购失败
            return;
        }
        
        // 2. 创建订单
        Order order = new Order();
        order.setItemId(itemId);
        order.setUserId(userId);
        order.setQuantity(1);
        order.setStatus(OrderStatus.UNPAID);
        order.setCreateTime(LocalDateTime.now());
        orderMapper.insert(order);
        
        // 3. 发送延迟消息检查支付（防少卖 - 超时回滚）
        sendPaymentTimeoutCheck(order.getId());
        
        log.info("订单创建成功: orderId={}, userId={}", order.getId(), userId);
    }
}
```

### 六、方案对比总结

| 方案 | 防超卖 | 性能 | 复杂度 | 适用场景 |
|-----|--------|------|--------|---------|
| 数据库悲观锁 | ✅ 强 | ❌ 差 | 低 | 低并发业务 |
| 数据库乐观锁 | ✅ 强 | ⚠️ 中 | 低 | 中等并发 |
| Redis DECR | ⚠️ 有缺陷 | ✅ 高 | 低 | 不推荐单独使用 |
| Redis Lua | ✅ 强 | ✅ 高 | 中 | **高并发首选** |
| 分布式锁 | ✅ 强 | ⚠️ 中 | 高 | 复杂业务场景 |

### 七、面试加分回答

**完整回答思路**：

> "针对超卖和少卖问题，我会采用**多层防护**策略：
> 
> **防超卖**：
> 1. 第一层：Redis Lua 脚本原子扣减预库存，这是主要防线
> 2. 第二层：数据库乐观锁兜底，`WHERE stock >= quantity`
> 3. 辅助：唯一索引防止重复订单
> 
> **防少卖**：
> 1. MQ 发送失败时主动回滚 Redis 库存
> 2. 订单超时未支付，延迟消息触发库存回滚
> 3. 定时对账任务，比对 Redis 和数据库库存差异
> 
> 这套方案在我们线上系统经过验证，QPS 达到 5 万时，超卖率为 0，少卖率控制在 0.01% 以内（主要是极端异常导致）。"
