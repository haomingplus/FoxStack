---
title: Java线程池参数设置与队列配置最佳实践
published: 2022-05-14T09:30:00Z
description: 深入解析线程池7大参数配置原则，重点讲解最大线程数与队列的设置策略，包含实战案例和动态调优方案
tags: [Java, 并发编程, 线程池, 性能优化, JUC]
category: Java并发编程
draft: false
---

# 线程池参数怎么设置，队列满了怎么办

线程池是Java并发编程的核心，参数配置不当可能导致任务积压、线程资源浪费，甚至系统崩溃。本文将系统性讲解线程池参数设置和队列配置策略。

## 一、线程池的7大参数

### 1. ThreadPoolExecutor构造函数

```java
public ThreadPoolExecutor(
    int corePoolSize,              // 核心线程数
    int maximumPoolSize,           // 最大线程数
    long keepAliveTime,            // 线程空闲存活时间
    TimeUnit unit,                 // 时间单位
    BlockingQueue<Runnable> workQueue,        // 工作队列
    ThreadFactory threadFactory,   // 线程工厂
    RejectedExecutionHandler handler          // 拒绝策略
)
```

### 2. 参数详解

#### （1）corePoolSize - 核心线程数

**定义：** 线程池中常驻的线程数量，即使空闲也不会被销毁。

```java
// 示例
ThreadPoolExecutor executor = new ThreadPoolExecutor(
    10,        // 核心线程数=10
    20,        
    60L,       
    TimeUnit.SECONDS,
    new LinkedBlockingQueue<>(100)
);

// 行为：
// - 线程数 < 10：新任务来了，创建新线程
// - 线程数 = 10：新任务来了，放入队列
// - 队列满了 && 线程数 < 20：创建新线程
```

#### （2）maximumPoolSize - 最大线程数

**定义：** 线程池允许创建的最大线程数（包括核心线程）。

```java
maximumPoolSize = 20;  // 最多20个线程
```

**注意：** 
- 只有当队列满了之后，才会创建超过核心线程数的线程
- 如果使用无界队列（LinkedBlockingQueue），maximumPoolSize参数无效

#### （3）keepAliveTime - 线程空闲存活时间

**定义：** 非核心线程空闲超过这个时间，会被回收。

```java
keepAliveTime = 60L;      // 60秒
unit = TimeUnit.SECONDS;
```

#### （4）workQueue - 工作队列

**定义：** 存放待执行任务的队列。

常见队列类型：

| 队列类型 | 特点 | 适用场景 |
|---------|------|---------|
| **ArrayBlockingQueue** | 有界队列，数组实现 | 明确任务上限，防止内存溢出 |
| **LinkedBlockingQueue** | 可选有界/无界，链表实现 | 通用场景，推荐设置容量 |
| **SynchronousQueue** | 容量为0，直接交付 | 任务立即执行，不缓存 |
| **PriorityBlockingQueue** | 无界优先级队列 | 任务有优先级要求 |
| **DelayQueue** | 延迟队列 | 定时任务 |

#### （5）threadFactory - 线程工厂

**定义：** 用于创建线程，可以自定义线程名称、优先级等。

```java
ThreadFactory factory = new ThreadFactory() {
    private final AtomicInteger threadNumber = new AtomicInteger(1);
    
    @Override
    public Thread newThread(Runnable r) {
        Thread t = new Thread(r, "my-pool-" + threadNumber.getAndIncrement());
        t.setDaemon(false);  // 非守护线程
        t.setPriority(Thread.NORM_PRIORITY);
        return t;
    }
};
```

#### （6）handler - 拒绝策略

**定义：** 当队列满且线程数达到最大值时，如何处理新任务。

JDK提供的4种拒绝策略：

```java
// 1. AbortPolicy（默认）：抛出异常
new ThreadPoolExecutor.AbortPolicy();
// 行为：抛出 RejectedExecutionException

// 2. CallerRunsPolicy：调用者线程执行
new ThreadPoolExecutor.CallerRunsPolicy();
// 行为：由提交任务的线程（如主线程）执行任务

// 3. DiscardPolicy：静默丢弃
new ThreadPoolExecutor.DiscardPolicy();
// 行为：直接丢弃任务，不抛异常

// 4. DiscardOldestPolicy：丢弃最旧的任务
new ThreadPoolExecutor.DiscardOldestPolicy();
// 行为：丢弃队列头部最旧的任务，然后重新提交当前任务
```

