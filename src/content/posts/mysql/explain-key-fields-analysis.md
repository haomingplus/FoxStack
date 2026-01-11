---
title: EXPLAIN执行计划关键字段深度解析
published: 2023-08-15
description: 全面剖析EXPLAIN输出的核心字段，掌握type访问类型、key索引选择、Extra额外信息等关键指标的解读与优化技巧
tags: [MySQL, SQL优化, EXPLAIN, 索引, 执行计划, 性能调优]
category: MySQL性能优化
draft: false
---

## 面试题

使用 EXPLAIN 分析 SQL 时，你主要关注哪些字段或指标？

## 一句话回答

主要关注 **type**（访问类型）、**key**（实际使用的索引）、**rows**（扫描行数）、**Extra**（额外信息），其中 type 决定了查询效率的上限，Extra 中的 `Using filesort` 和 `Using temporary` 是重点优化目标。

## 详细解析

### 一、EXPLAIN 输出字段全览

```sql
EXPLAIN SELECT * FROM orders WHERE user_id = 10086 AND status = 'PAID';
```

```
+----+-------------+--------+------------+------+---------------+---------+---------+-------+------+----------+-------------+
| id | select_type | table  | partitions | type | possible_keys | key     | key_len | ref   | rows | filtered | Extra       |
+----+-------------+--------+------------+------+---------------+---------+---------+-------+------+----------+-------------+
|  1 | SIMPLE      | orders | NULL       | ref  | idx_user      | idx_user| 8       | const | 156  |    10.00 | Using where |
+----+-------------+--------+------------+------+---------------+---------+---------+-------+------+----------+-------------+
```

**字段重要程度分级**：

```
⭐⭐⭐⭐⭐ 必看：type、key、rows、Extra
⭐⭐⭐⭐   重要：key_len、ref、filtered
⭐⭐⭐     了解：id、select_type、table、possible_keys
⭐⭐       一般：partitions
```

### 二、核心字段详解

---

#### 2.1 type（访问类型）⭐⭐⭐⭐⭐

**这是最重要的字段**，表示 MySQL 如何查找表中的行。

```
性能从好到差排序：

system > const > eq_ref > ref > range > index > ALL
  │       │        │       │      │       │      │
  │       │        │       │      │       │      └── 全表扫描 ❌
  │       │        │       │      │       └── 全索引扫描
  │       │        │       │      └── 索引范围扫描 ✅
  │       │        │       └── 非唯一索引等值查询 ✅
  │       │        └── 唯一索引等值连接 ✅✅
  │       └── 主键/唯一索引等值查询 ✅✅✅
  └── 表只有一行（系统表）
```

**各类型详细说明**：

| type | 说明 | 示例 | 性能 |
|------|------|------|------|
| **system** | 表只有一行记录 | 系统表 | 最快 |
| **const** | 通过主键或唯一索引访问一行 | `WHERE id = 1` | 极快 |
| **eq_ref** | 连接时使用主键或唯一索引 | `JOIN ON a.id = b.id` (id是主键) | 很快 |
| **ref** | 非唯一索引等值查询 | `WHERE user_id = 100` | 快 |
| **range** | 索引范围扫描 | `WHERE id > 100`、`WHERE id IN (1,2,3)` | 较快 |
| **index** | 全索引扫描 | `SELECT id FROM t` (id有索引) | 一般 |
| **ALL** | 全表扫描 | 无索引或索引失效 | **最慢** ❌ |

**实战示例**：

```sql
-- const：主键等值查询
EXPLAIN SELECT * FROM users WHERE id = 1;
-- type: const

-- ref：普通索引等值查询
EXPLAIN SELECT * FROM orders WHERE user_id = 10086;
-- type: ref

-- range：索引范围查询
EXPLAIN SELECT * FROM orders WHERE create_time > '2024-01-01';
-- type: range

-- ALL：全表扫描（需要优化！）
EXPLAIN SELECT * FROM orders WHERE YEAR(create_time) = 2024;
-- type: ALL（函数导致索引失效）
```

