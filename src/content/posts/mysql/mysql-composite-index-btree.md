---
title: MySQL联合索引在B+树中的存储原理与最左前缀匹配
published: 2020-03-22T14:20:00Z
description: 深入剖析MySQL联合索引的B+树存储结构，理解最左前缀原则、索引覆盖和查询优化策略
tags: [MySQL, 数据库, 联合索引, 索引优化, B+树, 最左前缀]
category: 数据库原理
draft: false
---

# 联合索引在B+树中如何存储？

联合索引（也叫复合索引、组合索引）是MySQL性能优化的重要手段，理解它在B+树中的存储方式是掌握索引优化的关键。

## 一、什么是联合索引

联合索引是对表中**多个列**建立的索引。

```sql
-- 创建联合索引
CREATE INDEX idx_name_age_city ON user(name, age, city);

-- 等价于
ALTER TABLE user ADD INDEX idx_name_age_city(name, age, city);
```

这个索引包含三个字段：name、age、city，它们的**顺序非常重要**。

## 二、联合索引在B+树中的存储结构

### 1. 单列索引 vs 联合索引

**单列索引（age）的B+树：**
```
非叶子节点：[18, 25, 30, 35, ...]
              |
叶子节点：  [18→PK] → [20→PK] → [25→PK] → [28→PK] → [30→PK] → ...
```

**联合索引（name, age, city）的B+树：**
```
非叶子节点：[(Alice,18,BJ), (Bob,25,SH), (Charlie,30,GZ), ...]
                          |
叶子节点：  [(Alice,18,BJ)→PK] → [(Alice,20,BJ)→PK] → [(Alice,25,SH)→PK]
                ↓
           [(Bob,18,BJ)→PK] → [(Bob,25,SH)→PK] → [(Bob,30,GZ)→PK]
                ↓
           [(Charlie,20,BJ)→PK] → [(Charlie,25,SH)→PK] → ...
```

### 2. 核心存储原理

联合索引在B+树中的存储遵循**从左到右的排序规则**：

```
排序优先级：第一列 > 第二列 > 第三列

具体规则：
1. 先按 name 排序（字典序）
2. name 相同时，按 age 排序
3. name 和 age 都相同时，按 city 排序
```

### 3. 实际存储示例

假设有如下数据：

```sql
| id | name    | age | city |
|----|---------|-----|------|
| 1  | Alice   | 25  | BJ   |
| 2  | Bob     | 20  | SH   |
| 3  | Alice   | 20  | SH   |
| 4  | Charlie | 30  | GZ   |
| 5  | Bob     | 25  | BJ   |
| 6  | Alice   | 25  | SH   |
```

**在B+树中的存储顺序（按name, age, city）：**

```
叶子节点链表（从左到右）：
┌─────────────────────────────────────────────────────────────┐
│ (Alice,20,SH,id=3) → (Alice,25,BJ,id=1) → (Alice,25,SH,id=6) │
│      ↓                                                        │
│ (Bob,20,SH,id=2) → (Bob,25,BJ,id=5)                         │
│      ↓                                                        │
│ (Charlie,30,GZ,id=4)                                         │
└─────────────────────────────────────────────────────────────┘
```

**关键观察：**
- Alice的所有记录聚集在一起
- 在Alice内部，按age排序：20 < 25 < 25
- age相同时（两个25），再按city排序：BJ < SH

## 三、最左前缀匹配原则

### 1. 为什么存在最左前缀原则？

因为B+树的排序是**从左到右**的，只有从最左边开始的查询条件才能利用索引的有序性进行二分查找。

### 2. 可以使用索引的查询

```sql
-- ✅ 使用到索引：匹配第一列
SELECT * FROM user WHERE name = 'Alice';

-- ✅ 使用到索引：匹配前两列
SELECT * FROM user WHERE name = 'Alice' AND age = 25;

-- ✅ 使用到索引：匹配所有列
SELECT * FROM user WHERE name = 'Alice' AND age = 25 AND city = 'BJ';

-- ✅ 使用到索引：顺序无关，优化器会调整
SELECT * FROM user WHERE age = 25 AND name = 'Alice';

-- ✅ 部分使用索引：只用到name
SELECT * FROM user WHERE name = 'Alice' AND city = 'BJ';
-- 说明：name可以走索引，但city需要额外过滤
```

