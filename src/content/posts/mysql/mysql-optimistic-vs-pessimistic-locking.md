---
title: MySQL乐观锁vs悲观锁深度解析
published: 2020-06-18 09:30:00
description: 深入理解MySQL乐观锁与悲观锁的实现原理、使用场景及最佳实践
tags: [MySQL, 锁机制, 并发控制, 事务]
category: MySQL
draft: false
---

# MySQL乐观锁 vs 悲观锁深度解析

## 一、锁的基本概念

### 1.1 什么是锁？

锁是数据库系统为了保证数据一致性，在并发事务访问同一资源时，通过某种机制让访问有序进行。

```
┌─────────────────────────────────────────────────────────────┐
│                    并发访问场景                              │
│                                                             │
│   事务A ──┐                                                 │
│            ├──→ 同一行数据 ──→ 数据混乱？                   │
│   事务B ──┘                                                 │
│                                                             │
│   解决方案: 加锁 → 串行化访问 → 保证一致性                  │
└─────────────────────────────────────────────────────────────┘
```

### 1.2 乐观锁 vs 悲观锁对比

| 维度 | 悲观锁 | 乐观锁 |
|------|--------|--------|
| **核心思想** | 假定冲突必定发生 | 假定冲突很少发生 |
| **实现方式** | 数据库层面的锁机制 | 版本号/CAS机制 |
| **锁粒度** | 行锁、表锁、间隙锁 | 无锁/逻辑锁 |
| **适用场景** | 写操作多、冲突多 | 读操作多、冲突少 |
| **性能开销** | 加锁、解锁、死锁检测 | 冲突重试 |
| **死锁风险** | 存在 | 不存在 |

---

## 二、悲观锁深度解析

### 2.1 什么是悲观锁？

悲观锁假定并发冲突概率很高，因此在操作数据前直接加锁，其他事务必须等待。

```
悲观锁思维模型：
┌────────────────────────────────────────────────────────────┐
│                                                             │
│   事务A: 取数据 → 加锁 ──┐                                  │
│                          ├── 锁持有中，事务B等待           │
│   事务B: 取数据 → 等待 ──┘                                  │
│                                                             │
│   事务A: 修改数据 → 释放锁                                  │
│   事务B: 获取锁 → 继续执行                                  │
│                                                             │
└────────────────────────────────────────────────────────────┘
```

### 2.2 MySQL悲观锁类型

```
                    MySQL 锁体系
                          │
         ┌────────────────┼────────────────┐
         │                │                │
    全局锁             表级锁            行级锁
         │                │                │
         │         ┌──────┴──────┐         │
         │         │             │    ┌────┴────┐
         │    表锁(MDL)     自增锁   记录锁  间隙锁 临键锁
         │         │             │    │         │
         │    元数据锁    AUTO_INC
         │
    FLUSH TABLES
    WITH READ LOCK
```

### 2.3 行级锁详解

#### 2.3.1 共享锁(S锁) 与 排他锁(X锁)

```sql
-- 共享锁（Shared Lock，S锁）
-- 允许其他事务也加共享锁，但禁止加排他锁
SELECT * FROM user WHERE id = 1 LOCK IN SHARE MODE;
-- MySQL 8.0 新语法
SELECT * FROM user WHERE id = 1 FOR SHARE;

-- 排他锁（Exclusive Lock，X锁）
-- 禁止其他事务加任何锁
SELECT * FROM user WHERE id = 1 FOR UPDATE;

-- 锁兼容矩阵
┌─────────┬─────────┬─────────┐
│         │   S锁   │   X锁   │
├─────────┼─────────┼─────────┤
│  S锁    │   兼容   │  不兼容  │
│  X锁    │  不兼容  │  不兼容  │
└─────────┴─────────┴─────────┘
```

#### 2.3.2 意向锁

意向锁是表级锁，用于快速判断是否能加行级锁。

