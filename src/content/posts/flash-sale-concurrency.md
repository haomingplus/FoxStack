---
title: 抢购场景下的高并发处理方案
published: 2025-01-08
description: 深入解析电商抢购场景下的高并发挑战与多层次解决方案，涵盖限流、缓存、队列、库存扣减等核心技术
tags: [Java, 并发编程, 高并发, 秒杀系统, Redis, 分布式]
category: Java并发编程
draft: false
---

## 回答思路

这是一道综合性很强的架构设计题，考察候选人对高并发系统的整体把控能力。回答时应该从**流量入口到数据存储**的全链路角度，层层递进地阐述解决方案。

## 参考答案

### 一、抢购场景的核心挑战

抢购（秒杀）场景具有以下特点：

1. **瞬时流量巨大**：可能在几秒内涌入百万级请求
2. **读多写少**：大量用户查询商品，少量用户能成功下单
3. **库存有限**：必须保证不超卖、不少卖
4. **业务简单**：核心就是"扣减库存 + 创建订单"

### 二、全链路高并发处理方案

我通常从以下几个层面来设计抢购系统的并发处理：

#### 2.1 前端层：削峰与拦截

```javascript
// 1. 按钮防重复点击
let isSubmitting = false;
function submitOrder() {
    if (isSubmitting) return;
    isSubmitting = true;
    // 提交逻辑...
    setTimeout(() => isSubmitting = false, 3000);
}

// 2. 验证码/答题 - 分散请求时间
// 3. 倒计时结束才能点击，避免时间不同步导致提前请求
```

**关键策略**：
- 静态资源 CDN 化，减少服务器压力
- 页面动静分离，商品详情页静态化
- 客户端随机延迟（如 0-500ms），打散请求

#### 2.2 网关层：限流与过滤

```java
// 使用 Sentinel 进行限流
@SentinelResource(value = "flashSale", 
    blockHandler = "handleBlock",
    fallback = "handleFallback")
public Result flashSale(Long itemId, Long userId) {
    // 业务逻辑
}

// 限流规则配置
FlowRule rule = new FlowRule();
rule.setResource("flashSale");
rule.setGrade(RuleConstant.FLOW_GRADE_QPS);
rule.setCount(10000); // QPS 阈值
rule.setControlBehavior(RuleConstant.CONTROL_BEHAVIOR_WARM_UP);
```

**关键策略**：
- **用户级限流**：同一用户 N 秒内只能请求一次
- **IP 级限流**：防止恶意刷接口
- **接口级限流**：保护后端服务不被压垮
- **黑名单过滤**：拦截已知的恶意用户/IP

#### 2.3 服务层：缓存与预热

```java
@Service
public class FlashSaleService {
    
    @Autowired
    private StringRedisTemplate redisTemplate;
    
    // 活动开始前预热库存到 Redis
    public void preheatStock(Long itemId, Integer stock) {
        String key = "flash:stock:" + itemId;
        redisTemplate.opsForValue().set(key, String.valueOf(stock));
    }
    
    // 使用 Redis 原子操作扣减库存
    public boolean deductStock(Long itemId) {
        String key = "flash:stock:" + itemId;
        // DECR 是原子操作，返回扣减后的值
        Long remain = redisTemplate.opsForValue().decrement(key);
        if (remain != null && remain >= 0) {
            return true;
        }
        // 库存不足，需要回滚
        if (remain != null && remain < 0) {
            redisTemplate.opsForValue().increment(key);
        }
        return false;
    }
}
```

**更优雅的 Lua 脚本方案**：

```lua
-- deduct_stock.lua
local key = KEYS[1]
local quantity = tonumber(ARGV[1])

local stock = tonumber(redis.call('GET', key))
if stock == nil then
    return -1  -- 商品不存在
end

if stock < quantity then
    return -2  -- 库存不足
end

redis.call('DECRBY', key, quantity)
return stock - quantity  -- 返回剩余库存
```

```java
// Java 调用 Lua 脚本
public Long deductStockByLua(Long itemId, int quantity) {
    String script = loadLuaScript("deduct_stock.lua");
    String key = "flash:stock:" + itemId;
    
    return redisTemplate.execute(
        new DefaultRedisScript<>(script, Long.class),
        Collections.singletonList(key),
        String.valueOf(quantity)
    );
}
```

#### 2.4 消息队列：异步解耦

