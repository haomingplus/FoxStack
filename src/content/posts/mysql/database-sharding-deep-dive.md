---
title: MySQL分库分表深度解析
published: 2021-02-08 16:45:00
description: 深入解析MySQL分库分表的实现策略、分片算法、最佳实践及常见问题解决方案
tags: [MySQL, 分库分表, 架构设计, 高并发]
category: MySQL
draft: false
---

# MySQL分库分表深度解析

---

## 一、为什么要分库分表？

### 1.1 单机数据库的瓶颈

```
┌─────────────────────────────────────────────────────────────┐
│                      单机MySQL瓶颈                           │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  性能瓶颈：                                                  │
│  - QPS瓶颈：单机QPS上限约1-2万（复杂SQL更低）               │
│  - TPS瓶颈：单机TPS上限约5000-10000                         │
│  - 连接数瓶颈：默认最大连接数151，可调至10000但有风险       │
│  - IO瓶颈：单块盘IOPS约1000-3000                            │
│                                                             │
│  容量瓶颈：                                                  │
│  - 单表数据量超过1000万，性能明显下降                       │
│  - 单表超过1亿，维护变得困难                                 │
│  - 单库数据量超过500G，备份恢复耗时                         │
│  - B+树层数增加，查询效率下降                               │
│                                                             │
│  运维瓶颈：                                                  │
│  - DDL操作锁表时间长                                        │
│  - 主从延迟增加                                             │
│  - 备份恢复耗时长                                           │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 1.2 分库分表能解决什么？

```
分库分表的核心目标：

1. 提升并发能力
   ┌────────────────────────────────────────────────────────┐
   │ 单机：1万 QPS                                           │
   │                                                         │
   │ 分4库：4万 QPS ✓                                        │
   │    (db1:1万 + db2:1万 + db3:1万 + db4:1万)              │
   └────────────────────────────────────────────────────────┘

2. 提升存储容量
   ┌────────────────────────────────────────────────────────┐
   │ 单机：1TB 存储上限                                      │
   │                                                         │
   │ 分10库：10TB ✓                                          │
   └────────────────────────────────────────────────────────┘

3. 提升可用性
   ┌────────────────────────────────────────────────────────┐
   │ 单机：挂了全部挂 ✗                                      │
   │                                                         │
   │ 多库：一个库挂了，只影响部分用户 ✓                      │
   └────────────────────────────────────────────────────────┘
```

---

## 二、分库分表策略

### 2.1 分库分表的两种维度

```
┌─────────────────────────────────────────────────────────────┐
│                     分库分表策略                             │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│    垂直（纵向）                    水平（横向）              │
│         │                              │                    │
│         ▼                              ▼                    │
│  ┌──────────┐                   ┌──────────┐               │
│  │ 垂直分库 │                   │ 水平分库 │               │
│  │ 按业务拆分│                   │ 按数据分片│               │
│  └──────────┘                   └──────────┘               │
│         │                              │                    │
│         ▼                              ▼                    │
│  ┌──────────┐                   ┌──────────┐               │
│  │ 垂直分表 │                   │ 水平分表 │               │
│  │ 按字段拆分│                   │ 按数据行分片│             │
│  └──────────┘                   └──────────┘               │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 2.2 垂直分库（按业务拆分）

**核心思想**：将不同业务表拆分到不同数据库

```
单库单表架构：
┌─────────────────────────────────────────────────────────────┐
│                        database                             │
│  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐           │
│  │ user    │ │ order   │ │ product │ │ payment │           │
│  │ table   │ │ table   │ │ table   │ │ table   │           │
│  └─────────┘ └─────────┘ └─────────┘ └─────────┘           │
│                                                             │
│  问题：所有业务共享一个库，互相影响                          │
└─────────────────────────────────────────────────────────────┘
                            ▼
垂直分库架构：
┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│ user_db      │  │ order_db     │  │ product_db   │
│ ┌──────────┐ │  │ ┌──────────┐ │  │ ┌──────────┐ │
│ │ user     │ │  │ │ order    │ │  │ │ product  │ │
│ │ user_ext │ │  │ │ order_item│ │  │ │ sku      │ │
│ └──────────┘ │  │ └──────────┘ │  │ └──────────┘ │
└──────────────┘  └──────────────┘  └──────────────┘
        │                  │                  │
    用户服务           订单服务           商品服务

优势：
✓ 业务隔离，互不影响
✓ 职责清晰，便于团队协作
✓ 可针对不同业务优化
✓ 独立扩缩容

劣势：
✗ 跨库事务问题
✗ 跨库JOIN问题
✗ 需要引入分布式事务
```

**实践案例**：

```java
// 订单系统垂直分库设计

// 拆分前：单库所有表
database = order_system
tables = [orders, order_items, payments, refunds, logistics, reviews]

// 拆分后：按业务域分库
// 1. 订单主库
database = order_core
tables = [orders, order_items]

// 2. 支付库
database = payment_center
tables = [payments, payment_records, refunds]

// 3. 物流库
database = logistics_center
tables = [shipments, delivery_records, logistics_tracks]

// 4. 评价库
database = review_center
tables = [reviews, review_replies]

// 代码实现：配置多数据源
@Configuration
public class DataSourceConfig {

    @Bean
    @Primary
    public DataSource orderDataSource() {
        return createDataSource("order_core");
    }

    @Bean
    public DataSource paymentDataSource() {
        return createDataSource("payment_center");
    }

    @Bean
    public DataSource logisticsDataSource() {
        return createDataSource("logistics_center");
    }
}

// 不同Service使用不同数据源
@Service
public class OrderService {
    @Autowired
    private OrderMapper orderMapper;  // 使用orderDataSource
}

@Service
public class PaymentService {
    @Autowired
    private PaymentMapper paymentMapper;  // 使用paymentDataSource
}
```

### 2.3 水平分库（按数据分片）

**核心思想**：将同一表的数据按规则拆分到不同库

```
单库（5000万订单）：
┌─────────────────────────────────────────────────────────────┐
│                    order_db                                 │
│  ┌───────────────────────────────────────────────────────┐  │
│  │                    orders (5000万行)                   │  │
│  └───────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
                           ▼ 按user_id取模分片
水平分库（4库）：
┌──────────────────┐  ┌──────────────────┐
│   order_db_0     │  │   order_db_1     │
│ ┌──────────────┐ │  │ ┌──────────────┐ │
│ │ orders       │ │  │ │ orders       │ │
│ │ (1250万行)   │ │  │ │ (1250万行)   │ │
│ │ user_id%4=0  │ │  │ │ user_id%4=1  │ │
│ └──────────────┘ │  │ └──────────────┘ │
└──────────────────┘  └──────────────────┘
┌──────────────────┐  ┌──────────────────┐
│   order_db_2     │  │   order_db_3     │
│ ┌──────────────┐ │  │ ┌──────────────┐ │
│ │ orders       │ │  │ │ orders       │ │
│ │ (1250万行)   │ │  │ │ (1250万行)   │ │
│ │ user_id%4=2  │ │  │ │ user_id%4=3  │ │
│ └──────────────┘ │  │ └──────────────┘ │
└──────────────────┘  └──────────────────┘

优势：
✓ 数据量分散到多个库
✓ 提升并发能力
✓ 单库维护更容易
```


