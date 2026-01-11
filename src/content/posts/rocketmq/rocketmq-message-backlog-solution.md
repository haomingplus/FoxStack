---
title: RocketMQ消息积压问题排查与解决方案全攻略
published: 2021-11-18T16:45:00Z
description: 系统讲解RocketMQ消息积压的排查思路、根因分析和多种解决方案，包含监控、优化和应急处理策略
tags: [RocketMQ, 消息队列, 性能优化, 故障排查, 分布式系统]
category: 消息中间件
draft: false
---

# RocketMQ消息积压问题怎么排查和解决

消息积压是使用RocketMQ时最常见的生产问题之一，处理不当可能导致系统雪崩。本文将系统性地讲解排查思路和解决方案。

## 一、什么是消息积压

### 1. 定义

消息积压指的是：**生产者发送消息的速度 > 消费者消费消息的速度**，导致未消费消息堆积在Broker中。

```
正常情况：
生产速度 1000条/s  ≈  消费速度 1000条/s  →  积压量稳定

积压情况：
生产速度 1000条/s  >  消费速度 100条/s   →  积压量持续增长
```

### 2. 危害

- **消息延迟**：新消息等待时间过长
- **内存压力**：Broker内存占用过高，可能OOM
- **磁盘压力**：消息持久化占用大量磁盘空间
- **系统雪崩**：消费者处理不过来，进一步拖慢整个系统

## 二、如何发现消息积压

### 1. RocketMQ控制台（推荐）

访问 RocketMQ Dashboard 查看关键指标：

```
消费者组详情页面：
├── Consumer TPS：消费速度（条/秒）
├── Producer TPS：生产速度（条/秒）
├── Diff Total：积压消息总数 ⚠️ 重点关注
├── Last Consume Time：最后消费时间
└── Consume RT：消费耗时
```

**告警阈值：**
```
- Diff Total > 10000：需要关注
- Diff Total > 100000：需要立即处理
- Diff Total 持续增长：严重问题
```

### 2. 命令行查询

```bash
# 查看消费者组消费情况
sh mqadmin consumerProgress -g your_consumer_group -n 127.0.0.1:9876

# 输出示例
#Topic                  #Broker         #Queue  #Offset   #LastTimestamp      #Diff
order_topic            broker-a        0       1234567   2024-01-09 10:30:00  150000  ← 积压15万
order_topic            broker-a        1       1234568   2024-01-09 10:30:01  120000
```

### 3. 监控系统（Prometheus + Grafana）

```yaml
# 关键监控指标
- rocketmq_consumer_tps              # 消费TPS
- rocketmq_producer_tps              # 生产TPS
- rocketmq_group_diff_total          # 消息积压量
- rocketmq_consumer_latency          # 消费延迟
- rocketmq_message_size              # 消息大小
```

**告警规则示例：**
```promql
# 积压超过10万条
rocketmq_group_diff_total > 100000

# 消费TPS低于生产TPS的50%
rocketmq_consumer_tps < rocketmq_producer_tps * 0.5

# 消费延迟超过10秒
rocketmq_consumer_latency > 10000
```

## 三、排查思路（5个维度）

### 维度1：消费端问题

#### （1）消费者挂了

```bash
# 检查消费者是否在线
sh mqadmin consumerConnection -g your_consumer_group -n 127.0.0.1:9876

# 查看消费者实例列表
# 如果列表为空或数量不对，说明消费者下线了
```

**解决方案：**
- 重启消费者应用
- 检查消费者日志，排查宕机原因
- 配置健康检查和自动重启

#### （2）消费逻辑太慢

```java
// 🔍 排查：查看消费耗时
@Override
public ConsumeConcurrentlyStatus consumeMessage(
    List<MessageExt> msgs, ConsumeConcurrentlyContext context) {
    
    long startTime = System.currentTimeMillis();
    try {
        // 业务处理
        processMessage(msgs.get(0));
    } finally {
        long cost = System.currentTimeMillis() - startTime;
        if (cost > 1000) {
            log.warn("消费耗时过长: {}ms, msgId: {}", cost, msgs.get(0).getMsgId());
        }
    }
}
```

**常见耗时操作：**
- 同步调用外部接口（数据库、HTTP、RPC）
- 复杂的业务逻辑计算
- 大文件处理、图片转码
- 没有合理使用批处理