**优化目标**：
```
生产环境 SQL 的 type 至少要达到 range 级别
最理想是 ref 或 const
如果出现 ALL，必须优化！
```

---

#### 2.2 key & possible_keys（索引选择）⭐⭐⭐⭐⭐

```
possible_keys：可能使用的索引（候选）
key：实际使用的索引（最终选择）
```

**常见情况分析**：

```sql
-- 情况一：有候选但未使用（需关注）
possible_keys: idx_user_id,idx_status
key: NULL  ← 说明索引失效或优化器认为全表更快

-- 情况二：多个候选选择了一个
possible_keys: idx_user_id,idx_status  
key: idx_user_id  ← 优化器选择了 idx_user_id

-- 情况三：没有候选
possible_keys: NULL
key: NULL  ← 需要添加索引
```

**为什么有索引却不用？**

```sql
-- 原因一：数据量太小，全表更快
-- 原因二：查询结果占比太高（>30%），回表代价大
-- 原因三：索引失效（函数、隐式转换等）

-- 强制使用某个索引（不推荐，优化器通常更聪明）
SELECT * FROM orders FORCE INDEX(idx_user_id) WHERE user_id = 10086;
```

---

#### 2.3 key_len（索引使用长度）⭐⭐⭐⭐

**用于判断联合索引使用了多少个字段**。

**计算规则**：

```
┌────────────────┬─────────────────────────────────────┐
│    数据类型     │              字节数                  │
├────────────────┼─────────────────────────────────────┤
│ TINYINT        │ 1                                   │
│ SMALLINT       │ 2                                   │
│ INT            │ 4                                   │
│ BIGINT         │ 8                                   │
│ DATE           │ 3                                   │
│ DATETIME       │ 8                                   │
│ TIMESTAMP      │ 4                                   │
│ CHAR(n)        │ n * 字符集字节数                     │
│ VARCHAR(n)     │ n * 字符集字节数 + 2                 │
├────────────────┼─────────────────────────────────────┤
│ 允许 NULL      │ +1 字节                              │
└────────────────┴─────────────────────────────────────┘

字符集字节数：
- latin1: 1
- utf8: 3  
- utf8mb4: 4
```

**实战计算**：

```sql
-- 表结构
CREATE TABLE orders (
    id BIGINT NOT NULL,
    user_id BIGINT NOT NULL,
    status VARCHAR(20) NOT NULL,
    create_time DATETIME NOT NULL,
    INDEX idx_composite (user_id, status, create_time)
);

-- 字符集：utf8mb4

-- 计算 key_len：
-- user_id: BIGINT NOT NULL = 8
-- status: VARCHAR(20) NOT NULL = 20 * 4 + 2 = 82
-- create_time: DATETIME NOT NULL = 8

-- 场景一：只用 user_id
EXPLAIN SELECT * FROM orders WHERE user_id = 10086;
-- key_len = 8 ← 只用了联合索引的第一个字段

-- 场景二：用 user_id + status
EXPLAIN SELECT * FROM orders WHERE user_id = 10086 AND status = 'PAID';
-- key_len = 8 + 82 = 90 ← 用了前两个字段

-- 场景三：全部使用
EXPLAIN SELECT * FROM orders 
WHERE user_id = 10086 AND status = 'PAID' AND create_time > '2024-01-01';
-- key_len = 8 + 82 + 8 = 98 ← 全部字段都用上了
```

**key_len 的价值**：

```
通过 key_len 可以判断：
1. 联合索引用了几个字段
2. 是否达到了预期的索引使用效果
3. 帮助验证最左前缀原则
```

---

#### 2.4 rows（扫描行数估算）⭐⭐⭐⭐⭐

```
表示 MySQL 预估需要扫描的行数
这是一个估算值，不是精确值
rows 越小越好
```

**对比优化效果**：

