---
title: MySQL深度分页问题与5种优化方案详解
published: 2021-07-25T11:20:00Z
description: 系统讲解MySQL深度分页的性能问题、原理分析和5种优化方案，包含延迟关联、子查询、标签记录法等实战技巧
tags: [MySQL, 数据库, 性能优化, 分页查询, 索引优化]
category: 数据库原理
draft: false
---

# MySQL深度分页有什么解决方案？

深度分页（Deep Pagination）是MySQL性能优化中的经典问题。当 `LIMIT offset` 很大时，查询会变得非常慢。本文将深入分析问题原因，并提供5种优化方案。

## 一、什么是深度分页问题

### 1. 问题现象

```sql
-- 第1页：很快
SELECT * FROM user ORDER BY id LIMIT 0, 10;     -- 耗时: 0.01s

-- 第10页：很快
SELECT * FROM user ORDER BY id LIMIT 90, 10;    -- 耗时: 0.01s

-- 第10000页：开始变慢
SELECT * FROM user ORDER BY id LIMIT 99990, 10; -- 耗时: 0.5s

-- 第100万页：非常慢 ❌
SELECT * FROM user ORDER BY id LIMIT 9999990, 10; -- 耗时: 5s+
```

### 2. 性能对比

假设表中有1000万条数据：

| 查询 | 偏移量 | 耗时 |
|------|--------|------|
| LIMIT 0, 10 | 0 | 0.01s |
| LIMIT 100, 10 | 100 | 0.01s |
| LIMIT 10000, 10 | 10000 | 0.1s |
| LIMIT 100000, 10 | 100000 | 0.5s |
| LIMIT 1000000, 10 | 1000000 | 3s |
| LIMIT 5000000, 10 | 5000000 | 15s ⚠️ |

**结论：** offset越大，查询越慢，呈线性增长。

## 二、深度分页慢的根本原因

### 1. MySQL执行流程分析

```sql
SELECT * FROM user ORDER BY id LIMIT 1000000, 10;
```

**执行步骤：**
```
1. 根据ORDER BY排序，扫描索引
2. 从头开始遍历，取出前 1000010 条记录  ← 关键问题
3. 丢弃前 1000000 条记录
4. 返回最后 10 条记录
```

**问题：** 虽然只需要10条数据，但MySQL必须先取出1000010条，然后丢弃前1000000条。

### 2. 深入原理

#### （1）索引扫描过程

```
索引树（id索引）：
         [5000]
        /      \
    [2500]      [7500]
    /    \      /    \
 [1250] [3750][6250][8750]
   ...

扫描流程（LIMIT 1000000, 10）：
1. 从最左叶子节点开始
2. 遍历: id=1 → id=2 → id=3 → ... → id=1000000  ← 扫描100万次
3. 每次扫描都需要回表（如果SELECT *）
4. 丢弃前100万条，只返回最后10条
```

#### （2）回表成本

```sql
-- 如果是 SELECT *，需要回表
SELECT * FROM user ORDER BY id LIMIT 1000000, 10;

-- 过程：
-- 1. 扫描索引，找到100万个主键id
-- 2. 根据主键id回表，查询100万次  ← 耗时主要在这里
-- 3. 丢弃前100万条
-- 4. 返回10条
```

**回表代价：**
- 二级索引 → 主键索引 → 完整数据行
- 随机IO访问，效率低下
- offset越大，回表次数越多

### 3. 慢查询日志分析

```sql
# Time: 2024-01-10T10:30:00.123456Z
# Query_time: 5.234567  Lock_time: 0.000123  Rows_sent: 10  Rows_examined: 1000010
SELECT * FROM user ORDER BY id LIMIT 1000000, 10;
```

**关键指标：**
- `Query_time: 5.23s` - 查询耗时5秒
- `Rows_sent: 10` - 只返回10条
- `Rows_examined: 1000010` - 实际扫描了100万条 ⚠️

## 三、5种优化方案

### 方案1：子查询优化（延迟关联）⭐⭐⭐⭐⭐

#### 原理

先在索引上完成分页，再根据主键ID回表查询完整数据。

#### 实现

