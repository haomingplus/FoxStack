---
title: MySQL当前读与快照读详解
published: 2019-09-27
description: 深入理解InnoDB两种读取模式的区别、触发条件、锁机制及在不同隔离级别下的表现
tags: [MySQL, MVCC, 当前读, 快照读, 锁机制, InnoDB]
category: MySQL性能优化
draft: false
---

## 面试题

说一下当前读和快照读

## 一句话回答

**快照读**是普通 SELECT，通过 MVCC 读取历史版本，不加锁；**当前读**是 `SELECT ... FOR UPDATE`、DML 操作等，读取最新已提交数据并加锁。快照读性能高但可能读到旧数据，当前读保证数据最新但会阻塞其他事务。

## 详细解析

### 一、核心区别对比

| 对比项 | 快照读（Snapshot Read） | 当前读（Current Read） |
|-------|------------------------|----------------------|
| **读取内容** | MVCC 版本链中的历史版本 | 数据库中最新的已提交版本 |
| **是否加锁** | ❌ 不加锁 | ✅ 加锁（共享锁或排他锁） |
| **并发性能** | 高（读写不阻塞） | 较低（可能阻塞） |
| **数据实时性** | 可能是旧数据 | 一定是最新数据 |
| **实现机制** | MVCC + Read View | 锁机制 |

### 二、快照读（Snapshot Read）

#### 2.1 定义

快照读是指读取数据的**某个历史版本**，通过 MVCC 机制实现，不需要加锁。

#### 2.2 触发 SQL

```sql
-- 普通 SELECT 就是快照读
SELECT * FROM user WHERE id = 1;
SELECT name, age FROM user WHERE status = 1;
SELECT COUNT(*) FROM orders;
```

#### 2.3 工作原理

```
快照读流程：

┌─────────────┐
│  SELECT ... │
└──────┬──────┘
       │
       ▼
┌─────────────────────────┐
│  生成/复用 Read View    │
│  （取决于隔离级别）       │
└──────┬──────────────────┘
       │
       ▼
┌─────────────────────────┐
│  遍历版本链              │
│  根据可见性规则判断       │
└──────┬──────────────────┘
       │
       ▼
┌─────────────────────────┐
│  返回第一个可见版本       │
│  （不一定是最新版本）     │
└─────────────────────────┘
```

#### 2.4 示例演示

```
事务A                              事务B
─────────────────────────────────────────────────
                                   BEGIN;
                                   UPDATE user SET name='李四' 
                                   WHERE id=1;  -- 原值'张三'
BEGIN;
SELECT * FROM user WHERE id=1;
-- 快照读：看到 '张三'（事务B未提交）

                                   COMMIT;

SELECT * FROM user WHERE id=1;
-- RR级别：仍然看到 '张三'（复用 Read View）
-- RC级别：看到 '李四'（新生成 Read View）

COMMIT;
```

#### 2.5 快照读的优势

```
✅ 不加锁，读写不阻塞
✅ 高并发性能
✅ 避免脏读
✅ RR 级别下保证可重复读
```

---

### 三、当前读（Current Read）

#### 3.1 定义

当前读是指读取数据的**最新已提交版本**，并对读取的数据加锁。

#### 3.2 触发 SQL

```sql
-- 1. 共享锁（S锁）当前读
SELECT * FROM user WHERE id = 1 LOCK IN SHARE MODE;  -- MySQL 5.x
SELECT * FROM user WHERE id = 1 FOR SHARE;           -- MySQL 8.0+

-- 2. 排他锁（X锁）当前读
SELECT * FROM user WHERE id = 1 FOR UPDATE;

-- 3. DML 操作都是当前读
INSERT INTO user(name) VALUES('张三');
UPDATE user SET name = '李四' WHERE id = 1;
DELETE FROM user WHERE id = 1;
```

#### 3.3 工作原理