**解决方案：**
```java
// ✅ 方案1：异步化处理
@Override
public ConsumeConcurrentlyStatus consumeMessage(List<MessageExt> msgs, ...) {
    // 将耗时操作提交到线程池异步执行
    threadPoolExecutor.submit(() -> {
        processSlowOperation(msgs.get(0));
    });
    return ConsumeConcurrentlyStatus.CONSUME_SUCCESS;  // 快速返回
}

// ✅ 方案2：批量处理
consumer.setConsumeMessageBatchMaxSize(10);  // 一次拉取10条
@Override
public ConsumeConcurrentlyStatus consumeMessage(List<MessageExt> msgs, ...) {
    // 批量插入数据库，减少网络开销
    batchInsertToDatabase(msgs);
    return ConsumeConcurrentlyStatus.CONSUME_SUCCESS;
}

// ✅ 方案3：优化外部调用
// 使用缓存减少数据库查询
// 使用连接池复用HTTP连接
// 并行调用多个外部服务
```

#### （3）消费线程数不足

```java
// 查看配置
DefaultMQPushConsumer consumer = new DefaultMQPushConsumer("consumer_group");

// ⚠️ 默认消费线程数只有20
consumer.setConsumeThreadMin(20);   // 最小线程数
consumer.setConsumeThreadMax(20);   // 最大线程数

// ✅ 增加消费线程数
consumer.setConsumeThreadMin(64);   // 最小64个线程
consumer.setConsumeThreadMax(128);  // 最大128个线程
```

**如何确定合理的线程数？**
```
消费线程数 = (生产TPS / 单线程消费TPS) * 1.5

示例：
- 生产速度：1000 msg/s
- 单条消息消费耗时：50ms
- 单线程TPS：1000ms / 50ms = 20 msg/s
- 所需线程数：(1000 / 20) * 1.5 = 75个线程
```

#### （4）消费者频繁重试

```bash
# 查看重试队列积压情况
# 重试队列命名规则：%RETRY%consumer_group_name
sh mqadmin topicStatus -n 127.0.0.1:9876 -t %RETRY%your_consumer_group
```

**原因分析：**
- 消费失败后不断重试
- 业务异常导致消费失败
- 数据库连接池耗尽

**解决方案：**
```java
@Override
public ConsumeConcurrentlyStatus consumeMessage(List<MessageExt> msgs, ...) {
    try {
        processMessage(msgs.get(0));
        return ConsumeConcurrentlyStatus.CONSUME_SUCCESS;
    } catch (BusinessException e) {
        // 业务异常，不重试，直接进入死信队列
        log.error("业务处理失败，放弃重试", e);
        return ConsumeConcurrentlyStatus.CONSUME_SUCCESS;
    } catch (Exception e) {
        // 系统异常，进行重试
        log.error("系统异常，稍后重试", e);
        return ConsumeConcurrentlyStatus.RECONSUME_LATER;
    }
}

// 设置重试次数
consumer.setMaxReconsumeTimes(3);  // 最多重试3次
```

### 维度2：生产端问题

#### （1）生产速度突增

```java
// 监控生产者TPS
log.info("当前生产TPS: {}", producerTps);

// 如果突然从 100/s 涨到 10000/s，需要排查：
// - 是否有批量任务在跑？
// - 是否有重复发送？
// - 是否有死循环发送？
```

**解决方案：**
- 限流：在生产端加限流器
- 削峰填谷：将批量任务拆分成多批，错峰发送
- 检查代码逻辑，避免重复发送

#### （2）消息体过大

```java
// ⚠️ 单条消息过大（比如1MB），会降低消费速度
String largeContent = buildLargeContent();  // 1MB
Message msg = new Message("topic", largeContent.getBytes());

// ✅ 解决方案：拆分消息或存储到外部
// 方案1：拆分成多条小消息
splitAndSend(largeContent);

// 方案2：上传到OSS，只传URL
String url = uploadToOSS(largeContent);
Message msg = new Message("topic", url.getBytes());
```

### 维度3：Broker问题

#### （1）Broker性能瓶颈

```bash
# 查看Broker状态
sh mqadmin brokerStatus -n 127.0.0.1:9876 -b broker-a:10911

# 关注指标：
# - commitLogDiskRatio：CommitLog磁盘使用率
# - putMessageDistributeTime：写消息耗时分布
# - getMessageDistributeTime：读消息耗时分布
```

**常见问题：**
- 磁盘IO慢（机械硬盘 vs SSD）
- PageCache不足，频繁刷盘
- 磁盘空间不足

**解决方案：**
- 升级为SSD磁盘
- 增加Broker节点，分散压力
- 清理过期消息

#### （2）网络问题

```bash
# 检查网络延迟
ping broker_ip

# 检查网络带宽
iftop
```

### 维度4：队列分配问题

#### 队列数量不合理

```bash
# 查看Topic的队列数量
sh mqadmin topicStatus -n 127.0.0.1:9876 -t your_topic

# 假设只有4个队列，但有8个消费者
# 结果：只有4个消费者工作，另外4个空闲
```

