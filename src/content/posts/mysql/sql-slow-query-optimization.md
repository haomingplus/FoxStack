---
title: SQL慢查询排查与优化实战指南
published: 2025-01-08
description: 从慢查询日志分析到EXPLAIN执行计划解读，全面讲解SQL性能问题的排查思路、索引优化策略及实际案例
tags: [Java, MySQL, SQL优化, 索引, EXPLAIN, 慢查询]
category: MySQL性能优化
draft: false
---

## 面试题

在项目中是否遇到过 SQL 查询性能问题？如何排查和优化慢查询？

## 题目分析

这是一道实战经验题，面试官想考察：
1. 是否有真实的性能优化经验
2. 排查问题的方法论和思路
3. 对 MySQL 执行计划、索引原理的理解
4. 常见优化手段的掌握程度

**回答策略**：先讲排查流程，再讲优化方法，最后结合实际案例。

## 参考答案

### 一、慢查询排查的完整流程

```
┌─────────────────────────────────────────────────────────────────┐
│                      慢查询排查流程                              │
└─────────────────────────────────────────────────────────────────┘
                              │
          ┌───────────────────┼───────────────────┐
          ▼                   ▼                   ▼
    ┌──────────┐        ┌──────────┐        ┌──────────┐
    │ 发现问题  │        │ 定位SQL  │        │ 分析原因  │
    └──────────┘        └──────────┘        └──────────┘
    │ - 监控告警        │ - 慢查询日志      │ - EXPLAIN
    │ - 用户反馈        │ - APM 工具        │ - SHOW PROFILE
    │ - 巡检发现        │ - 数据库监控      │ - 索引分析
          │                   │                   │
          └───────────────────┼───────────────────┘
                              ▼
                        ┌──────────┐
                        │ 优化执行  │
                        └──────────┘
                        │ - 索引优化
                        │ - SQL 重写
                        │ - 表结构调整
                        │ - 架构优化
```

### 二、开启和分析慢查询日志

#### 2.1 开启慢查询日志

```sql
-- 查看慢查询配置
SHOW VARIABLES LIKE '%slow_query%';
SHOW VARIABLES LIKE '%long_query_time%';

-- 动态开启慢查询日志
SET GLOBAL slow_query_log = ON;
SET GLOBAL long_query_time = 1;  -- 超过 1 秒记录
SET GLOBAL log_queries_not_using_indexes = ON;  -- 记录未使用索引的查询

-- 查看慢查询日志位置
SHOW VARIABLES LIKE 'slow_query_log_file';
-- 通常在 /var/lib/mysql/hostname-slow.log
```

```ini
# my.cnf 永久配置
[mysqld]
slow_query_log = ON
slow_query_log_file = /var/lib/mysql/slow.log
long_query_time = 1
log_queries_not_using_indexes = ON
```

#### 2.2 分析慢查询日志

**使用 mysqldumpslow 工具**：

```bash
# 按查询时间排序，显示前 10 条
mysqldumpslow -s t -t 10 /var/lib/mysql/slow.log

# 按查询次数排序
mysqldumpslow -s c -t 10 /var/lib/mysql/slow.log

# 按平均查询时间排序
mysqldumpslow -s at -t 10 /var/lib/mysql/slow.log

# 输出示例
# Count: 156  Time=3.50s (546s)  Lock=0.00s (0s)  Rows=1.0 (156)
# SELECT * FROM orders WHERE user_id = N AND status = S
```

**使用 pt-query-digest（推荐）**：

```bash
# 安装 Percona Toolkit
yum install percona-toolkit

# 分析慢查询日志
pt-query-digest /var/lib/mysql/slow.log > slow_report.txt

# 分析指定时间段
pt-query-digest --since '2024-01-01 00:00:00' --until '2024-01-02 00:00:00' slow.log
```

### 三、EXPLAIN 执行计划详解

#### 3.1 EXPLAIN 基本使用

```sql
EXPLAIN SELECT * FROM orders 
WHERE user_id = 10086 AND status = 'PAID' 
ORDER BY create_time DESC 
LIMIT 10;
```

