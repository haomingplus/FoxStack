---
title: 如果系统每秒最多处理10个请求，但秒杀瞬间来了300个请求，你会如何设计系统来应对？
published: 2025-01-08
description: 面对30倍于系统承载能力的瞬时流量，如何通过限流、排队、降级等策略保护系统并最大化用户体验
tags: [Java, 并发编程, 限流, 削峰填谷, 令牌桶, 消息队列]
category: Java并发编程
draft: false
---

## 题目分析

这道题的核心矛盾是：**系统容量（10 QPS）与瞬时流量（300 QPS）相差 30 倍**。

面试官想考察的是：
1. 你对限流算法的理解深度
2. 如何在"保护系统"与"用户体验"之间取得平衡
3. 系统设计的全局视野

## 参考答案

### 一、明确设计目标

面对这种场景，我们需要达成三个目标：

| 目标 | 说明 |
|-----|------|
| **系统稳定** | 绝不能让 300 个请求同时打到后端，必须保护系统 |
| **公平处理** | 先到先得，让部分用户成功，而非全部失败 |
| **快速响应** | 无论成功或失败，都要快速给用户反馈 |

### 二、整体设计思路

我会采用 **"漏斗模型"** 来设计：

```
300 请求/秒
     │
     ▼
┌─────────────┐
│  第一层     │  前端拦截 + 随机丢弃 → 过滤到 100 个
│  快速过滤   │
└─────────────┘
     │
     ▼
┌─────────────┐
│  第二层     │  令牌桶限流 → 放行 10 个/秒
│  精确限流   │
└─────────────┘
     │
     ▼
┌─────────────┐
│  第三层     │  消息队列 → 平滑处理
│  异步排队   │
└─────────────┘
     │
     ▼
  后端服务（10 QPS）
```

### 三、具体实现方案

#### 3.1 第一层：前端快速过滤

在请求到达后端之前，先过滤掉大部分流量：

```javascript
// 前端随机过滤 - 只有部分用户的请求能发出去
function submitFlashSale() {
    // 随机放行 30%，其他直接提示"系统繁忙"
    if (Math.random() > 0.3) {
        showMessage("当前人数过多，请稍后再试");
        return;
    }
    
    // 按钮置灰，防止重复点击
    disableButton();
    
    // 发起请求
    doRequest();
}
```

**效果**：300 个请求 → 约 90 个请求到达后端

#### 3.2 第二层：网关精确限流

使用**令牌桶算法**进行精确限流，这是应对此场景最合适的算法：

```java
/**
 * 令牌桶限流器
 * 特点：允许一定程度的突发流量，同时保证平均速率
 */
public class TokenBucketRateLimiter {
    
    private final long capacity;        // 桶容量
    private final long refillRate;      // 每秒填充令牌数
    private long availableTokens;       // 当前可用令牌
    private long lastRefillTime;        // 上次填充时间
    
    public TokenBucketRateLimiter(long capacity, long refillRate) {
        this.capacity = capacity;
        this.refillRate = refillRate;
        this.availableTokens = capacity;
        this.lastRefillTime = System.nanoTime();
    }
    
    public synchronized boolean tryAcquire() {
        refill();
        if (availableTokens > 0) {
            availableTokens--;
            return true;
        }
        return false;
    }
    
    private void refill() {
        long now = System.nanoTime();
        long elapsedNanos = now - lastRefillTime;
        // 计算这段时间应该补充多少令牌
        long tokensToAdd = elapsedNanos * refillRate / 1_000_000_000L;
        if (tokensToAdd > 0) {
            availableTokens = Math.min(capacity, availableTokens + tokensToAdd);
            lastRefillTime = now;
        }
    }
}
```

**实际项目中使用 Guava RateLimiter**：

```java
@Component
public class FlashSaleRateLimiter {
    
    // 每秒 10 个令牌，允许突发 20 个
    private final RateLimiter rateLimiter = RateLimiter.create(10);
    
    /**
     * 非阻塞获取令牌
     * @return true-获取成功，false-被限流
     */
    public boolean tryAcquire() {
        // 不等待，立即返回
        return rateLimiter.tryAcquire();
    }
    
    /**
     * 带超时的获取令牌
     * @param timeout 最大等待时间
     * @return true-获取成功，false-超时
     */
    public boolean tryAcquire(long timeout, TimeUnit unit) {
        return rateLimiter.tryAcquire(timeout, unit);
    }
}
```

**结合 Sentinel 实现更完善的限流**：

