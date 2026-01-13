---
title: Redis ZSet 底层实现深度解析
published: 2023-08-17
description: 深入剖析Redis有序集合的双编码机制、跳表源码实现与高性能设计原理
tags: [Redis, 数据结构, 跳表, 源码分析, 缓存]
category: Redis原理
draft: false
---

# Redis ZSet 底层实现深度解析

## 1. ZSet 概述

ZSet（Sorted Set，有序集合）是Redis中最强大的数据结构之一，它同时具备Set的去重特性和按分数排序的能力。每个元素关联一个double类型的分数（score），Redis通过分数对集合成员进行从小到大的排序。

**核心特性：**
- 成员唯一性：同一个ZSet中不会出现重复成员
- 分数可重复：不同成员可以拥有相同的分数
- 有序性：成员按分数从小到大排列，分数相同时按成员字典序排列
- O(logN)复杂度：支持高效的插入、删除、范围查询操作

## 2. 底层数据结构选型

ZSet根据数据规模采用两种不同的底层编码方式：

| 编码方式 | 触发条件 | 时间复杂度 | 空间效率 |
|---------|---------|-----------|---------|
| ziplist/listpack | 元素数量 < 128 且 所有元素长度 < 64字节 | O(N) | 高 |
| skiplist + dict | 超出上述任一条件 | O(logN) | 较低 |

配置参数（redis.conf）：
```
zset-max-ziplist-entries 128    # 最大元素数量
zset-max-ziplist-value 64       # 单个元素最大字节数
```

> **注意**：Redis 7.0+ 已将ziplist替换为listpack，但核心设计思想一致。

## 3. 紧凑型编码：Ziplist/Listpack

### 3.1 Ziplist 内存布局

```
+--------+------+------+-------+-------+-------+-------+-----+-------+------+
| zlbytes| zltail| zllen| entry1| entry2| entry3| entry4| ... | entryN| zlend|
+--------+------+------+-------+-------+-------+-------+-----+-------+------+
   4B       4B     2B     变长     变长     变长     变长          变长     1B
```

**头部字段说明：**
- `zlbytes`：整个ziplist占用的字节数
- `zltail`：最后一个entry的偏移量，支持O(1)反向遍历
- `zllen`：entry数量（小于65535时有效）
- `zlend`：结束标记，固定为0xFF

### 3.2 Entry 结构

```
+------------------+----------+--------+
| prevlen          | encoding | data   |
+------------------+----------+--------+
   1B or 5B          1-5B       变长
```

- `prevlen`：前一个entry的长度，用于反向遍历
  - 前一entry长度 < 254：使用1字节存储
  - 前一entry长度 ≥ 254：使用5字节（首字节0xFE + 4字节长度）
  
- `encoding`：编码类型和数据长度
  - 整数编码：支持int16/int32/int64等多种类型
  - 字符串编码：根据长度选择1/2/5字节头

### 3.3 ZSet 在 Ziplist 中的存储方式

ZSet使用相邻的两个entry存储一个元素：

```
+----------+---------+----------+---------+----------+---------+
|  member1 | score1  |  member2 | score2  |  member3 | score3  |
+----------+---------+----------+---------+----------+---------+
    entry     entry      entry     entry      entry     entry
```

**查找过程（O(N)）：**
1. 从头遍历ziplist
2. 每次跳过2个entry（member + score）
3. 比较member值，找到目标元素

### 3.4 连锁更新问题（Cascade Update）

当插入或删除元素导致某个entry的prevlen字段需要从1字节扩展到5字节时，可能触发连锁更新：

```
场景示例：
原始状态：[entry1(253B)] [entry2(prevlen=253, 1B)] [entry3] ...
插入后：  [entry1(253B)] [new_entry(255B)] [entry2(prevlen需要5B)] [entry3(prevlen也需更新)] ...
```

**最坏情况下时间复杂度为O(N²)**，但实际场景中极少发生。

### 3.5 Listpack（Redis 7.0+）

Listpack是ziplist的改进版本，主要改进：

```
+------------+------+---------+---------+-----+---------+------+
| total-bytes| nelem| entry1  | entry2  | ... | entryN  | end  |
+------------+------+---------+---------+-----+---------+------+
     4B        2B      变长      变长           变长       1B
```

Entry结构变化：
```
+----------+---------+-------------+
| encoding | data    | entry-len   |
+----------+---------+-------------+
   变长       变长        变长
```