```sql
-- ❌ 原始慢查询
SELECT * FROM user ORDER BY id LIMIT 1000000, 10;
-- 耗时: 5s, 扫描100万行并回表

-- ✅ 优化后（延迟关联）
SELECT * FROM user
WHERE id >= (
    SELECT id FROM user ORDER BY id LIMIT 1000000, 1
)
ORDER BY id LIMIT 10;
-- 耗时: 0.5s, 只回表10次
```

**性能对比：**
```sql
-- 方法1：直接LIMIT（慢）
EXPLAIN SELECT * FROM user ORDER BY id LIMIT 1000000, 10;
+----+-------+------+------+---------+-------+----------+
| id | type  | key  | rows | Extra                       |
+----+-------+------+------+---------+-------+----------+
| 1  | index | id   | 1000010 | Using index; Using filesort |
+----+-------+------+------+---------+-------+----------+

-- 方法2：子查询优化（快）
EXPLAIN SELECT * FROM user
WHERE id >= (SELECT id FROM user ORDER BY id LIMIT 1000000, 1)
ORDER BY id LIMIT 10;
+----+-------+------+------+---------+-------+
| id | type  | key  | rows | Extra         |
+----+-------+------+------+---------+-------+
| 1  | range | id   | 10   | Using where   |  ← 只扫描10行
| 2  | index | id   | 1000001 | Using index |  ← 只扫描索引
+----+-------+------+------+---------+-------+
```

#### 更通用的写法

```sql
-- 适用于任意ORDER BY字段
SELECT a.* FROM user a
INNER JOIN (
    SELECT id FROM user ORDER BY create_time LIMIT 1000000, 10
) b ON a.id = b.id;
```

**优点：**
- ✅ 性能提升明显（5-10倍）
- ✅ 适用于大部分场景
- ✅ 改动最小

**缺点：**
- ❌ 仍需扫描100万条索引记录
- ❌ offset过大时依然较慢（如500万）

### 方案2：标签记录法（游标分页）⭐⭐⭐⭐⭐

#### 原理

记录上一页的最后一条记录ID，下一页从这个ID开始查询。

#### 实现

```sql
-- 第1页（首次查询）
SELECT * FROM user ORDER BY id LIMIT 10;
-- 返回: id=1~10, 记录最后一个id=10

-- 第2页（基于上一页最后的id）
SELECT * FROM user WHERE id > 10 ORDER BY id LIMIT 10;
-- 返回: id=11~20, 记录最后一个id=20

-- 第3页
SELECT * FROM user WHERE id > 20 ORDER BY id LIMIT 10;
-- 返回: id=21~30

-- 第N页
SELECT * FROM user WHERE id > {last_id} ORDER BY id LIMIT 10;
```

#### 性能对比

```sql
-- ❌ 传统分页：offset=1000000
SELECT * FROM user ORDER BY id LIMIT 1000000, 10;
-- 耗时: 5s

-- ✅ 标签记录法：基于last_id
SELECT * FROM user WHERE id > 1000000 ORDER BY id LIMIT 10;
-- 耗时: 0.01s  ← 快500倍！
```

#### 执行计划对比

```sql
EXPLAIN SELECT * FROM user WHERE id > 1000000 ORDER BY id LIMIT 10;
+----+-------+------+------+-------+-------------+
| id | type  | key  | rows | Extra               |
+----+-------+------+------+-------+-------------+
| 1  | range | id   | 10   | Using where; Using index |  ← 只扫描10行
+----+-------+------+------+-------+-------------+
```

#### 复杂排序的处理

```sql
-- 按创建时间排序 + ID保证唯一性
-- 第1页
SELECT * FROM user ORDER BY create_time, id LIMIT 10;
-- 返回最后一条: create_time='2024-01-10 10:30:00', id=12345

-- 第2页
SELECT * FROM user 
WHERE (create_time > '2024-01-10 10:30:00') 
   OR (create_time = '2024-01-10 10:30:00' AND id > 12345)
ORDER BY create_time, id LIMIT 10;

-- 需要建立联合索引
CREATE INDEX idx_create_time_id ON user(create_time, id);
```

#### 前端实现

