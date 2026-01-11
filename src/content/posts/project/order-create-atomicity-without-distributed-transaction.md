---
title: 订单创建链路原子性保障：不用分布式事务的实战方案
published: 2024-05-12
description: 深入解析营销交易平台如何通过本地事务+消息表+补偿机制保障"扣库存→创建订单→发消息"的最终一致性
tags: [Java, 分布式事务, 最终一致性, 本地消息表, 补偿机制, 高并发]
category: 系统架构
draft: false
---

## 面试题

在订单创建链路中，需要保证"扣库存 → 创建订单 → 发消息"这三步的原子性，但又不想用分布式事务，你们是怎么处理的？如果中间某一步失败了呢？

---

## 一、问题分析

### 1.1 业务场景

```
订单创建核心流程：

用户下单
    │
    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  Step 1: 扣库存                                                              │
│  └── 调用库存服务，扣减商品库存                                               │
├─────────────────────────────────────────────────────────────────────────────┤
│  Step 2: 创建订单                                                            │
│  └── 写入订单数据库，状态=待支付                                              │
├─────────────────────────────────────────────────────────────────────────────┤
│  Step 3: 发消息                                                              │
│  └── 发送 MQ 消息，通知下游系统（优惠券核销、积分发放、通知等）                  │
└─────────────────────────────────────────────────────────────────────────────┘
    │
    ▼
返回下单成功
```

### 1.2 核心挑战

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          一致性问题场景                                       │
└─────────────────────────────────────────────────────────────────────────────┘

场景1：扣库存成功，创建订单失败
├── 库存扣了，但订单没创建
├── 用户没买到，库存却少了
└── 需要：回滚库存

场景2：扣库存成功，创建订单成功，发消息失败
├── 订单创建了，但下游没收到通知
├── 优惠券没核销、积分没发放
└── 需要：重试发消息 或 补偿

场景3：发消息成功，但 MQ 消费失败
├── 消息发了，但消费者处理失败
└── 需要：消费重试 + 幂等

场景4：网络超时，不知道成功还是失败
├── 调用库存服务超时，实际可能成功了
└── 需要：查询确认 + 幂等
```

### 1.3 为什么不用分布式事务？

```
分布式事务方案：

1. 2PC/XA
   ├── 性能差：同步阻塞，吞吐量低
   ├── 可用性差：协调者单点
   └── 不适合高并发场景

2. TCC
   ├── 侵入性强：需要实现 Try/Confirm/Cancel
   ├── 开发成本高
   └── 适合强一致性场景，但复杂

3. Seata AT
   ├── 有性能开销
   ├── 需要额外组件
   └── 增加系统复杂度

我们的选择：
└── 本地事务 + 消息表 + 补偿机制
    ├── 实现简单
    ├── 性能高
    ├── 最终一致性（业务可接受）
    └── 经过双11验证
```

---

## 二、整体方案设计

### 2.1 核心思路

```
核心原则：
1. 能用本地事务的，绝不跨服务
2. 消息必达：本地消息表 + 定时重试
3. 操作幂等：任何步骤都可以安全重试
4. 补偿兜底：失败后有补偿机制

设计思路：
┌─────────────────────────────────────────────────────────────────────────────┐
│                                                                             │
│    扣库存（Redis）   创建订单 + 消息表（本地事务）     异步发消息             │
│         │                      │                         │                 │
│         │                      │                         │                 │
│    先扣 Redis      ───────────►│ 同一个事务保证原子性     │                 │
│    预占库存                     │ - 写订单表              │                 │
│         │                      │ - 写本地消息表          │──────►  定时扫描  │
│         │                      │                         │       发送MQ    │
│         │                      │                         │                 │
│    失败则快速返回               成功后异步落库存到DB       │                 │
│                                                          │                 │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 2.2 架构图

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        订单创建链路架构                                       │
└─────────────────────────────────────────────────────────────────────────────┘

                            ┌─────────────────┐
                            │    用户下单     │
                            └────────┬────────┘
                                     │
                                     ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                            订单服务                                          │