### 2.4 垂直分表（按字段拆分）

**核心思想**：将大表的字段拆分到多个表

```
单表（宽表）：
┌─────────────────────────────────────────────────────────────┐
│                        orders                               │
├─────────────────────────────────────────────────────────────┤
│ id, order_no, user_id, merchant_id, status, amount,        │
│ create_time, update_time,                                   │
│ receiver_name, receiver_phone, receiver_address,            │
│ remark, source, channel, coupon_id, discount_amount,       │
│ payment_method, payment_time,                              │
│ ... (共50+字段)                                             │
└─────────────────────────────────────────────────────────────┘

问题：
- 字段多，数据页利用率低
- 查询时读取不必要的字段
- 热点数据和冷数据混在一起

垂直分表后：
┌─────────────────────┐  ┌───────────────────────┐
│   orders_main       │  │   orders_ext          │
├─────────────────────┤  ├───────────────────────┤
│ id                  │  │ id                    │
│ order_no            │  │ receiver_name         │
│ user_id             │  │ receiver_phone        │
│ merchant_id         │  │ receiver_address      │
│ status              │  │ remark                │
│ amount              │  │ source                │
│ create_time         │  │ channel               │
│ update_time         │  │ coupon_id             │
└─────────────────────┘  │ discount_amount       │
                        │ payment_method        │
                        │ payment_time          │
                        └───────────────────────┘

查询主流程：只查orders_main
查询详情页：JOIN orders_ext
```

**拆分原则**：

```
1. 按使用频率拆分
   - 热点字段：orders_main (订单号、状态、金额等)
   - 冷字段：orders_ext (收货地址、备注等)

2. 按字段大小拆分
   - 小字段：主表
   - 大字段：扩展表（如文本、JSON）

3. 按业务属性拆分
   - 核心字段：主表
   - 扩展属性：扩展表
```

### 2.5 水平分表（按行拆分）

**核心思想**：同一库内将表拆分成多个物理表

```
单表（2000万用户）：
┌─────────────────────────────────────────────────────────────┐
│                      user_db                                │
│  ┌───────────────────────────────────────────────────────┐  │
│  │                    users (2000万行)                    │  │
│  └───────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘

水平分表（10表）：
┌─────────────────────────────────────────────────────────────┐
│                      user_db                                │
│  ┌─────────┐ ┌─────────┐ ┌─────────┐ ... ┌─────────┐       │
│  │ users_0 │ │ users_1 │ │ users_2 │     │ users_9 │       │
│  │ 200万行 │ │ 200万行 │ │ 200万行 │     │ 200万行 │       │
│  │ id%10=0 │ │ id%10=1 │ │ id%10=2 │     │ id%10=9 │       │
│  └─────────┘ └─────────┘ └─────────┘     └─────────┘       │
└─────────────────────────────────────────────────────────────┘

优势：
✓ 单表数据量可控
✓ 查询性能提升
✓ 索引树深度降低
```

---

## 三、分片算法详解

### 3.1 取模分片（最常用）

```java
/**
 * 取模分片算法
 * 适用场景：数据分布均匀，扩容不频繁
 */
public class ModShardingAlgorithm {

    /**
     * 计算分片
     * @param shardingValue 分片键值
     * @param shardingCount 分片数量
     * @return 分片索引 [0, shardingCount-1]
     */
    public static int calculateSharding(Long shardingValue, int shardingCount) {
        // 取模算法
        return (int) (shardingValue % shardingCount);
    }

    /**
     * 获取实际表名
     */
    public static String getActualTableName(String logicTable, Long shardingValue, int tableCount) {
        int index = calculateSharding(shardingValue, tableCount);
        return logicTable + "_" + index;
    }
}

// 使用示例
Long userId = 123456L;
String tableName = ModShardingAlgorithm.getActualTableName("orders", userId, 4);
// 结果：orders_0 (因为 123456 % 4 = 0)

// 分库分表组合：先分库再分表
public class ShardingStrategy {
    private static final int DB_COUNT = 2;
    private static final int TABLE_COUNT = 2;

    public static TableInfo getTableInfo(Long userId) {
        int dbIndex = (int) (userId % DB_COUNT);
        int tableIndex = (int) (userId / DB_COUNT % TABLE_COUNT);

        return new TableInfo(
            "order_db_" + dbIndex,
            "orders_" + tableIndex
        );
    }
}

// ShardingSphere配置示例
spring:
  shardingsphere:
    datasource:
      names: ds0,ds1
      ds0:
        type: com.zaxxer.hikari.HikariDataSource
        jdbc-url: jdbc:mysql://localhost:3306/order_db_0
      ds1:
        type: com.zaxxer.hikari.HikariDataSource
        jdbc-url: jdbc:mysql://localhost:3306/order_db_1
    rules:
      sharding:
        tables:
          orders:
            actual-data-nodes: ds$->{0..1}.orders_$->{0..1}
            table-strategy:
              standard:
                sharding-column: user_id
                sharding-algorithm-name: table-mod
            database-strategy:
              standard:
                sharding-column: user_id
                sharding-algorithm-name: db-mod
        sharding-algorithms:
          table-mod:
            type: MOD
            props:
              sharding-count: 2
          db-mod:
            type: MOD
            props:
              sharding-count: 2
```

**取模分片的扩容问题**：

```
问题：扩容需要数据迁移

2库扩容到4库：
┌──────────────────┐  ┌──────────────────┐
│   db_0           │  │   db_1           │
│  user_id % 2 = 0 │  │  user_id % 2 = 1 │
│                  │  │                  │
│  user_id: 2,4,6  │  │  user_id: 1,3,5  │
│  8,10,12...      │  │  7,9,11...       │
└──────────────────┘  └──────────────────┘
         │ 扩容                 │
         ▼                      ▼
┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐
│   db_0           │  │   db_1           │  │   db_2           │  │   db_3           │
│  user_id % 4 = 0 │  │  user_id % 4 = 1 │  │  user_id % 4 = 2 │  │  user_id % 4 = 3 │
│                  │  │                  │  │                  │  │                  │
│  user_id: 4,8,12 │  │  user_id: 1,5,9  │  │  user_id: 2,6,10 │  │  user_id: 3,7,11 │
│                  │  │  (不动)          │  │  (从db0迁移)     │  │  (从db1迁移)     │
└──────────────────┘  └──────────────────┘  └──────────────────┘  └──────────────────┘

需要迁移50%的数据！

解决方式1：倍数扩容
- 扩容时按2的幂次扩：2→4→8→16
- 这样只需要迁移一半数据到新库

解决方式2：一致性Hash
- 扩容时只迁移部分数据

解决方式3：预约扩容
- 提前规划，按2^n的倍数建表
- 例如：在建表时就建32个表，初期只使用2个
```

### 3.2 范围分片

