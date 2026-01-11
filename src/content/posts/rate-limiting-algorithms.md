---
title: 接口限流的常见策略与算法详解
published: 2025-01-08
description: 全面解析固定窗口、滑动窗口、漏桶、令牌桶四大限流算法的原理、实现与适用场景，以及分布式环境下的限流方案
tags: [Java, 并发编程, 限流算法, 令牌桶, 漏桶, Sentinel, Gateway]
category: Java并发编程
draft: false
---

## 题目分析

这是一道高频面试题，考察候选人对**流量控制**的理解。回答时应该：

1. 能说清楚每种算法的**原理**和**实现**
2. 能对比各算法的**优缺点**
3. 能结合实际场景说明**如何选择**
4. 最好能延伸到**分布式限流**方案

## 参考答案

### 一、限流的本质

限流的核心目的是**保护系统**，确保在流量超出系统承载能力时，系统仍能稳定运行。

```
没有限流：
100万请求 → 系统崩溃 → 所有用户受影响

有限流：
100万请求 → 限流放行1万 → 系统稳定 → 1万用户正常服务
```

### 二、四大经典限流算法

#### 2.1 固定窗口计数器（Fixed Window Counter）

**原理**：将时间划分为固定大小的窗口，每个窗口内维护一个计数器，超过阈值则拒绝。

```
时间窗口：1秒    阈值：100

|-------- 窗口1 --------|-------- 窗口2 --------|
0s                     1s                     2s
    [计数: 0→100]            [计数: 0→...]
    第101个请求被拒绝
```

**Java 实现**：

```java
public class FixedWindowRateLimiter {
    
    private final long windowSize;      // 窗口大小（毫秒）
    private final int maxRequests;      // 窗口内最大请求数
    private long windowStart;           // 当前窗口起始时间
    private int requestCount;           // 当前窗口请求计数
    
    public FixedWindowRateLimiter(long windowSizeMs, int maxRequests) {
        this.windowSize = windowSizeMs;
        this.maxRequests = maxRequests;
        this.windowStart = System.currentTimeMillis();
        this.requestCount = 0;
    }
    
    public synchronized boolean tryAcquire() {
        long now = System.currentTimeMillis();
        
        // 进入新窗口，重置计数器
        if (now - windowStart >= windowSize) {
            windowStart = now;
            requestCount = 0;
        }
        
        // 判断是否超过阈值
        if (requestCount < maxRequests) {
            requestCount++;
            return true;
        }
        return false;
    }
}
```

**致命缺陷 —— 临界突刺问题**：

```
阈值：100请求/秒

时间线：
|-------- 窗口1 --------|-------- 窗口2 --------|
0.0s                   1.0s                   2.0s
              ↑                    ↑
           0.9s: 100请求        1.0s: 100请求
           
在 0.9s ~ 1.0s 这 0.1 秒内，实际通过了 200 个请求！
```

**优点**：实现简单，内存占用少
**缺点**：临界问题严重，可能瞬间通过 2 倍流量
**适用场景**：对精度要求不高的简单限流

---

#### 2.2 滑动窗口计数器（Sliding Window Counter）

**原理**：将大窗口细分为多个小窗口，统计时考虑所有小窗口的请求总和。

```
大窗口：1秒，分成 10 个小窗口（每个 100ms）

|--w1--|--w2--|--w3--|--w4--|--w5--|--w6--|--w7--|--w8--|--w9--|--w10-|
0    100   200   300   400   500   600   700   800   900   1000ms

当前时间 950ms，统计窗口 [0, 950] 内所有请求
```

**Java 实现**：