```sql
-- 意向共享锁（IS）：事务打算在某些行上加共享锁
SELECT * FROM user WHERE id = 1 LOCK IN SHARE MODE;
-- 自动加：表级IS锁 + 行级S锁

-- 意向排他锁（IX）：事务打算在某些行上加排他锁
SELECT * FROM user WHERE id = 1 FOR UPDATE;
-- 自动加：表级IX锁 + 行级X锁

-- 意向锁兼容矩阵
┌─────────┬────┬────┬────┬────┐
│         │ IS │ IX │  S │  X │
├─────────┼────┼────┼────┼────┤
│   IS    │ ✓  │ ✓  │ ✓  │ ✗  │
│   IX    │ ✓  │ ✓  │ ✗  │ ✗  │
│    S    │ ✓  │ ✗  │ ✓  │ ✗  │
│    X    │ ✗  │ ✗  │ ✗  │ ✗  │
└─────────┴────┴────┴────┴────┘
```

#### 2.3.3 记录锁(Record Lock)

锁定索引记录，而非数据行本身。

```sql
-- 锁定id=1的记录（假设id是主键/唯一索引）
SELECT * FROM user WHERE id = 1 FOR UPDATE;
-- 加锁：Record Lock on id=1

-- 如果不是索引列，会退化为表锁或扫描所有行加锁
SELECT * FROM user WHERE name = 'Tom' FOR UPDATE;  -- name无索引
-- 结果：全表扫描，所有记录加X锁
```

#### 2.3.4 间隙锁(Gap Lock)

锁定索引记录之间的间隙，防止幻读。

```
索引数据: 1, 5, 10, 15

间隙分布:
  (-∞, 1)  (1, 5)  (5, 10)  (10, 15)  (15, +∞)
     │       │       │        │        │
     └───────┴───────┴────────┴────────┘
              间隙锁可以锁定这些范围

-- 示例
SELECT * FROM user WHERE id > 5 AND id < 10 FOR UPDATE;
-- 在RR隔离级别下，加锁：
--   1. Record Lock on id=5
--   2. Gap Lock (5, 10)
--   3. Record Lock on id=10
```

#### 2.3.5 临键锁(Next-Key Lock)

记录锁 + 间隙锁的组合，默认锁定记录及其前面的间隙。

```
索引数据: 1, 5, 10, 15

-- 范围查询
SELECT * FROM user WHERE id >= 5 AND id < 15 FOR UPDATE;

-- 加锁分析（RR级别）：
┌────────────────────────────────────────────────────────┐
│  (1, 5]  Next-Key Lock    ← 锁定间隙(1,5) + 记录5       │
│  (5, 10] Next-Key Lock    ← 锁定间隙(5,10) + 记录10     │
│  (10, 15) Gap Lock        ← 只锁定间隙，不包括15        │
└────────────────────────────────────────────────────────┘

效果：
- 不能插入 id=2,3,4           ← 被(1,5]锁定
- 不能插入 id=6,7,8,9         ← 被(5,10]锁定
- 不能插入 id=11,12,13,14     ← 被(10,15)锁定
- 但可以插入/修改 id=15       ← 15本身未锁定
```

### 2.4 悲观锁实战示例

```sql
-- ========== 场景：库存扣减 ==========

-- 表结构
CREATE TABLE inventory (
    id BIGINT PRIMARY KEY,
    product_id BIGINT NOT NULL,
    quantity INT NOT NULL,
    version INT DEFAULT 0,
    UNIQUE KEY uk_product (product_id)
) ENGINE=InnoDB;

-- ========== 悲观锁实现 ==========

-- 事务A：扣减库存
BEGIN;

-- 1. 先查询并加排他锁（必须使用索引）
SELECT quantity FROM inventory
WHERE product_id = 1001 FOR UPDATE;

-- 2. 检查库存
-- 假设查到 quantity = 50

-- 3. 扣减库存
UPDATE inventory
SET quantity = quantity - 10
WHERE product_id = 1001 AND quantity >= 10;

-- 4. 提交事务
COMMIT;

-- ========== 事务B：同时扣减库存 ==========
-- 此时会被阻塞，等待事务A释放锁

BEGIN;

SELECT quantity FROM inventory
WHERE product_id = 1001 FOR UPDATE;
-- 等待中... (Lock wait timeout exceeded)

-- 当事务A提交后，事务B获取到锁
-- 继续执行，基于事务A修改后的值
```