```java
/**
 * 范围分片算法
 * 适用场景：数据有明显的范围特征，如时间、ID段
 */
public class RangeShardingAlgorithm {

    /**
     * 按ID范围分片
     */
    public static String getDbByUserId(Long userId) {
        if (userId >= 1 && userId < 1000000) {
            return "order_db_0";  // 1-100万
        } else if (userId < 2000000) {
            return "order_db_1";  // 100万-200万
        } else if (userId < 3000000) {
            return "order_db_2";  // 200万-300万
        } else {
            return "order_db_3";  // 300万以上
        }
    }

    /**
     * 按时间范围分片（适合时间序列数据）
     */
    public static String getTableByTime(LocalDateTime dateTime) {
        int year = dateTime.getYear();
        int month = dateTime.getMonthValue();

        // 按月分表
        return String.format("orders_%d%02d", year, month);
    }

    /**
     * 按日期自动创建表（适合日志类数据）
     */
    public static String getOrCreateTableByDate(LocalDate date) {
        String tableName = "logs_" + date.toString().replace("-", "");

        // 检查表是否存在
        if (!tableExists(tableName)) {
            createTable(tableName);
        }

        return tableName;
    }
}

// ShardingSphere范围分片配置
spring:
  shardingsphere:
    rules:
      sharding:
        tables:
          orders:
            actual-data-nodes: ds$->{0..3}.orders
            table-strategy:
              standard:
                sharding-column: create_time
                sharding-algorithm-name: date-range
        sharding-algorithms:
          date-range:
            type: INTERVAL
            props:
              datetime-pattern: yyyy-MM-dd HH:mm:ss
              datetime-lower: 2024-01-01 00:00:00
              datetime-upper: 2024-12-31 23:59:59
              sharding-suffix-pattern: yyyyMM
              datetime-interval-amount: 1
              datetime-interval-unit: MONTHS
```

**范围分片的优缺点**：

```
优势：
✓ 范围查询效率高，单次查询可能只涉及一个分片
✓ 容易理解，便于运维
✓ 历史数据归档方便

劣势：
✗ 数据分布不均
✗ 热点问题（新数据总是集中在最新的分片）
✗ 扩容时需要规划

适用场景：
- 订单表按时间分表（便于归档）
- 日志表按日期分表（定期清理）
- 用户表按ID段分表（预留ID段）
```

### 3.3 一致性Hash分片

```java
/**
 * 一致性Hash分片算法
 * 适用场景：需要平滑扩容
 */
public class ConsistentHashShardingAlgorithm {

    // 虚拟节点数量（解决数据分布不均问题）
    private static final int VIRTUAL_NODE_COUNT = 150;

    // Hash环
    private final TreeMap<Long, String> hashRing = new TreeMap<>();

    public ConsistentHashShardingAlgorithm(List<String> nodes) {
        for (String node : nodes) {
            addNode(node);
        }
    }

    /**
     * 添加节点
     */
    public void addNode(String node) {
        for (int i = 0; i < VIRTUAL_NODE_COUNT; i++) {
            String virtualNode = node + "#" + i;
            long hash = hash(virtualNode);
            hashRing.put(hash, node);
        }
    }

    /**
     * 移除节点
     */
    public void removeNode(String node) {
        for (int i = 0; i < VIRTUAL_NODE_COUNT; i++) {
            String virtualNode = node + "#" + i;
            long hash = hash(virtualNode);
            hashRing.remove(hash);
        }
    }

    /**
     * 获取分片节点
     */
    public String getShardingNode(String key) {
        long hash = hash(key);

        // 顺时针查找第一个大于等于hash的节点
        Map.Entry<Long, String> entry = hashRing.ceilingEntry(hash);

        // 如果没找到，取第一个节点（环状）
        if (entry == null) {
            entry = hashRing.firstEntry();
        }

        return entry.getValue();
    }

    /**
     * Hash函数（使用FNV1A或MurmurHash）
     */
    private long hash(String key) {
        // FNV1A 32bit Hash
        final int FNV_PRIME = 16777619;
        int hash = (int) 2166136261L;

        for (byte b : key.getBytes()) {
            hash ^= b & 0xFF;
            hash *= FNV_PRIME;
        }

        return hash & 0xFFFFFFFFL;
    }
}

// 使用示例
public class ConsistentHashExample {
    public static void main(String[] args) {
        List<String> databases = Arrays.asList("db0", "db1", "db2");
        ConsistentHashShardingAlgorithm algorithm =
            new ConsistentHashShardingAlgorithm(databases);

        // 查询分片
        String db = algorithm.getShardingNode("user_123456");
        System.out.println(db);  // 输出：db0 或 db1 或 db2

        // 扩容：添加db3
        algorithm.addNode("db3");
        // 只有部分数据需要迁移到db3
    }
}

// ShardingSphere配置
spring:
  shardingsphere:
    rules:
      sharding:
        sharding-algorithms:
          consistent-hash:
            type: CONSISTENT_HASH
            props:
              sharding-count: 4  # 虚拟节点数
```

**一致性Hash扩容示意图**：

```
初始状态（3库）：
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│                    Hash环                                   │
│                   ╱    ╲                                    │
│                 ╱        ╲                                  │
│               ▼            ▼                                │
│            db0            db2                               │
│                                                             │
│               ▲            ▲                                │
│                ╲          ╱                                 │
│                  ╲      ╱                                   │
│                    db1                                      │
│                                                             │
│  数据分布：                                                 │
│  - db0: 33.3%                                              │
│  - db1: 33.3%                                              │
│  - db2: 33.3%                                              │
└────────────────────────────────────────────────────────────┘

扩容后（4库）：
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│                    Hash环                                   │
│                  ╱  ║  ╲                                    │
│                ╱    ║    ╲                                  │
│              ▼      ║      ▼                                │
│           db0      ║     db2                                │
│                    ║                                        │
│              ▲      ║      ▲                                │
│                ╲    ║    ╱                                  │
│                  ╲ ║ ╱                                     │
│                  db1│db3  ← 新增                           │
│                                                             │
│  数据迁移：                                                 │
│  - db0 → db3: 约8% (需要迁移)                               │
│  - db1 → db3: 约8% (需要迁移)                               │
│  - db2 → db3: 约8% (需要迁移)                               │
│  - db0/db1/db2: 各保留约25% (不需要迁移)                    │
│                                                             │
│  迁移数据量：25% (相比取模分片的50%更少)                     │
└─────────────────────────────────────────────────────────────┘
```

### 3.4 地理位置分片

```java
/**
 * 地理位置/业务分片算法
 * 适用场景：有明显的地理或业务属性
 */
public class LocationShardingAlgorithm {

    /**
     * 按地区分片
     */
    public static String getDbByRegion(String region) {
        // 华东
        if (isEastChina(region)) {
            return "order_db_east";
        }
        // 华南
        else if (isSouthChina(region)) {
            return "order_db_south";
        }
        // 华北
        else if (isNorthChina(region)) {
            return "order_db_north";
        }
        // 其他
        else {
            return "order_db_other";
        }
    }

    /**
     * 按商家类型分片
     */
    public static String getDbByMerchantType(Integer merchantType) {
        switch (merchantType) {
            case 1:  // 餐饮
                return "order_db_food";
            case 2:  // 零售
                return "order_db_retail";
            case 3:  // 服务
                return "order_db_service";
            default:
                return "order_db_default";
        }
    }

    /**
     * 按用户等级分片（VIP用户单独库）
     */
    public static String getDbByUserLevel(Integer userLevel) {
        if (userLevel >= 5) {
            return "order_db_vip";  // VIP用户专用库，配置更好
        } else {
            return "order_db_normal";  // 普通用户库
        }
    }
}
```