### 3. 无法使用索引的查询

```sql
-- ❌ 无法使用索引：跳过了第一列
SELECT * FROM user WHERE age = 25;

-- ❌ 无法使用索引：跳过了第一列
SELECT * FROM user WHERE city = 'BJ';

-- ❌ 无法使用索引：跳过了第一列
SELECT * FROM user WHERE age = 25 AND city = 'BJ';
```

### 4. 原理解释

以 `WHERE age = 25` 为例，为什么不能用索引？

```
B+树中的存储顺序：
(Alice,20,SH) → (Alice,25,BJ) → (Alice,25,SH) → (Bob,20,SH) → (Bob,25,BJ) → (Charlie,30,GZ)
        ↑              ↑              ↑                              ↑
      不是25         是25           是25                           是25
```

**问题：** age=25的数据分散在整个B+树中，无法通过二分查找快速定位，必须扫描全部叶子节点。

而 `WHERE name = 'Alice'` 可以快速定位：

```
B+树中的存储顺序：
(Alice,20,SH) → (Alice,25,BJ) → (Alice,25,SH) → (Bob,20,SH) → ...
└────────────── Alice 区域 ──────────────┘     └── 后面都不是 Alice ──
        ↑ 从这里开始                            ↑ 到这里结束
```

## 四、索引覆盖（Covering Index）

### 1. 什么是索引覆盖

查询的所有字段都在索引中，不需要回表查询。

```sql
-- ✅ 索引覆盖：所有字段都在索引 idx_name_age_city 中
SELECT name, age, city FROM user WHERE name = 'Alice';

-- ✅ 索引覆盖：包含主键id（二级索引叶子节点存储主键）
SELECT id, name, age FROM user WHERE name = 'Alice' AND age = 25;

-- ❌ 非索引覆盖：需要查询 phone 字段
SELECT name, age, phone FROM user WHERE name = 'Alice';
-- 执行流程：先在索引中找到主键id → 回表查询phone
```

### 2. 执行计划对比

```sql
EXPLAIN SELECT name, age, city FROM user WHERE name = 'Alice';
```

```
+----+-------+------+-------+---------+-------------+
| id | type  | key  | rows  | filtered| Extra       |
+----+-------+------+-------+---------+-------------+
| 1  | ref   | idx_*| 3     | 100     | Using index |  ← 索引覆盖
+----+-------+------+-------+---------+-------------+
```

```sql
EXPLAIN SELECT name, age, phone FROM user WHERE name = 'Alice';
```

```
+----+-------+------+-------+---------+-------------+
| id | type  | key  | rows  | filtered| Extra       |
+----+-------+------+-------+---------+-------------+
| 1  | ref   | idx_*| 3     | 100     | NULL        |  ← 需要回表
+----+-------+------+-------+---------+-------------+
```

## 五、联合索引的查询优化案例

### 案例1：范围查询的影响

```sql
-- 索引：idx_name_age_city (name, age, city)

-- ✅ 全部列都能用到索引
SELECT * FROM user WHERE name = 'Alice' AND age = 25 AND city = 'BJ';

-- ⚠️ 只有 name 和 age 能用到索引，city 无法使用
SELECT * FROM user WHERE name = 'Alice' AND age > 20 AND city = 'BJ';
```

**原因：** 
- `age > 20` 是范围查询，导致后续的 city 无法利用索引有序性
- B+树定位到 name='Alice' 后，age>20 的记录不再连续，city 是分散的

```
(Alice,21,BJ) → (Alice,21,SH) → (Alice,25,BJ) → (Alice,25,SH) → (Alice,30,GZ)
                      ↑ city混乱，无法二分查找
```

### 案例2：索引列顺序的选择

**场景：** 需要频繁查询以下SQL

```sql
-- 查询1：按名字和年龄查询（高频）
SELECT * FROM user WHERE name = 'Alice' AND age = 25;

-- 查询2：只按名字查询（中频）
SELECT * FROM user WHERE name = 'Bob';

-- 查询3：只按年龄查询（低频）
SELECT * FROM user WHERE age = 30;
```

**应该建立哪个索引？**