```java
@RestController
public class FlashSaleController {
    
    @GetMapping("/flash-sale")
    @SentinelResource(
        value = "flashSale",
        blockHandler = "handleBlock"
    )
    public Result flashSale(@RequestParam Long itemId) {
        // 业务逻辑
        return flashSaleService.doFlashSale(itemId);
    }
    
    /**
     * 被限流时的处理
     */
    public Result handleBlock(Long itemId, BlockException e) {
        return Result.fail("系统繁忙，请稍后重试");
    }
}

// 配置限流规则
@PostConstruct
public void initFlowRules() {
    FlowRule rule = new FlowRule();
    rule.setResource("flashSale");
    rule.setGrade(RuleConstant.FLOW_GRADE_QPS);
    rule.setCount(10);  // QPS 阈值
    // 排队等待模式，超时时间 500ms
    rule.setControlBehavior(RuleConstant.CONTROL_BEHAVIOR_RATE_LIMITER);
    rule.setMaxQueueingTimeMs(500);
    FlowRuleManager.loadRules(Collections.singletonList(rule));
}
```

#### 3.3 第三层：消息队列削峰

对于通过限流的请求，放入消息队列异步处理：

```java
@Service
public class FlashSaleService {
    
    @Autowired
    private RocketMQTemplate mqTemplate;
    
    @Autowired
    private FlashSaleRateLimiter rateLimiter;
    
    public Result doFlashSale(Long itemId, Long userId) {
        // 1. 限流检查
        if (!rateLimiter.tryAcquire()) {
            return Result.fail("系统繁忙，请稍后重试");
        }
        
        // 2. 基础校验（库存是否售罄等）
        if (isStockEmpty(itemId)) {
            return Result.fail("商品已售罄");
        }
        
        // 3. 发送到消息队列
        FlashSaleMessage msg = new FlashSaleMessage(itemId, userId);
        mqTemplate.syncSend("flash-sale-topic", msg);
        
        // 4. 返回排队状态
        return Result.success("正在排队处理，请稍后查询结果");
    }
}

/**
 * 消费者 - 按系统能力平滑消费
 */
@RocketMQMessageListener(
    topic = "flash-sale-topic",
    consumerGroup = "flash-sale-group"
)
public class FlashSaleConsumer implements RocketMQListener<FlashSaleMessage> {
    
    // 控制消费速度，与系统处理能力匹配
    private final RateLimiter consumeRateLimiter = RateLimiter.create(10);
    
    @Override
    public void onMessage(FlashSaleMessage message) {
        // 控制消费速率
        consumeRateLimiter.acquire();
        
        // 执行真正的业务逻辑
        processFlashSale(message);
    }
}
```

### 四、不同限流算法对比

针对这道题，我们来对比四种常见的限流算法：

#### 4.1 固定窗口计数器

```java
public class FixedWindowCounter {
    private final long windowSize = 1000; // 1秒窗口
    private final int limit = 10;
    private long windowStart;
    private int count;
    
    public synchronized boolean tryAcquire() {
        long now = System.currentTimeMillis();
        if (now - windowStart >= windowSize) {
            windowStart = now;
            count = 0;
        }
        if (count < limit) {
            count++;
            return true;
        }
        return false;
    }
}
```

❌ **问题**：存在临界问题，窗口交界处可能瞬间通过 2 倍流量

#### 4.2 滑动窗口计数器

```java
public class SlidingWindowCounter {
    private final int windowCount = 10;      // 分成 10 个小窗口
    private final long windowSize = 1000;    // 总窗口 1 秒
    private final int limit = 10;
    private final int[] counters = new int[windowCount];
    private long lastTime;
    
    public synchronized boolean tryAcquire() {
        long now = System.currentTimeMillis();
        int currentWindow = (int) ((now / (windowSize / windowCount)) % windowCount);
        
        // 清理过期窗口
        clearExpiredWindows(now);
        
        int total = Arrays.stream(counters).sum();
        if (total < limit) {
            counters[currentWindow]++;
            lastTime = now;
            return true;
        }
        return false;
    }
}
```

✅ 解决了临界问题，但实现相对复杂

#### 4.3 漏桶算法

```java
public class LeakyBucket {
    private final long capacity = 10;     // 桶容量
    private final long leakRate = 10;     // 每秒漏出数量
    private long water = 0;               // 当前水量
    private long lastLeakTime = System.currentTimeMillis();
    
    public synchronized boolean tryAcquire() {
        leak(); // 先漏水
        if (water < capacity) {
            water++;
            return true;
        }
        return false;
    }
    
    private void leak() {
        long now = System.currentTimeMillis();
        long elapsed = now - lastLeakTime;
        long leaked = elapsed * leakRate / 1000;
        water = Math.max(0, water - leaked);
        lastLeakTime = now;
    }
}
```

✅ 输出速率恒定，但不允许突发流量

#### 4.4 令牌桶算法（推荐）

```
特点：
- 以固定速率生成令牌
- 桶满则丢弃新令牌
- 请求需获取令牌才能通过
- 允许一定突发（桶内积累的令牌）
```

✅ **最适合本场景**：既能应对突发，又能保证平均速率

### 五、300 请求的完整流转

让我们追踪这 300 个请求的命运：