```sql
-- 优化前：无索引
EXPLAIN SELECT * FROM orders WHERE user_id = 10086;
-- rows: 500000（全表扫描）

-- 优化后：添加索引
CREATE INDEX idx_user_id ON orders(user_id);
EXPLAIN SELECT * FROM orders WHERE user_id = 10086;
-- rows: 156（索引精确定位）

-- 优化效果：扫描行数降低 3000+ 倍！
```

**rows 与实际性能**：

```
rows 只是估算，实际性能还要考虑：
1. 是否需要回表（覆盖索引可避免）
2. 数据是否在 Buffer Pool 中
3. 磁盘 I/O 情况
```

---

#### 2.5 filtered（过滤比例）⭐⭐⭐⭐

```
表示经过条件过滤后，剩余数据的百分比
结合 rows 可以计算最终结果行数
最终行数 ≈ rows × filtered%
```

```sql
EXPLAIN SELECT * FROM orders WHERE user_id = 10086 AND status = 'PAID';
-- rows: 1000, filtered: 10.00

-- 预估最终结果：1000 × 10% = 100 行
```

**filtered 的意义**：

```
filtered 低说明：
1. 条件过滤掉了大量数据
2. 可能有部分条件没有走索引
3. 考虑优化联合索引包含更多条件字段
```

---

#### 2.6 Extra（额外信息）⭐⭐⭐⭐⭐

**这是第二重要的字段**，包含很多关键的优化提示。

**需要重点关注的值**：

| Extra 值 | 含义 | 是否需要优化 |
|---------|------|-------------|
| **Using index** | 覆盖索引，无需回表 | ✅ 很好 |
| **Using where** | 使用 WHERE 过滤 | 正常 |
| **Using index condition** | 索引条件下推（ICP） | ✅ 较好 |
| **Using temporary** | 使用了临时表 | ⚠️ **需要优化** |
| **Using filesort** | 使用了文件排序 | ⚠️ **需要优化** |
| **Using join buffer** | 连接使用了缓冲区 | ⚠️ 可能缺索引 |
| **Select tables optimized away** | 优化器直接返回结果 | ✅ 最好 |
| **Impossible WHERE** | WHERE 条件永远为假 | 检查 SQL 逻辑 |

**重点一：Using filesort（文件排序）**

```sql
-- 出现 filesort 的情况
EXPLAIN SELECT * FROM orders WHERE status = 'PAID' ORDER BY create_time;
-- Extra: Using filesort ← 排序字段没有索引

-- 优化方案：联合索引覆盖 WHERE + ORDER BY
CREATE INDEX idx_status_time ON orders(status, create_time);

-- 优化后
EXPLAIN SELECT * FROM orders WHERE status = 'PAID' ORDER BY create_time;
-- Extra: Using index condition ← filesort 消除
```

**重点二：Using temporary（临时表）**

```sql
-- 出现 temporary 的情况
EXPLAIN SELECT DISTINCT user_id FROM orders WHERE status = 'PAID';
-- Extra: Using temporary

-- 常见于：
-- 1. DISTINCT
-- 2. GROUP BY
-- 3. UNION
-- 4. 复杂子查询

-- 优化方案：让 DISTINCT/GROUP BY 的字段有索引
CREATE INDEX idx_status_user ON orders(status, user_id);
```

**重点三：Using index（覆盖索引）**

```sql
-- 查询字段都在索引中，无需回表
-- 索引：(user_id, status, amount)
EXPLAIN SELECT user_id, status, amount FROM orders WHERE user_id = 10086;
-- Extra: Using index ← 覆盖索引，最高效

-- 如果 SELECT * 就需要回表
EXPLAIN SELECT * FROM orders WHERE user_id = 10086;
-- Extra: Using index condition ← 需要回表取其他字段
```

---

#### 2.7 id & select_type（查询标识）⭐⭐⭐

**id**：查询序列号，表示执行顺序

```sql
EXPLAIN 
SELECT * FROM users u 
WHERE u.id IN (SELECT user_id FROM orders WHERE amount > 1000);

-- id=1: users 表查询（主查询）
-- id=2: orders 子查询

-- 规则：
-- id 相同：从上往下执行
-- id 不同：id 大的先执行
```