**核心改进：** entry-len记录的是当前entry的长度而非前一个entry的长度，彻底消除了连锁更新问题。

## 4. 高性能编码：Skiplist + Dict

当数据量较大时，ZSet采用跳表（Skiplist）和哈希表（Dict）的组合结构：

```c
typedef struct zset {
    dict *dict;         // 哈希表：member -> score，O(1)查分数
    zskiplist *zsl;     // 跳表：按score排序，O(logN)范围查询
} zset;
```

### 4.1 为什么是双结构？

| 操作 | 仅Dict | 仅Skiplist | Dict + Skiplist |
|-----|--------|-----------|-----------------|
| ZSCORE | O(1) | O(logN) | O(1) ✓ |
| ZRANK | O(N) | O(logN) | O(logN) ✓ |
| ZRANGE | O(NlogN) | O(logN+M) | O(logN+M) ✓ |
| ZADD | O(1) | O(logN) | O(logN) |

**空间换时间**：虽然存储了两份数据指针，但显著提升了综合查询性能。

### 4.2 跳表（Skiplist）深度解析

#### 4.2.1 基本概念

跳表是一种概率型数据结构，通过多层链表实现O(logN)的查找效率：

```
Level 3:    HEAD ─────────────────────────────────────> 50 ──────────────> NULL
                                                         │
Level 2:    HEAD ─────────────> 20 ────────────────────> 50 ──────────────> NULL
                                 │                        │
Level 1:    HEAD ────> 10 ────> 20 ────> 30 ────> 40 ──> 50 ────> 60 ────> NULL
                        │        │        │        │      │        │
Level 0:    HEAD ────> 10 ────> 20 ────> 30 ────> 40 ──> 50 ────> 60 ────> NULL
```

#### 4.2.2 Redis 跳表节点定义

```c
typedef struct zskiplistNode {
    sds ele;                      // 成员值（SDS字符串）
    double score;                 // 分数
    struct zskiplistNode *backward;  // 后退指针（仅level[0]有效）
    struct zskiplistLevel {
        struct zskiplistNode *forward;  // 前进指针
        unsigned long span;              // 跨度（用于计算rank）
    } level[];                    // 柔性数组，层级动态分配
} zskiplistNode;

typedef struct zskiplist {
    struct zskiplistNode *header, *tail;  // 头尾指针
    unsigned long length;                  // 节点数量
    int level;                             // 当前最大层数
} zskiplist;
```

#### 4.2.3 层高随机算法

```c
#define ZSKIPLIST_MAXLEVEL 32    // 最大层数
#define ZSKIPLIST_P 0.25         // 晋升概率

int zslRandomLevel(void) {
    int level = 1;
    while ((random() & 0xFFFF) < (ZSKIPLIST_P * 0xFFFF))
        level += 1;
    return (level < ZSKIPLIST_MAXLEVEL) ? level : ZSKIPLIST_MAXLEVEL;
}
```

**概率分析：**
- P(level=1) = 1 - 0.25 = 75%
- P(level=2) = 0.25 × 0.75 = 18.75%
- P(level=3) = 0.25² × 0.75 ≈ 4.69%
- P(level=k) = 0.25^(k-1) × 0.75

**期望层高 = 1/(1-P) = 1.33**

#### 4.2.4 Span（跨度）的作用

Span记录两个节点之间跳过的节点数，用于快速计算ZRANK：

```
Level 2:    HEAD ──────────[span=2]──────────> 20 ────[span=3]────> 50
Level 1:    HEAD ──[span=1]──> 10 ──[span=1]──> 20 ──[span=1]──> 30 ...
```

计算节点50的rank：沿查找路径累加span = 2 + 3 = 5（0-indexed为4）

#### 4.2.5 核心操作实现

**插入操作（ZADD）：**