```sql
-- 方案A：idx_name_age
-- ✅ 查询1 和 查询2 都能用到
-- ❌ 查询3 用不到

-- 方案B：idx_age_name
-- ✅ 查询1 能用到
-- ❌ 查询2 用不到
-- ✅ 查询3 能用到

-- 推荐：方案A + 单独的 idx_age
```

**选择原则：**
1. 区分度高的列放前面（name的唯一值多，age的唯一值少）
2. 最常用的查询条件放前面
3. 等值查询的列放在范围查询的列前面

### 案例3：IN查询的优化

```sql
-- 索引：idx_name_age_city

-- ✅ 所有列都能用到索引（IN会转换为多个等值查询）
SELECT * FROM user WHERE name IN ('Alice', 'Bob') AND age = 25 AND city = 'BJ';

-- 内部执行相当于：
-- (name='Alice' AND age=25 AND city='BJ') OR (name='Bob' AND age=25 AND city='BJ')
```

## 六、联合索引 vs 多个单列索引

### 对比

```sql
-- 方案A：联合索引
CREATE INDEX idx_name_age ON user(name, age);

-- 方案B：两个单列索引
CREATE INDEX idx_name ON user(name);
CREATE INDEX idx_age ON user(age);
```

**查询性能对比：**

```sql
SELECT * FROM user WHERE name = 'Alice' AND age = 25;
```

| 方案 | 执行方式 | 效率 |
|------|---------|------|
| 联合索引 | 直接在一棵B+树中精确定位 | ⭐⭐⭐⭐⭐ 最优 |
| 多个单列索引 | 使用索引合并（Index Merge），查两棵树后取交集 | ⭐⭐⭐ 较差 |

**结论：** 对于经常一起查询的列，联合索引优于多个单列索引。

## 七、实战建议

### 1. 索引设计原则

```sql
-- ✅ 推荐：高频查询组合建联合索引
CREATE INDEX idx_status_create_time ON orders(status, create_time);
-- 适用于：WHERE status = 1 ORDER BY create_time DESC

-- ❌ 不推荐：过长的联合索引（超过5列）
CREATE INDEX idx_too_long ON user(a, b, c, d, e, f, g);
-- 问题：索引太大，维护成本高
```

### 2. 检查索引使用情况

```sql
-- 查看索引统计信息
SHOW INDEX FROM user;

-- 查看执行计划
EXPLAIN SELECT * FROM user WHERE name = 'Alice' AND age = 25;

-- 查看索引使用情况（MySQL 8.0+）
SELECT * FROM sys.schema_unused_indexes WHERE object_schema = 'your_db';
```

### 3. 常见错误

```sql
-- ❌ 错误1：在联合索引的中间列上使用函数
SELECT * FROM user WHERE name = 'Alice' AND YEAR(create_time) = 2024;
-- 问题：create_time 上的函数导致索引失效

-- ✅ 正确：改写查询条件
SELECT * FROM user 
WHERE name = 'Alice' 
  AND create_time >= '2024-01-01' 
  AND create_time < '2025-01-01';

-- ❌ 错误2：隐式类型转换
CREATE INDEX idx_phone ON user(phone);  -- phone 是 VARCHAR
SELECT * FROM user WHERE phone = 13800138000;  -- 传入的是数字
-- 问题：发生类型转换，索引失效

-- ✅ 正确：保持类型一致
SELECT * FROM user WHERE phone = '13800138000';
```

## 八、总结

### 联合索引的B+树存储核心要点

1. **排序规则**：从左到右依次排序，形成有序的B+树结构
2. **最左前缀**：只有从最左列开始的查询才能有效利用索引
3. **索引覆盖**：查询字段都在索引中时，性能最优
4. **范围查询**：范围查询会导致后续列无法使用索引
5. **列顺序**：区分度高、常用的列应该放在前面

### 记忆口诀

```
联合索引要记牢，从左到右排序好
最左前缀是关键，断了前缀索引废
等值查询排前面，范围查询放后边
覆盖索引性能高，回表操作要减少
```

### 进阶思考

- 为什么MySQL的索引最多16列？
- 如何选择合适的索引列顺序？
- 联合索引和索引下推（ICP）的关系？
- 什么时候应该使用前缀索引？

理解联合索引的存储原理，是写出高性能SQL的基础！