### 2.5 悲观锁的死锁问题

```sql
-- ========== 死锁场景 ==========

-- 事务A
BEGIN;
UPDATE inventory SET quantity = quantity - 10 WHERE product_id = 1001;
-- 持有 product_id=1001 的X锁
-- 等待 product_id=1002 的X锁
UPDATE inventory SET quantity = quantity - 5 WHERE product_id = 1002;

-- 事务B
BEGIN;
UPDATE inventory SET quantity = quantity - 5 WHERE product_id = 1002;
-- 持有 product_id=1002 的X锁
-- 等待 product_id=1001 的X锁
UPDATE inventory SET quantity = quantity - 10 WHERE product_id = 1001;

-- 结果：互相等待，MySQL检测到死锁，回滚其中一个事务
-- ERROR 1213 (40001): Deadlock found when trying to get lock

-- ========== 死锁预防 ==========
-- 1. 固定加锁顺序
-- 2. 使用较低隔离级别（READ COMMITTED）
-- 3. 减小事务范围
-- 4. 设置锁超时时间
SET innodb_lock_wait_timeout = 10;
```

---

## 三、乐观锁深度解析

### 3.1 什么是乐观锁？

乐观锁假定并发冲突概率很低，操作时先不加锁，在更新时检查数据是否被修改过。

```
乐观锁思维模型：
┌────────────────────────────────────────────────────────────┐
│                                                             │
│   事务A: 读数据(v1) ──┐                                    │
│                       ├── 各自处理                          │
│   事务B: 读数据(v1) ──┘                                    │
│                                                             │
│   事务A: 更新时检查(v1==v_current?) → 成功 → v2            │
│   事务B: 更新时检查(v1==v_current?) → 失败 → v1!=v2       │
│                                                             │
│   事务B: 重试 → 读数据(v2) → 更新检查(v2==v_current?)      │
│                                                             │
└────────────────────────────────────────────────────────────┘
```

### 3.2 乐观锁实现方式

#### 3.2.1 版本号机制（推荐）

```sql
-- ========== 表结构设计 ==========
CREATE TABLE goods (
    id BIGINT PRIMARY KEY,
    name VARCHAR(100),
    stock INT NOT NULL,
    version INT NOT NULL DEFAULT 0 COMMENT '乐观锁版本号',
    create_time DATETIME DEFAULT CURRENT_TIMESTAMP,
    update_time DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
) ENGINE=InnoDB;

-- ========== 乐观锁更新 ==========

-- 1. 先查询当前数据
SELECT id, stock, version FROM goods WHERE id = 1;
-- 结果: id=1, stock=100, version=5

-- 2. 应用层处理
-- new_stock = stock - 10 = 90
-- new_version = version + 1 = 6

-- 3. 使用版本号作为条件更新
UPDATE goods
SET stock = stock - 10,
    version = version + 1
WHERE id = 1
  AND version = 5;  -- 关键：检查版本号是否变化

-- 受影响行数 = 1 → 成功
-- 受影响行数 = 0 → 失败，数据已被其他事务修改
```

#### 3.2.2 时间戳机制

```sql
-- ========== 表结构设计 ==========
CREATE TABLE orders (
    id BIGINT PRIMARY KEY,
    amount DECIMAL(10, 2),
    status TINYINT NOT NULL DEFAULT 0,
    update_time TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
) ENGINE=InnoDB;

-- ========== 乐观锁更新 ==========

-- 1. 查询
SELECT id, amount, status, update_time FROM orders WHERE id = 1001;
-- 结果: update_time = '2024-01-01 10:00:00'

-- 2. 更新（使用时间戳）
UPDATE orders
SET status = 1,
    amount = 100
WHERE id = 1001
  AND update_time = '2024-01-01 10:00:00';

-- 注意：时间戳精度可能导致问题，建议使用版本号
```