## 二、线程池执行流程

### 1. 任务提交的完整流程

```
                     提交任务
                        ↓
        ┌──────────────────────────────┐
        │  线程数 < corePoolSize ?     │
        └──────────────────────────────┘
                 ↓ YES                ↓ NO
          创建核心线程              队列是否已满？
                                      ↓ NO
                                  加入队列等待
                                      ↓ YES
                        ┌──────────────────────────┐
                        │ 线程数 < maximumPoolSize?│
                        └──────────────────────────┘
                             ↓ YES          ↓ NO
                    创建非核心线程       执行拒绝策略
```

### 2. 代码示例

```java
public class ThreadPoolDemo {
    public static void main(String[] args) {
        ThreadPoolExecutor executor = new ThreadPoolExecutor(
            2,      // 核心线程数
            5,      // 最大线程数
            60L, TimeUnit.SECONDS,
            new LinkedBlockingQueue<>(3),  // 队列容量3
            new ThreadPoolExecutor.AbortPolicy()
        );
        
        // 提交8个任务
        for (int i = 1; i <= 8; i++) {
            final int taskId = i;
            try {
                executor.execute(() -> {
                    System.out.println("任务" + taskId + " 执行中，线程：" 
                        + Thread.currentThread().getName());
                    sleep(2000);
                });
                System.out.println("任务" + taskId + " 已提交");
            } catch (RejectedExecutionException e) {
                System.out.println("任务" + taskId + " 被拒绝");
            }
        }
    }
}

/* 执行结果分析：
任务1：线程数0 → 创建核心线程1 → 执行
任务2：线程数1 → 创建核心线程2 → 执行
任务3：线程数2 = corePoolSize → 放入队列[任务3]
任务4：队列[任务3, 任务4]
任务5：队列[任务3, 任务4, 任务5] (队列已满)
任务6：队列满 && 线程数2 < 5 → 创建非核心线程3 → 执行
任务7：创建非核心线程4 → 执行
任务8：创建非核心线程5 → 执行 (达到最大线程数)
任务9：线程数=5 && 队列满 → 触发拒绝策略 ❌
*/
```

## 三、核心问题：最大线程数满了，队列怎么设置合适？

### 场景1：IO密集型任务（推荐配置）

**特点：** 大量网络IO、文件IO、数据库操作，CPU利用率低。

```java
// 示例：HTTP接口调用、数据库查询
ThreadPoolExecutor executor = new ThreadPoolExecutor(
    20,       // 核心线程数 = CPU核心数 * 2
    100,      // 最大线程数 = CPU核心数 * 10
    60L, TimeUnit.SECONDS,
    new LinkedBlockingQueue<>(500),  // 有界队列，容量500
    new ThreadFactoryBuilder().setNameFormat("io-pool-%d").build(),
    new ThreadPoolExecutor.CallerRunsPolicy()  // 调用者执行
);
```

**推荐配置：**
```
核心线程数 = CPU核心数 * 2
最大线程数 = CPU核心数 * 10  (根据IO等待比调整)
队列容量 = 最大线程数 * 5 ~ 10
拒绝策略 = CallerRunsPolicy (降速处理)
```

**队列设置原则：**
- ✅ **使用有界队列**（ArrayBlockingQueue或LinkedBlockingQueue设置容量）
- ✅ **队列容量 = 最大线程数的5-10倍**
- ✅ **防止内存溢出**，明确系统承载上限

### 场景2：CPU密集型任务

**特点：** 大量计算，少量IO，CPU利用率高。

```java
// 示例：加密解密、图像处理、复杂计算
ThreadPoolExecutor executor = new ThreadPoolExecutor(
    Runtime.getRuntime().availableProcessors(),  // 核心线程数 = CPU核心数
    Runtime.getRuntime().availableProcessors(),  // 最大线程数 = CPU核心数
    0L, TimeUnit.MILLISECONDS,
    new LinkedBlockingQueue<>(100),
    new ThreadPoolExecutor.AbortPolicy()
);
```

**推荐配置：**
```
核心线程数 = CPU核心数
最大线程数 = CPU核心数 + 1
队列容量 = 根据任务量评估（不宜过大）
拒绝策略 = AbortPolicy (快速失败，反压)
```