│                                                                             │
│   ┌─────────────────────────────────────────────────────────────────────┐  │
│   │                    @Transactional（本地事务）                        │  │
│   │                                                                     │  │
│   │   ┌─────────────┐     ┌─────────────┐     ┌─────────────┐         │  │
│   │   │ 1.Redis扣库存│ ──► │ 2.创建订单  │ ──► │ 3.写消息表  │         │  │
│   │   │   (预扣)     │     │   (DB)      │     │   (DB)      │         │  │
│   │   └─────────────┘     └─────────────┘     └─────────────┘         │  │
│   │         │                    │                   │                 │  │
│   │         │                    └───────────────────┘                 │  │
│   │         │                           同一个本地事务                  │  │
│   └─────────│───────────────────────────────────────────────────────────┘  │
│             │                                                               │
│             │  失败回滚 Redis                                               │
│             ▼                                                               │
│   ┌─────────────────────────────────────────────────────────────────────┐  │
│   │                      异步任务                                        │  │
│   │                                                                     │  │
│   │   ┌─────────────┐     ┌─────────────┐     ┌─────────────┐         │  │
│   │   │ 4.扫描消息表│ ──► │ 5.发送MQ   │ ──► │ 6.更新状态  │         │  │
│   │   │   (定时)    │     │             │     │   (已发送)  │         │  │
│   │   └─────────────┘     └─────────────┘     └─────────────┘         │  │
│   │                                                                     │  │
│   │   ┌─────────────┐                                                   │  │
│   │   │ 7.异步落库存│  Redis 库存同步到 MySQL（最终一致）                 │  │
│   │   │   到MySQL   │                                                   │  │
│   │   └─────────────┘                                                   │  │
│   └─────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
                                     │
                                     ▼
                            ┌─────────────────┐
                            │    RocketMQ     │
                            └────────┬────────┘
                                     │
                    ┌────────────────┼────────────────┐
                    ▼                ▼                ▼
             ┌──────────┐     ┌──────────┐     ┌──────────┐
             │ 优惠券服务│     │ 积分服务 │     │ 通知服务 │
             └──────────┘     └──────────┘     └──────────┘
```

---

## 三、核心实现

### 3.1 数据库设计

```sql
-- 订单表
CREATE TABLE t_order (
    id BIGINT PRIMARY KEY,
    order_no VARCHAR(32) NOT NULL UNIQUE,
    user_id BIGINT NOT NULL,
    product_id BIGINT NOT NULL,
    quantity INT NOT NULL,
    amount DECIMAL(10,2) NOT NULL,
    status TINYINT NOT NULL DEFAULT 0,  -- 0:待支付 1:已支付 2:已取消
    create_time DATETIME NOT NULL,
    update_time DATETIME NOT NULL,
    INDEX idx_user_id(user_id),
    INDEX idx_order_no(order_no)
) ENGINE=InnoDB;

-- 本地消息表
CREATE TABLE t_local_message (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    message_id VARCHAR(64) NOT NULL UNIQUE,  -- 消息唯一ID，用于幂等
    business_type VARCHAR(32) NOT NULL,       -- 业务类型：ORDER_CREATE
    business_id VARCHAR(64) NOT NULL,         -- 业务ID：订单号
    topic VARCHAR(64) NOT NULL,               -- MQ Topic
    message_body TEXT NOT NULL,               -- 消息内容（JSON）
    status TINYINT NOT NULL DEFAULT 0,        -- 0:待发送 1:发送中 2:已发送 3:失败
    retry_count INT NOT NULL DEFAULT 0,       -- 重试次数
    max_retry INT NOT NULL DEFAULT 5,         -- 最大重试次数
    next_retry_time DATETIME,                 -- 下次重试时间
    create_time DATETIME NOT NULL,
    update_time DATETIME NOT NULL,
    INDEX idx_status_retry(status, next_retry_time),
    INDEX idx_business(business_type, business_id)
) ENGINE=InnoDB;

