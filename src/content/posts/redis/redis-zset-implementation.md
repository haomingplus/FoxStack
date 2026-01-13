---
title: Redis ZSet 底层实现深度解析11
published: 2023-08-17
description: 深入剖析Redis有序集合的双编码机制、跳表源码实现与高性能设计原理
tags: [Redis, 数据结构, 跳表, 源码分析, 缓存]
category: Redis原理
draft: false
---

## 目录

- [1. 概述](#1-概述)
- [2. ZSet 数据结构](#2-zset-数据结构)
- [3. 压缩列表（ziplist）实现](#3-压缩列表ziplist实现)
- [4. 跳表（skiplist）实现](#4-跳表skiplist实现)
- [5. 字典（dict）辅助结构](#5-字典dict辅助结构)
- [6. 编码转换](#6-编码转换)
- [7. 为什么选择跳表](#7-为什么选择跳表)
- [8. 核心操作源码分析](#8-核心操作源码分析)
- [9. 时间复杂度分析](#9-时间复杂度分析)
- [10. 常见面试题](#10-常见面试题)

---

## 1. 概述

### 1.1 ZSet 是什么

ZSet（Sorted Set）是 Redis 提供的有序集合类型，具有以下特性：

- **唯一性**：每个元素唯一（通过 member 标识）
- **有序性**：每个元素关联一个 score，按 score 排序
- **分数可重复**：不同 member 可以有相同 score

### 1.2 应用场景

| 场景 | 描述 |
|------|------|
| 排行榜 | 游戏排名、热度排行 |
| 延迟队列 | score 作为执行时间戳 |
| 范围查询 | 按分数区间查找 |
| 优先级队列 | score 表示优先级 |

---

## 2. ZSet 数据结构

### 2.1 整体架构

```
┌─────────────────────────────────────────────────────────────┐
│                         ZSet                                 │
├─────────────────────────────────────────────────────────────┤
│  ┌─────────────────┐         ┌─────────────────────────┐   │
│  │   ziplist       │   OR    │   skiplist + dict        │   │
│  │   (元素少时)     │         │   (元素多时)              │   │
│  └─────────────────┘         └─────────────────────────┘   │
│                                                              │
│  ziplist: 连续内存块，member 和 score 紧密存储               │
│  skiplist: 按 score 有序存储，支持范围查询                   │
│  dict: member -> score 映射，O(1) 查找                       │
└─────────────────────────────────────────────────────────────┘
```

### 2.2 源码定义

```c
/* server.h */
typedef struct zset {
    dict *dict;          // member -> score 的映射
    zskiplist *zsl;      // 跳表结构，按 score 排序
} zset;

/* ziplist 编码时没有对应的 C 结构，直接使用 robj */
/* OBJ_ENCODING_ZIPLIST */
```

---

## 3. 压缩列表（ziplist）实现

### 3.1 ziplist 存储格式

当元素较少时，ZSet 使用 ziplist 存储，节省内存。

```
┌──────┬────────────┬────────────┬────────────┬─────┬────────────┬────────────┬────────────┬─────┐
│zlbytes│zltail│zlend│ entry1    │ entry2    │ ... │ entryN    │zlend│
│      │      │     │ (member1) │ (score1)  │     │ (memberN) │ (scoreN)  │     │
└──────┴──────┴─────┴───────────┴───────────┴─────┴───────────┴───────────┴─────┘
```

**存储规则**：
- member 和 score 相邻存储
- 按 score 从小到大排序
- score 使用 double 类型存储

### 3.2 ziplist 使用条件

```c
/* t_zset.c */
#define ZSET_MAX_ZIPLIST_VALUE 64       // 单个元素最大字节数
#define ZSET_MAX_ZIPLIST_ENTRIES 128    // 最大元素数量

/* Redis 使用 ziplist 的条件 */
/* 1. 有序集合保存的元素数量小于 128 个 */
/* 2. 有序集合保存的所有元素成员的长度都小于 64 字节 */
```

配置参数（redis.conf）：
```conf
zset-max-ziplist-entries 128
zset-max-ziplist-value 64
```

### 3.3 ziplist 优缺点

| 优点 | 缺点 |
|------|------|
| 内存紧凑，无指针开销 | 插入/删除需要内存重分配 |
| CPU 缓存友好 | 元素数量多时性能下降 |
| 适合小数据集 | 不支持范围查询优化 |

---

## 4. 跳表（skiplist）实现

### 4.1 跳表概述

跳表（Skip List）是 William Pugh 在 1990 年提出的一种数据结构，用于替代平衡树。

```
                   Level 4:    1 → 6
                   Level 3:    1 → 4 → 6
                   Level 4:    1 → 3 → 4 → 6
                   Level 1:    1 → 2 → 3 → 4 → 5 → 6 → 7
                   Bottom:     1 → 2 → 3 → 4 → 5 → 6 → 7 → NULL

                   头节点 (header)
                      │
        ┌─────────────┼─────────────┐
        ↓             ↓             ↓
      L4            L4            L4
      1 → → → → →  6 → → → → → → NULL
      ↓             ↓
      L3            L3
      1 → → → 4 → → 6 → → → → → NULL
      ↓       ↓     ↓
      L2      L2    L2
      1 → → → 3 → → 4 → → → 6 → → → NULL
      ↓       ↓     ↓       ↓
      L1      L1    L1      L1
      1 → 2 → 3 → 4 → 5 → 6 → 7 → NULL
```

### 4.2 跳表节点结构

```c
/* server.h - 跳表节点 */
typedef struct zskiplistNode {
    sds ele;                      // 成员对象
    double score;                 // 分值
    struct zskiplistNode *backward; // 后退指针

    // 层
    struct zskiplistLevel {
        struct zskiplistNode *forward; // 前进指针
        unsigned long span;            // 跨度（用于计算排名）
    } level[];                   // 柔性数组，节点有多少层就有多少个元素
} zskiplistNode;
```

**字段详解**：

| 字段 | 说明 |
|------|------|
| `ele` | 存储的 member（SDS 类型） |
| `score` | 排序分数 |
| `backward` | 指向链表前一个节点，用于反向遍历 |
| `level[]` | 层数组，每层包含 forward 指针和 span |

### 4.3 跳表结构

```c
/* server.h - 跳表 */
typedef struct zskiplist {
    struct zskiplistNode *header, *tail;  // 头节点、尾节点
    unsigned long length;                 // 节点数量
    int level;                            // 最大层数
} zskiplist;
```

**结构图示**：

```
zskiplist
┌──────────────────────────────────────────────────┐
│ header ───────────────────────────────→ tail     │
│   ↓                                            ↓  │
│  ┌───┐                                        ┌───┐
│  │ H │ (最大 32 层的 dummy 节点)                │ T │
│  └───┘                                        └───┘
│                                                  │
│ length: 6 (有效节点数，不含 header)               │
│ level: 4 (当前最大层数)                           │
└──────────────────────────────────────────────────┘
```

### 4.4 跳表层数确定

Redis 使用概率算法决定新节点的层数：

```c
/* t_zset.c - 随机层数生成 */
#define ZSKIPLIST_MAXLEVEL 32     /* 最大层数 */
#define ZSKIPLIST_P 0.25          /* 25% 概率提升一层 */

int zslRandomLevel(void) {
    int level = 1;

    /* 25% 概率层数 +1，75% 概率停止 */
    while ((random() & 0xFFFF) < (ZSKIPLIST_P * 0xFFFF))
        level += 1;

    return (level < ZSKIPLIST_MAXLEVEL) ? level : ZSKIPLIST_MAXLEVEL;
}
```

**数学期望**：

```
E(level) = 1 / p = 1 / 0.25 = 4

层数为 k 的概率：P(level = k) = p^(k-1) * (1-p)

P(level = 1) = 0.75
P(level = 2) = 0.1875
P(level = 3) = 0.046875
P(level = 4) = 0.01171875
...
```

### 4.5 跳表节点查找

```c
/* t_zset.c - 查找 score 相同且 member 匹配的节点 */
/* 返回每一层中小于目标且最接近的节点（保存在 update 数组） */
zskiplistNode* zslGetElementByRank(zskiplist *zsl, unsigned long rank) {
    zskiplistNode *x;
    unsigned long traversed = 0;

    x = zsl->header;

    /* 从最高层开始查找 */
    for (int i = zsl->level-1; i >= 0; i--) {
        /* 沿着 forward 指针前进 */
        while (x->level[i].forward &&
               (traversed + x->level[i].span) <= rank) {
            traversed += x->level[i].span;
            x = x->level[i].forward;
        }
        if (traversed == rank) {
            return x;
        }
    }
    return NULL;
}
```

**查找过程示例**：

```
查找 score = 5 的节点：

Level 3:  1 → → → → → → → 6 → NULL
Level 2:  1 → → → 4 → → → 6 → NULL
Level 1:  1 → 2 → 3 → 4 → 5 → 6 → NULL

步骤：
1. 从 header 最高层 (Level 2) 开始
2. Level 2: 1 → 4 (4 < 5，继续) → 6 (6 > 5，停止，下降到 Level 1)
3. Level 1: 4 → 5 (5 == 5，找到！)
```

### 4.6 跳表节点插入

```c
/* t_zsl.c - 插入节点 */
zskiplistNode *zslInsert(zskiplist *zsl, double score, sds ele) {
    zskiplistNode *update[ZSKIPLIST_MAXLEVEL], *x;
    unsigned int rank[ZSKIPLIST_MAXLEVEL];
    int i, level;

    x = zsl->header;

    /* 1. 从高到低查找插入位置 */
    for (i = zsl->level-1; i >= 0; i--) {
        rank[i] = (i == (zsl->level-1)) ? 0 : rank[i+1];

        while (x->level[i].forward &&
               (x->level[i].forward->score < score ||
                (x->level[i].forward->score == score &&
                 sdscmp(x->level[i].forward->ele, ele) < 0))) {
            rank[i] += x->level[i].span;
            x = x->level[i].forward;
        }
        update[i] = x;  /* 记录每层的前驱节点 */
    }

    /* 2. 随机生成新节点层数 */
    level = zslRandomLevel();

    /* 3. 如果新层数超过当前最大层数，初始化超出部分 */
    if (level > zsl->level) {
        for (i = zsl->level; i < level; i++) {
            update[i] = zsl->header;
            update[i]->level[i].span = zsl->length;
        }
        zsl->level = level;
    }

    /* 4. 创建新节点 */
    x = zslCreateNode(level, score, ele);

    /* 5. 插入到各层链表中 */
    for (i = 0; i < level; i++) {
        x->level[i].forward = update[i]->level[i].forward;
        update[i]->level[i].forward = x;

        /* 更新 span */
        x->level[i].span = update[i]->level[i].span - (rank[0] - rank[i]);
        update[i]->level[i].span = (rank[0] - rank[i]) + 1;
    }

    /* 6. 更新更高层的 span */
    for (i = level; i < zsl->level; i++) {
        update[i]->level[i].span++;
    }

    /* 7. 设置后退指针 */
    x->backward = (update[0] == zsl->header) ? NULL : update[0];
    if (x->level[0].forward)
        x->level[0].forward->backward = x;
    else
        zsl->tail = x;

    zsl->length++;
    return x;
}
```

### 4.7 跳表节点删除

```c
/* t_zsl.c - 删除节点 */
void zslDeleteNode(zskiplist *zsl, zskiplistNode *x, zskiplistNode **update) {
    int i;

    /* 1. 从各层链表中移除节点 */
    for (i = 0; i < zsl->level; i++) {
        if (update[i]->level[i].forward == x) {
            update[i]->level[i].span += x->level[i].span - 1;
            update[i]->level[i].forward = x->level[i].forward;
        } else {
            update[i]->level[i].span -= 1;
        }
    }

    /* 2. 更新后退指针 */
    if (x->level[0].forward) {
        x->level[0].forward->backward = x->backward;
    } else {
        zsl->tail = x->backward;
    }

    /* 3. 如果删除导致高层为空，降低层数 */
    while (zsl->level > 1 && zsl->header->level[zsl->level-1].forward == NULL) {
        zsl->level--;
    }

    zsl->length--;
}

/* 删除指定 score 和 member 的节点 */
int zslDelete(zskiplist *zsl, double score, sds ele, zskiplistNode **node) {
    zskiplistNode *update[ZSKIPLIST_MAXLEVEL], *x;
    int i;

    x = zsl->header;

    /* 查找待删除节点 */
    for (i = zsl->level-1; i >= 0; i--) {
        while (x->level[i].forward &&
               (x->level[i].forward->score < score ||
                (x->level[i].forward->score == score &&
                 sdscmp(x->level[i].forward->ele, ele) < 0))) {
            x = x->level[i].forward;
        }
        update[i] = x;
    }

    x = x->level[0].forward;

    /* 检查是否找到 */
    if (x && score == x->score && sdscmp(x->ele, ele) == 0) {
        zslDeleteNode(zsl, x, update);
        if (!node)
            zslFreeNode(x);
        else
            *node = x;
        return 1;
    }
    return 0;
}
```

---

## 5. 字典（dict）辅助结构

### 5.1 为什么需要 dict

虽然 skiplist 已经可以完成所有功能，但 Redis 仍然维护了一个 dict：

```
需求对比：
┌────────────────┬─────────────────┬─────────────────┐
│     操作       │    skiplist     │      dict       │
├────────────────┼─────────────────┼─────────────────┤
│ 按 score 范围  │   O(log N)      │   不支持        │
│ 按 member 查找 │   O(log N)      │   O(1)          │
│ 计算 member排名 │   O(log N)      │   不支持        │
│ 获取 score     │   O(log N)      │   O(1)          │
└────────────────┴─────────────────┴─────────────────┘
```

### 5.2 dict 结构

```c
/* dict.h */
typedef struct dict {
    dictType *type;
    void *privdata;
    dictht ht[2];         /* 两个哈希表，用于 rehash */
    long rehashidx;       /* rehash 索引，-1 表示未进行 */
    unsigned long iterators; /* 正在运行的迭代器数量 */
} dict;

typedef struct dictht {
    dictEntry **table;    /* 哈希表数组 */
    unsigned long size;   /* 哈希表大小 */
    unsigned long sizemask;
    unsigned long used;   /* 已有节点数量 */
} dictht;

typedef struct dictEntry {
    void *key;
    union {
        void *val;
        uint64_t u64;
        int64_t s64;
        double d;
    } v;
    struct dictEntry *next;
} dictEntry;
```

### 5.3 ZSet 中的 dict 使用

```c
/* ZSet 中 dict 的 key-value 结构 */
key:  member (sds)
val:  score (double)
```

**空间换时间策略**：
- dict 占用额外内存（约每个元素多 32-64 字节）
- 但 ZSCORE 操作从 O(log N) 降为 O(1)

---

## 6. 编码转换

### 6.1 转换触发条件

```c
/* t_zset.c */
/* 当满足以下任一条件时，从 ziplist 转为 skiplist+dict */
void zsetConvert(robj *zobj, int encoding) {
    ...

    if (encoding == OBJ_ENCODING_SKIPLIST) {
        /* 转换为跳表编码 */
        zset *zs = zmalloc(sizeof(*zs));
        zs->dict = dictCreate(&zsetDictType, NULL);
        zs->zsl = zslCreate();

        /* 遍历 ziplist，插入到 skiplist 和 dict */
        ...
    }
}
```

**转换条件**：

| 条件 | 默认阈值 |
|------|----------|
| ziplist 元素数量超过 | `zset-max-ziplist-entries 128` |
| 任意 member 长度超过 | `zset-max-ziplist-value 64` |

### 6.2 单向转换

```
┌─────────────┐           触发阈值          ┌─────────────────────┐
│   ziplist   │ ────────────────────────→  │  skiplist + dict     │
│  (省内存)   │                              │  (高性能)           │
└─────────────┘                              └─────────────────────┘
                  ⬆
                  │ 不可逆（一旦转成跳表，不会转回 ziplist）
```

**原因**：
1. 数据量已经较大，转回 ziplist 成本高
2. 如果数据量减少，内存节省有限

---

## 7. 为什么选择跳表

### 7.1 跳表 vs 平衡树

| 对比维度 | 跳表 | 平衡树 (AVL/红黑树) |
|----------|------|---------------------|
| 查找复杂度 | O(log N) | O(log N) |
| 插入/删除 | O(log N) | O(log N) |
| 实现复杂度 | 简单 | 复杂（旋转操作） |
| 范围查询 | 简单（按链表遍历） | 需要中序遍历 |
| 内存开销 | 多层指针（25%概率） | 固定指针 |
| 并发友好 | 是（局部修改） | 否（整体调整） |

### 7.2 Redis 作者的说明

Redis 作者 antirez 在博客中提到：

> "Skip lists are implemented in a way that they use less memory,
> and they are simpler to implement than balanced trees.
>
> The probability to have a complex tree with skip lists is lower
> because of the randomization involved."

### 7.3 跳表的优势

1. **实现简单**：无需复杂的旋转操作
2. **范围查询友好**：底层是链表，范围查询只需遍历
3. **内存局部性**：按分数相邻的节点在物理上也相邻
4. **无锁优化**：并发修改时影响范围小

---

## 8. 核心操作源码分析

### 8.1 ZADD - 添加/更新元素

```c
/* t_zset.c */
void zaddCommand(client *c) {
    robj *key = c->argv[1];
    robj *zobj;

    // 1. 获取或创建 ZSet 对象
    if ((zobj = lookupKeyWrite(c->db, key)) == NULL) {
        zobj = createZsetObject();
        dbAdd(c->db, key, zobj);
    }

    // 2. 添加元素
    for (j = 0; j < elements; j++) {
        // 根据编码类型调用不同的添加函数
        if (zobj->encoding == OBJ_ENCODING_ZIPLIST) {
            zsetAddEntryToZipList(zobj, ...);
            // 检查是否需要转换编码
            if (zzlLength(zobj->ptr) > server.zset_max_ziplist_entries ||
                sdslen(ele) > server.zset_max_ziplist_value)
                zsetConvert(zobj, OBJ_ENCODING_SKIPLIST);
        } else {
            zsetAddEntryToSkiplist(zobj, ...);
        }
    }
}
```

### 8.2 ZRANGE - 范围查询

```c
/* t_zset.c */
void zrangeCommand(client *c) {
    robj *key = c->argv[1];
    robj *zobj = lookupKeyRead(c->db, key);

    if (zobj->encoding == OBJ_ENCODING_ZIPLIST) {
        // ziplist 实现：遍历获取
        zzlRange(zobj->ptr, start, end, ...);
    } else {
        // skiplist 实现：利用 span 快速定位
        zset *zsetobj = zobj->ptr;
        zskiplist *zsl = zsetobj->zsl;
        zskiplistNode *ln;

        ln = zslGetElementByRank(zsl, start);
        // 沿着 level[0] 遍历
        while (ln && range-- > 0) {
            // 输出结果
            ln = ln->level[0].forward;
        }
    }
}
```

### 8.3 ZRANK - 获取排名

```c
/* t_zset.c - 获取 member 的排名（从 0 开始） */
unsigned long zsetRank(robj *zobj, sds ele, int reverse) {
    unsigned long rank = 0;

    if (zobj->encoding == OBJ_ENCODING_ZIPLIST) {
        // ziplist: 遍历计数
        rank = zzlFindRank(zobj->ptr, ele, NULL);
    } else {
        zset *zs = zobj->ptr;
        zskiplist *zsl = zs->zsl;
        zskiplistNode *x;
        zskiplistNode *update[ZSKIPLIST_MAXLEVEL];

        /* skiplist: 利用 span 累加计算排名 */
        x = zsl->header;
        for (int i = zsl->level-1; i >= 0; i--) {
            while (x->level[i].forward &&
                   (x->level[i].forward->score < score ||
                    (x->level[i].forward->score == score &&
                     sdscmp(x->level[i].forward->ele, ele) <= 0))) {
                rank += x->level[i].span;
                x = x->level[i].forward;
            }
            if (x->ele && sdscmp(x->ele, ele) == 0) {
                break;
            }
        }
    }
    return rank;
}
```

---

## 9. 时间复杂度分析

### 9.1 各操作复杂度汇总

| 命令 | ziplist | skiplist+dict |
|------|---------|---------------|
| ZADD | O(N) | O(log N) |
| ZREM | O(N) | O(log N) |
| ZSCORE | O(N) | O(1) |
| ZRANGE | O(N+log(N)) | O(log N + M) |
| ZRANK | O(N) | O(log N) |
| ZRANGEBYSCORE | O(N) | O(log N + M) |

*注：M 为返回的元素数量*

### 9.2 复杂度证明

**跳表查找复杂度**：

```
设 n 个节点，最大层数 h = O(log n)

每层期望节点数：
- 第 1 层: n 个节点
- 第 2 层: n * p 个节点
- 第 h 层: n * p^(h-1) 个节点

查找路径：
- 从最高层开始，每层最多前进 1/p 个节点
- 总复杂度: (h / p) = O(log n / p) = O(log n)
```

**跳表空间复杂度**：

```
期望节点数 = n * (1 + p + p^2 + ...) = n * (1 / (1 - p))

当 p = 0.25: 期望空间 = n * (1 / 0.75) ≈ 1.33n

每个节点平均层数 = 1 / p = 4
```

---

## 10. 常见面试题

### Q1: 为什么 ZSet 同时使用 skiplist 和 dict？

**答**：这是空间换时间的策略：

- **skiplist**：支持按 score 范围查询和排序
- **dict**：支持 O(1) 时间复杂度的 ZSCORE 操作

如果没有 dict，ZSCORE 需要 O(log N) 时间查找 skiplist。

### Q2: 跳表和平衡树相比有什么优势？

**答**：
1. **实现简单**：无需复杂的旋转操作，代码更易维护
2. **范围查询友好**：底层是有序链表，范围查询直接遍历
3. **内存局部性好**：相邻节点物理位置可能相近
4. **并发友好**：插入/删除只需修改部分指针，不影响整体结构

### Q3: Redis 跳表为什么最大层数是 32？

**答**：
```c
#define ZSKIPLIST_MAXLEVEL 32
/* 这个值足够大，使得 2^32 远超实际存储的元素数量 */
/* 即使存储数十亿元素，32 层也足够 */
```

计算：p = 0.25 时，平均层数约 4，32 层可以容纳 2^32 个元素。

### Q4: 跳表的 span 是如何计算的？

**答**：
```
span[i] = 当前节点到第 i 层下一个节点之间跨越的节点数

例如：
        Level 2: 1 ———span=3———→ 4
        Level 1: 1 → 2 → 3 → 4

节点 1 的 level[2].span = 3（跨越了 2、3、4）
```

span 用于快速计算排名（ZRANK）和范围查询（ZRANGE）。

### Q5: 为什么 p = 0.25？

**答**：
- p 越大：层数越高，查找越快，但空间开销越大
- p 越小：层数越低，空间节省，但查找变慢
- p = 0.25 是经验值，平衡了时间和空间

```c
#define ZSKIPLIST_P 0.25
```

---

## 参考资料

- Redis 源码: https://github.com/redis/redis
- William Pugh 论文: "Skip Lists: A Probabilistic Alternative to Balanced Trees"
- Redis 设计与实现 (黄健宏)
- server.h - Redis 源码
- t_zset.c - ZSet 实现代码