#### 3.2.3 CAS（Compare And Swap）条件更新

```sql
-- ========== CAS方式：基于当前值更新 ==========

-- 场景：扣减库存，保证库存不变成负数

-- 方式1：直接在SQL中判断
UPDATE inventory
SET stock = stock - 10
WHERE product_id = 1001
  AND stock >= 10;  -- CAS条件

-- 受影响行数 > 0 → 成功
-- 受影响行数 = 0 → 库存不足或已被修改

-- 方式2：获取并设置
UPDATE inventory
SET stock = (@new_stock := stock - 10)
WHERE product_id = 1001
  AND stock >= 10;

SELECT @new_stock;  -- 获取更新后的值

-- 方式3：使用变量获取更新前后的值
UPDATE inventory
SET stock = stock - 10
WHERE product_id = 1001 AND stock >= 10
  AND (@old_stock := stock) IS NOT NULL;

SELECT ROW_COUNT(), @old_stock;
-- ROW_COUNT() = 1 → 成功，@old_stock是更新前的值
-- ROW_COUNT() = 0 → 失败
```

### 3.3 乐观锁实战示例

```sql
-- ========== 完整的乐观锁扣减库存流程 ==========

-- 事务A
-- Step 1: 查询
SELECT id, product_id, stock, version
FROM inventory
WHERE product_id = 1001;
-- 结果: stock=100, version=5

-- Step 2: 尝试更新
UPDATE inventory
SET stock = stock - 10,
    version = version + 1,
    update_time = NOW()
WHERE product_id = 1001
  AND version = 5;
-- 受影响行数: 1 → 成功


-- ========== 事务B并发执行 ==========
-- Step 1: 查询（在事务A提交前查询）
SELECT id, product_id, stock, version
FROM inventory
WHERE product_id = 1001;
-- 结果: stock=100, version=5

-- Step 2: 尝试更新（此时事务A已提交）
UPDATE inventory
SET stock = stock - 5,
    version = version + 1,
    update_time = NOW()
WHERE product_id = 1001
  AND version = 5;
-- 受影响行数: 0 → 失败！版本号已变为6

-- Step 3: 重试
-- 重新查询
SELECT id, product_id, stock, version
FROM inventory
WHERE product_id = 1001;
-- 结果: stock=90, version=6

-- 重新更新
UPDATE inventory
SET stock = stock - 5,
    version = version + 1,
    update_time = NOW()
WHERE product_id = 1001
  AND version = 6;
-- 受影响行数: 1 → 成功
```

### 3.4 应用层代码示例

```java
// ========== Java Spring Boot + MyBatis 实现 ==========

@Service
public class InventoryService {

    @Autowired
    private InventoryMapper inventoryMapper;

    /**
     * 乐观锁扣减库存（带重试）
     */
    @Transactional
    public boolean deductStock(Long productId, int quantity) {
        int maxRetries = 3;
        for (int i = 0; i < maxRetries; i++) {
            // 1. 查询当前库存
            Inventory inventory = inventoryMapper.selectByProductId(productId);
            if (inventory == null) {
                throw new BusinessException("商品不存在");
            }

            // 2. 检查库存
            if (inventory.getStock() < quantity) {
                throw new BusinessException("库存不足");
            }

            // 3. 尝试更新（使用版本号）
            int rows = inventoryMapper.updateWithVersion(
                inventory.getId(),
                quantity,
                inventory.getVersion()
            );

            // 4. 判断是否成功
            if (rows > 0) {
                return true;  // 更新成功
            }

            // 5. 版本号冲突，重试
            if (i < maxRetries - 1) {
                // 随机等待，避免活锁
                Thread.sleep(ThreadLocalRandom.current().nextInt(10, 100));
            }
        }

        throw new BusinessException("系统繁忙，请稍后重试");
    }
}

// ========== MyBatis Mapper ==========
@Mapper
public interface InventoryMapper {

    @Select("SELECT id, product_id, stock, version " +
            "FROM inventory WHERE product_id = #{productId}")
    Inventory selectByProductId(@Param("productId") Long productId);

    @Update("UPDATE inventory " +
            "SET stock = stock - #{quantity}, " +
            "    version = version + 1, " +
            "    update_time = NOW() " +
            "WHERE id = #{id} " +
            "  AND version = #{version} " +
            "  AND stock >= #{quantity}")
    int updateWithVersion(@Param("id") Long id,
                         @Param("quantity") int quantity,
                         @Param("version") Integer version);
}
```