-- 库存表（MySQL，用于最终一致性校验）
CREATE TABLE t_stock (
    id BIGINT PRIMARY KEY,
    product_id BIGINT NOT NULL UNIQUE,
    total_stock INT NOT NULL,       -- 总库存
    locked_stock INT NOT NULL,      -- 锁定库存（已下单未支付）
    sold_stock INT NOT NULL,        -- 已售库存
    available_stock INT AS (total_stock - locked_stock - sold_stock) STORED,  -- 可用库存
    version INT NOT NULL DEFAULT 0, -- 乐观锁版本号
    INDEX idx_product_id(product_id)
) ENGINE=InnoDB;
```

### 3.2 核心服务代码

```java
@Service
@Slf4j
public class OrderService {

    @Autowired
    private StockService stockService;
    
    @Autowired
    private OrderMapper orderMapper;
    
    @Autowired
    private LocalMessageMapper messageMapper;
    
    @Autowired
    private TransactionTemplate transactionTemplate;

    /**
     * 创建订单（核心方法）
     */
    public OrderResult createOrder(CreateOrderRequest request) {
        String orderNo = generateOrderNo();
        
        // Step 1: Redis 预扣库存（快速失败）
        boolean stockDeducted = false;
        try {
            stockDeducted = stockService.deductStockInRedis(
                    request.getProductId(), 
                    request.getQuantity(),
                    orderNo  // 幂等键
            );
            
            if (!stockDeducted) {
                return OrderResult.fail("库存不足");
            }
            
            // Step 2 & 3: 本地事务（创建订单 + 写消息表）
            Order order = executeLocalTransaction(request, orderNo);
            
            // Step 4: 异步发送消息（由定时任务处理）
            // 这里不直接发，而是让定时任务扫描消息表发送
            
            return OrderResult.success(order);
            
        } catch (Exception e) {
            log.error("创建订单失败, orderNo={}", orderNo, e);
            
            // 回滚 Redis 库存
            if (stockDeducted) {
                stockService.rollbackStockInRedis(
                        request.getProductId(), 
                        request.getQuantity(),
                        orderNo
                );
            }
            
            return OrderResult.fail("系统繁忙，请稍后重试");
        }
    }

    /**
     * 本地事务：创建订单 + 写消息表
     */
    private Order executeLocalTransaction(CreateOrderRequest request, String orderNo) {
        return transactionTemplate.execute(status -> {
            // 创建订单
            Order order = buildOrder(request, orderNo);
            orderMapper.insert(order);
            
            // 写本地消息表
            LocalMessage message = buildLocalMessage(order);
            messageMapper.insert(message);
            
            return order;
        });
    }

    /**
     * 构建本地消息
     */
    private LocalMessage buildLocalMessage(Order order) {
        LocalMessage message = new LocalMessage();
        message.setMessageId(UUID.randomUUID().toString());
        message.setBusinessType("ORDER_CREATE");
        message.setBusinessId(order.getOrderNo());
        message.setTopic("order-create-topic");
        message.setMessageBody(JSON.toJSONString(buildOrderMessage(order)));
        message.setStatus(0);  // 待发送
        message.setRetryCount(0);
        message.setMaxRetry(5);
        message.setNextRetryTime(new Date());
        return message;
    }
}
```

### 3.3 库存服务：Redis 预扣

```java
@Service
@Slf4j
public class StockService {

    @Autowired
    private StringRedisTemplate redisTemplate;
    
    @Autowired
    private StockMapper stockMapper;

    // Lua 脚本：原子扣减库存
    private static final String DEDUCT_STOCK_SCRIPT =
        "local stock = tonumber(redis.call('GET', KEYS[1])) " +
        "if stock == nil then return -1 end " +                    // 库存不存在
        "if stock < tonumber(ARGV[1]) then return -2 end " +      // 库存不足
        "redis.call('DECRBY', KEYS[1], ARGV[1]) " +
        "redis.call('SADD', KEYS[2], ARGV[2]) " +                 // 记录订单号（幂等）
        "return stock - tonumber(ARGV[1])";