```
+----+-------------+--------+------+---------------+------+---------+------+--------+-----------------------------+
| id | select_type | table  | type | possible_keys | key  | key_len | ref  | rows   | Extra                       |
+----+-------------+--------+------+---------------+------+---------+------+--------+-----------------------------+
|  1 | SIMPLE      | orders | ALL  | idx_user_id   | NULL | NULL    | NULL | 500000 | Using where; Using filesort |
+----+-------------+--------+------+---------------+------+---------+------+--------+-----------------------------+
```

#### 3.2 关键字段详解

**type 字段（访问类型，从好到差）**：

```
┌─────────┬────────────────────────────────────────────────────┐
│  type   │                        说明                         │
├─────────┼────────────────────────────────────────────────────┤
│ system  │ 表只有一行（系统表），最快                           │
│ const   │ 通过主键或唯一索引访问一行，非常快                    │
│ eq_ref  │ 主键/唯一索引的等值连接，每次只返回一行               │
│ ref     │ 非唯一索引的等值查询                                 │
│ range   │ 索引范围扫描（>、<、BETWEEN、IN）                    │
│ index   │ 全索引扫描（比 ALL 好，因为索引文件小）               │
│ ALL     │ 全表扫描，最慢 ⚠️                                    │
└─────────┴────────────────────────────────────────────────────┘

优化目标：至少达到 range 级别，最好是 ref 或 const
```

**Extra 字段（重要提示）**：

```
┌────────────────────────┬────────────────────────────────────┐
│         Extra          │               说明                  │
├────────────────────────┼────────────────────────────────────┤
│ Using index            │ 覆盖索引，不需要回表 ✅              │
│ Using where            │ 使用了 WHERE 过滤                   │
│ Using index condition  │ 索引条件下推（ICP）                  │
│ Using temporary        │ 使用了临时表 ⚠️ 需优化              │
│ Using filesort         │ 使用了文件排序 ⚠️ 需优化            │
│ Using join buffer      │ 连接缓冲，可能缺少索引               │
│ Select tables optimized| 仅通过索引就能获取结果               │
└────────────────────────┴────────────────────────────────────┘
```

**key_len 计算**：

```sql
-- 用于判断联合索引使用了几个字段
-- 计算规则：
-- INT: 4 字节
-- BIGINT: 8 字节
-- VARCHAR(n): 3n + 2（UTF-8）或 4n + 2（UTF8MB4）
-- 可为 NULL: +1 字节

-- 示例：联合索引 (user_id BIGINT, status VARCHAR(20), create_time DATETIME)
-- user_id: 8
-- status: 20 * 4 + 2 = 82
-- create_time: 8
-- 全部使用: 8 + 82 + 8 = 98
```

#### 3.3 EXPLAIN 实战分析

```sql
-- 原始 SQL（问题 SQL）
EXPLAIN SELECT * FROM orders 
WHERE user_id = 10086 
  AND status = 'PAID' 
ORDER BY create_time DESC 
LIMIT 10;

-- 结果分析
+----+-------------+--------+------+---------------+-------------+---------+-------+------+-----------------------------+
| id | select_type | table  | type | possible_keys | key         | key_len | ref   | rows | Extra                       |
+----+-------------+--------+------+---------------+-------------+---------+-------+------+-----------------------------+
|  1 | SIMPLE      | orders | ref  | idx_user_id   | idx_user_id | 8       | const | 5000 | Using where; Using filesort |
+----+-------------+--------+------+---------------+-------------+---------+-------+------+-----------------------------+

问题：
1. type=ref 还可以接受
2. Using filesort ⚠️ 需要对 5000 条记录排序
3. 只用到了 idx_user_id 索引，status 和 create_time 没有利用
```