---

## 四、分库分表实战方案

### 4.1 完整的分库分表方案设计

```
┌─────────────────────────────────────────────────────────────┐
│                 订单系统分库分表架构                          │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  垂直分库（业务域）：                                        │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │
│  │ 订单库       │  │ 支付库       │  │ 物流库       │      │
│  └──────────────┘  └──────────────┘  └──────────────┘      │
│                                                             │
│  水平分库（数据分片）：                                      │
│  订单库按user_id分4个库：                                    │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐       │
│  │order_db_0│ │order_db_1│ │order_db_2│ │order_db_3│       │
│  └──────────┘ └──────────┘ └──────────┘ └──────────┘       │
│                                                             │
│  水平分表（数据分片）：                                      │
│  每个库内按user_id分4个表：                                  │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐          │
│  │orders_0     │ │orders_1     │ │orders_2     │          │
│  │orders_3     │ │order_items_0│ │order_items_1│ ...      │
│  └─────────────┘ └─────────────┘ └─────────────┘          │
│                                                             │
└─────────────────────────────────────────────────────────────┘

分片规则：
┌─────────────────────────────────────────────────────────────┐
│  表名            │ 分片键    │ 分片算法          │ 分片数   │
├─────────────────────────────────────────────────────────────┤
│  orders         │ user_id   │ user_id % 4      │ 4库x4表  │
│  order_items    │ order_id  │ order_id % 4     │ 4库x4表  │
│  payments       │ order_id  │ order_id % 2     │ 2库x2表  │
│  shipments      │ order_id  │ 按月分表         │ 按月      │
└─────────────────────────────────────────────────────────────┘

分片键选择原则：
1. 查询条件中常用的字段
2. 数据分布均匀的字段
3. 不会频繁变更的字段
4. 业务维度清晰（user_id、merchant_id、order_id）
```

### 4.2 ShardingSphere实现

```yaml
# ShardingSphere配置完整示例
spring:
  shardingsphere:
    # 数据源配置
    datasource:
      names: ds0,ds1,ds2,ds3
      ds0:
        type: com.zaxxer.hikari.HikariDataSource
        driver-class-name: com.mysql.cj.jdbc.Driver
        jdbc-url: jdbc:mysql://localhost:3306/order_db_0?useSSL=false&serverTimezone=Asia/Shanghai
        username: root
        password: password
      ds1:
        type: com.zaxxer.hikari.HikariDataSource
        jdbc-url: jdbc:mysql://localhost:3306/order_db_1?useSSL=false&serverTimezone=Asia/Shanghai
        username: root
        password: password
      ds2:
        type: com.zaxxer.hikari.HikariDataSource
        jdbc-url: jdbc:mysql://localhost:3306/order_db_2?useSSL=false&serverTimezone=Asia/Shanghai
        username: root
        password: password
      ds3:
        type: com.zaxxer.hikari.HikariDataSource
        jdbc-url: jdbc:mysql://localhost:3306/order_db_3?useSSL=false&serverTimezone=Asia/Shanghai
        username: root
        password: password

    # 分片规则
    rules:
      sharding:
        # 广播表（每个库都存在相同数据）
        broadcast-tables:
          - dict_config
          - region_config

        # 表配置
        tables:
          # 订单主表
          orders:
            # 实际数据节点：ds0.orders_0, ds0.orders_1, ..., ds3.orders_3
            actual-data-nodes: ds$->{0..3}.orders_$->{0..3}
            # 分库策略
            database-strategy:
              standard:
                sharding-column: user_id
                sharding-algorithm-name: db-mod
            # 分表策略
            table-strategy:
              standard:
                sharding-column: user_id
                sharding-algorithm-name: table-mod
            # 主键生成策略
            key-generate-strategy:
              column: id
              key-generator-name: snowflake

          # 订单明细表（绑定表，与orders一起分片）
          order_items:
            actual-data-nodes: ds$->{0..3}.order_items_$->{0..3}
            table-strategy:
              standard:
                sharding-column: order_id
                sharding-algorithm-name: table-mod

          # 支付表（独立分片）
          payments:
            actual-data-nodes: ds$->{0..1}.payments_$->{0..1}
            database-strategy:
              standard:
                sharding-column: order_id
                sharding-algorithm-name: db-mod-2
            table-strategy:
              standard:
                sharding-column: order_id
                sharding-algorithm-name: table-mod-2

        # 分片算法
        sharding-algorithms:
          # 模4分片
          db-mod:
            type: MOD
            props:
              sharding-count: 4
          table-mod:
            type: MOD
            props:
              sharding-count: 4
          # 模2分片
          db-mod-2:
            type: MOD
            props:
              sharding-count: 2
          table-mod-2:
            type: MOD
            props:
              sharding-count: 2

        # 主键生成器
        key-generators:
          snowflake:
            type: SNOWFLAKE
            props:
              max-vibration-offset: 1
              worker-id: 1

        # 绑定表配置（避免笛卡尔积）
        binding-tables:
          - orders,order_items

    # 属性配置
    props:
      sql-show: true
      check-table-metadata-enabled: true
```