**队列分配原则：**
```
队列数 >= 消费者数

推荐配置：
- Topic队列数：16-32个（根据消息量调整）
- 保证队列数是消费者数的倍数
```

**调整队列数：**
```bash
# 更新Topic队列数
sh mqadmin updateTopic -n 127.0.0.1:9876 \
  -t your_topic \
  -r 8 \    # 读队列数
  -w 8 \    # 写队列数
  -b broker-a:10911
```

### 维度5：配置问题

```java
// ❌ 错误配置示例
consumer.setPullBatchSize(1);              // 每次只拉1条，太少
consumer.setPullInterval(1000);            // 拉取间隔1秒，太长
consumer.setConsumeMessageBatchMaxSize(1); // 每次只消费1条

// ✅ 推荐配置
consumer.setPullBatchSize(32);             // 每次拉取32条
consumer.setPullInterval(0);               // 无间隔，立即拉取
consumer.setConsumeMessageBatchMaxSize(10);// 批量消费10条
consumer.setConsumeThreadMin(64);          // 最小64个消费线程
consumer.setConsumeThreadMax(128);         // 最大128个消费线程
```

## 四、应急处理方案

### 方案1：快速扩容消费者（推荐）

```yaml
# Kubernetes环境快速扩容
kubectl scale deployment consumer-app --replicas=10

# 扩容前：2个实例
# 扩容后：10个实例
# 消费速度提升5倍
```

**注意事项：**
- 确保队列数 >= 消费者数
- 监控系统资源（CPU、内存）

### 方案2：临时增加消费线程

```java
// 运行时动态调整（需要代码支持）
consumer.setConsumeThreadMax(256);  // 临时提升到256个线程
```

### 方案3：暂停生产者

```java
// 如果消费速度远低于生产速度，可以临时限流
RateLimiter rateLimiter = RateLimiter.create(100.0);  // 限制100条/秒

public void sendMessage(Message msg) {
    rateLimiter.acquire();  // 限流
    producer.send(msg);
}
```

### 方案4：跳过部分消息（极端情况）

```java
// ⚠️ 仅在业务允许的情况下使用
@Override
public ConsumeConcurrentlyStatus consumeMessage(List<MessageExt> msgs, ...) {
    MessageExt msg = msgs.get(0);
    
    // 跳过超过1小时的旧消息
    long messageTime = msg.getBornTimestamp();
    if (System.currentTimeMillis() - messageTime > 3600_000) {
        log.warn("跳过旧消息: {}", msg.getMsgId());
        return ConsumeConcurrentlyStatus.CONSUME_SUCCESS;
    }
    
    // 正常处理
    processMessage(msg);
    return ConsumeConcurrentlyStatus.CONSUME_SUCCESS;
}
```

### 方案5：重置消费位点（慎用）

```bash
# ⚠️ 会丢失消息，仅在极端情况下使用
# 重置到当前最大偏移量（跳过所有积压消息）
sh mqadmin resetOffsetByTime -n 127.0.0.1:9876 \
  -g your_consumer_group \
  -t your_topic \
  -s -1  # -1表示重置到最新位置
```

## 五、预防措施

### 1. 监控告警体系

```yaml
# Prometheus 告警规则
groups:
  - name: rocketmq_alerts
    rules:
      # 积压超过10万
      - alert: MessageBacklog
        expr: rocketmq_group_diff_total > 100000
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "消息积压严重"
          
      # 消费速度低于生产速度50%
      - alert: SlowConsumer
        expr: rocketmq_consumer_tps < rocketmq_producer_tps * 0.5
        for: 10m
        labels:
          severity: warning
```

### 2. 容量规划

```
评估公式：
所需消费能力 = 峰值生产TPS * 1.5

示例：
- 峰值生产TPS：1000 msg/s
- 所需消费能力：1500 msg/s
- 单消费者能力：100 msg/s
- 所需消费者数：15个
```

### 3. 消费者最佳实践

```java
@Service
public class OptimizedConsumer {
    
    // 1. 异步处理耗时操作
    @Resource
    private ThreadPoolExecutor asyncExecutor;
    
    // 2. 使用缓存减少数据库查询
    @Resource
    private Cache<String, Object> localCache;
    
    // 3. 批量处理
    private final List<MessageExt> buffer = new ArrayList<>();
    
    @Override
    public ConsumeConcurrentlyStatus consumeMessage(
        List<MessageExt> msgs, ConsumeConcurrentlyContext context) {
        
        try {
            // 批量消费
            for (MessageExt msg : msgs) {
                // 快速校验
                if (!validate(msg)) {
                    continue;
                }
                
                // 异步处理
                asyncExecutor.submit(() -> {
                    processMessageAsync(msg);
                });
            }
            
            return ConsumeConcurrentlyStatus.CONSUME_SUCCESS;
            
        } catch (Exception e) {
            log.error("消费异常", e);
            return ConsumeConcurrentlyStatus.RECONSUME_LATER;
        }
    }
    
    private void processMessageAsync(MessageExt msg) {
        // 使用缓存
        String key = msg.getKeys();
        Object cached = localCache.getIfPresent(key);
        if (cached != null) {
            // 使用缓存数据
            return;
        }
        
        // 业务处理
        // ...
    }
}
```