```java
public class SlidingWindowRateLimiter {
    
    private final long windowSize;          // 大窗口大小（毫秒）
    private final int subWindowCount;       // 小窗口数量
    private final int maxRequests;          // 大窗口内最大请求数
    private final long subWindowSize;       // 小窗口大小
    private final int[] counters;           // 每个小窗口的计数器
    private long lastUpdateTime;            // 上次更新时间
    
    public SlidingWindowRateLimiter(long windowSizeMs, int subWindowCount, int maxRequests) {
        this.windowSize = windowSizeMs;
        this.subWindowCount = subWindowCount;
        this.maxRequests = maxRequests;
        this.subWindowSize = windowSizeMs / subWindowCount;
        this.counters = new int[subWindowCount];
        this.lastUpdateTime = System.currentTimeMillis();
    }
    
    public synchronized boolean tryAcquire() {
        long now = System.currentTimeMillis();
        
        // 清理过期的小窗口
        clearExpiredWindows(now);
        
        // 计算当前所有小窗口的请求总数
        int totalCount = 0;
        for (int count : counters) {
            totalCount += count;
        }
        
        // 判断是否超过阈值
        if (totalCount < maxRequests) {
            // 找到当前时间对应的小窗口，增加计数
            int currentIndex = (int) ((now / subWindowSize) % subWindowCount);
            counters[currentIndex]++;
            return true;
        }
        return false;
    }
    
    /**
     * 清理过期的小窗口
     */
    private void clearExpiredWindows(long now) {
        // 计算需要清理多少个小窗口
        long elapsed = now - lastUpdateTime;
        int windowsToClear = (int) (elapsed / subWindowSize);
        
        if (windowsToClear > 0) {
            int startIndex = (int) ((lastUpdateTime / subWindowSize) % subWindowCount);
            for (int i = 0; i < Math.min(windowsToClear, subWindowCount); i++) {
                int index = (startIndex + i + 1) % subWindowCount;
                counters[index] = 0;
            }
            lastUpdateTime = now;
        }
    }
}
```

**更优雅的实现 —— 基于时间戳队列**：

```java
public class SlidingWindowLogRateLimiter {
    
    private final long windowSize;
    private final int maxRequests;
    private final LinkedList<Long> timestamps = new LinkedList<>();
    
    public SlidingWindowLogRateLimiter(long windowSizeMs, int maxRequests) {
        this.windowSize = windowSizeMs;
        this.maxRequests = maxRequests;
    }
    
    public synchronized boolean tryAcquire() {
        long now = System.currentTimeMillis();
        long windowStart = now - windowSize;
        
        // 移除窗口外的过期时间戳
        while (!timestamps.isEmpty() && timestamps.peekFirst() <= windowStart) {
            timestamps.pollFirst();
        }
        
        // 判断窗口内请求数
        if (timestamps.size() < maxRequests) {
            timestamps.addLast(now);
            return true;
        }
        return false;
    }
}
```

**优点**：解决了临界突刺问题，精度更高
**缺点**：小窗口越多越精确，但内存占用也越大
**适用场景**：需要较高精度的限流，如 API 网关

---

#### 2.3 漏桶算法（Leaky Bucket）

**原理**：请求像水一样流入桶中，桶以固定速率漏出。桶满则溢出（拒绝请求）。

```
      请求流入（不均匀）
          │ │ │
          ▼ ▼ ▼
       ┌─────────┐
       │ ░░░░░░░ │  ← 桶（缓冲区）
       │ ░░░░░░░ │
       │ ░░░░░░░ │
       └────┬────┘
            │
            ▼
       恒定速率流出（均匀）
```

**Java 实现**：

```java
public class LeakyBucketRateLimiter {
    
    private final long capacity;        // 桶容量
    private final long leakRate;        // 漏出速率（每秒）
    private long water;                 // 当前水量
    private long lastLeakTime;          // 上次漏水时间
    
    public LeakyBucketRateLimiter(long capacity, long leakRatePerSecond) {
        this.capacity = capacity;
        this.leakRate = leakRatePerSecond;
        this.water = 0;
        this.lastLeakTime = System.currentTimeMillis();
    }
    
    public synchronized boolean tryAcquire() {
        // 先漏水
        leak();
        
        // 判断能否加水
        if (water < capacity) {
            water++;
            return true;
        }
        return false;  // 桶满，拒绝
    }
    
    /**
     * 漏水：根据时间流逝计算漏出的水量
     */
    private void leak() {
        long now = System.currentTimeMillis();
        long elapsed = now - lastLeakTime;
        
        // 计算这段时间应该漏出多少水
        long leaked = elapsed * leakRate / 1000;
        
        if (leaked > 0) {
            water = Math.max(0, water - leaked);
            lastLeakTime = now;
        }
    }
}
```

**特点分析**：

```
场景：漏出速率 10/秒，桶容量 100

时刻 T0：瞬间来了 200 个请求
├── 100 个进入桶中排队
└── 100 个直接拒绝（桶满）

T0 ~ T10：桶中请求以 10/秒 的速率被处理
└── 输出非常平滑，系统负载稳定
```

**优点**：输出速率恒定，能很好地保护下游系统
**缺点**：无法应对突发流量，即使系统有余力也不能加速处理
**适用场景**：需要严格平滑流量的场景，如消息推送、短信发送

---

#### 2.4 令牌桶算法（Token Bucket）⭐ 推荐

**原理**：系统以固定速率向桶中放入令牌，请求需要获取令牌才能通过。桶满则丢弃新令牌。