```java
// Java代码配置方式（更灵活）
@Configuration
public class ShardingConfig {

    @Bean
    public ShardingSphereDataSource shardingSphereDataSource() throws SQLException {
        // 1. 配置数据源
        Map<String, DataSource> dataSourceMap = new HashMap<>();
        for (int i = 0; i < 4; i++) {
            dataSourceMap.put("ds" + i, createDataSource(i));
        }

        // 2. 配置分片规则
        ShardingRuleConfiguration shardingRule = new ShardingRuleConfiguration();

        // 订单表配置
        TableRuleConfiguration orderTableRule = new TableRuleConfiguration(
            "orders",
            "ds$->{0..3}.orders_$->{0..3}"
        );

        // 分库策略
        orderTableRule.setDatabaseShardingStrategyConfig(
            new StandardShardingStrategyConfiguration(
                "user_id",
                new ModShardingAlgorithm("db-mod", 4)
            )
        );

        // 分表策略
        orderTableRule.setTableShardingStrategyConfig(
            new StandardShardingStrategyConfiguration(
                "user_id",
                new ModShardingAlgorithm("table-mod", 4)
            )
        );

        // 主键生成
        orderTableRule.setKeyGenerateStrategyConfig(
            new KeyGenerateStrategyConfiguration("id", "snowflake")
        );

        shardingRule.getTableRuleConfigs().add(orderTableRule);

        // 绑定表配置
        shardingRule.getBindingTableGroups().add("orders,order_items");

        // 3. 配置主键生成器
        shardingRule.getKeyGenerators().put("snowflake", new SnowflakeKeyGenerateAlgorithm());

        // 4. 创建ShardingSphereDataSource
        return ShardingSphereDataSource.create(dataSourceMap, Collections.singleton(shardingRule));
    }

    private DataSource createDataSource(int dbIndex) {
        HikariConfig config = new HikariConfig();
        config.setJdbcUrl("jdbc:mysql://localhost:3306/order_db_" + dbIndex);
        config.setUsername("root");
        config.setPassword("password");
        config.setDriverClassName("com.mysql.cj.jdbc.Driver");
        config.setMaximumPoolSize(20);
        config.setMinimumIdle(5);
        return new HikariDataSource(config);
    }
}

// 自定义分片算法
public class ModShardingAlgorithm implements PreciseShardingAlgorithm<Long> {

    private final String algorithmName;
    private final int shardingCount;

    public ModShardingAlgorithm(String algorithmName, int shardingCount) {
        this.algorithmName = algorithmName;
        this.shardingCount = shardingCount;
    }

    @Override
    public String doSharding(Collection<String> availableTargetNames,
                             PreciseShardingValue<Long> shardingValue) {
        Long value = shardingValue.getValue();
        if (value == null) {
            throw new IllegalArgumentException("Sharding value cannot be null");
        }

        // 计算分片索引
        int index = (int) (value % shardingCount);

        // 匹配目标名称
        for (String target : availableTargetNames) {
            if (target.endsWith(String.valueOf(index))) {
                return target;
            }
        }

        throw new IllegalArgumentException("No target found for sharding value: " + value);
    }

    @Override
    public String getType() {
        return "MOD";
    }
}
```

### 4.3 数据迁移方案

```java
/**
 * 分库分表数据迁移服务
 */
@Service
public class DataMigrationService {

    /**
     * 双写方案迁移（推荐用于生产环境）
     *
     * 迁移步骤：
     * 1. 应用层支持双写（同时写老库和新库）
     * 2. 历史数据全量迁移（停机或凌晨）
     * 3. 数据校验（比对老库和新库数据）
     * 4. 切换读（先切一部分流量验证）
     * 5. 停止双写，下线老库
     */
    public void migrateWithDualWrite() {
        // 阶段1：应用层支持双写
        enableDualWriteMode();

        // 阶段2：历史数据全量迁移
        migrateHistoryData();

        // 阶段3：增量数据同步
        syncIncrementalData();

        // 阶段4：数据校验
        validateData();

        // 阶段5：灰度切换读
        switchReadGradually();

        // 阶段6：完全切换，停止双写
        completeMigration();
    }

    /**
     * 双写模式实现
     */
    @Component
    public class DualWriteOrderService {

        @Autowired
        private OrderMapper oldOrderMapper;

        @Autowired
        private OrderMapper newOrderMapper;

        private volatile boolean dualWriteEnabled = true;
        private volatile boolean readFromNew = false;

        /**
         * 双写插入
         */
        @Transactional
        public void insertWithDualWrite(Order order) {
            // 先写新库
            try {
                newOrderMapper.insert(order);
            } catch (Exception e) {
                log.error("写入新库失败", e);
                // 新库失败不影响老库
            }

            // 再写老库（保证数据不丢）
            oldOrderMapper.insert(order);
        }

        /**
         * 双写更新
         */
        @Transactional
        public void updateWithDualWrite(Order order) {
            try {
                newOrderMapper.updateById(order);
            } catch (Exception e) {
                log.error("更新新库失败", e);
            }

            oldOrderMapper.updateById(order);
        }

        /**
         * 读操作（根据配置决定读哪个库）
         */
        public Order selectById(Long id) {
            if (readFromNew) {
                return newOrderMapper.selectById(id);
            } else {
                return oldOrderMapper.selectById(id);
            }
        }
    }

    /**
     * 历史数据迁移
     */
    public void migrateHistoryData() {
        int batchSize = 1000;
        long lastId = 0;
        int totalMigrated = 0;

        while (true) {
            // 批量查询老库数据
            List<Order> orders = oldOrderMapper.selectBatch(lastId, batchSize);

            if (orders.isEmpty()) {
                break;
            }

            // 批量写入新库
            for (List<Order> batch : Lists.partition(orders, 100)) {
                try {
                    newOrderMapper.batchInsert(batch);
                    totalMigrated += batch.size();
                    lastId = batch.get(batch.size() - 1).getId();

                    log.info("迁移进度: total={}, lastId={}", totalMigrated, lastId);

                } catch (Exception e) {
                    log.error("批量写入失败, lastId={}", lastId, e);
                    // 记录失败ID，后续重试
                    recordFailedBatch(batch);
                }
            }
        }

        log.info("历史数据迁移完成, total={}", totalMigrated);
    }

    /**
     * 数据校验
     */
    public void validateData() {
        int batchSize = 1000;
        long lastId = 0;
        int errorCount = 0;

        while (true) {
            List<Order> oldOrders = oldOrderMapper.selectBatch(lastId, batchSize);

            if (oldOrders.isEmpty()) {
                break;
            }

            // 查询新库数据
            List<Long> ids = oldOrders.stream()
                .map(Order::getId)
                .collect(Collectors.toList());

            Map<Long, Order> newOrderMap = newOrderMapper.selectByIds(ids)
                .stream()
                .collect(Collectors.toMap(Order::getId, o -> o));

            // 逐一比对
            for (Order oldOrder : oldOrders) {
                Order newOrder = newOrderMap.get(oldOrder.getId());

                if (newOrder == null) {
                    log.error("数据缺失: orderId={}", oldOrder.getId());
                    errorCount++;
                } else if (!compareOrder(oldOrder, newOrder)) {
                    log.error("数据不一致: orderId={}", oldOrder.getId());
                    errorCount++;
                }

                lastId = oldOrder.getId();
            }
        }

        if (errorCount > 0) {
            throw new RuntimeException("数据校验失败, 错误数=" + errorCount);
        }

        log.info("数据校验通过");
    }

    private boolean compareOrder(Order o1, Order o2) {
        return Objects.equals(o1.getOrderNo(), o2.getOrderNo()) &&
               Objects.equals(o1.getUserId(), o2.getUserId()) &&
               Objects.equals(o1.getStatus(), o2.getStatus()) &&
               Objects.equals(o1.getAmount(), o2.getAmount());
    }
}

/**
 * 使用Binlog同步方案（推荐用于大表）
 */
@Service
public class BinlogSyncMigration {

    /**
     * 使用Canal监听Binlog，实时同步到新库
     */
    public void migrateWithBinlog() {
        // 1. 历史数据全量迁移（使用工具如mysqldump + mydumper）
        // 2. 记录迁移时的Binlog位置
        // 3. 使用Canal从该位置开始增量同步
        // 4. 验证数据一致性
        // 5. 切换流量

        // Canal配置示例（连接Canal Server）
        CanalConnector connector = CanalConnectors.newSingleConnector(
            new InetSocketAddress("canal-server", 11111),
            "example",
            "",
            ""
        );

        try {
            connector.connect();
            connector.subscribe("order_db\\.orders");
            connector.rollback();

            while (true) {
                Message message = connector.getWithoutAck(100);
                long batchId = message.getId();
                List<CanalEntry.Entry> entries = message.getEntries();

                if (batchId != -1 && !entries.isEmpty()) {
                    for (CanalEntry.Entry entry : entries) {
                        if (entry.getEntryType() == CanalEntry.EntryType.ROWDATA) {
                            CanalEntry.RowChange rowChange =
                                CanalEntry.RowChange.parseFrom(entry.getStoreValue());

                            for (CanalEntry.RowData rowData : rowChange.getRowDatasList()) {
                                if (rowChange.getEventType() == CanalEntry.EventType.INSERT) {
                                    // 处理插入
                                    handleInsert(rowData.getAfterColumnsList());
                                } else if (rowChange.getEventType() == CanalEntry.EventType.UPDATE) {
                                    // 处理更新
                                    handleUpdate(rowData.getAfterColumnsList());
                                } else if (rowChange.getEventType() == CanalEntry.EventType.DELETE) {
                                    // 处理删除
                                    handleDelete(rowData.getBeforeColumnsList());
                                }
                            }
                        }
                    }
                }

                connector.ack(batchId);
            }
        } finally {
            connector.disconnect();
        }
    }
}
```