```
当前读流程：

┌─────────────────────────┐
│  SELECT ... FOR UPDATE  │
│  或 UPDATE / DELETE     │
└──────┬──────────────────┘
       │
       ▼
┌─────────────────────────┐
│  读取最新已提交数据       │
│  （忽略 MVCC 版本链）     │
└──────┬──────────────────┘
       │
       ▼
┌─────────────────────────┐
│  对数据加锁              │
│  （行锁 + 可能的间隙锁）  │
└──────┬──────────────────┘
       │
       ▼
┌─────────────────────────┐
│  返回最新数据            │
│  其他事务被阻塞          │
└─────────────────────────┘
```

#### 3.4 示例演示

```
事务A                              事务B
─────────────────────────────────────────────────
BEGIN;
SELECT * FROM user WHERE id=1 FOR UPDATE;
-- 当前读：加排他锁，读到最新值

                                   BEGIN;
                                   UPDATE user SET name='李四' 
                                   WHERE id=1;
                                   -- 阻塞！等待事务A释放锁

COMMIT;  -- 释放锁

                                   -- 事务B继续执行
                                   COMMIT;
```

#### 3.5 当前读的锁类型

```sql
-- 共享锁（S锁）：允许其他事务读，不允许写
SELECT * FROM user WHERE id = 1 FOR SHARE;
-- 其他事务可以：SELECT / SELECT ... FOR SHARE
-- 其他事务不可以：UPDATE / DELETE / SELECT ... FOR UPDATE

-- 排他锁（X锁）：不允许其他事务读写
SELECT * FROM user WHERE id = 1 FOR UPDATE;
-- 其他事务都会被阻塞
```

---

### 四、图解对比

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         快照读 vs 当前读                                     │
└─────────────────────────────────────────────────────────────────────────────┘

数据库状态：
┌─────────────────────────────────────┐
│  最新数据: name = '王五' (TRX=103)  │ ← 当前读从这里读
├─────────────────────────────────────┤
│  历史版本: name = '李四' (TRX=102)  │
├─────────────────────────────────────┤
│  历史版本: name = '张三' (TRX=101)  │ ← 快照读可能从这里读
└─────────────────────────────────────┘

事务A (TRX=100，在 TRX=102 之前开始)：

快照读：SELECT * FROM user WHERE id=1;
        → 返回 '李四' 或 '张三'（取决于 Read View）

当前读：SELECT * FROM user WHERE id=1 FOR UPDATE;
        → 返回 '王五'（最新值）+ 加锁
```

---

### 五、为什么 UPDATE 要用当前读？

```sql
-- 假设 balance = 100

-- 事务A
BEGIN;
UPDATE account SET balance = balance + 50 WHERE id = 1;
-- 如果用快照读（读到旧值100）：balance = 100 + 50 = 150
-- 实际用当前读（读最新值）：balance = 最新值 + 50

-- 事务B（同时执行）
BEGIN;
UPDATE account SET balance = balance - 30 WHERE id = 1;
COMMIT;
```

**如果 UPDATE 用快照读会怎样？**

```
时间线：

T1: 事务A 快照读 balance = 100
T2: 事务B 快照读 balance = 100
T3: 事务B 更新 balance = 100 - 30 = 70，提交
T4: 事务A 更新 balance = 100 + 50 = 150，提交

最终结果：150
正确结果：100 + 50 - 30 = 120

问题：事务B 的修改被覆盖了！（丢失更新）
```

**UPDATE 用当前读的正确流程**：

```
时间线：

T1: 事务A 当前读 balance = 100，加排他锁
T2: 事务B 当前读 → 阻塞（等待锁）
T3: 事务A 更新 balance = 150，提交，释放锁
T4: 事务B 当前读 balance = 150
T5: 事务B 更新 balance = 150 - 30 = 120，提交

最终结果：120 ✓
```

---

### 六、RR 级别下的间隙锁

当前读在 RR 级别下会触发**间隙锁（Gap Lock）**，防止幻读。

```sql
-- 假设 id 索引现有值：1, 5, 10