```python
# ========== Python + SQLAlchemy 实现 ==========

from sqlalchemy import Column, BigInteger, Integer, String, DateTime, create_engine
from sqlalchemy.ext.declarative import declarative_base
from sqlalchemy.orm import sessionmaker
from sqlalchemy import func
from datetime import datetime
import time
import random

Base = declarative_base()

class Inventory(Base):
    __tablename__ = 'inventory'

    id = Column(BigInteger, primary_key=True)
    product_id = Column(BigInteger, nullable=False, unique=True)
    stock = Column(Integer, nullable=False)
    version = Column(Integer, nullable=False, default=0)
    update_time = Column(DateTime, default=datetime.now, onupdate=datetime.now)

def deduct_stock(session, product_id: int, quantity: int, max_retries: int = 3) -> bool:
    """乐观锁扣减库存"""
    for attempt in range(max_retries):
        try:
            # 1. 查询当前数据
            inventory = session.query(Inventory).filter(
                Inventory.product_id == product_id
            ).with_for_update().first()

            if not inventory:
                raise ValueError("商品不存在")

            if inventory.stock < quantity:
                raise ValueError("库存不足")

            # 2. 使用版本号更新
            old_version = inventory.version
            rows = session.query(Inventory).filter(
                Inventory.id == inventory.id,
                Inventory.version == old_version,
                Inventory.stock >= quantity
            ).update({
                Inventory.stock: Inventory.stock - quantity,
                Inventory.version: Inventory.version + 1,
                Inventory.update_time: func.now()
            }, synchronize_session=False)

            session.commit()

            if rows > 0:
                return True
            else:
                session.rollback()
                # 冲突，等待后重试
                if attempt < max_retries - 1:
                    time.sleep(random.uniform(0.01, 0.1))

        except Exception as e:
            session.rollback()
            raise

    raise ValueError("系统繁忙，请稍后重试")
```

### 3.5 乐观锁的ABA问题

```
什么是ABA问题？
时间线:
  T1: 事务A读取 version=1
  T2: 事务B读取 version=1
  T2: 事务B更新 version=2
  T3: 事务C读取 version=2
  T3: 事务C更新 version=1
  T4: 事务A检查 version=1 → 以为没变 → 更新成功

虽然版本号相同，但数据实际被修改过！

解决方案：
1. 版本号只增不减
   - 使用递增的version，而非toggle方式

2. 加入时间戳
   - 同时检查版本号和时间戳

3. 使用多个版本字段
   - 例如 version + update_time 组合判断
```

---

## 四、乐观锁 vs 悲观锁详细对比

### 4.1 性能对比

```
┌─────────────────────────────────────────────────────────────┐
│                      低并发场景（冲突少）                     │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  乐观锁:  无锁开销 → 快速完成 ✓                              │
│  悲观锁:  加锁等待 → 额外开销 ✗                              │
│                                                             │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│                      高并发场景（冲突多）                     │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  乐观锁:  频繁失败重试 → CPU浪费 ✗                           │
│  悲观锁:  队列化执行 → 有序完成 ✓                             │
│                                                             │
└─────────────────────────────────────────────────────────────┘

临界点经验值：
- 冲突率 < 10%：乐观锁更优
- 冲突率 > 20%：悲观锁更优
- 10% ~ 20%：根据实际业务测试决定
```

### 4.2 使用场景对比