---

## 五、分库分表带来的问题及解决方案

### 5.1 跨库JOIN问题

```java
/**
 * 问题：订单表和用户表在不同的库，无法直接JOIN
 *
 * 解决方案：
 * 1. 应用层组装（查询两次，在代码中组装）
 * 2. 数据冗余（订单表冗余用户信息）
 * 3. 字段摘要（只冗余必要字段）
 * 4. 全局表（小表在每个库都保存）
 */

@Service
public class OrderQueryService {

    @Autowired
    private OrderMapper orderMapper;

    @Autowired
    private UserServiceClient userServiceClient;  // Feign客户端

    /**
     * 方案1：应用层组装
     */
    public OrderVO getOrderWithUser(Long orderId) {
        // 1. 查询订单（从分片库）
        Order order = orderMapper.selectById(orderId);

        if (order == null) {
            return null;
        }

        // 2. 查询用户信息（通过RPC或HTTP调用用户服务）
        User user = userServiceClient.getUserById(order.getUserId());

        // 3. 组装结果
        OrderVO vo = new OrderVO();
        BeanUtils.copyProperties(order, vo);
        vo.setUserName(user.getName());
        vo.setUserPhone(user.getPhone());

        return vo;
    }

    /**
     * 方案2：数据冗余
     *
     * 订单表冗余用户信息：
     * CREATE TABLE orders (
     *   id BIGINT PRIMARY KEY,
     *   order_no VARCHAR(32),
     *   user_id BIGINT,
     *   user_name VARCHAR(64),    -- 冗余字段
     *   user_phone VARCHAR(16),   -- 冗余字段
     *   user_avatar VARCHAR(256), -- 冗余字段
     *   ...
     *   INDEX idx_user_id (user_id)
     * );
     *
     * 注意：
     * 1. 冗余字段不能频繁变更
     * 2. 用户信息变更时需要异步更新订单表
     * 3. 只冗余查询必需的字段
     */

    /**
     * 方案3：全局表（小表在每个库都存在）
     *
     * 字典表、配置表等数据量小的表，可以在每个库都保存完整数据
     * CREATE TABLE dict_config (
     *   id INT PRIMARY KEY,
     *   dict_code VARCHAR(32),
     *   dict_value VARCHAR(128)
     * ) ENGINE=InnoDB;
     *
     * 每个库都执行相同的INSERT/UPDATE
     */
}
```

### 5.2 分布式事务问题

```java
/**
 * 问题：订单在db0，支付在db1，如何保证事务一致性？
 *
 * 解决方案：
 * 1. 本地消息表
 * 2. MQ事务消息
 * 3. Saga模式
 * 4. TCC模式
 */

@Service
public class OrderPaymentService {

    @Autowired
    private OrderMapper orderMapper;

    @Autowired
    private PaymentMapper paymentMapper;

    @Autowired
    private LocalMessageMapper localMessageMapper;

    @Autowired
    private RocketMQTemplate rocketMQTemplate;

    /**
     * 方案1：本地消息表
     *
     * 原理：将分布式事务拆分为多个本地事务 + 定时任务补偿
     */
    @Transactional(rollbackFor = Exception.class)
    public void createOrderWithPayment(Order order, Payment payment) {
        // 1. 创建订单（本地事务）
        orderMapper.insert(order);

        // 2. 记录本地消息表（本地事务）
        LocalMessage message = new LocalMessage();
        message.setTopic("payment-create");
        message.setContent(JSON.toJSONString(payment));
        message.setStatus(MessageStatus.PENDING);
        localMessageMapper.insert(message);

        // 3. 发送MQ消息（异步）
        // 注意：这里可能失败，靠定时任务兜底
        try {
            rocketMQTemplate.syncSend("payment-create", payment);
            message.setStatus(MessageStatus.SENT);
            localMessageMapper.updateById(message);
        } catch (Exception e) {
            log.error("发送MQ失败，等待定时任务重试", e);
        }
    }

    /**
     * 定时任务：扫描待发送消息
     */
    @Scheduled(fixedRate = 5000)
    public void scanPendingMessages() {
        List<LocalMessage> messages = localMessageMapper.selectPendingMessages(100);

        for (LocalMessage message : messages) {
            try {
                rocketMQTemplate.syncSend(message.getTopic(),
                    JSON.parseObject(message.getContent(), Payment.class));

                message.setStatus(MessageStatus.SENT);
                localMessageMapper.updateById(message);

            } catch (Exception e) {
                log.error("重试发送失败, messageId={}", message.getId(), e);
            }
        }
    }
}

/**
 * 方案2：RocketMQ事务消息
 */
@Service
public class TransactionalMessageService {

    @Autowired
    private RocketMQTemplate rocketMQTemplate;

    public void createOrder(Order order, Payment payment) {
        // 发送事务消息
        rocketMQTemplate.sendMessageInTransaction(
            "order-group",
            "payment-create",
            MessageBuilder.withPayload(payment)
                .setHeader("orderId", order.getId())
                .build(),
            null
        );
    }
}

// 事务监听器
@RocketMQTransactionListener(rocketMQTemplateBeanName = "rocketMQTemplate")
public class OrderTransactionListener implements RocketMQLocalTransactionListener {

    @Autowired
    private OrderMapper orderMapper;

    @Override
    @Transactional
    public RocketMQLocalTransactionState executeLocalTransaction(Message msg, Object arg) {
        try {
            // 执行本地事务：创建订单
            Order order = JSON.parseObject(new String((byte[]) msg.getPayload()), Order.class);
            orderMapper.insert(order);

            return RocketMQLocalTransactionState.COMMIT;

        } catch (Exception e) {
            log.error("本地事务执行失败", e);
            return RocketMQLocalTransactionState.ROLLBACK;
        }
    }

    @Override
    public RocketMQLocalTransactionState checkLocalTransaction(Message msg) {
        // 回查本地事务状态
        Long orderId = (Long) msg.getHeaders().get("orderId");
        Order order = orderMapper.selectById(orderId);

        if (order != null) {
            return RocketMQLocalTransactionState.COMMIT;
        } else {
            return RocketMQLocalTransactionState.UNKNOWN;
        }
    }
}
```