```c
zskiplistNode *zslInsert(zskiplist *zsl, double score, sds ele) {
    zskiplistNode *update[ZSKIPLIST_MAXLEVEL];  // 每层的前驱节点
    unsigned int rank[ZSKIPLIST_MAXLEVEL];       // 累计rank
    zskiplistNode *x = zsl->header;
    
    // 1. 从最高层开始，找到每层的插入位置
    for (int i = zsl->level-1; i >= 0; i--) {
        rank[i] = (i == zsl->level-1) ? 0 : rank[i+1];
        while (x->level[i].forward &&
               (x->level[i].forward->score < score ||
                (x->level[i].forward->score == score &&
                 sdscmp(x->level[i].forward->ele, ele) < 0))) {
            rank[i] += x->level[i].span;
            x = x->level[i].forward;
        }
        update[i] = x;
    }
    
    // 2. 随机生成层高
    int level = zslRandomLevel();
    if (level > zsl->level) {
        for (int i = zsl->level; i < level; i++) {
            rank[i] = 0;
            update[i] = zsl->header;
            update[i]->level[i].span = zsl->length;
        }
        zsl->level = level;
    }
    
    // 3. 创建新节点并插入各层
    x = zslCreateNode(level, score, ele);
    for (int i = 0; i < level; i++) {
        x->level[i].forward = update[i]->level[i].forward;
        update[i]->level[i].forward = x;
        x->level[i].span = update[i]->level[i].span - (rank[0] - rank[i]);
        update[i]->level[i].span = (rank[0] - rank[i]) + 1;
    }
    
    // 4. 更新未触及层的span
    for (int i = level; i < zsl->level; i++) {
        update[i]->level[i].span++;
    }
    
    // 5. 设置后退指针
    x->backward = (update[0] == zsl->header) ? NULL : update[0];
    if (x->level[0].forward)
        x->level[0].forward->backward = x;
    else
        zsl->tail = x;
    
    zsl->length++;
    return x;
}
```

**范围查询（ZRANGEBYSCORE）：**

```c
// 查找第一个score >= min的节点
zskiplistNode *zslFirstInRange(zskiplist *zsl, zrangespec *range) {
    zskiplistNode *x = zsl->header;
    
    // 从高层向下查找
    for (int i = zsl->level-1; i >= 0; i--) {
        while (x->level[i].forward &&
               !zslValueGteMin(x->level[i].forward->score, range))
            x = x->level[i].forward;
    }
    
    x = x->level[0].forward;
    if (x && !zslValueLteMax(x->score, range)) return NULL;
    return x;
}
```

### 4.3 Dict（哈希表）

#### 4.3.1 结构定义

```c
typedef struct dictEntry {
    void *key;              // member指针（与skiplist节点共享）
    union {
        void *val;
        uint64_t u64;
        int64_t s64;
        double d;           // 直接存储score
    } v;
    struct dictEntry *next; // 哈希冲突链表
} dictEntry;

typedef struct dictht {
    dictEntry **table;      // 哈希桶数组
    unsigned long size;     // 桶数量（2的幂次）
    unsigned long sizemask; // size - 1，用于取模
    unsigned long used;     // 已使用节点数
} dictht;

typedef struct dict {
    dictType *type;
    void *privdata;
    dictht ht[2];           // 两个哈希表，用于渐进式rehash
    long rehashidx;         // rehash进度（-1表示未进行）
    int16_t pauserehash;    // rehash暂停标记
} dict;
```

#### 4.3.2 渐进式Rehash

当负载因子（used/size）过高或过低时，触发rehash：

```
扩容条件：used / size > 1（无后台进程）或 used / size > 5（有后台进程）
缩容条件：used / size < 0.1
```

**渐进式迁移过程：**

```
Step 1: 分配新哈希表 ht[1]，大小为第一个 >= used*2 的 2^n

Step 2: 每次CRUD操作时，迁移ht[0]中rehashidx位置的桶到ht[1]
        rehashidx++

Step 3: 迁移期间，查找先查ht[0]再查ht[1]
        新增只写入ht[1]

Step 4: 迁移完成，释放ht[0]，ht[1]变为ht[0]，rehashidx = -1
```

## 5. 编码转换机制

### 5.1 转换触发条件

```c
// 检查是否需要从ziplist转换为skiplist
void zsetConvert(robj *zobj, int encoding) {
    if (zobj->encoding == OBJ_ENCODING_ZIPLIST) {
        if (encoding == OBJ_ENCODING_SKIPLIST) {
            zset *zs = zmalloc(sizeof(*zs));
            zs->dict = dictCreate(&zsetDictType, NULL);
            zs->zsl = zslCreate();
            
            // 遍历ziplist，插入到skiplist和dict
            unsigned char *zl = zobj->ptr;
            unsigned char *eptr = ziplistIndex(zl, 0);
            while (eptr != NULL) {
                sds ele = ziplistGetSds(eptr);
                eptr = ziplistNext(zl, eptr);
                double score = zzlGetScore(eptr);
                
                zskiplistNode *node = zslInsert(zs->zsl, score, ele);
                dictAdd(zs->dict, ele, &node->score);
                
                eptr = ziplistNext(zl, eptr);
            }
            
            zfree(zobj->ptr);
            zobj->ptr = zs;
            zobj->encoding = OBJ_ENCODING_SKIPLIST;
        }
    }
}
```