```java
@Service
public class FlashSaleService {
    
    @Autowired
    private RocketMQTemplate rocketMQTemplate;
    
    public Result flashSale(Long itemId, Long userId) {
        // 1. 校验用户是否已购买（Redis Set 判断）
        String buyerKey = "flash:buyers:" + itemId;
        Boolean isMember = redisTemplate.opsForSet().isMember(buyerKey, userId.toString());
        if (Boolean.TRUE.equals(isMember)) {
            return Result.fail("您已参与过此活动");
        }
        
        // 2. Redis 预扣库存
        Long remain = deductStockByLua(itemId, 1);
        if (remain == null || remain < 0) {
            return Result.fail("商品已售罄");
        }
        
        // 3. 记录用户已购买
        redisTemplate.opsForSet().add(buyerKey, userId.toString());
        
        // 4. 发送消息到 MQ，异步创建订单
        FlashSaleMessage message = new FlashSaleMessage(itemId, userId, 1);
        rocketMQTemplate.asyncSend("flash-sale-topic", message, new SendCallback() {
            @Override
            public void onSuccess(SendResult sendResult) {
                log.info("消息发送成功: {}", sendResult.getMsgId());
            }
            @Override
            public void onException(Throwable e) {
                // 消息发送失败，回滚库存和用户购买记录
                redisTemplate.opsForValue().increment("flash:stock:" + itemId);
                redisTemplate.opsForSet().remove(buyerKey, userId.toString());
                log.error("消息发送失败", e);
            }
        });
        
        // 5. 返回排队中，前端轮询订单状态
        return Result.success("排队中，请稍后查询订单");
    }
}

// 消费者：异步创建订单
@RocketMQMessageListener(topic = "flash-sale-topic", consumerGroup = "flash-sale-consumer")
public class FlashSaleConsumer implements RocketMQListener<FlashSaleMessage> {
    
    @Override
    public void onMessage(FlashSaleMessage message) {
        // 1. 扣减数据库库存（带乐观锁）
        // 2. 创建订单
        // 3. 发送延迟消息检查支付状态
    }
}
```

#### 2.5 数据库层：兜底保护

即使 Redis 层做了预扣，数据库层仍需保证数据一致性：

```java
@Service
public class OrderService {
    
    @Transactional(rollbackFor = Exception.class)
    public void createOrder(FlashSaleMessage message) {
        // 1. 乐观锁扣减库存
        int affected = itemMapper.deductStock(message.getItemId(), message.getQuantity());
        if (affected == 0) {
            throw new BusinessException("库存不足");
        }
        
        // 2. 创建订单
        Order order = buildOrder(message);
        orderMapper.insert(order);
        
        // 3. 发送延迟消息，30分钟后检查支付状态
        sendDelayMessage(order.getId(), 30 * 60 * 1000);
    }
}
```

```xml
<!-- 乐观锁更新，避免超卖 -->
<update id="deductStock">
    UPDATE item 
    SET stock = stock - #{quantity},
        version = version + 1
    WHERE id = #{itemId} 
      AND stock >= #{quantity}
      AND version = #{version}
</update>

<!-- 或者不依赖 version，直接用库存判断 -->
<update id="deductStockSimple">
    UPDATE item 
    SET stock = stock - #{quantity}
    WHERE id = #{itemId} 
      AND stock >= #{quantity}
</update>
```

### 三、整体架构图

```
用户请求
    │
    ▼
┌─────────────┐
│   CDN/WAF   │  ← 静态资源、DDoS 防护
└─────────────┘
    │
    ▼
┌─────────────┐
│   Nginx     │  ← 负载均衡、限流
└─────────────┘
    │
    ▼
┌─────────────┐
│   Gateway   │  ← 用户鉴权、接口限流、黑名单
└─────────────┘
    │
    ▼
┌─────────────┐
│   Redis     │  ← 库存预扣、用户去重、分布式锁
│   Cluster   │
└─────────────┘
    │
    ▼
┌─────────────┐
│  RocketMQ   │  ← 削峰填谷、异步处理
└─────────────┘
    │
    ▼
┌─────────────┐
│   MySQL     │  ← 订单持久化、库存最终一致
│   (主从)     │
└─────────────┘
```

### 四、关键技术点总结

| 层级 | 技术方案 | 解决的问题 |
|------|---------|-----------|
| 前端 | 按钮禁用、验证码、CDN | 减少无效请求、分散流量 |
| 网关 | 限流、熔断、降级 | 保护后端服务 |
| 缓存 | Redis 原子操作、Lua 脚本 | 高性能库存扣减 |
| 队列 | RocketMQ 异步 | 削峰填谷、解耦 |
| 数据库 | 乐观锁、分库分表 | 数据一致性兜底 |

### 五、常见追问及回答

**Q1：Redis 和数据库库存不一致怎么办？**

采用"最终一致性"方案：
- Redis 作为预扣减，快速返回结果
- MQ 消费时扣减数据库，若失败则重试
- 定时任务对账，修正差异

**Q2：如何防止用户重复下单？**

多层校验：
1. 前端按钮置灰
2. Redis Set 记录已购用户
3. 数据库唯一索引（user_id + item_id + activity_id）

**Q3：Redis 挂了怎么办？**

- Redis Cluster 保证高可用
- 降级方案：直接走数据库（此时需更严格限流）
- 本地缓存兜底

**Q4：如何保证库存不超卖？**

核心是"扣减操作原子化"：
- Redis：DECR 或 Lua 脚本
- 数据库：`WHERE stock >= quantity`

## 总结

抢购场景的高并发处理是一个**系统工程**，需要从前端到数据库的全链路优化：

1. **前端削峰**：减少到达后端的请求量
2. **网关限流**：保护系统不被压垮
3. **缓存预扣**：Redis 承载主要读写压力
4. **队列异步**：削峰填谷，保证最终一致性
5. **数据库兜底**：乐观锁保证不超卖

面试时建议根据自己的项目经验，结合具体数据（如 QPS、RT）来回答，会更有说服力。