### 5.3 分页查询问题

```java
/**
 * 问题：如何实现分页查询？
 *
 * 场景1：已知分片键，只查询单个分片
 * 场景2：不知道分片键，需要查询所有分片
 */

@Service
public class OrderQueryService {

    /**
     * 场景1：已知user_id（分片键）
     * 只需要查询单个分片
     */
    public PageResult<Order> queryByUserId(Long userId, int pageNo, int pageSize) {
        // ShardingSphere会自动路由到正确的分片
        List<Order> orders = orderMapper.selectByUserId(userId, pageNo, pageSize);
        Long total = orderMapper.countByUserId(userId);

        return new PageResult<>(orders, total);
    }

    /**
     * 场景2：需要查询所有分片（如按商家查询）
     *
     * 方案1：查询所有分片后合并（适合数据量小）
     * 方案2：带分片键查询，让用户先选择
     * 方案3：使用ES实现全局搜索
     */

    /**
     * 方案1：聚合所有分片
     */
    public PageResult<Order> queryByMerchantId(Long merchantId, int pageNo, int pageSize) {
        // ShardingSphere会查询所有分片并聚合
        // 注意：性能较差，只适合低频查询
        List<Order> orders = orderMapper.selectByMerchantId(merchantId, pageNo, pageSize);
        Long total = orderMapper.countByMerchantId(merchantId);

        return new PageResult<>(orders, total);
    }

    /**
     * 深分页问题优化
     *
     * 问题：SELECT * FROM orders LIMIT 1000000, 10
     * ShardingSphere需要查询所有分片，每个分片都要查询100万+10条
     *
     * 解决方案：使用游标分页
     */
    public PageResult<Order> queryByCursor(Long userId, Long lastId, int pageSize) {
        // 使用id作为游标，而不是offset
        List<Order> orders = orderMapper.selectByIdGreaterThan(userId, lastId, pageSize);

        boolean hasMore = orders.size() == pageSize;
        Long newLastId = orders.isEmpty() ? lastId :
            orders.get(orders.size() - 1).getId();

        return new PageResult<>(orders, hasMore, newLastId);
    }

    /**
     * 方案3：使用Elasticsearch实现全局搜索
     *
     * 架构：
     * 1. 写数据：MySQL → Canal → ES
     * 2. 读数据：直接查询ES
     */
    @Autowired
    private ElasticsearchRestTemplate elasticsearchTemplate;

    public PageResult<Order> searchFromES(OrderSearchRequest request) {
        NativeSearchQuery query = new NativeSearchQueryBuilder()
            .withQuery(QueryBuilders.boolQuery()
                .must(QueryBuilders.termQuery("merchantId", request.getMerchantId()))
                .filter(QueryBuilders.rangeQuery("createTime")
                    .gte(request.getStartTime())
                    .lte(request.getEndTime()))
            )
            .withPageable(PageRequest.of(request.getPageNo() - 1, request.getPageSize()))
            .withSorts(Sort.by(Sort.Direction.DESC, "createTime"))
            .build();

        SearchHits<Order> hits = elasticsearchTemplate.search(query, Order.class);

        List<Order> orders = hits.stream()
            .map(SearchHit::getContent)
            .collect(Collectors.toList());

        return new PageResult<>(orders, hits.getTotalHits());
    }
}
```

### 5.4 全局唯一ID问题

```java
/**
 * 问题：分库分表后，如何保证ID全局唯一？
 *
 * 解决方案：
 * 1. UUID（太长，无序，不推荐）
 * 2. 数据库自增（设置不同步长）
 * 3. Redis生成
 * 4. 雪花算法（推荐）
 * 5. 号段模式（美团Leaf）
 */

/**
 * 方案1：数据库自增（设置步长）
 *
 * db1: auto_increment_increment=1, auto_increment_offset=1  → 1,4,7,10...
 * db2: auto_increment_increment=1, auto_increment_offset=2  → 2,5,8,11...
 * db3: auto_increment_increment=1, auto_increment_offset=3  → 3,6,9,12...
 *
 * 问题：
 * - 扩容时需要重新配置
 * - ID不连续
 */

/**
 * 方案2：雪花算法（推荐）
 */
@Component
public class SnowflakeIdGenerator {

    // 起始时间戳 (2024-01-01 00:00:00)
    private static final long EPOCH = 1704067200000L;

    // 各部分位数
    private static final long SEQUENCE_BITS = 12L;
    private static final long WORKER_ID_BITS = 10L;
    private static final long DATACENTER_ID_BITS = 0L;  // 不用数据中心

    // 最大值
    private static final long MAX_SEQUENCE = ~(-1L << SEQUENCE_BITS);
    private static final long MAX_WORKER_ID = ~(-1L << WORKER_ID_BITS);

    // 位移
    private static final long WORKER_ID_SHIFT = SEQUENCE_BITS;
    private static final long TIMESTAMP_SHIFT = SEQUENCE_BITS + WORKER_ID_BITS;

    private final long workerId;
    private long sequence = 0L;
    private long lastTimestamp = -1L;

    public SnowflakeIdGenerator(long workerId) {
        if (workerId < 0 || workerId > MAX_WORKER_ID) {
            throw new IllegalArgumentException("workerId out of range");
        }
        this.workerId = workerId;
    }

    public synchronized long nextId() {
        long timestamp = System.currentTimeMillis();

        // 时钟回拨处理
        if (timestamp < lastTimestamp) {
            long offset = lastTimestamp - timestamp;
            if (offset <= 5) {
                try {
                    Thread.sleep(offset << 1);
                    timestamp = System.currentTimeMillis();
                    if (timestamp < lastTimestamp) {
                        throw new RuntimeException("Clock moved backwards");
                    }
                } catch (InterruptedException e) {
                    throw new RuntimeException("Clock moved backwards");
                }
            } else {
                throw new RuntimeException("Clock moved backwards");
            }
        }

        // 同一毫秒内，序列号自增
        if (timestamp == lastTimestamp) {
            sequence = (sequence + 1) & MAX_SEQUENCE;
            if (sequence == 0) {
                // 序列号溢出，等待下一毫秒
                timestamp = tilNextMillis(lastTimestamp);
            }
        } else {
            sequence = 0L;
        }

        lastTimestamp = timestamp;

        // 组装ID
        return ((timestamp - EPOCH) << TIMESTAMP_SHIFT)
            | (workerId << WORKER_ID_SHIFT)
            | sequence;
    }

    private long tilNextMillis(long lastTimestamp) {
        long timestamp = System.currentTimeMillis();
        while (timestamp <= lastTimestamp) {
            timestamp = System.currentTimeMillis();
        }
        return timestamp;
    }

    // 解析ID（用于调试）
    public static IdInfo parseId(long id) {
        long timestamp = ((id >> TIMESTAMP_SHIFT) & ~(-1L << 41)) + EPOCH;
        long workerId = (id >> WORKER_ID_SHIFT) & MAX_WORKER_ID;
        long sequence = id & MAX_SEQUENCE;

        return new IdInfo(timestamp, workerId, sequence);
    }

    @Data
    @AllArgsConstructor
    public static class IdInfo {
        private long timestamp;
        private long workerId;
        private long sequence;
    }
}

/**
 * 方案3：号段模式（美团Leaf）
 *
 * 原理：
 * 1. 预先从数据库分配一段ID（如：1-10000）
 * 2. 在内存中分配，快用完时再获取下一段
 * 3. 双Buffer预加载，保证服务不中断
 */
@Service
public class SegmentIdService {

    @Autowired
    private IdSegmentMapper segmentMapper;

    private volatile AtomicLong currentId = new AtomicLong(0);
    private volatile long maxId = 0;
    private volatile boolean loading = false;
    private final Lock lock = new ReentrantLock();

    /**
     * 获取ID
     */
    public Long nextId(String bizTag) {
        // 检查是否需要加载新号段
        if (currentId.get() >= maxId) {
            loadSegment(bizTag);
        }

        return currentId.getAndIncrement();
    }

    /**
     * 加载号段
     */
    private void loadSegment(String bizTag) {
        if (loading) {
            // 正在加载，等待
            while (loading) {
                try {
                    Thread.sleep(10);
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                }
            }
            return;
        }

        lock.lock();
        try {
            // 双重检查
            if (currentId.get() < maxId) {
                return;
            }

            loading = true;

            // 从数据库获取新号段
            IdSegment segment = segmentMapper.selectAndUpdate(bizTag, 10000);

            currentId.set(segment.getMaxId() - 10000 + 1);
            maxId = segment.getMaxId();

            log.info("加载号段: bizTag={}, currentId={}, maxId={}",
                bizTag, currentId.get(), maxId);

        } finally {
            loading = false;
            lock.unlock();
        }
    }
}

// 数据库表设计
// CREATE TABLE id_segment (
//   biz_tag VARCHAR(32) PRIMARY KEY,
//   max_id BIGINT NOT NULL,
//   step INT NOT NULL,
//   version INT NOT NULL,
//   update_time DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
// );

// 获取并更新号段的SQL
// UPDATE id_segment
// SET max_id = max_id + #{step},
//     version = version + 1
// WHERE biz_tag = #{bizTag}
```