| 场景 | 推荐方案 | 原因 |
|------|----------|------|
| 读取频率远高于写入 | 乐观锁 | 减少锁开销 |
| 写入频繁、冲突率高 | 悲观锁 | 避免频繁重试 |
| 响应时间要求极高 | 乐观锁 | 无等待 |
| 数据一致性要求极高 | 悲观锁 | 强一致性 |
| 单个用户操作自己的数据 | 乐观锁 | 冲突概率低 |
| 抢购、秒杀 | 悲观锁/队列化 | 冲突极高 |
| 批量操作 | 悲观锁 | 避免部分失败 |

### 4.3 决策树

```
                    是否需要强一致性？
                         │
            ┌────────────┴────────────┐
            │ 否                      │ 是
            ▼                         ▼
       使用乐观锁             冲突率是否高？
                                   │
                      ┌────────────┴────────────┐
                      │ 否                      │ 是
                      ▼                         ▼
                 乐观锁+重试                使用悲观锁
```

---

## 五、混合方案与最佳实践

### 5.1 分段锁（降低冲突）

```sql
-- ========== 库存分段优化 ==========

-- 原方案：一个商品一条库存记录
-- 问题：所有用户争抢同一行，冲突率高

-- 优化方案：库存分段
CREATE TABLE inventory_segment (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    product_id BIGINT NOT NULL,
    segment_no INT NOT NULL,
    stock INT NOT NULL,
    version INT NOT NULL DEFAULT 0,
    UNIQUE KEY uk_product_segment (product_id, segment_no)
) ENGINE=InnoDB;

-- 将库存分成多个段
INSERT INTO inventory_segment (product_id, segment_no, stock, version) VALUES
(1001, 1, 100, 0),
(1001, 2, 100, 0),
(1001, 3, 100, 0),
(1001, 4, 100, 0);

-- 抢购时随机选择一个段
SELECT id, stock, version
FROM inventory_segment
WHERE product_id = 1001
  AND stock > 0
ORDER BY RAND()
LIMIT 1;

-- 更新选中的段
UPDATE inventory_segment
SET stock = stock - 1,
    version = version + 1
WHERE id = #{segmentId}
  AND version = #{version};

-- 优势：
-- 1. 将单点竞争分散到多个段
-- 2. 冲突率降低为原来的 1/N（N为段数）
```

### 5.2 Redis + MySQL 组合方案

```java
// ========== 高并发扣减库存方案 ==========

@Service
public class SeckillService {

    @Autowired
    private RedisTemplate<String, Object> redisTemplate;
    @Autowired
    private InventoryMapper inventoryMapper;

    private static final String STOCK_KEY = "seckill:stock:";

    /**
     * 预热：将库存加载到Redis
     */
    public void preloadStock(Long productId) {
        Integer stock = inventoryMapper.getStock(productId);
        redisTemplate.opsForValue().set(STOCK_KEY + productId, stock);
    }

    /**
     * 秒杀扣减
     */
    @Transactional
    public boolean seckill(Long productId, Long userId) {
        String key = STOCK_KEY + productId;

        // 1. Redis原子扣减（Lua脚本保证原子性）
        Long remaining = redisTemplate.opsForValue().decrement(key);

        if (remaining < 0) {
            // 库存不足，回滚
            redisTemplate.opsForValue().increment(key);
            throw new BusinessException("库存不足");
        }

        try {
            // 2. 异步/批量扣减MySQL库存
            // 可以使用消息队列异步处理
            // 也可以使用定时任务批量处理
            return true;

        } catch (Exception e) {
            // 异常时回滚Redis
            redisTemplate.opsForValue().increment(key);
            throw e;
        }
    }
}

// Redis Lua脚本：原子性检查+扣减
// local stock = redis.call('get', KEYS[1])
// if tonumber(stock) >= tonumber(ARGV[1]) then
//     return redis.call('decrby', KEYS[1], ARGV[1])
// else
//     return -1
// end
```

### 5.3 分布式锁方案