### 场景3：高并发短任务（不推荐大队列）

**特点：** 任务执行时间很短（毫秒级），吞吐量要求高。

```java
// 示例：缓存操作、简单校验
ThreadPoolExecutor executor = new ThreadPoolExecutor(
    50,       // 核心线程数较大
    100,      // 最大线程数
    30L, TimeUnit.SECONDS,
    new SynchronousQueue<>(),  // 容量为0，直接交付 ⭐
    new ThreadPoolExecutor.CallerRunsPolicy()
);
```

**推荐配置：**
```
核心线程数 = 较大值（50+）
最大线程数 = 核心线程数 * 2
队列 = SynchronousQueue (零容量，不缓存)
拒绝策略 = CallerRunsPolicy
```

**原因：** 任务执行很快，没必要排队，直接创建线程执行效率更高。

### 场景4：低频大任务（大队列）

**特点：** 任务到达频率低，但每个任务耗时长。

```java
// 示例：数据批处理、报表生成
ThreadPoolExecutor executor = new ThreadPoolExecutor(
    5,        // 核心线程数较小
    10,       // 最大线程数
    300L, TimeUnit.SECONDS,
    new LinkedBlockingQueue<>(10000),  // 大队列 ⭐
    new ThreadPoolExecutor.AbortPolicy()
);
```

**推荐配置：**
```
核心线程数 = 较小值（5-10）
最大线程数 = 核心线程数 * 2
队列容量 = 较大值（1000-10000）
拒绝策略 = AbortPolicy
```

## 四、队列满了的4种应对策略

### 策略1：CallerRunsPolicy（推荐⭐⭐⭐⭐⭐）

**原理：** 由提交任务的线程（调用者）执行任务，形成负反馈。

```java
ThreadPoolExecutor executor = new ThreadPoolExecutor(
    10, 20, 60L, TimeUnit.SECONDS,
    new LinkedBlockingQueue<>(100),
    new ThreadPoolExecutor.CallerRunsPolicy()  // 调用者执行
);

// 执行流程
// 1. 线程池满负荷运行
// 2. 新任务到达，触发CallerRunsPolicy
// 3. 提交任务的线程（如主线程）执行该任务
// 4. 主线程被占用，无法继续提交新任务
// 5. 形成负反馈，降低任务提交速度
```

**优点：**
- ✅ 不丢失任务
- ✅ 自动降速，不会压垮系统
- ✅ 适合大部分场景

**缺点：**
- ❌ 调用者线程被占用
- ❌ 响应时间可能变长

**适用场景：** Web应用、微服务、通用业务系统

### 策略2：动态扩容队列

**原理：** 实时监控队列使用率，动态扩容线程或拒绝任务。

```java
public class MonitoredThreadPool extends ThreadPoolExecutor {
    
    private static final Logger log = LoggerFactory.getLogger(MonitoredThreadPool.class);
    
    public MonitoredThreadPool(int corePoolSize, int maximumPoolSize,
                               long keepAliveTime, TimeUnit unit,
                               BlockingQueue<Runnable> workQueue) {
        super(corePoolSize, maximumPoolSize, keepAliveTime, unit, 
              workQueue, new ThreadPoolExecutor.AbortPolicy());
        
        // 启动监控线程
        startMonitor();
    }
    
    private void startMonitor() {
        ScheduledExecutorService monitor = Executors.newSingleThreadScheduledExecutor();
        monitor.scheduleAtFixedRate(() -> {
            int queueSize = getQueue().size();
            int queueCapacity = ((LinkedBlockingQueue<?>) getQueue()).remainingCapacity() + queueSize;
            double usage = (double) queueSize / queueCapacity;
            
            log.info("线程池状态 - 活跃线程:{}, 队列任务:{}, 队列使用率:{:.2f}%",
                    getActiveCount(), queueSize, usage * 100);
            
            // 队列使用率超过80%，告警
            if (usage > 0.8) {
                log.warn("⚠️ 队列使用率过高: {:.2f}%", usage * 100);
                // 可以发送告警、动态调整参数等
            }
        }, 0, 10, TimeUnit.SECONDS);
    }
    
    @Override
    protected void beforeExecute(Thread t, Runnable r) {
        super.beforeExecute(t, r);
        log.debug("任务开始执行: {}", r);
    }
    
    @Override
    protected void afterExecute(Runnable r, Throwable t) {
        super.afterExecute(r, t);
        if (t != null) {
            log.error("任务执行异常", t);
        }
    }
}
```