```sql
-- 优化方案：创建联合索引
CREATE INDEX idx_user_status_time ON orders(user_id, status, create_time);

-- 再次分析
EXPLAIN SELECT * FROM orders 
WHERE user_id = 10086 
  AND status = 'PAID' 
ORDER BY create_time DESC 
LIMIT 10;

+----+-------------+--------+------+----------------------+---------------------+---------+-------------+------+-----------------------+
| id | select_type | table  | type | possible_keys        | key                 | key_len | ref         | rows | Extra                 |
+----+-------------+--------+------+----------------------+---------------------+---------+-------------+------+-----------------------+
|  1 | SIMPLE      | orders | ref  | idx_user_status_time | idx_user_status_time| 90      | const,const |  100 | Using index condition |
+----+-------------+--------+------+----------------------+---------------------+---------+-------------+------+-----------------------+

优化效果：
1. rows 从 5000 降到 100
2. 消除了 Using filesort
3. 索引覆盖了 user_id + status，排序走索引
```

### 四、SHOW PROFILE 性能分析

```sql
-- 开启 profiling
SET profiling = 1;

-- 执行查询
SELECT * FROM orders WHERE user_id = 10086 LIMIT 100;

-- 查看所有查询
SHOW PROFILES;
+----------+------------+-----------------------------------------------------+
| Query_ID | Duration   | Query                                               |
+----------+------------+-----------------------------------------------------+
|        1 | 0.00123400 | SELECT * FROM orders WHERE user_id = 10086 LIMIT 100|
+----------+------------+-----------------------------------------------------+

-- 查看具体查询的各阶段耗时
SHOW PROFILE FOR QUERY 1;
+----------------------+----------+
| Status               | Duration |
+----------------------+----------+
| starting             | 0.000023 |
| checking permissions | 0.000005 |
| Opening tables       | 0.000015 |
| init                 | 0.000025 |
| System lock          | 0.000008 |
| optimizing           | 0.000012 |
| statistics           | 0.000035 |
| preparing            | 0.000010 |
| executing            | 0.000002 |
| Sending data         | 0.001050 |  ← 主要耗时在这里
| end                  | 0.000005 |
| query end            | 0.000006 |
| closing tables       | 0.000008 |
| freeing items        | 0.000018 |
| cleaning up          | 0.000012 |
+----------------------+----------+

-- 查看 CPU、IO 等详细信息
SHOW PROFILE CPU, BLOCK IO FOR QUERY 1;
```

### 五、常见慢查询场景及优化

#### 5.1 场景一：索引失效

```sql
-- ❌ 对索引列使用函数
SELECT * FROM orders WHERE DATE(create_time) = '2024-01-01';
-- ✅ 改为范围查询
SELECT * FROM orders 
WHERE create_time >= '2024-01-01 00:00:00' 
  AND create_time < '2024-01-02 00:00:00';

-- ❌ 隐式类型转换（phone 是 VARCHAR）
SELECT * FROM users WHERE phone = 13800138000;
-- ✅ 使用正确类型
SELECT * FROM users WHERE phone = '13800138000';

-- ❌ LIKE 左模糊
SELECT * FROM products WHERE name LIKE '%手机%';
-- ✅ 使用全文索引或搜索引擎
ALTER TABLE products ADD FULLTEXT INDEX ft_name(name);
SELECT * FROM products WHERE MATCH(name) AGAINST('手机');

-- ❌ OR 条件导致索引失效
SELECT * FROM orders WHERE user_id = 1 OR amount > 1000;
-- ✅ 改为 UNION
SELECT * FROM orders WHERE user_id = 1
UNION
SELECT * FROM orders WHERE amount > 1000;

-- ❌ 联合索引不满足最左前缀
-- 索引：(a, b, c)
SELECT * FROM t WHERE b = 1 AND c = 2;  -- 索引失效
-- ✅ 包含最左列
SELECT * FROM t WHERE a = 1 AND b = 2;  -- 使用索引
```

#### 5.2 场景二：深度分页

```sql
-- ❌ 深度分页性能差（需要扫描 100000 + 10 行）
SELECT * FROM orders ORDER BY id LIMIT 100000, 10;

-- ✅ 方案一：游标分页（推荐）
SELECT * FROM orders WHERE id > 100000 ORDER BY id LIMIT 10;

-- ✅ 方案二：延迟关联
SELECT o.* FROM orders o
INNER JOIN (
    SELECT id FROM orders ORDER BY id LIMIT 100000, 10
) AS tmp ON o.id = tmp.id;

-- ✅ 方案三：业务限制（不允许跳页太远）
-- 只允许上一页/下一页，不允许跳转到第 10000 页
```