### 5.2 转换时机

```c
// 在ZADD等操作中检查
int zsetAdd(robj *zobj, double score, sds ele, ...) {
    if (zobj->encoding == OBJ_ENCODING_ZIPLIST) {
        // 检查元素长度
        if (sdslen(ele) > server.zset_max_ziplist_value)
            zsetConvert(zobj, OBJ_ENCODING_SKIPLIST);
        
        // 检查元素数量
        if (ziplistLen(zobj->ptr)/2 >= server.zset_max_ziplist_entries)
            zsetConvert(zobj, OBJ_ENCODING_SKIPLIST);
    }
    // ... 执行插入操作
}
```

**注意：Redis不支持从skiplist反向转换为ziplist**

## 6. 典型命令实现分析

### 6.1 ZADD

```
时间复杂度：O(logN)（skiplist）或 O(N)（ziplist）

执行流程：
1. 查找member是否存在（dict: O(1)）
2. 若存在且score变化：
   - 删除旧节点
   - 插入新位置
   - 更新dict中的score
3. 若不存在：
   - skiplist插入
   - dict插入
```

### 6.2 ZRANGE / ZREVRANGE

```
时间复杂度：O(logN + M)，M为返回元素数量

执行流程：
1. 通过累计span定位到第start个节点：O(logN)
2. 沿level[0]链表顺序遍历M个节点：O(M)
```

### 6.3 ZRANGEBYSCORE

```
时间复杂度：O(logN + M)

执行流程：
1. 从header开始，找到第一个score >= min的节点
2. 沿level[0]遍历，直到score > max
3. 可选LIMIT offset count参数优化
```

### 6.4 ZRANK

```
时间复杂度：O(logN)

执行流程：
1. 从header的最高层开始查找
2. 沿路径累加每层的span值
3. 最终累加值 - 1 即为rank
```

### 6.5 ZINCRBY

```
时间复杂度：O(logN)

执行流程：
1. 通过dict O(1)获取当前score
2. 计算新score
3. 若新score导致排序位置不变：直接更新dict
4. 若位置变化：删除旧节点，插入新位置
```

## 7. 跳表 vs 平衡树

Redis选择跳表而非红黑树/AVL树的原因：

| 对比维度 | 跳表 | 红黑树 |
|---------|------|-------|
| 实现复杂度 | 简单，约100行核心代码 | 复杂，旋转操作繁琐 |
| 范围查询 | 天然支持，O(logN+M) | 需要中序遍历，常数较大 |
| 内存局部性 | 较差（指针跳转） | 较好 |
| 并发友好 | CAS即可实现无锁 | 需要复杂的锁机制 |
| 空间开销 | 平均1.33个指针/节点 | 固定3个指针/节点 |
| 调试难度 | 直观，易于打印 | 抽象，难以可视化 |

> **Antirez（Redis作者）的解释：**
> "Skiplist和平衡树在这个场景下性能相当，但skiplist更简单，实现范围操作更直观。"

## 8. 性能优化建议

### 8.1 合理设置阈值

```bash
# 对于大量小元素场景，适当提高阈值
config set zset-max-ziplist-entries 256
config set zset-max-ziplist-value 128
```

### 8.2 避免热点Key

```python
# 分片策略示例
def get_shard_key(base_key, member):
    shard_id = hash(member) % 16
    return f"{base_key}:{shard_id}"

# 写入
redis.zadd(get_shard_key("leaderboard", user_id), {user_id: score})

# 全局TopN需要合并多个分片
```

### 8.3 批量操作

```bash
# 避免
ZADD key score1 member1
ZADD key score2 member2
ZADD key score3 member3

# 推荐
ZADD key score1 member1 score2 member2 score3 member3
```

### 8.4 ZRANGESTORE（Redis 6.2+）

```bash
# 将范围查询结果直接存入新key，减少网络往返
ZRANGESTORE dest src 0 99 BYSCORE
```

## 9. 实战案例

### 9.1 排行榜系统