```
    固定速率放入令牌
          │
          ▼
       ┌─────────┐
       │ ● ● ● ● │  ← 令牌桶
       │ ● ● ● ● │
       │ ● ●     │
       └────┬────┘
            │
            ▼
       请求获取令牌 ← 有令牌就通过，没有就拒绝/等待
```

**Java 实现**：

```java
public class TokenBucketRateLimiter {
    
    private final long capacity;            // 桶容量
    private final long refillRate;          // 每秒填充令牌数
    private double availableTokens;         // 当前可用令牌（用 double 提高精度）
    private long lastRefillTime;            // 上次填充时间
    
    public TokenBucketRateLimiter(long capacity, long refillRatePerSecond) {
        this.capacity = capacity;
        this.refillRate = refillRatePerSecond;
        this.availableTokens = capacity;    // 初始满桶
        this.lastRefillTime = System.currentTimeMillis();
    }
    
    public synchronized boolean tryAcquire() {
        return tryAcquire(1);
    }
    
    public synchronized boolean tryAcquire(int permits) {
        // 先填充令牌
        refill();
        
        // 判断令牌是否足够
        if (availableTokens >= permits) {
            availableTokens -= permits;
            return true;
        }
        return false;
    }
    
    /**
     * 带等待的获取令牌
     */
    public synchronized boolean tryAcquire(int permits, long timeoutMs) throws InterruptedException {
        long deadline = System.currentTimeMillis() + timeoutMs;
        
        while (true) {
            refill();
            
            if (availableTokens >= permits) {
                availableTokens -= permits;
                return true;
            }
            
            // 计算需要等待多久才能有足够令牌
            double tokensNeeded = permits - availableTokens;
            long waitTime = (long) (tokensNeeded * 1000 / refillRate);
            
            if (System.currentTimeMillis() + waitTime > deadline) {
                return false;  // 超时
            }
            
            wait(waitTime);
        }
    }
    
    /**
     * 填充令牌
     */
    private void refill() {
        long now = System.currentTimeMillis();
        long elapsed = now - lastRefillTime;
        
        // 计算这段时间应该填充多少令牌
        double tokensToAdd = (double) elapsed * refillRate / 1000;
        
        if (tokensToAdd > 0) {
            availableTokens = Math.min(capacity, availableTokens + tokensToAdd);
            lastRefillTime = now;
        }
    }
}
```

**令牌桶 vs 漏桶的关键区别**：

```
令牌桶允许突发：
├── 桶中积累了 100 个令牌
├── 瞬间来了 100 个请求
└── 全部立即通过！（消耗积累的令牌）

漏桶不允许突发：
├── 桶中积累了 100 个请求
├── 无论如何都以固定速率处理
└── 100 个请求需要 10 秒才能全部处理完
```

**优点**：允许一定程度的突发流量，同时保证平均速率
**缺点**：实现相对复杂
**适用场景**：**大多数限流场景的首选**，特别是秒杀、API 网关

---

### 三、四种算法对比

| 算法 | 突发流量 | 平滑度 | 精度 | 实现复杂度 | 适用场景 |
|-----|---------|--------|------|-----------|---------|
| 固定窗口 | ❌ 临界问题 | ⚠️ 一般 | ❌ 低 | ⭐ 简单 | 简单场景 |
| 滑动窗口 | ✅ 解决临界 | ✅ 较好 | ✅ 高 | ⭐⭐ 中等 | API 网关 |
| 漏桶 | ❌ 不支持 | ✅✅ 最平滑 | ✅ 高 | ⭐⭐ 中等 | 流量整形 |
| 令牌桶 | ✅ 支持 | ✅ 较好 | ✅ 高 | ⭐⭐ 中等 | **通用首选** |

### 四、生产环境限流方案

#### 4.1 单机限流：Guava RateLimiter

```java
@Component
public class ApiRateLimiter {
    
    // 令牌桶：每秒生成 100 个令牌
    private final RateLimiter rateLimiter = RateLimiter.create(100);
    
    /**
     * 非阻塞获取
     */
    public boolean tryAcquire() {
        return rateLimiter.tryAcquire();
    }
    
    /**
     * 阻塞获取（会等待）
     */
    public double acquire() {
        return rateLimiter.acquire();  // 返回等待时间（秒）
    }
    
    /**
     * 带超时的获取
     */
    public boolean tryAcquire(long timeout, TimeUnit unit) {
        return rateLimiter.tryAcquire(timeout, unit);
    }
}

// 使用方式
@RestController
public class ApiController {
    
    @Autowired
    private ApiRateLimiter rateLimiter;
    
    @GetMapping("/api/data")
    public Result getData() {
        if (!rateLimiter.tryAcquire()) {
            return Result.fail("系统繁忙，请稍后重试");
        }
        // 业务逻辑
        return Result.success(data);
    }
}
```