---

## 六、面试回答模板

### 6.1 面试问题：你负责的分库分表是怎么设计的？

```
回答框架：

1. 先说背景
   "我们当时的订单表单表数据量达到5000万，QPS峰值2万，
    单机数据库已经无法支撑，查询RT超过500ms，所以决定分库分表。"

2. 再说方案
   "我们采用的是垂直分库+水平分表的方案：
    - 首先垂直分库，按业务域拆分成订单库、支付库、物流库等
    - 然后水平分表，订单表按user_id取模分4个库，每个库再分4个表
    - 总共16个分片，支撑10万QPS"

3. 说分片键选择
   "分片键选择的是user_id，原因：
    - 订单查询90%都是按user_id查询
    - user_id分布均匀
    - user_id不会变更
    - 保证同一用户的订单在一个分片，便于查询"

4. 说遇到的问题
   "实施过程中主要遇到3个问题：
    1. 跨库JOIN：通过应用层组装+数据冗余解决
    2. 分布式事务：采用RocketMQ事务消息实现最终一致性
    3. 分页查询：已知分片键直接查，未知分片键用ES"

5. 说效果
   "改造后的效果：
    - QPS从2万提升到10万
    - RT从500ms降到50ms
    - 支撑了双12大促，零故障"
```

### 6.2 面试问题：为什么选择取模分片而不是其他？

```
回答：

"我们选择取模分片主要考虑以下几点：

1. 数据分布均匀
   user_id是递增的，取模后能均匀分布到各个分片
   不会有热点问题

2. 实现简单
   ShardingSphere等中间件原生支持
   不需要复杂的路由逻辑

3. 查询效率
   知道分片键的情况下，能直接定位到分片
   不需要扫描所有分片

4. 扩容考虑
   虽然取模分片扩容需要数据迁移，
   但我们采用了倍数扩容策略（2→4→8→16），
   配合ShardingSphere的数据迁移工具，
   扩容时只需要迁移50%的数据。

当然，取模分片也有缺点，
就是扩容时需要迁移数据。
所以我们规划时一次性建了足够的分片（16个），
预留了2年的增长空间。"
```

### 6.3 面试问题：分库分表后如何保证数据一致性？

```
回答：

"分库分表后的数据一致性主要涉及两个场景：

1. 分布式事务
   比如订单在db0，支付在db1。
   我们采用的是最终一致性方案：
   - 使用RocketMQ事务消息
   - 本地事务 + 消息发送
   - 消费者处理补偿
   - 定时对账兜底

2. 数据冗余一致性
   订单表冗余了用户姓名、手机号等字段。
   当用户信息变更时：
   - 发送用户变更消息
   - 订单服务消费消息
   - 批量更新订单表的冗余字段
   - 定时全量比对兜底

3. 对账机制
   每天凌晨进行全量对账：
   - 比对订单库和支付库
   - 比对订单库和物流库
   - 发现差异自动修复或报警"
```

---

## 七、总结

```
┌─────────────────────────────────────────────────────────────┐
│                    分库分表决策树                            │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  数据量是否超过千万？                                        │
│       否 → 只需优化索引、SQL                                │
│       是                                                    │
│         ↓                                                   │
│  QPS是否超过单机上限？                                       │
│       否 → 只需读写分离                                     │
│       是                                                    │
│         ↓                                                   │
│  选择分片策略：                                              │
│  - 有明显业务边界 → 垂直分库                                 │
│  - 有明显时间特征 → 按时间分表                               │
│  - 需要范围查询 → 按ID范围分片                               │
│  - 数据分布均匀 → 按取模分片（最常用）                        │
│  - 需要平滑扩容 → 一致性Hash                                 │
│                                                             │
│  选择分片键：                                                │
│  - 查询条件中常用的字段                                      │
│  - 数据分布均匀                                              │
│  - 不会频繁变更                                              │
│  - 业务维度清晰                                             │
│                                                             │
│  解决衍生问题：                                              │
│  - 跨库JOIN → 应用层组装 / 数据冗余 / ES                     │
│  - 分布式事务 → MQ / Saga / TCC                             │
│  - 全局ID → 雪花算法 / 号段模式                              │
│  - 分页查询 → 游标分页 / ES                                 │
│                                                             │
└─────────────────────────────────────────────────────────────┘

核心要点：
1. 不要过早分库分表，先优化
2. 选择合适的分片键最重要
3. 预留扩容空间
4. 数据迁移要有双写方案
5. 充分利用中间件能力
```