```java
// ========== 使用Redis分布式锁 ==========

@Service
public class DistributedLockService {

    @Autowired
    private RedissonClient redissonClient;

    public boolean deductStockWithDistributedLock(Long productId, int quantity) {
        String lockKey = "lock:inventory:" + productId;
        RLock lock = redissonClient.getLock(lockKey);

        try {
            // 尝试获取锁，最多等待3秒，锁10秒后自动释放
            boolean acquired = lock.tryLock(3, 10, TimeUnit.SECONDS);

            if (!acquired) {
                throw new BusinessException("系统繁忙，请稍后重试");
            }

            // 执行业务逻辑
            // 此时只有一个进程能执行
            Inventory inventory = inventoryMapper.selectByProductId(productId);
            if (inventory.getStock() < quantity) {
                throw new BusinessException("库存不足");
            }

            inventoryMapper.updateStock(productId, quantity);
            return true;

        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            throw new BusinessException("系统异常");
        } finally {
            // 确保释放锁
            if (lock.isHeldByCurrentThread()) {
                lock.unlock();
            }
        }
    }
}
```

---

## 六、监控与诊断

### 6.1 锁等待监控

```sql
-- ========== 查看当前锁等待 ==========

-- 1. 查看正在等待锁的事务
SELECT * FROM performance_schema.data_locks
WHERE LOCKED_BY = NULL;

-- 2. 查看持有锁的事务
SELECT
    r.TRX_ID AS waiting_trx_id,
    r.TRX_MYSQL_THREAD_ID AS waiting_thread,
    r.TRX_QUERY AS waiting_query,
    b.TRX_ID AS blocking_trx_id,
    b.TRX_MYSQL_THREAD_ID AS blocking_thread,
    b.TRX_QUERY AS blocking_query
FROM information_schema.innodb_lock_waits w
JOIN information_schema.innodb_trx b ON b.TRX_ID = w.BLOCKING_TRX_ID
JOIN information_schema.innodb_trx r ON r.TRX_ID = w.REQUESTING_TRX_ID;

-- 3. 查看锁等待超时设置
SHOW VARIABLES LIKE 'innodb_lock_wait_timeout';

-- 4. 查看死锁检测
SHOW VARIABLES LIKE 'innodb_deadlock_detect';

-- 5. 查看最近一次死锁信息
SHOW ENGINE INNODB STATUS;
-- 在输出中查找 "LATEST DETECTED DEADLOCK" 部分
```

### 6.2 慢查询分析

```sql
-- ========== 慢查询相关配置 ==========

-- 1. 开启慢查询
SET GLOBAL slow_query_log = ON;
SET GLOBAL long_query_time = 1;  -- 超过1秒记录

-- 2. 查看慢查询日志位置
SHOW VARIABLES LIKE 'slow_query_log_file';

-- 3. 分析锁相关的慢查询
-- 检查是否有：
--   - 全表扫描导致锁表
--   - 未使用索引导致锁升级
--   - 长事务持有锁时间过长

-- 4. 查看当前运行的事务
SELECT
    TRX_ID,
    TRX_STATE,
    TRX_STARTED,
    TRX_REQUESTED_LOCK_ID,
    TIMESTAMPDIFF(SECOND, TRX_STARTED, NOW()) AS running_seconds,
    TRX_MYSQL_THREAD_ID,
    TRX_QUERY
FROM information_schema.innodb_trx
ORDER BY TRX_STARTED;

-- 5. 杀掉长时间运行的事务
KILL <TRX_MYSQL_THREAD_ID>;
```

---

## 七、高频面试题

**Q1: 什么是ABA问题？如何解决？**

```
A: ABA问题是指：
- 原值为A
- 被改成B后又改回A
- 检查时发现还是A，以为没变，实际已经变了

在MySQL乐观锁场景：
解决方式1: 版本号递增，不重复
  version: 1 → 2 → 3，不会回到1

解决方式2: 添加时间戳字段
  WHERE id=1 AND version=1 AND update_time='2024-01-01 10:00:00'

解决方式3: 使用多版本控制
  version + checksum 组合
```