**深度分页优化原理**：

```
原始查询 LIMIT 100000, 10：
┌─────────────────────────────────────────────────────┐
│  扫描 100000 行（丢弃）  │  返回 10 行              │
└─────────────────────────────────────────────────────┘
                           ↑
                     大量无效 IO

游标分页 WHERE id > 100000 LIMIT 10：
┌─────────────────────────────────────────────────────┐
│  直接定位到 id=100000   │  向后扫描 10 行并返回      │
└─────────────────────────────────────────────────────┘
                           ↑
                     索引直接定位，高效
```

#### 5.3 场景三：JOIN 优化

```sql
-- ❌ 被驱动表没有索引
SELECT * FROM orders o
LEFT JOIN order_items i ON o.id = i.order_id
WHERE o.user_id = 10086;

-- 检查执行计划
EXPLAIN SELECT ...
-- 如果 order_items 表 type=ALL，说明全表扫描

-- ✅ 为关联字段创建索引
CREATE INDEX idx_order_id ON order_items(order_id);

-- ✅ 小表驱动大表
-- MySQL 优化器通常会自动选择，但可以用 STRAIGHT_JOIN 强制
SELECT * FROM small_table s
STRAIGHT_JOIN large_table l ON s.id = l.ref_id;
```

**JOIN 优化原则**：

```
1. 被驱动表的关联字段必须有索引
2. 小表驱动大表
3. 避免过多表 JOIN（一般不超过 3 张）
4. 考虑是否可以拆分为多个简单查询
```

#### 5.4 场景四：SELECT * 问题

```sql
-- ❌ SELECT * 的问题
SELECT * FROM orders WHERE user_id = 10086;

-- 问题：
-- 1. 无法使用覆盖索引
-- 2. 传输大量不需要的数据
-- 3. 可能导致回表

-- ✅ 只查询需要的字段
SELECT id, order_no, amount, status FROM orders WHERE user_id = 10086;

-- ✅ 如果字段都在索引中，可以实现覆盖索引
-- 索引：(user_id, status, amount)
SELECT status, amount FROM orders WHERE user_id = 10086;
-- Extra: Using index（覆盖索引，无需回表）
```

#### 5.5 场景五：ORDER BY 优化

```sql
-- ❌ 排序字段没有索引，导致 filesort
SELECT * FROM orders WHERE status = 'PAID' ORDER BY create_time;

-- ✅ 方案一：联合索引覆盖 WHERE 和 ORDER BY
CREATE INDEX idx_status_time ON orders(status, create_time);

-- ✅ 方案二：只取主键排序后再回表
SELECT o.* FROM orders o
INNER JOIN (
    SELECT id FROM orders WHERE status = 'PAID' ORDER BY create_time LIMIT 100
) AS tmp ON o.id = tmp.id;
```

#### 5.6 场景六：COUNT 优化

```sql
-- ❌ COUNT(*) 全表扫描
SELECT COUNT(*) FROM orders WHERE status = 'PAID';

-- ✅ 方案一：添加索引
CREATE INDEX idx_status ON orders(status);

-- ✅ 方案二：预计算存储
-- 使用 Redis 或单独的统计表维护计数
-- 每次状态变更时更新计数

-- ✅ 方案三：近似值（如果业务允许）
SHOW TABLE STATUS LIKE 'orders';  -- 查看 Rows 近似值
```

### 六、实战案例分析

#### 案例一：订单列表查询优化

**问题描述**：用户订单列表接口响应超过 3 秒

```sql
-- 原始 SQL
SELECT * FROM orders 
WHERE user_id = 10086 
  AND status IN ('PAID', 'SHIPPED', 'DELIVERED')
  AND create_time > '2024-01-01'
ORDER BY create_time DESC
LIMIT 20;
```

**排查过程**：