```python
class Leaderboard:
    def __init__(self, redis_client, key):
        self.redis = redis_client
        self.key = key
    
    def update_score(self, user_id, score):
        """更新用户分数"""
        self.redis.zadd(self.key, {user_id: score})
    
    def increment_score(self, user_id, delta):
        """增加分数"""
        return self.redis.zincrby(self.key, delta, user_id)
    
    def get_rank(self, user_id):
        """获取排名（从0开始）"""
        rank = self.redis.zrevrank(self.key, user_id)
        return rank + 1 if rank is not None else None
    
    def get_top_n(self, n):
        """获取Top N"""
        return self.redis.zrevrange(self.key, 0, n-1, withscores=True)
    
    def get_around_me(self, user_id, count=5):
        """获取用户前后各count名"""
        rank = self.redis.zrevrank(self.key, user_id)
        if rank is None:
            return []
        start = max(0, rank - count)
        end = rank + count
        return self.redis.zrevrange(self.key, start, end, withscores=True)
```

### 9.2 延迟队列

```python
class DelayQueue:
    def __init__(self, redis_client, key):
        self.redis = redis_client
        self.key = key
    
    def add_task(self, task_id, execute_at):
        """添加延迟任务"""
        self.redis.zadd(self.key, {task_id: execute_at})
    
    def poll_tasks(self, batch_size=100):
        """获取到期任务"""
        now = time.time()
        # 原子操作：获取并删除
        with self.redis.pipeline() as pipe:
            while True:
                try:
                    pipe.watch(self.key)
                    tasks = pipe.zrangebyscore(
                        self.key, 0, now, 
                        start=0, num=batch_size
                    )
                    if not tasks:
                        pipe.unwatch()
                        return []
                    pipe.multi()
                    pipe.zrem(self.key, *tasks)
                    pipe.execute()
                    return tasks
                except WatchError:
                    continue
```

### 9.3 滑动窗口限流

```python
class SlidingWindowRateLimiter:
    def __init__(self, redis_client, window_seconds, max_requests):
        self.redis = redis_client
        self.window = window_seconds
        self.max_requests = max_requests
    
    def is_allowed(self, key):
        now = time.time()
        window_start = now - self.window
        
        pipe = self.redis.pipeline()
        # 移除窗口外的记录
        pipe.zremrangebyscore(key, 0, window_start)
        # 统计窗口内请求数
        pipe.zcard(key)
        # 添加当前请求
        pipe.zadd(key, {str(uuid.uuid4()): now})
        # 设置过期时间
        pipe.expire(key, self.window + 1)
        
        results = pipe.execute()
        current_count = results[1]
        
        return current_count < self.max_requests
```

## 10. 面试高频问题

### Q1: 为什么ZSet同时使用skiplist和dict？

**答：** 为了在不同操作场景下都能获得最优时间复杂度。dict提供O(1)的成员查找和分数获取（ZSCORE），skiplist提供O(logN)的有序操作（ZRANK、ZRANGE）。两者共享成员的SDS指针，空间开销可接受。

### Q2: 跳表的查找过程？

**答：** 从最高层header开始，在每层尽可能向右移动，直到下一个节点的score大于目标值或为NULL时下沉一层。重复此过程直到第0层找到目标节点或确认不存在。

### Q3: 为什么选择0.25作为晋升概率？

**答：** 0.25是空间和时间的平衡点。概率越高，层数越多，查找越快但空间开销越大。0.25时平均每个节点约1.33个指针，与二叉树的2个指针相比更省空间，同时保持O(logN)复杂度。

### Q4: ziplist的连锁更新是什么？

**答：** ziplist中每个entry存储前一个entry的长度（prevlen）。当插入/删除导致某entry长度从253字节变为254+字节时，其后继entry的prevlen需要从1字节扩展到5字节，这可能级联影响后续所有entry，最坏O(N²)。Listpack通过改变设计解决了此问题。

### Q5: ZSet如何保证分数相同时的稳定排序？

**答：** 当score相同时，Redis使用member的字典序（memcmp）作为第二排序键。这保证了排序的确定性和一致性。

## 11. 总结

Redis ZSet通过精心设计的双编码策略（ziplist/listpack + skiplist/dict）实现了高效的有序集合操作。小数据量时使用紧凑型编码节省内存，大数据量时切换为skiplist+dict组合提供O(logN)的综合性能。理解这些底层实现对于高并发场景下的性能调优和问题排查至关重要。

---

*作者：技术深度解析系列*  
*最后更新：2025年*