    // Lua 脚本：回滚库存
    private static final String ROLLBACK_STOCK_SCRIPT =
        "local exists = redis.call('SISMEMBER', KEYS[2], ARGV[2]) " +
        "if exists == 1 then " +
        "  redis.call('INCRBY', KEYS[1], ARGV[1]) " +
        "  redis.call('SREM', KEYS[2], ARGV[2]) " +
        "  return 1 " +
        "end " +
        "return 0";  // 已回滚或不存在

    /**
     * Redis 扣减库存
     * @return true 成功，false 失败
     */
    public boolean deductStockInRedis(Long productId, int quantity, String orderNo) {
        String stockKey = "stock:" + productId;
        String deductedOrdersKey = "stock:deducted:" + productId;
        
        // 幂等检查：该订单是否已扣过库存
        if (redisTemplate.opsForSet().isMember(deductedOrdersKey, orderNo)) {
            log.info("订单已扣过库存，幂等返回成功, orderNo={}", orderNo);
            return true;
        }
        
        Long result = redisTemplate.execute(
                new DefaultRedisScript<>(DEDUCT_STOCK_SCRIPT, Long.class),
                Arrays.asList(stockKey, deductedOrdersKey),
                String.valueOf(quantity),
                orderNo
        );
        
        if (result == null || result == -1) {
            log.error("库存数据不存在, productId={}", productId);
            return false;
        }
        if (result == -2) {
            log.info("库存不足, productId={}, quantity={}", productId, quantity);
            return false;
        }
        
        log.info("Redis扣库存成功, productId={}, quantity={}, remaining={}", 
                productId, quantity, result);
        return true;
    }

    /**
     * 回滚 Redis 库存
     */
    public boolean rollbackStockInRedis(Long productId, int quantity, String orderNo) {
        String stockKey = "stock:" + productId;
        String deductedOrdersKey = "stock:deducted:" + productId;
        
        Long result = redisTemplate.execute(
                new DefaultRedisScript<>(ROLLBACK_STOCK_SCRIPT, Long.class),
                Arrays.asList(stockKey, deductedOrdersKey),
                String.valueOf(quantity),
                orderNo
        );
        
        log.info("回滚Redis库存, productId={}, quantity={}, result={}", 
                productId, quantity, result);
        return result != null && result == 1;
    }
}
```

### 3.4 本地消息表：定时发送

```java
@Component
@Slf4j
public class LocalMessageSender {

    @Autowired
    private LocalMessageMapper messageMapper;
    
    @Autowired
    private RocketMQTemplate mqTemplate;

    /**
     * 定时扫描待发送消息
     * 每秒执行一次
     */
    @Scheduled(fixedDelay = 1000)
    public void scanAndSendMessages() {
        // 查询待发送的消息（状态=0 或 状态=3且可重试）
        List<LocalMessage> messages = messageMapper.selectPendingMessages(100);
        
        for (LocalMessage message : messages) {
            try {
                sendMessage(message);
            } catch (Exception e) {
                log.error("发送消息失败, messageId={}", message.getMessageId(), e);
                handleSendFailure(message);
            }
        }
    }

    /**
     * 发送单条消息
     */
    private void sendMessage(LocalMessage message) {
        // 更新状态为发送中（防止重复发送）
        int updated = messageMapper.updateStatus(
                message.getId(), 
                0,  // 原状态：待发送
                1   // 新状态：发送中
        );
        
        if (updated == 0) {
            // 已被其他实例处理
            return;
        }
        
        try {
            // 发送 MQ
            SendResult result = mqTemplate.syncSend(
                    message.getTopic(),
                    MessageBuilder
                            .withPayload(message.getMessageBody())
                            .setHeader("messageId", message.getMessageId())
                            .build()
            );
            
            if (result.getSendStatus() == SendStatus.SEND_OK) {
                // 发送成功，更新状态
                messageMapper.updateStatus(message.getId(), 1, 2);
                log.info("消息发送成功, messageId={}", message.getMessageId());
            } else {
                throw new RuntimeException("MQ发送状态异常: " + result.getSendStatus());
            }
            
        } catch (Exception e) {
            // 发送失败，回滚状态，等待重试
            messageMapper.updateStatus(message.getId(), 1, 0);
            throw e;
        }
    }