```javascript
// Vue/React 示例
export default {
  data() {
    return {
      list: [],
      lastId: 0,  // 记录最后一个ID
      hasMore: true
    }
  },
  methods: {
    async loadMore() {
      const res = await axios.get('/api/users', {
        params: { lastId: this.lastId, pageSize: 10 }
      });
      
      this.list.push(...res.data.list);
      
      if (res.data.list.length > 0) {
        // 记录最后一个ID
        this.lastId = res.data.list[res.data.list.length - 1].id;
      }
      
      this.hasMore = res.data.hasMore;
    }
  }
}
```

**优点：**
- ✅ 性能极佳，恒定时间复杂度O(1)
- ✅ 适合无限滚动、瀑布流
- ✅ 移动端App常用

**缺点：**
- ❌ 无法跳页（不能直接跳到第100页）
- ❌ 不适合需要跳页的场景
- ❌ 前端需要配合改造

### 方案3：使用覆盖索引优化⭐⭐⭐⭐

#### 原理

只查询索引列，避免回表。

#### 实现

```sql
-- ❌ 原始查询：需要回表
SELECT id, name, age, email, phone, address 
FROM user ORDER BY id LIMIT 1000000, 10;
-- 耗时: 5s, 回表100万次

-- ✅ 优化1：只查询索引字段
SELECT id FROM user ORDER BY id LIMIT 1000000, 10;
-- 耗时: 0.5s, 无需回表，直接在索引上完成

-- ✅ 优化2：先查ID，再IN查询（推荐）
SELECT * FROM user WHERE id IN (
    SELECT id FROM user ORDER BY id LIMIT 1000000, 10
);
-- 耗时: 0.6s, 只回表10次
```

#### 建立覆盖索引

```sql
-- 如果经常查询 id, name, age
CREATE INDEX idx_id_name_age ON user(id, name, age);

-- 查询走覆盖索引
SELECT id, name, age FROM user ORDER BY id LIMIT 1000000, 10;
-- 无需回表，性能提升显著
```

#### 执行计划

```sql
EXPLAIN SELECT id FROM user ORDER BY id LIMIT 1000000, 10;
+----+-------+------+----------+-------------+
| id | type  | key  | rows     | Extra       |
+----+-------+------+----------+-------------+
| 1  | index | id   | 1000010  | Using index |  ← Using index表示覆盖索引
+----+-------+------+----------+-------------+
```

**优点：**
- ✅ 避免回表，性能提升明显
- ✅ 适合只查部分字段的场景

**缺点：**
- ❌ 仍需扫描100万条索引
- ❌ 只能查询索引字段

### 方案4：分段查询⭐⭐⭐

#### 原理

将大偏移量拆分成多个小偏移量查询。

#### 实现

```sql
-- ❌ 原始查询：一次查询offset=1000000
SELECT * FROM user ORDER BY id LIMIT 1000000, 10;

-- ✅ 分段查询：分10次，每次offset=100000
-- 第1段: LIMIT 0, 100000 取最大ID
-- 第2段: WHERE id > max_id_1 LIMIT 0, 100000 取最大ID
-- ...
-- 第10段: WHERE id > max_id_9 LIMIT 0, 10

-- 伪代码
SET @last_id = 0;
SET @batch_size = 100000;
SET @total_offset = 1000000;

-- 循环10次
WHILE @last_id < @total_offset DO
    SELECT @last_id := id FROM user 
    WHERE id > @last_id 
    ORDER BY id LIMIT @batch_size, 1;
END WHILE;

-- 最后查询10条
SELECT * FROM user WHERE id > @last_id ORDER BY id LIMIT 10;
```

**优点：**
- ✅ 将大问题拆分成小问题
- ✅ 每次查询更快

**缺点：**
- ❌ 实现复杂
- ❌ 需要多次查询
- ❌ 实际应用较少

### 方案5：ES/NoSQL方案⭐⭐⭐⭐

#### 原理

使用专门的搜索引擎或NoSQL数据库处理深度分页。

#### 方案A：Elasticsearch