**Guava RateLimiter 的预热模式**：

```java
// 预热模式：系统启动后逐渐达到最大速率
// 预热时间 3 秒，最终速率 100/秒
RateLimiter warmUpLimiter = RateLimiter.create(100, 3, TimeUnit.SECONDS);

// 启动初期速率较低，3 秒后达到 100/秒
// 适合需要预热的系统（如数据库连接池、缓存预热）
```

#### 4.2 分布式限流：Redis + Lua

单机限流在集群环境下会失效（每台机器独立计数），需要分布式限流。

**滑动窗口实现**：

```lua
-- sliding_window_rate_limit.lua
-- KEYS[1]: 限流 key
-- ARGV[1]: 窗口大小（毫秒）
-- ARGV[2]: 最大请求数
-- ARGV[3]: 当前时间戳

local key = KEYS[1]
local window = tonumber(ARGV[1])
local limit = tonumber(ARGV[2])
local now = tonumber(ARGV[3])

-- 移除窗口外的数据
local windowStart = now - window
redis.call('ZREMRANGEBYSCORE', key, 0, windowStart)

-- 获取当前窗口内的请求数
local count = redis.call('ZCARD', key)

if count < limit then
    -- 未超限，添加当前请求
    redis.call('ZADD', key, now, now .. '-' .. math.random())
    redis.call('PEXPIRE', key, window)
    return 1  -- 允许通过
else
    return 0  -- 拒绝
end
```

```java
@Component
public class RedisRateLimiter {
    
    @Autowired
    private StringRedisTemplate redisTemplate;
    
    private static final String SCRIPT = "..."; // 上面的 Lua 脚本
    
    /**
     * 分布式限流
     * @param key      限流 key（如 "rate:user:123" 或 "rate:api:/order"）
     * @param windowMs 窗口大小（毫秒）
     * @param limit    窗口内最大请求数
     */
    public boolean tryAcquire(String key, long windowMs, int limit) {
        Long result = redisTemplate.execute(
            new DefaultRedisScript<>(SCRIPT, Long.class),
            Collections.singletonList(key),
            String.valueOf(windowMs),
            String.valueOf(limit),
            String.valueOf(System.currentTimeMillis())
        );
        return result != null && result == 1;
    }
}
```

**令牌桶实现**：

```lua
-- token_bucket_rate_limit.lua
-- KEYS[1]: 令牌桶 key
-- ARGV[1]: 桶容量
-- ARGV[2]: 填充速率（每秒）
-- ARGV[3]: 当前时间戳（毫秒）
-- ARGV[4]: 请求令牌数

local key = KEYS[1]
local capacity = tonumber(ARGV[1])
local refillRate = tonumber(ARGV[2])
local now = tonumber(ARGV[3])
local requested = tonumber(ARGV[4])

-- 获取上次填充时间和当前令牌数
local data = redis.call('HMGET', key, 'tokens', 'lastRefill')
local tokens = tonumber(data[1]) or capacity
local lastRefill = tonumber(data[2]) or now

-- 计算需要填充的令牌数
local elapsed = now - lastRefill
local tokensToAdd = elapsed * refillRate / 1000
tokens = math.min(capacity, tokens + tokensToAdd)

-- 判断是否有足够令牌
if tokens >= requested then
    tokens = tokens - requested
    redis.call('HMSET', key, 'tokens', tokens, 'lastRefill', now)
    redis.call('PEXPIRE', key, 60000)  -- 60秒过期
    return 1
else
    -- 更新令牌数（即使未获取成功也要更新填充）
    redis.call('HMSET', key, 'tokens', tokens, 'lastRefill', now)
    redis.call('PEXPIRE', key, 60000)
    return 0
end
```