### 策略3：拒绝任务，记录日志并告警

**原理：** 无法处理时，拒绝任务并记录，避免系统崩溃。

```java
public class LoggingRejectHandler implements RejectedExecutionHandler {
    
    private static final Logger log = LoggerFactory.getLogger(LoggingRejectHandler.class);
    
    @Override
    public void rejectedExecution(Runnable r, ThreadPoolExecutor executor) {
        // 记录拒绝日志
        log.error("❌ 任务被拒绝 - 队列大小:{}, 活跃线程:{}, 任务:{}",
                executor.getQueue().size(),
                executor.getActiveCount(),
                r.toString());
        
        // 发送告警（可接入监控系统）
        sendAlert("线程池任务被拒绝", executor);
        
        // 记录到数据库或MQ，后续补偿
        saveRejectedTask(r);
    }
    
    private void sendAlert(String message, ThreadPoolExecutor executor) {
        // 发送钉钉/邮件告警
    }
    
    private void saveRejectedTask(Runnable task) {
        // 保存到数据库或消息队列，后续补偿执行
    }
}

// 使用
ThreadPoolExecutor executor = new ThreadPoolExecutor(
    10, 20, 60L, TimeUnit.SECONDS,
    new LinkedBlockingQueue<>(100),
    new LoggingRejectHandler()
);
```

### 策略4：降级到消息队列

**原理：** 线程池处理不过来时，将任务发送到消息队列（如RocketMQ）。

```java
public class MQFallbackHandler implements RejectedExecutionHandler {
    
    private final RocketMQProducer mqProducer;
    
    @Override
    public void rejectedExecution(Runnable r, ThreadPoolExecutor executor) {
        // 将任务序列化后发送到MQ
        TaskWrapper task = (TaskWrapper) r;
        String taskJson = JSON.toJSONString(task);
        
        mqProducer.send(new Message("task_topic", taskJson.getBytes()));
        
        log.info("任务降级到MQ: {}", task.getTaskId());
    }
}
```

## 五、实战案例

### 案例1：电商订单处理系统

**需求：**
- 高峰期每秒1000个订单
- 订单处理涉及：库存扣减、优惠计算、生成订单、发送MQ（IO密集）
- 平均处理时间：200ms

**参数计算：**
```java
// 1. 核心线程数
// 假设8核CPU，IO密集型，核心线程数 = 8 * 2 = 16
int corePoolSize = 16;

// 2. 最大线程数
// IO密集型，最大线程数 = 8 * 10 = 80
int maximumPoolSize = 80;

// 3. 队列容量
// 高峰期TPS=1000，每个任务200ms，每秒最多完成 80 * (1000/200) = 400个任务
// 积压量 = (1000 - 400) * 10秒 = 6000
// 队列容量设置为 5000-10000
int queueCapacity = 5000;

// 4. 拒绝策略
// 使用CallerRunsPolicy，避免丢失订单
RejectedExecutionHandler handler = new ThreadPoolExecutor.CallerRunsPolicy();
```

**实现：**
```java
@Configuration
public class ThreadPoolConfig {
    
    @Bean("orderProcessPool")
    public ThreadPoolExecutor orderProcessPool() {
        return new ThreadPoolExecutor(
            16,                                    // 核心线程数
            80,                                    // 最大线程数
            60L, TimeUnit.SECONDS,                // 空闲线程存活时间
            new LinkedBlockingQueue<>(5000),      // 队列容量
            new ThreadFactoryBuilder()
                .setNameFormat("order-process-%d")
                .setDaemon(false)
                .build(),
            new ThreadPoolExecutor.CallerRunsPolicy()  // 调用者执行
        );
    }
}

@Service
public class OrderService {
    
    @Resource(name = "orderProcessPool")
    private ThreadPoolExecutor orderProcessPool;
    
    public void processOrder(Order order) {
        orderProcessPool.execute(() -> {
            try {
                // 扣减库存
                stockService.deduct(order.getProductId(), order.getQuantity());
                
                // 计算优惠
                BigDecimal discount = promotionService.calculate(order);
                
                // 创建订单
                orderMapper.insert(order);
                
                // 发送MQ
                mqProducer.send(order);
                
            } catch (Exception e) {
                log.error("订单处理失败: {}", order.getOrderId(), e);
                // 补偿逻辑
            }
        });
    }
}
```