    /**
     * 处理发送失败
     */
    private void handleSendFailure(LocalMessage message) {
        int retryCount = message.getRetryCount() + 1;
        
        if (retryCount >= message.getMaxRetry()) {
            // 超过最大重试次数，标记为失败
            messageMapper.updateToFailed(message.getId());
            
            // 发送告警
            alertService.send(String.format(
                    "本地消息发送失败，已达最大重试次数, messageId=%s, businessId=%s",
                    message.getMessageId(), message.getBusinessId()
            ));
        } else {
            // 更新重试次数和下次重试时间（指数退避）
            Date nextRetryTime = calculateNextRetryTime(retryCount);
            messageMapper.updateForRetry(message.getId(), retryCount, nextRetryTime);
        }
    }

    /**
     * 计算下次重试时间（指数退避）
     * 1s, 2s, 4s, 8s, 16s...
     */
    private Date calculateNextRetryTime(int retryCount) {
        long delaySeconds = (long) Math.pow(2, retryCount);
        return new Date(System.currentTimeMillis() + delaySeconds * 1000);
    }
}
```

### 3.5 消费者：幂等处理

```java
@Component
@RocketMQMessageListener(
        topic = "order-create-topic",
        consumerGroup = "coupon-consumer-group"
)
@Slf4j
public class OrderCreateConsumer implements RocketMQListener<String> {

    @Autowired
    private StringRedisTemplate redisTemplate;
    
    @Autowired
    private CouponService couponService;

    @Override
    public void onMessage(String messageBody) {
        OrderMessage message = JSON.parseObject(messageBody, OrderMessage.class);
        String messageId = message.getMessageId();
        
        // 幂等检查
        String idempotentKey = "msg:consumed:" + messageId;
        Boolean isNew = redisTemplate.opsForValue()
                .setIfAbsent(idempotentKey, "1", 7, TimeUnit.DAYS);
        
        if (Boolean.FALSE.equals(isNew)) {
            log.info("消息已消费，幂等跳过, messageId={}", messageId);
            return;
        }
        
        try {
            // 业务处理：核销优惠券
            couponService.useCoupon(message.getCouponId(), message.getOrderNo());
            log.info("优惠券核销成功, couponId={}, orderNo={}", 
                    message.getCouponId(), message.getOrderNo());
                    
        } catch (Exception e) {
            // 处理失败，删除幂等标记，允许重试
            redisTemplate.delete(idempotentKey);
            throw e;  // 抛出异常，触发 MQ 重试
        }
    }
}
```

---

## 四、异常场景处理

### 4.1 各步骤失败处理

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          异常场景处理矩阵                                    │
├───────────────────────┬─────────────────────────────────────────────────────┤
│       失败步骤         │                    处理方式                         │
├───────────────────────┼─────────────────────────────────────────────────────┤
│ Step1: Redis扣库存失败 │ 直接返回失败，无需回滚                              │
│                       │ （还没做任何变更）                                   │
├───────────────────────┼─────────────────────────────────────────────────────┤
│ Step2: 创建订单失败    │ 事务回滚 + 回滚 Redis 库存                          │
│ (DB写入失败)          │ try-catch 中调用 rollbackStockInRedis               │
├───────────────────────┼─────────────────────────────────────────────────────┤
│ Step3: 写消息表失败    │ 同 Step2，本地事务一起回滚                          │
│ (与订单同事务)         │ Redis 库存也回滚                                    │
├───────────────────────┼─────────────────────────────────────────────────────┤
│ Step4: 发MQ失败        │ 消息表状态保持"待发送"                              │
│ (异步任务)            │ 定时任务重试，指数退避                               │
│                       │ 超过最大重试次数则告警人工处理                       │
├───────────────────────┼─────────────────────────────────────────────────────┤
│ Step5: MQ消费失败      │ MQ 自动重试                                         │
│ (下游消费者)          │ 消费者保证幂等                                       │
│                       │ 死信队列兜底                                         │
└───────────────────────┴─────────────────────────────────────────────────────┘
```