```
时间点 T0：300 个请求同时到达

第一层（前端过滤 70%）
├── 210 个 → 直接返回"系统繁忙"
└── 90 个 → 进入后端

第二层（令牌桶限流 10 QPS）
├── 10 个 → 立即获得令牌，进入处理
├── 40 个 → 排队等待（最多等 500ms）
│   ├── 5 个 → 等到令牌，进入处理（T0+100ms ~ T0+500ms）
│   └── 35 个 → 等待超时，返回"系统繁忙"
└── 40 个 → 队列已满，直接拒绝

第三层（消息队列）
├── 15 个请求进入 MQ 排队
└── 消费者按 10/秒 速度消费

最终结果：
- 15 个请求成功处理（约 1.5 秒内完成）
- 285 个请求快速返回失败（< 500ms）
```

### 六、更极端情况的优化

如果流量更大（如 3000 QPS），还可以引入：

#### 6.1 本地缓存 + 库存预分配

```java
@Component
public class LocalStockCache {
    // 本地库存缓存，减少 Redis 访问
    private final AtomicInteger localStock = new AtomicInteger(0);
    
    /**
     * 从中心 Redis 分配库存到本地
     * 假设有 10 个实例，每个实例分配 10 个库存
     */
    @Scheduled(fixedRate = 1000)
    public void allocateStock() {
        // 从 Redis 获取并扣减一批库存到本地
        Long allocated = redisTemplate.execute(
            ALLOCATE_SCRIPT, 
            Collections.singletonList("global:stock"),
            "10"  // 每次分配 10 个
        );
        if (allocated != null && allocated > 0) {
            localStock.addAndGet(allocated.intValue());
        }
    }
    
    public boolean deductLocal() {
        return localStock.decrementAndGet() >= 0;
    }
}
```

#### 6.2 多级限流

```java
// 用户级别：同一用户 5 秒内只能请求 1 次
// IP 级别：同一 IP 1 秒内最多 10 次
// 接口级别：整体 QPS 上限
@Configuration
public class MultiLevelRateLimitConfig {
    
    @Bean
    public FilterRegistrationBean<RateLimitFilter> rateLimitFilter() {
        FilterRegistrationBean<RateLimitFilter> bean = new FilterRegistrationBean<>();
        bean.setFilter(new RateLimitFilter());
        bean.addUrlPatterns("/api/flash-sale/*");
        return bean;
    }
}

public class RateLimitFilter implements Filter {
    
    // 用户级限流器
    private final Cache<Long, RateLimiter> userLimiters = Caffeine.newBuilder()
        .expireAfterAccess(1, TimeUnit.MINUTES)
        .build();
    
    // IP 级限流器
    private final Cache<String, RateLimiter> ipLimiters = Caffeine.newBuilder()
        .expireAfterAccess(1, TimeUnit.MINUTES)
        .build();
    
    @Override
    public void doFilter(ServletRequest req, ServletResponse resp, FilterChain chain) {
        HttpServletRequest request = (HttpServletRequest) req;
        
        // 1. 用户级限流
        Long userId = getUserId(request);
        RateLimiter userLimiter = userLimiters.get(userId, 
            k -> RateLimiter.create(0.2)); // 5秒1个
        if (!userLimiter.tryAcquire()) {
            reject(resp, "操作太频繁");
            return;
        }
        
        // 2. IP 级限流
        String ip = getClientIp(request);
        RateLimiter ipLimiter = ipLimiters.get(ip, 
            k -> RateLimiter.create(10)); // 每秒10个
        if (!ipLimiter.tryAcquire()) {
            reject(resp, "请求过于频繁");
            return;
        }
        
        chain.doFilter(req, resp);
    }
}
```

### 七、方案总结

| 策略 | 处理的请求 | 响应时间 | 用户感知 |
|-----|-----------|---------|---------|
| 前端过滤 | 70% 请求 | < 10ms | 系统繁忙（可重试） |
| 限流拒绝 | 25% 请求 | < 500ms | 系统繁忙（可重试） |
| 队列处理 | 5% 请求 | 1-2s | 排队中/成功 |

**核心思想**：

1. **快速失败优于长时间等待** - 被拒绝的用户可以快速重试
2. **分层过滤逐级削减** - 每一层都减少流量，保护下一层
3. **异步化削峰填谷** - 用队列将瞬时流量转为平滑流量
4. **保证公平性** - 先到的请求有更大概率成功

## 面试加分点

回答时可以补充：

1. **具体数据**："我们线上系统用这套方案，秒杀峰值 10 万 QPS，系统稳定运行"
2. **监控告警**："配合 Prometheus + Grafana 监控限流指标，触发阈值及时告警"
3. **动态调整**："限流阈值支持动态配置，通过配置中心实时调整"
4. **A/B 测试**："不同限流策略可以灰度验证效果"