### 案例2：数据导入系统

**需求：**
- 每天凌晨批量导入100万条数据
- 每条数据需要：解析、校验、入库（CPU + IO混合）
- 平均处理时间：50ms

**参数计算：**
```java
// 1. 不要求实时性，可以用较小的核心线程数
int corePoolSize = 10;

// 2. 最大线程数适当放大
int maximumPoolSize = 20;

// 3. 大队列，缓存任务
int queueCapacity = 10000;

// 4. 任务不能丢失，使用CallerRunsPolicy
RejectedExecutionHandler handler = new ThreadPoolExecutor.CallerRunsPolicy();
```

**实现：**
```java
@Service
public class DataImportService {
    
    private final ThreadPoolExecutor importPool = new ThreadPoolExecutor(
        10, 20, 300L, TimeUnit.SECONDS,
        new LinkedBlockingQueue<>(10000),
        new ThreadFactoryBuilder().setNameFormat("import-pool-%d").build(),
        new ThreadPoolExecutor.CallerRunsPolicy()
    );
    
    public void importData(List<DataRecord> records) {
        CountDownLatch latch = new CountDownLatch(records.size());
        
        for (DataRecord record : records) {
            importPool.execute(() -> {
                try {
                    // 解析
                    ParsedData data = parseData(record);
                    // 校验
                    validateData(data);
                    // 入库
                    dataMapper.insert(data);
                } catch (Exception e) {
                    log.error("数据导入失败: {}", record.getId(), e);
                } finally {
                    latch.countDown();
                }
            });
        }
        
        // 等待所有任务完成
        try {
            latch.await(1, TimeUnit.HOURS);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
        
        log.info("数据导入完成，总数: {}", records.size());
    }
}
```

## 六、动态调整线程池（高级）

### 1. 支持动态调整的参数

```java
ThreadPoolExecutor executor = new ThreadPoolExecutor(...);

// ✅ 可以动态调整的参数
executor.setCorePoolSize(20);          // 调整核心线程数
executor.setMaximumPoolSize(50);       // 调整最大线程数
executor.setKeepAliveTime(120, TimeUnit.SECONDS);  // 调整存活时间

// ❌ 不可动态调整的参数
// workQueue - 队列无法动态更换
// threadFactory - 线程工厂无法动态更换
// handler - 拒绝策略可以通过setRejectedExecutionHandler()更换
```

### 2. 基于配置中心动态调整

```java
@Component
public class DynamicThreadPoolManager {
    
    @Resource
    private ThreadPoolExecutor orderProcessPool;
    
    @Resource
    private NacosConfigService nacosConfig;
    
    @PostConstruct
    public void init() {
        // 监听配置变化
        nacosConfig.addListener("thread-pool-config", new Listener() {
            @Override
            public void receiveConfigInfo(String configInfo) {
                ThreadPoolConfig config = JSON.parseObject(configInfo, ThreadPoolConfig.class);
                adjustThreadPool(config);
            }
        });
    }
    
    private void adjustThreadPool(ThreadPoolConfig config) {
        log.info("动态调整线程池参数: {}", config);
        
        orderProcessPool.setCorePoolSize(config.getCorePoolSize());
        orderProcessPool.setMaximumPoolSize(config.getMaximumPoolSize());
        orderProcessPool.setKeepAliveTime(config.getKeepAliveTime(), TimeUnit.SECONDS);
    }
}
```

### 3. 基于监控自动调整