### 4.2 流程图：失败场景

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                      失败场景处理流程                                        │
└─────────────────────────────────────────────────────────────────────────────┘

场景1：Redis 扣库存失败
─────────────────────────
    Redis 扣库存 ──失败──► 直接返回"库存不足" ✓
         │
      无后续操作


场景2：本地事务失败
─────────────────────────
    Redis 扣库存 ──成功──► 创建订单 ──失败──┐
         │                                  │
         │                                  ▼
         │                           事务回滚
         │                                  │
         │◄─────── 回滚库存 ◄───────────────┘
         │
      返回"系统繁忙"


场景3：MQ 发送失败（不影响下单）
─────────────────────────
    Redis 扣库存 ──成功──► 创建订单 ──成功──► 写消息表 ──成功──► 返回成功 ✓
                                                  │
                                                  │ 消息状态=待发送
                                                  ▼
                                           定时任务扫描
                                                  │
                                           发送 MQ ──失败──► 重试（指数退避）
                                                  │              │
                                               成功 ✓         超过最大次数
                                                              │
                                                           告警人工处理
```

---

## 五、补偿机制

### 5.1 库存最终一致性补偿

```java
/**
 * 库存一致性校验任务
 * 定期对比 Redis 和 MySQL 库存，修复不一致
 */
@Component
@Slf4j
public class StockConsistencyChecker {

    @Autowired
    private StringRedisTemplate redisTemplate;
    
    @Autowired
    private StockMapper stockMapper;

    /**
     * 每小时执行一次库存一致性校验
     */
    @Scheduled(cron = "0 0 * * * ?")
    public void checkStockConsistency() {
        List<Stock> allStocks = stockMapper.selectAll();
        
        for (Stock dbStock : allStocks) {
            String redisKey = "stock:" + dbStock.getProductId();
            String redisValue = redisTemplate.opsForValue().get(redisKey);
            
            if (redisValue == null) {
                // Redis 没有，从 DB 同步
                redisTemplate.opsForValue().set(redisKey, 
                        String.valueOf(dbStock.getAvailableStock()));
                log.warn("Redis库存缺失，已从DB同步, productId={}", dbStock.getProductId());
                continue;
            }
            
            int redisStock = Integer.parseInt(redisValue);
            int dbAvailableStock = dbStock.getAvailableStock();
            
            // 允许小误差（因为异步同步有延迟）
            if (Math.abs(redisStock - dbAvailableStock) > 10) {
                log.error("库存不一致! productId={}, redis={}, db={}", 
                        dbStock.getProductId(), redisStock, dbAvailableStock);
                
                // 发送告警，人工确认后修复
                alertService.send(String.format(
                        "库存不一致告警: productId=%d, redis=%d, db=%d",
                        dbStock.getProductId(), redisStock, dbAvailableStock
                ));
            }
        }
    }
}
```

### 5.2 订单超时未支付补偿

```java
/**
 * 订单超时关闭任务
 * 处理超时未支付的订单，释放库存
 */
@Component
@Slf4j
public class OrderTimeoutHandler {

    @Autowired
    private OrderMapper orderMapper;
    
    @Autowired
    private StockService stockService;

    /**
     * 每分钟扫描超时订单
     */
    @Scheduled(fixedDelay = 60000)
    public void handleTimeoutOrders() {
        // 查询 30 分钟前创建且未支付的订单
        Date timeoutTime = new Date(System.currentTimeMillis() - 30 * 60 * 1000);
        List<Order> timeoutOrders = orderMapper.selectTimeoutOrders(timeoutTime);
        
        for (Order order : timeoutOrders) {
            try {
                cancelOrder(order);
            } catch (Exception e) {
                log.error("关闭超时订单失败, orderNo={}", order.getOrderNo(), e);
            }
        }
    }