**select_type**：查询类型

```
┌─────────────────────┬────────────────────────────────────┐
│     select_type     │               说明                  │
├─────────────────────┼────────────────────────────────────┤
│ SIMPLE              │ 简单查询（不含子查询或 UNION）      │
│ PRIMARY             │ 最外层查询                         │
│ SUBQUERY            │ 子查询中的第一个 SELECT            │
│ DERIVED             │ 派生表（FROM 子句中的子查询）       │
│ UNION               │ UNION 中的第二个及后续 SELECT      │
│ UNION RESULT        │ UNION 的结果                       │
│ DEPENDENT SUBQUERY  │ 依赖外部查询的子查询 ⚠️            │
└─────────────────────┴────────────────────────────────────┘
```

```sql
-- DEPENDENT SUBQUERY 需要特别注意，性能通常较差
EXPLAIN 
SELECT * FROM users u 
WHERE EXISTS (SELECT 1 FROM orders o WHERE o.user_id = u.id);

-- 可能改写为 JOIN 提升性能
SELECT DISTINCT u.* FROM users u 
INNER JOIN orders o ON u.id = o.user_id;
```

---

### 三、EXPLAIN 分析实战流程

```
┌─────────────────────────────────────────────────────────────────┐
│                    EXPLAIN 分析流程                              │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
                    ┌─────────────────┐
                    │ 1. 先看 type    │
                    │ 是否为 ALL？    │
                    └─────────────────┘
                         │
            ┌────────────┴────────────┐
            │ 是 ALL                  │ 否
            ▼                         ▼
    ┌───────────────┐         ┌───────────────┐
    │ 检查索引是否  │         │ 2. 看 key     │
    │ 存在或失效    │         │ 用对索引了吗？ │
    └───────────────┘         └───────────────┘
                                      │
                                      ▼
                              ┌───────────────┐
                              │ 3. 看 key_len │
                              │ 联合索引用全了？│
                              └───────────────┘
                                      │
                                      ▼
                              ┌───────────────┐
                              │ 4. 看 rows    │
                              │ 扫描行数合理？ │
                              └───────────────┘
                                      │
                                      ▼
                              ┌───────────────┐
                              │ 5. 看 Extra   │
                              │ 有 filesort？ │
                              │ 有 temporary？│
                              └───────────────┘
```

### 四、EXPLAIN 速查表

```
┌──────────────────────────────────────────────────────────────────────┐
│                        EXPLAIN 速查表                                 │
├────────────┬─────────────────────────────────────────────────────────┤
│   字段      │                     关注点                              │
├────────────┼─────────────────────────────────────────────────────────┤
│ type       │ 至少 range，避免 ALL                                    │
│ key        │ 是否使用了预期索引                                       │
│ key_len    │ 联合索引是否充分使用                                     │
│ rows       │ 越小越好，对比优化前后                                   │
│ filtered   │ 结合 rows 计算最终行数                                   │
│ Extra      │ ❌ Using filesort  ❌ Using temporary                   │
│            │ ✅ Using index     ✅ Using index condition              │
└────────────┴─────────────────────────────────────────────────────────┘
```

### 五、总结

**面试回答模板**：

> "使用 EXPLAIN 分析 SQL 时，我主要关注以下几个字段：
>
> **第一看 type**：访问类型，至少要达到 range 级别。如果出现 ALL 全表扫描，必须优化。
>
> **第二看 key**：确认是否使用了预期的索引。配合 key_len 判断联合索引用了几个字段。
>
> **第三看 rows**：预估扫描行数，越小越好。这是衡量优化效果的重要指标。
>
> **第四看 Extra**：重点关注 `Using filesort` 和 `Using temporary`，这两个是性能杀手，需要通过调整索引或改写 SQL 来消除。如果看到 `Using index` 则说明用了覆盖索引，是最理想的状态。
>
> 在实际优化中，我会先看 type 和 Extra 定位问题，再通过调整索引观察 rows 的变化来验证优化效果。"