```java
@Component
public class AutoTuningThreadPool {
    
    @Scheduled(fixedRate = 60000)  // 每分钟检查一次
    public void autoTune() {
        ThreadPoolExecutor executor = getExecutor();
        
        int activeCount = executor.getActiveCount();
        int poolSize = executor.getPoolSize();
        int queueSize = executor.getQueue().size();
        
        double utilization = (double) activeCount / poolSize;
        
        // 利用率持续高于90%，扩容
        if (utilization > 0.9 && poolSize < executor.getMaximumPoolSize()) {
            int newCoreSize = Math.min(executor.getCorePoolSize() + 5, 
                                       executor.getMaximumPoolSize());
            executor.setCorePoolSize(newCoreSize);
            log.info("自动扩容线程池，新核心线程数: {}", newCoreSize);
        }
        
        // 利用率持续低于30%，缩容
        if (utilization < 0.3 && poolSize > 10) {
            int newCoreSize = Math.max(executor.getCorePoolSize() - 5, 10);
            executor.setCorePoolSize(newCoreSize);
            log.info("自动缩容线程池，新核心线程数: {}", newCoreSize);
        }
    }
}
```

## 七、最佳实践总结

### 1. 参数配置口诀

```
IO密集型：线程数多，队列大
CPU密集型：线程少，队列小
短任务：零队列，直接执行
长任务：大队列，削峰填谷
```

### 2. 队列选择指南

| 场景 | 队列类型 | 容量 | 理由 |
|-----|---------|-----|------|
| **IO密集型** | LinkedBlockingQueue | 500-5000 | 有界队列，防止OOM |
| **CPU密集型** | LinkedBlockingQueue | 100-500 | 小队列，避免任务堆积 |
| **短任务高并发** | SynchronousQueue | 0 | 零缓存，立即执行 |
| **低频大任务** | LinkedBlockingQueue | 5000-10000 | 大队列，削峰 |
| **优先级任务** | PriorityBlockingQueue | 无界 | 根据优先级排序 |

### 3. 拒绝策略选择

| 策略 | 适用场景 | 优点 | 缺点 |
|-----|---------|-----|------|
| **CallerRunsPolicy** | 通用业务系统 | 不丢任务，自动降速 | 调用者被阻塞 |
| **AbortPolicy** | 关键业务 | 快速失败，及时告警 | 抛异常 |
| **DiscardPolicy** | 非关键业务 | 性能好 | 丢失任务 |
| **自定义** | 需要补偿 | 灵活 | 需要开发 |

### 4. 监控指标

```java
// 关键监控指标
- 活跃线程数：executor.getActiveCount()
- 线程池大小：executor.getPoolSize()
- 队列大小：executor.getQueue().size()
- 已完成任务数：executor.getCompletedTaskCount()
- 总任务数：executor.getTaskCount()
- 拒绝次数：自定义拒绝策略统计

// 告警阈值
- 队列使用率 > 80%
- 线程池使用率 > 90%
- 拒绝次数 > 0
```

### 5. 避坑指南

```java
// ❌ 错误1：使用无界队列 + 大最大线程数
new ThreadPoolExecutor(10, 1000, 60L, TimeUnit.SECONDS,
    new LinkedBlockingQueue<>());  // 无界队列，maximumPoolSize=1000永远用不上

// ❌ 错误2：核心线程数 = 最大线程数 + 有界队列
new ThreadPoolExecutor(10, 10, 60L, TimeUnit.SECONDS,
    new LinkedBlockingQueue<>(100));  // 队列满了也不会扩容，直接拒绝

// ❌ 错误3：使用Executors工具类
Executors.newFixedThreadPool(10);      // 无界队列，OOM风险
Executors.newCachedThreadPool();       // 无限创建线程，OOM风险

// ✅ 正确：自定义线程池，明确参数
new ThreadPoolExecutor(10, 20, 60L, TimeUnit.SECONDS,
    new LinkedBlockingQueue<>(500),
    new ThreadPoolExecutor.CallerRunsPolicy());
```

## 八、快速决策表

**根据场景快速选择参数：**

| 任务类型 | 核心线程数 | 最大线程数 | 队列容量 | 拒绝策略 |
|---------|-----------|-----------|---------|---------|
| Web请求处理 | CPU*2 | CPU*10 | 500 | CallerRuns |
| 数据库批处理 | 10 | 20 | 5000 | CallerRuns |
| 消息消费 | CPU*2 | CPU*5 | 1000 | CallerRuns |
| 定时任务 | 5 | 10 | 100 | Abort |
| 异步通知 | 20 | 50 | 500 | Discard |
| 文件上传 | CPU | CPU*2 | 200 | Abort |

记住核心原则：**宁可慢一点，也不能丢任务和压垮系统**！

---

理解了这些原理和实践，你就能根据实际场景灵活配置线程池，避免生产事故！