```json
// ES的 search_after 分页
POST /user/_search
{
  "size": 10,
  "query": { "match_all": {} },
  "sort": [
    { "id": "asc" }
  ]
}

// 返回结果
{
  "hits": {
    "hits": [
      {
        "_source": { "id": 1, "name": "Alice" },
        "sort": [1]  ← 记录sort值
      },
      ...
    ]
  }
}

// 下一页：使用search_after
POST /user/_search
{
  "size": 10,
  "query": { "match_all": {} },
  "sort": [{ "id": "asc" }],
  "search_after": [10]  ← 从上一页的sort值开始
}
```

**优点：**
- ✅ 性能极佳，支持亿级数据
- ✅ 支持复杂搜索
- ✅ 深度分页性能稳定

**缺点：**
- ❌ 需要额外维护ES集群
- ❌ 数据同步成本

#### 方案B：MongoDB

```javascript
// MongoDB的skip性能也不好，建议用cursor分页
// 类似MySQL的标签记录法
db.users.find({ _id: { $gt: lastId } })
  .sort({ _id: 1 })
  .limit(10);
```

#### 方案C：数据冷热分离

```sql
-- 热数据表（最近3个月，索引效率高）
CREATE TABLE user_hot (
    id BIGINT PRIMARY KEY,
    ...
) ENGINE=InnoDB;

-- 冷数据表（3个月以前，使用ES或归档）
-- 深度分页的需求，引导用户去搜索功能（ES）
```

**适用场景：**
- 数据量亿级以上
- 对搜索性能要求极高
- 已有ES/NoSQL基础设施

## 四、方案对比与选择

### 1. 性能对比（1000万数据，offset=100万）

| 方案 | 耗时 | 索引扫描 | 回表次数 | 难度 | 推荐指数 |
|------|------|---------|---------|------|---------|
| 原始LIMIT | 5000ms | 100万 | 100万 | ⭐ | ⭐ |
| 子查询优化 | 500ms | 100万 | 10 | ⭐⭐ | ⭐⭐⭐⭐⭐ |
| 标签记录法 | 10ms | 10 | 10 | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| 覆盖索引 | 400ms | 100万 | 0 | ⭐⭐ | ⭐⭐⭐⭐ |
| ES方案 | 20ms | - | - | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ |

### 2. 场景选择指南

```
场景1：后台管理系统，需要跳页
├─ 数据量 < 100万：原始LIMIT可以接受
├─ 数据量 100万-1000万：子查询优化
└─ 数据量 > 1000万：限制最大页数 + ES搜索

场景2：移动App/小程序，下拉刷新
└─ 标签记录法（游标分页）⭐⭐⭐⭐⭐

场景3：列表页，只显示前100页
└─ 子查询优化 + 限制最大offset

场景4：搜索功能，复杂查询
└─ Elasticsearch

场景5：只查询部分字段
└─ 覆盖索引优化
```

## 五、实战案例

### 案例1：订单列表优化（电商系统）

#### 问题

```sql
-- 原始SQL：商家查看订单列表
SELECT * FROM orders 
WHERE shop_id = 12345 
ORDER BY create_time DESC 
LIMIT 100000, 20;
-- 耗时: 3s+
```

#### 优化方案：子查询 + 覆盖索引

```sql
-- 1. 创建覆盖索引
CREATE INDEX idx_shop_time_id ON orders(shop_id, create_time, order_id);

-- 2. 优化查询
SELECT o.* FROM orders o
INNER JOIN (
    SELECT order_id FROM orders
    WHERE shop_id = 12345
    ORDER BY create_time DESC
    LIMIT 100000, 20
) t ON o.order_id = t.order_id;
-- 耗时: 0.3s
```

#### 代码实现

```java
@Service
public class OrderService {
    
    public PageResult<Order> queryOrders(Long shopId, int pageNo, int pageSize) {
        // 限制最大页数，避免深度分页
        if (pageNo > 1000) {
            throw new BusinessException("最多只能查看前1000页，请使用搜索功能");
        }
        
        int offset = (pageNo - 1) * pageSize;
        
        // 子查询优化
        String sql = """
            SELECT o.* FROM orders o
            INNER JOIN (
                SELECT order_id FROM orders
                WHERE shop_id = ?
                ORDER BY create_time DESC
                LIMIT ?, ?
            ) t ON o.order_id = t.order_id
            """;
        
        List<Order> orders = jdbcTemplate.query(sql, 
            new Object[]{shopId, offset, pageSize},
            new BeanPropertyRowMapper<>(Order.class));
        
        return new PageResult<>(orders, pageNo, pageSize);
    }
}
```