-- 事务A
BEGIN;
SELECT * FROM user WHERE id > 3 FOR UPDATE;
-- 加锁范围：
-- 行锁：id = 5, 10
-- 间隙锁：(3, 5), (5, 10), (10, +∞)

-- 事务B
BEGIN;
INSERT INTO user(id, name) VALUES(7, '张三');
-- 阻塞！7 在间隙 (5, 10) 中
```

```
间隙锁示意图：

     1        5        10       +∞
     │        │        │        │
─────●────────●────────●────────→
         │         │        │
         └────┬────┴────────┘
              │
    这些间隙都被锁住，无法插入新数据
```

---

### 七、不同隔离级别的表现

| 隔离级别 | 快照读 | 当前读 |
|---------|--------|--------|
| **RC** | 每次 SELECT 生成新 Read View | 只有行锁，无间隙锁 |
| **RR** | 第一次 SELECT 生成，后续复用 | 行锁 + 间隙锁（Next-Key Lock） |

```
RC 级别当前读：
SELECT ... FOR UPDATE → 只锁匹配的行

RR 级别当前读：
SELECT ... FOR UPDATE → 锁匹配的行 + 锁住间隙

这就是为什么 RC 级别并发性能更好，但可能有幻读
```

---

### 八、实际应用场景

#### 8.1 快照读场景

```sql
-- 普通查询，不需要最新数据
SELECT * FROM products WHERE category = 'phone';

-- 报表统计，允许轻微延迟
SELECT COUNT(*), SUM(amount) FROM orders WHERE date = '2024-01-01';

-- 列表展示
SELECT * FROM articles ORDER BY create_time DESC LIMIT 20;
```

#### 8.2 当前读场景

```sql
-- 扣减库存（必须读最新值）
SELECT stock FROM products WHERE id = 1 FOR UPDATE;
UPDATE products SET stock = stock - 1 WHERE id = 1;

-- 转账（防止并发修改）
SELECT balance FROM account WHERE id = 1 FOR UPDATE;
UPDATE account SET balance = balance - 100 WHERE id = 1;

-- 防止重复插入
SELECT * FROM user WHERE phone = '13800138000' FOR UPDATE;
-- 如果不存在则插入
INSERT INTO user(phone, name) VALUES('13800138000', '张三');
```

---

### 九、速查表

| SQL 语句 | 读取类型 | 加锁 | 读取版本 |
|---------|---------|------|---------|
| `SELECT ...` | 快照读 | ❌ | 历史版本 |
| `SELECT ... FOR SHARE` | 当前读 | S 锁 | 最新版本 |
| `SELECT ... FOR UPDATE` | 当前读 | X 锁 | 最新版本 |
| `INSERT ...` | 当前读 | X 锁 | - |
| `UPDATE ...` | 当前读 | X 锁 | 最新版本 |
| `DELETE ...` | 当前读 | X 锁 | 最新版本 |

---

### 十、总结

**面试回答模板**：

> "MySQL InnoDB 有两种读取模式：
>
> **快照读**：
> - 普通 SELECT 语句
> - 通过 MVCC 读取历史版本
> - 不加锁，读写不阻塞，并发性能高
> - 可能读到旧数据
>
> **当前读**：
> - `SELECT ... FOR UPDATE`、`SELECT ... FOR SHARE`、以及所有 DML 操作
> - 读取最新已提交数据
> - 加锁（共享锁或排他锁）
> - 保证数据最新，但会阻塞其他事务
>
> **为什么 UPDATE 必须用当前读**：如果读旧值再更新，会导致丢失更新问题。
>
> **RR 级别的间隙锁**：当前读会加间隙锁防止幻读，这也是 RR 比 RC 并发性能低的原因。
>
> **应用场景**：普通查询用快照读提高性能，涉及并发修改的场景（如扣库存、转账）用当前读保证正确性。"