**Q2: 乐观锁一定比悲观锁性能好吗？**

```
A: 不一定，取决于冲突率：

冲突率低（<10%）：乐观锁好
- 无需等待锁
- 重试次数少

冲突率高（>20%）：悲观锁好
- 乐观锁频繁重试，浪费CPU
- 悲观锁虽然等待，但有序执行

临界点需要实际压测确定：
- 不同业务场景不同
- 不同并发量下不同
- 建议进行AB测试
```

**Q3: 如何实现一个分布式库存扣减系统？**

```
A: 多层防护方案：

1. Redis前置缓存层
   - 原子扣减（Lua脚本）
   - 快速拒绝超卖请求

2. 消息队列异步处理
   - Redis扣减成功后发MQ
   - 消费者批量更新MySQL

3. 数据库最终一致性
   - 定时对账
   - 异常补偿

4. 分库分表
   - 按商品ID分片
   - 降低单点压力

5. 限流降级
   - 限流保护后端
   - 服务降级策略
```

**Q4: Next-Key Lock是怎么工作的？**

```
A: Next-Key Lock = Record Lock + Gap Lock

示例：数据为 1, 5, 10, 15
执行：SELECT * FROM t WHERE id >= 5 AND id < 15 FOR UPDATE;

加锁情况：
┌────────────────────────────────────────────────────┐
│ (1, 5]  Next-Key  → 锁定1~5之间 + 记录5            │
│ (5, 10] Next-Key  → 锁定5~10之间 + 记录10          │
│ (10, 15) Gap      → 锁定10~15之间（不含15）        │
└────────────────────────────────────────────────────┘

防止幻读：
- 无法插入 2,3,4     ← 被(1,5]锁定
- 无法插入 6,7,8,9   ← 被(5,10]锁定
- 无法插入 11,12,13,14 ← 被(10,15)锁定

注意：
- RC隔离级别没有间隙锁
- 唯一索引等值查询会退化为记录锁
```

**Q5: 为什么SELECT ... FOR UPDATE要使用索引？**

```
A: 如果没有索引：

1. 全表扫描
   - 需要扫描每一行
   - 对扫描到的每一行加锁
   - 相当于锁表

2. 锁等待时间长
   - 扫描时间 = 加锁时间
   - 其他事务全部等待

3. 容易死锁
   - 多个事务各自锁一部分
   - 互相等待

最佳实践：
- 确保 WHERE 条件使用索引
- 复合索引注意最左前缀
- 避免类型转换导致索引失效
- 使用 EXPLAIN 检查执行计划
```

**Q6: 如何排查死锁？**

```
A: 死锁排查步骤：

1. 查看死锁日志
   SHOW ENGINE INNODB STATUS;
   -- 查找 LATEST DETECTED DEADLOCK

2. 分析死锁原因
   - 哪些事务参与
   - 持有什么锁
   - 等待什么锁
   - 为什么形成循环等待

3. 解决方案
   a. 固定加锁顺序（按主键顺序）
   b. 减小事务范围
   c. 降低隔离级别（RR → RC）
   d. 使用乐观锁
   e. 添加索引减少锁范围

4. 监控预防
   - 开启 innodb_print_all_deadlocks
   - 使用 pt-deadlock-logger 工具
   - 定期分析慢查询
```

---

## 八、总结

```
┌─────────────────────────────────────────────────────────────┐
│                    选型决策矩阵                              │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  低冲突 + 读多写少 → 乐观锁                                  │
│  高冲突 + 写多读少 → 悲观锁                                  │
│  抢购秒杀       → 队列 + Redis + 异步                        │
│  普通CRUD       → 根据实际冲突率选择                         │
│  分布式场景     → 分布式锁 + 消息队列                        │
│                                                             │
└─────────────────────────────────────────────────────────────┘

核心要点：
1. 没有银弹，根据场景选择
2. 监控是关键，数据驱动决策
3. 复杂场景可以组合使用
4. 代码中要有降级和重试机制
5. 做好压测验证方案有效性
```