### 案例2：社交App动态列表（标签记录法）

#### 前端实现

```vue
<template>
  <div class="feed-list">
    <div v-for="post in posts" :key="post.id" class="feed-item">
      {{ post.content }}
    </div>
    
    <div v-if="loading" class="loading">加载中...</div>
    <div v-if="!hasMore" class="no-more">没有更多了</div>
  </div>
</template>

<script>
export default {
  data() {
    return {
      posts: [],
      lastId: 0,        // 游标：最后一条的ID
      hasMore: true,
      loading: false
    }
  },
  
  mounted() {
    this.loadPosts();
    // 滚动到底部时加载更多
    window.addEventListener('scroll', this.handleScroll);
  },
  
  methods: {
    async loadPosts() {
      if (this.loading || !this.hasMore) return;
      
      this.loading = true;
      try {
        const res = await axios.get('/api/posts', {
          params: {
            lastId: this.lastId,
            pageSize: 20
          }
        });
        
        const newPosts = res.data.list;
        this.posts.push(...newPosts);
        
        // 更新游标
        if (newPosts.length > 0) {
          this.lastId = newPosts[newPosts.length - 1].id;
        }
        
        // 判断是否还有更多
        this.hasMore = newPosts.length === 20;
      } finally {
        this.loading = false;
      }
    },
    
    handleScroll() {
      const scrollTop = window.pageYOffset;
      const windowHeight = window.innerHeight;
      const documentHeight = document.documentElement.scrollHeight;
      
      // 距离底部100px时触发加载
      if (scrollTop + windowHeight >= documentHeight - 100) {
        this.loadPosts();
      }
    }
  }
}
</script>
```

#### 后端实现

```java
@RestController
@RequestMapping("/api/posts")
public class PostController {
    
    @GetMapping
    public Result<List<Post>> getPosts(
            @RequestParam(defaultValue = "0") Long lastId,
            @RequestParam(defaultValue = "20") Integer pageSize) {
        
        // 基于lastId查询
        String sql = """
            SELECT * FROM posts
            WHERE id > ?
            ORDER BY id DESC
            LIMIT ?
            """;
        
        List<Post> posts = jdbcTemplate.query(sql, 
            new Object[]{lastId, pageSize},
            new BeanPropertyRowMapper<>(Post.class));
        
        return Result.success(posts);
    }
}
```

### 案例3：数据分析平台（限制+引导）

#### 策略：限制最大页数 + ES搜索

```java
@Service
public class DataAnalysisService {
    
    @Resource
    private ElasticsearchRestTemplate esTemplate;
    
    public PageResult<DataRecord> queryData(QueryParam param) {
        // 策略1：限制最大页数
        if (param.getPageNo() > 100) {
            // 引导用户使用ES搜索
            throw new BusinessException(
                "数据量过大，请使用高级搜索功能或导出数据"
            );
        }
        
        // 策略2：小于100页，使用MySQL
        if (param.getPageNo() <= 100) {
            return queryFromMySQL(param);
        }
        
        // 策略3：大于100页（如果允许），使用ES
        return queryFromES(param);
    }
    
    private PageResult<DataRecord> queryFromMySQL(QueryParam param) {
        // 子查询优化
        String sql = """
            SELECT d.* FROM data_records d
            INNER JOIN (
                SELECT id FROM data_records
                WHERE category = ?
                ORDER BY create_time DESC
                LIMIT ?, ?
            ) t ON d.id = t.id
            """;
        // ...
    }
    
    private PageResult<DataRecord> queryFromES(QueryParam param) {
        // ES的search_after分页
        SearchRequest request = new SearchRequest("data_records");
        // ...
    }
}
```

## 六、最佳实践总结

### 1. 通用优化建议