### 4. 压测验证

```bash
# 使用RocketMQ自带压测工具
sh mqadmin sendMessage -n 127.0.0.1:9876 \
  -t test_topic \
  -p "test message" \
  -c 10000  # 发送10000条消息

# 观察消费情况
sh mqadmin consumerProgress -g test_consumer_group -n 127.0.0.1:9876
```

## 六、排查流程总结

```
第一步：确认积压量和趋势
  ├─ 查看Dashboard：Diff Total
  ├─ 是否持续增长？
  └─ 积压了多长时间？

第二步：排查消费端
  ├─ 消费者是否在线？
  ├─ 消费TPS是多少？
  ├─ 消费耗时是多少？
  ├─ 消费线程数是否充足？
  └─ 是否有重试？

第三步：排查生产端
  ├─ 生产TPS是否异常？
  ├─ 是否有突发流量？
  └─ 消息体是否过大？

第四步：排查Broker
  ├─ Broker性能是否正常？
  ├─ 磁盘IO是否正常？
  └─ 网络是否通畅？

第五步：排查配置
  ├─ 队列数是否合理？
  ├─ 消费者配置是否合理？
  └─ 是否有性能配置问题？

第六步：应急处理
  ├─ 扩容消费者
  ├─ 增加消费线程
  ├─ 限流生产者
  └─ 极端情况考虑重置位点
```

## 七、实战案例

### 案例1：电商大促积压

**场景：** 双11期间，订单Topic积压200万条消息

**排查过程：**
1. 查看Dashboard：生产TPS 5000/s，消费TPS 500/s
2. 查看消费者：只有2个实例，消费线程20个
3. 查看消费耗时：单条消息耗时200ms（调用库存接口）

**解决方案：**
```java
// 1. 紧急扩容到10个实例
kubectl scale deployment order-consumer --replicas=10

// 2. 增加消费线程
consumer.setConsumeThreadMax(128);

// 3. 异步化库存查询
CompletableFuture.supplyAsync(() -> {
    return stockService.checkStock(orderId);
}, executor);

// 结果：消费TPS提升到5000/s，1小时内清空积压
```

### 案例2：消费死循环

**场景：** 消息不断重试，重试队列积压10万条

**排查过程：**
1. 查看重试队列：%RETRY%order_consumer_group 积压严重
2. 查看日志：发现数据库连接超时异常
3. 原因：数据库连接池耗尽，导致消费失败

**解决方案：**
```java
// 1. 增加数据库连接池
spring.datasource.hikari.maximum-pool-size=50  # 从20增加到50

// 2. 优化消费逻辑，捕获异常
try {
    processOrder(msg);
} catch (SQLTimeoutException e) {
    // 数据库超时，稍后重试
    return ConsumeConcurrentlyStatus.RECONSUME_LATER;
} catch (BusinessException e) {
    // 业务异常，不重试
    log.error("业务异常，消息进入死信队列", e);
    return ConsumeConcurrentlyStatus.CONSUME_SUCCESS;
}

// 3. 设置最大重试次数
consumer.setMaxReconsumeTimes(3);
```

## 八、总结

### 核心要点

1. **监控先行**：建立完善的监控告警体系
2. **快速定位**：从消费端、生产端、Broker三个维度排查
3. **治标治本**：应急处理 + 根因优化
4. **预防为主**：容量规划、压测、代码review

### 黄金法则

```
消费速度 >= 生产速度 * 1.5

关键公式：
所需消费能力 = 峰值TPS * 1.5
所需消费者数 = 所需消费能力 / 单消费者TPS
消费线程数 = 生产TPS / 单线程TPS * 1.5
```

### 常用命令速查

```bash
# 查看消费进度
sh mqadmin consumerProgress -g group_name -n namesrv_addr

# 查看Topic状态
sh mqadmin topicStatus -t topic_name -n namesrv_addr

# 更新队列数
sh mqadmin updateTopic -t topic_name -r 16 -w 16 -n namesrv_addr

# 重置消费位点（慎用）
sh mqadmin resetOffsetByTime -g group_name -t topic_name -s -1 -n namesrv_addr
```

掌握这些排查思路和解决方案，你就能从容应对RocketMQ消息积压问题！