    /**
     * 取消订单，释放库存
     */
    @Transactional
    public void cancelOrder(Order order) {
        // 更新订单状态为已取消
        int updated = orderMapper.updateStatus(
                order.getId(), 
                0,  // 原状态：待支付
                2   // 新状态：已取消
        );
        
        if (updated == 0) {
            // 状态已变更，跳过
            return;
        }
        
        // 释放 Redis 库存
        stockService.releaseStock(order.getProductId(), order.getQuantity());
        
        // 释放 MySQL 锁定库存
        stockMapper.releaseLocked(order.getProductId(), order.getQuantity());
        
        log.info("超时订单已关闭, orderNo={}", order.getOrderNo());
    }
}
```

### 5.3 消息表清理

```java
/**
 * 本地消息表清理任务
 */
@Component
@Slf4j
public class LocalMessageCleaner {

    @Autowired
    private LocalMessageMapper messageMapper;

    /**
     * 每天凌晨清理已发送的消息
     */
    @Scheduled(cron = "0 0 3 * * ?")
    public void cleanSentMessages() {
        // 删除 7 天前已发送的消息
        Date deadline = new Date(System.currentTimeMillis() - 7 * 24 * 60 * 60 * 1000L);
        int deleted = messageMapper.deleteSentBefore(deadline);
        log.info("清理已发送消息, count={}", deleted);
    }

    /**
     * 处理长期失败的消息
     */
    @Scheduled(cron = "0 0 4 * * ?")
    public void handleFailedMessages() {
        List<LocalMessage> failedMessages = messageMapper.selectFailed();
        
        for (LocalMessage message : failedMessages) {
            // 记录到失败日志表，供人工排查
            failedMessageLogMapper.insert(message);
            
            // 删除原消息
            messageMapper.delete(message.getId());
        }
        
        if (!failedMessages.isEmpty()) {
            alertService.send(String.format("有 %d 条消息发送失败，请排查", failedMessages.size()));
        }
    }
}
```

---

## 六、方案对比

### 6.1 与其他方案对比

| 方案 | 一致性 | 性能 | 复杂度 | 适用场景 |
|-----|--------|------|--------|---------|
| **2PC/XA** | 强一致 | 低 | 中 | 金融核心场景 |
| **TCC** | 强一致 | 中 | 高 | 需要强一致+高性能 |
| **Saga** | 最终一致 | 高 | 高 | 长事务场景 |
| **本地消息表** ⭐ | 最终一致 | 高 | 低 | 大多数业务场景 |
| **事务消息** | 最终一致 | 高 | 低 | 消息可靠投递 |

### 6.2 我们的选择理由

```
选择"本地消息表 + 补偿"的原因：

1. 业务可接受最终一致性
   └── 优惠券核销、积分发放晚几秒没问题

2. 高性能要求
   └── 双11 峰值 50W+ QPS
   └── 分布式事务会成为瓶颈

3. 实现简单
   └── 不需要引入额外组件（Seata等）
   └── 只需要多一张消息表 + 定时任务

4. 可靠性高
   └── 消息落库，不会丢失
   └── 定时重试，保证必达

5. 经过验证
   └── 双11/618 大促零故障
```

---

## 七、总结

**面试回答模板**：

> "在订单创建链路中，我们采用**本地消息表 + 补偿机制**来保证最终一致性：
>
> **整体流程**：
> 1. **Redis 预扣库存**：快速校验和扣减，失败则直接返回
> 2. **本地事务**：创建订单 + 写本地消息表，保证原子性
> 3. **异步发消息**：定时任务扫描消息表，发送 MQ
>
> **失败处理**：
> - Redis 扣库存失败：直接返回，无需回滚
> - 本地事务失败：事务回滚 + 回滚 Redis 库存
> - MQ 发送失败：消息表保底，定时重试（指数退避）
> - MQ 消费失败：消费者幂等 + MQ 重试 + 死信队列
>
> **补偿机制**：
> - 库存一致性校验：定时对比 Redis 和 MySQL
> - 超时订单处理：定时关闭未支付订单，释放库存
> - 消息表清理：清理已发送消息，处理失败消息
>
> **为什么不用分布式事务**：
> - 业务可接受最终一致性
> - 高性能要求（50W+ QPS）
> - 实现简单，不引入额外组件
>
> 这套方案经过双11/618 验证，能够在保证数据最终一致的前提下，支撑高并发场景。"