#### 4.3 网关级限流：Spring Cloud Gateway

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: order-service
          uri: lb://order-service
          predicates:
            - Path=/api/order/**
          filters:
            # 基于 Redis 的令牌桶限流
            - name: RequestRateLimiter
              args:
                redis-rate-limiter.replenishRate: 100    # 每秒填充 100 个令牌
                redis-rate-limiter.burstCapacity: 200    # 桶容量 200
                redis-rate-limiter.requestedTokens: 1    # 每个请求消耗 1 个令牌
                key-resolver: "#{@userKeyResolver}"      # 限流 key 解析器
```

```java
@Configuration
public class RateLimitConfig {
    
    /**
     * 基于用户 ID 限流
     */
    @Bean
    public KeyResolver userKeyResolver() {
        return exchange -> Mono.just(
            exchange.getRequest().getHeaders().getFirst("X-User-Id")
        );
    }
    
    /**
     * 基于 IP 限流
     */
    @Bean
    public KeyResolver ipKeyResolver() {
        return exchange -> Mono.just(
            exchange.getRequest().getRemoteAddress().getAddress().getHostAddress()
        );
    }
    
    /**
     * 基于接口路径限流
     */
    @Bean
    public KeyResolver pathKeyResolver() {
        return exchange -> Mono.just(
            exchange.getRequest().getPath().value()
        );
    }
}
```

#### 4.4 应用级限流：Sentinel

```java
@RestController
public class OrderController {
    
    @GetMapping("/order/{id}")
    @SentinelResource(
        value = "getOrder",
        blockHandler = "handleBlock",
        fallback = "handleFallback"
    )
    public Result getOrder(@PathVariable Long id) {
        return orderService.getOrder(id);
    }
    
    /**
     * 限流/熔断时的处理
     */
    public Result handleBlock(Long id, BlockException e) {
        return Result.fail("系统繁忙，请稍后重试");
    }
    
    /**
     * 异常降级处理
     */
    public Result handleFallback(Long id, Throwable t) {
        return Result.fail("服务暂时不可用");
    }
}

// 配置限流规则
@PostConstruct
public void initFlowRules() {
    List<FlowRule> rules = new ArrayList<>();
    
    // QPS 限流
    FlowRule qpsRule = new FlowRule();
    qpsRule.setResource("getOrder");
    qpsRule.setGrade(RuleConstant.FLOW_GRADE_QPS);
    qpsRule.setCount(100);  // QPS 阈值
    rules.add(qpsRule);
    
    // 并发线程数限流
    FlowRule threadRule = new FlowRule();
    threadRule.setResource("getOrder");
    threadRule.setGrade(RuleConstant.FLOW_GRADE_THREAD);
    threadRule.setCount(50);  // 最大并发线程数
    rules.add(threadRule);
    
    FlowRuleManager.loadRules(rules);
}
```

### 五、限流策略的多维度设计

实际项目中，通常需要**多维度组合限流**：

```java
@Component
public class MultiDimensionRateLimiter {
    
    @Autowired
    private RedisRateLimiter redisLimiter;
    
    /**
     * 多维度限流检查
     */
    public boolean checkRateLimit(HttpServletRequest request) {
        String userId = getUserId(request);
        String ip = getClientIp(request);
        String api = request.getRequestURI();
        
        // 1. 用户级限流：每个用户 100 次/分钟
        if (!redisLimiter.tryAcquire("rate:user:" + userId, 60000, 100)) {
            throw new RateLimitException("操作太频繁，请稍后再试");
        }
        
        // 2. IP 级限流：每个 IP 1000 次/分钟
        if (!redisLimiter.tryAcquire("rate:ip:" + ip, 60000, 1000)) {
            throw new RateLimitException("请求过于频繁");
        }
        
        // 3. 接口级限流：全局 10000 次/秒
        if (!redisLimiter.tryAcquire("rate:api:" + api, 1000, 10000)) {
            throw new RateLimitException("系统繁忙");
        }
        
        // 4. 全局限流：系统整体 50000 次/秒
        if (!redisLimiter.tryAcquire("rate:global", 1000, 50000)) {
            throw new RateLimitException("系统过载");
        }
        
        return true;
    }
}
```

```
限流层级：

全局限流（5万 QPS）
    │
    ├── 接口限流（/api/order: 1万 QPS）
    │       │
    │       ├── IP 限流（1000/分钟）
    │       │       │
    │       │       └── 用户限流（100/分钟）
    │       │
    │       └── ...
    │
    └── 接口限流（/api/product: 2万 QPS）
            │
            └── ...
```

### 六、总结

**面试回答模板**：

> "限流算法主要有四种：
>
> 1. **固定窗口**：实现简单，但有临界突刺问题
> 2. **滑动窗口**：解决了临界问题，精度更高
> 3. **漏桶**：输出恒定速率，适合流量整形
> 4. **令牌桶**：允许突发流量，是**大多数场景的首选**
>
> 生产环境中，单机用 Guava RateLimiter，分布式用 Redis + Lua 脚本，网关用 Gateway/Sentinel。
>
> 实际项目我们采用**多维度限流**：用户级、IP 级、接口级、全局级，层层防护，确保系统稳定。"