```sql
-- 1. 查看执行计划
EXPLAIN SELECT ...;

+----+-------------+--------+------+---------------+-------------+---------+------+-------+-----------------------------+
| id | select_type | table  | type | possible_keys | key         | key_len | ref  | rows  | Extra                       |
+----+-------------+--------+------+---------------+-------------+---------+------+-------+-----------------------------+
|  1 | SIMPLE      | orders | ref  | idx_user_id   | idx_user_id | 8       | const| 15000 | Using where; Using filesort |
+----+-------------+--------+------+---------------+-------------+---------+------+-------+-----------------------------+

-- 问题分析：
-- 1. 只用到了 idx_user_id 索引
-- 2. 需要对 15000 行进行 filesort
-- 3. status 和 create_time 条件没有走索引
```

**优化方案**：

```sql
-- 创建联合索引
CREATE INDEX idx_user_time ON orders(user_id, create_time);

-- 优化后执行计划
+----+-------------+--------+-------+---------------+--------------+---------+------+------+-------------+
| id | select_type | table  | type  | possible_keys | key          | key_len | ref  | rows | Extra       |
+----+-------------+--------+-------+---------------+--------------+---------+------+------+-------------+
|  1 | SIMPLE      | orders | range | idx_user_time | idx_user_time| 16      | NULL |  200 | Using where |
+----+-------------+--------+-------+---------------+--------------+---------+------+------+-------------+

-- 优化效果：
-- 1. rows 从 15000 降到 200
-- 2. 消除了 filesort（ORDER BY create_time DESC 走索引）
-- 3. 响应时间从 3s 降到 50ms
```

#### 案例二：商品搜索优化

**问题描述**：商品搜索接口在高峰期超时

```sql
-- 原始 SQL
SELECT * FROM products 
WHERE category_id = 100 
  AND price BETWEEN 100 AND 500
  AND name LIKE '%手机%'
ORDER BY sales DESC
LIMIT 50;
```

**多维度优化**：

```sql
-- 1. 放弃 LIKE 模糊查询，改用 ES 全文搜索
-- 2. 创建合适的联合索引
CREATE INDEX idx_cat_price_sales ON products(category_id, price, sales);

-- 3. 改造 SQL（去掉 LIKE）
SELECT id FROM products 
WHERE category_id = 100 
  AND price BETWEEN 100 AND 500
ORDER BY sales DESC
LIMIT 50;

-- 4. 模糊搜索走 ES，取交集
-- 应用层：
-- Set<Long> esIds = esSearch("手机");
-- List<Product> dbResults = query(...);
-- 取交集

-- 5. 最终架构
┌──────────┐     ┌──────────────┐     ┌─────────────┐
│  应用层   │ ──→ │ Elasticsearch │ ──→ │   MySQL     │
│          │     │  (全文搜索)    │     │ (精确查询)   │
└──────────┘     └──────────────┘     └─────────────┘
```

### 七、SQL 优化 Checklist

```
□ 是否开启慢查询日志？
□ 是否用 EXPLAIN 分析执行计划？
□ type 是否至少是 range 级别？
□ 是否有 Using filesort 或 Using temporary？
□ 索引是否满足最左前缀原则？
□ 是否存在索引失效的情况？
□ 是否存在深度分页问题？
□ JOIN 的被驱动表是否有索引？
□ 是否可以用覆盖索引避免回表？
□ 是否有 SELECT * 可以优化？
□ 大表是否考虑分库分表？
□ 热点数据是否可以缓存？
```

### 八、总结

**面试回答模板**：

> "我在项目中遇到过多次 SQL 性能问题，我的排查流程是：
>
> **第一步：发现问题**
> - 通过慢查询日志、APM 监控发现慢 SQL
> - 使用 pt-query-digest 分析 TOP 慢查询
>
> **第二步：分析原因**
> - 用 EXPLAIN 查看执行计划，关注 type、rows、Extra
> - 检查索引是否失效、是否有 filesort
>
> **第三步：优化执行**
> - 索引优化：创建合适的联合索引
> - SQL 重写：避免函数、隐式转换、深度分页
> - 架构优化：读写分离、缓存热点数据
>
> **实际案例**：之前订单列表查询慢，通过 EXPLAIN 发现只用到了 user_id 索引，15000 行需要 filesort。创建 (user_id, create_time) 联合索引后，扫描行数降到 200，消除了排序，响应时间从 3 秒降到 50 毫秒。"