```sql
-- ✅ 推荐做法
-- 1. 限制最大offset
SELECT * FROM user ORDER BY id LIMIT 10000, 10;  -- offset < 10000

-- 2. 提示用户使用搜索
if (offset > 10000) {
    return "请使用搜索功能精确查找";
}

-- 3. 建立合适的索引
CREATE INDEX idx_create_time ON user(create_time);

-- 4. 只查询必要的字段
SELECT id, name FROM user ORDER BY id LIMIT 1000000, 10;  -- 而不是SELECT *

-- ❌ 避免做法
-- 1. 无限制的深度分页
SELECT * FROM user ORDER BY id LIMIT 9999999, 10;  -- ❌

-- 2. 在非索引字段上排序
SELECT * FROM user ORDER BY random_column LIMIT 1000000, 10;  -- ❌

-- 3. 使用SELECT *
SELECT * FROM user ...  -- ❌
```

### 2. 产品设计建议

```
1. 移动端：
   ├─ 使用下拉刷新/加载更多（标签记录法）
   └─ 不要设计跳页功能

2. 后台管理：
   ├─ 限制最大100页
   ├─ 提供搜索/筛选功能
   └─ 提供导出功能（导出走异步任务）

3. 数据分析：
   ├─ 引导用户添加筛选条件
   ├─ 使用ES进行复杂查询
   └─ 超大数据集走离线计算

4. 用户体验：
   ├─ 显示"前100页"提示
   ├─ 超过限制时，引导使用搜索
   └─ 提供快速定位功能（比如按日期跳转）
```

### 3. 监控告警

```java
@Aspect
@Component
public class PageQueryMonitor {
    
    @Around("execution(* com.example..*.query*(..))")
    public Object monitor(ProceedingJoinPoint pjp) throws Throwable {
        Object[] args = pjp.getArgs();
        
        // 检查分页参数
        for (Object arg : args) {
            if (arg instanceof PageParam) {
                PageParam param = (PageParam) arg;
                int offset = (param.getPageNo() - 1) * param.getPageSize();
                
                // offset > 10000，记录日志并告警
                if (offset > 10000) {
                    log.warn("⚠️ 深度分页告警 - offset: {}, method: {}", 
                            offset, pjp.getSignature());
                    
                    // 发送监控告警
                    alertService.send("深度分页", 
                        "offset=" + offset + ", method=" + pjp.getSignature());
                }
            }
        }
        
        return pjp.proceed();
    }
}
```

### 4. 数据库参数优化

```sql
-- MySQL配置优化
[mysqld]
# 增加排序缓冲区
sort_buffer_size = 2M

# 增加索引缓冲区
key_buffer_size = 256M

# 增加InnoDB缓冲池
innodb_buffer_pool_size = 4G
```

## 七、快速决策树

```
需要深度分页？
  ├─ 是否需要跳页？
  │   ├─ 是 → 数据量多大？
  │   │   ├─ < 100万 → 子查询优化
  │   │   ├─ 100万-1000万 → 子查询 + 限制最大页数
  │   │   └─ > 1000万 → ES + 限制MySQL只查前100页
  │   │
  │   └─ 否（移动端/瀑布流）→ 标签记录法 ⭐
  │
  └─ 只查询少量字段？
      ├─ 是 → 覆盖索引
      └─ 否 → 子查询优化
```

## 八、总结

### 核心要点

1. **深度分页慢的根本原因**：需要扫描并丢弃大量数据
2. **最优方案**：标签记录法（游标分页）
3. **通用方案**：子查询优化（延迟关联）
4. **产品策略**：限制最大页数 + 引导搜索

### 记忆口诀

```
深度分页别硬抗，子查询来帮忙
游标分页最理想，无限滚动响应快
覆盖索引省回表，ES搜索应对大
限制页数是王道，引导搜索体验好
```

### 性能优化优先级

```
1. 产品设计：避免深度分页需求（最重要）
2. 标签记录法：适合90%的移动端场景
3. 子查询优化：适合需要跳页的场景
4. 限制最大页数：超过限制引导用户搜索
5. ES方案：超大数据量、复杂搜索场景
```

掌握这些方案，你就能灵活应对各种分页场景，既能满足业务需求，又能保证系统性能！
