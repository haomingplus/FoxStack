---
title: MySQL三大日志深度解析
published: 2020-03-22 10:15:00
description: 深入理解MySQL的binlog、redo log和undo log三大日志的设计原理、工作机制及解决的问题
tags: [MySQL, InnoDB, 事务, MVCC, 主从复制]
category: MySQL
draft: false
---

## 一、整体架构视角

```
┌─────────────────────────────────────────────────────────────┐
│                        MySQL Server                          │
│  ┌──────────────────┐         ┌──────────────────┐         │
│  │    Binlog        │         │   SQL Layer      │         │
│  │  (Server层)      │◄────────┤                  │         │
│  └──────────────────┘         └────────┬─────────┘         │
│                                        │                   │
│  ┌─────────────────────────────────────▼─────────────────┐ │
│  │                   InnoDB Storage Engine                │ │
│  │  ┌──────────────┐     ┌──────────────┐               │ │
│  │  │  Undo Log    │     │  Redo Log    │               │ │
│  │  │ (回滚段)     │     │  (IBlogfile)  │               │ │
│  │  └──────────────┘     └──────────────┘               │ │
│  │                                                       │ │
│  │  ┌─────────────────────────────────────────────────┐  │ │
│  │  │              Buffer Pool                        │  │ │
│  │  │  (数据页、索引页、插入缓存、自适应哈希索引)       │  │ │
│  │  └─────────────────────────────────────────────────┘  │ │
│  └───────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
```

---

## 二、Binlog 深度解析

### 2.1 为什么要设计Binlog？

**核心问题**：MySQL是插件式存储引擎架构，Server层本身不管理数据，数据由各存储引擎管理。

- Innodb、MyISAM、Memory等引擎各自管理自己的数据
- 如果没有Binlog，跨引擎的数据恢复和复制将无法统一实现
- **Binlog在Server层统一记录，屏蔽了底层引擎差异**

### 2.2 Binlog写入机制

```
事务执行过程：
┌────────────────────────────────────────────────────────────┐
│ 1. 事务执行，数据变更写入Buffer Pool                        │
│ 2. 事务提交时，将变更写入binlog cache（内存）               │
│ 3. 根据sync_binlog参数决定何时刷盘：                        │
│    - sync_binlog=0: 不主动同步，由OS决定                   │
│    - sync_binlog=1: 每次事务提交同步（默认，最安全）        │
│    - sync_binlog=N: N个事务提交后同步                       │
└────────────────────────────────────────────────────────────┘
```

### 2.3 Binlog Cache vs Binlog Buffer

|  | Binlog Cache | Binlog Buffer(IO Cache) |
|--|--------------|-------------------------|
| 位置 | 每个连接独占 | 全局共享 |
| 生命周期 | 连接期间 | 持久化 |
| 大小限制 | binlog_cache_size | - |
| 超限处理 | 写入临时文件 | - |

### 2.4 为什么Statement模式会有主从不一致？

**经典案例**：

```sql
-- 主库执行
UPDATE user SET create_time = NOW() WHERE id = 1;
-- Statement记录: UPDATE user SET create_time = NOW() WHERE id = 1;
-- 从库执行时时间不同，结果不同

-- 另一个案例
DELETE FROM t WHERE id > (SELECT MAX(id) FROM t) LIMIT 1;
-- 主从执行顺序不同，可能删除不同的行
```

**Row模式解决方式**：记录实际修改的行，如：
```
### UPDATE `test`.`user`
### WHERE
###   @1=1 /* id */
### SET
###   @2='2024-01-01 12:00:00' /* create_time */
```

### 2.5 GTID与Binlog

**GTID (Global Transaction ID)** 解决了传统Binlog位置的痛点：

```bash
# 传统方式：需要记录binlog文件名和position
CHANGE MASTER TO MASTER_LOG_FILE='bin.000003', MASTER_LOG_POS=1024;

# GTID方式：自动定位
SET GLOBAL GTID_PURGED='3E11FA47-71CA-11E1-9E33-C80AA9429562:1-56';
CHANGE MASTER TO MASTER_AUTO_POSITION = 1;
```

**GTID格式**：`source_id:transaction_id`
- 保证同一个事务在集群内唯一标识
- 自动处理主从切换，避免重复执行事务

---

## 三、Redo Log 深度解析

### 3.1 核心设计思想：WAL (Write-Ahead Logging)

```
传统方式（每次事务提交都要刷盘）：
事务 → 修改数据页 → 立即写数据页到磁盘（随机IO）
问题：一个事务可能修改多个页面，每次都是随机写，性能极差

WAL方式：
事务 → 写redo log（顺序IO） → 异步刷新数据页
优势：将随机写转化为顺序写，性能提升数倍
```

### 3.2 Redo Log 的物理结构

```
redo log file 结构：
┌────────────────────────────────────────────────────────────┐
│  header  │ block 2 │ block 3 │ ... │ block N │  header    │
│  (512B)  │ (512B)  │ (512B)  │     │ (512B)  │  (512B)    │
└────────────────────────────────────────────────────────────┘
                           ▲
                    循环写入，checkpoint推进

checkpoint: 表示数据页已经刷盘的位置
           checkpoint之前的redo可以覆盖
```

### 3.3 Redo Log Buffer 刷盘策略

```c
// MySQL源码简化逻辑
void write_redo_log() {
    // 1. 写入redo buffer（内存）
    mtr_commit(&mtr);  // mini-transaction commit

    // 2. 根据innodb_flush_log_at_trx_commit决定何时刷盘
    if (innodb_flush_log_at_trx_commit == 0) {
        // 每秒刷一次，可能丢失1秒事务
    } else if (innodb_flush_log_at_trx_commit == 1) {
        // 每次事务提交都刷盘（fsync），最安全
        fsync(log_file);
    } else if (innodb_flush_log_at_trx_commit == 2) {
        // 每次写入OS缓存，每秒fsync
        write(log_file);  // 但不fsync
    }
}
```

### 3.4 Redo Log 写入优化

**Mini-Transaction (MTR)**：

```
一次用户事务可能包含多个MTR：
BEGIN;
  INSERT INTO t1 VALUES (1);  -- MTR1: 修改叶子节点、非叶子节点
  INSERT INTO t2 VALUES (1);  -- MTR2: 修改t2的索引页
  UPDATE t3 SET a=1;          -- MTR3: 修改t3的数据页
COMMIT;

每个MTR产生一组redo log，作为一个原子单位写入
```

### 3.5 Checkpoint 机制

```
LSN (Log Sequence Number) 是redo log的核心概念：

              ┌─────────────────────────────────────────┐
Redo Log:     │  write_pos  │     checkpoint     │      │
              └─────────────────────────────────────────┘
                           ▲
                    可用空间 = write_pos - checkpoint

write_pos:  当前写入位置
checkpoint: 表示数据页已刷盘到这个LSN的位置
checkpoint_age: write_pos - checkpoint

checkpoint推进：
1. 脏页刷盘时，检查该页oldest_modification
2. 如果是oldest，则可以推进checkpoint
```

### 3.6 Redo Log 的空间计算

```
假设配置：
innodb_log_file_size = 1GB
innodb_log_files_in_group = 2
总空间 = 2GB

innodb_max_dirty_pages_pct = 75%
表示：脏页占比超过75%时，强制刷盘推进checkpoint

告警规则：
如果 checkpoint_age > 7/8 * total_size
  输出: "Cannot proceed with the flush"

如果 checkpoint_age > total_size
  输出: "The age of the checkpoint exceeds the log group capacity"
```

---

## 四、Undo Log 深度解析

### 4.1 Undo Log 的存储结构

```
Undo Table Space (undo表空间)
    │
    ├── Undo Segment (undo段，最多128个)
    │     │
    │     ├── Undo Page (undo页，每个段可以有很多页)
    │           │
    │           ├── Undo Log Record 1 (事务1的undo记录)
    │           ├── Undo Log Record 2 (事务2的undo记录)
    │           └── ...
    │
    └── Rollback Segment (回滚段，老版本概念)
```

### 4.2 Undo Log 版本链（MVCC核心）

```
数据行结构：
┌──────────────────────────────────────────────────────────┐
│  隐藏列 (DB_TRX_ID, DB_ROLL_PTR, DB_ROW_ID)              │
├──────────────────────────────────────────────────────────┤
│  DB_TRX_ID:   最后修改该行的事务ID                        │
│  DB_ROLL_PTR: 指向undo log中该行的旧版本                  │
│  DB_ROW_ID:   行ID（无主键时使用）                        │
└──────────────────────────────────────────────────────────┘

版本链示例：

最新记录: value='v5', trx_id=100, roll_ptr ─┐
                                          │
                                          ▼
Undo Log: value='v4', trx_id=90, roll_ptr ─┐
                                         │
                                         ▼
Undo Log: value='v3', trx_id=80, roll_ptr ─┐
                                         │
                                         ▼
Undo Log: value='v2', trx_id=70, roll_ptr ─┐
                                         │
                                         ▼
                                    value='v1', trx_id=60
```

### 4.3 Read View 可见性判断

```c
// Read View 结构
struct read_view_t {
    trx_id_t low_limit_id;      // 尚未活跃的最小事务ID (= max_active_trx_id + 1)
    trx_id_t up_limit_id;       // 活跃事务列表中最小ID
    vector<trx_id_t> m_ids;     // 创建Read View时的活跃事务列表
    trx_id_t creator_trx_id;    // 创建该Read View的事务ID
};

// 可见性判断算法
bool visible(trx_id_t trx_id, read_view_t* view) {
    // 情况1: 事务ID小于up_limit_id，可见
    if (trx_id < view->up_limit_id) return true;

    // 情况2: 事务ID大于等于low_limit_id，不可见
    if (trx_id >= view->low_limit_id) return false;

    // 情况3: 在m_ids中，不可见；否则可见
    return !view->m_ids.contains(trx_id);
}
```

### 4.4 Purge 线程

```
Undo Log 的清理不是在事务提交时立即进行，而是由Purge线程异步处理：

Purge触发条件：
1. Undo Log不再被任何活跃事务的Read View需要
2. Undo Log所在页空间可以重用

Purge工作流程：
1. 读取history list（提交事务的undo链表）
2. 判断undo记录是否可以清理
3. 清理undo记录
4. 清理delete mark的索引记录

参数控制：
innodb_purge_threads = 4          # purge线程数
innodb_purge_batch_size = 300     # 每次清理的undo记录数
```

### 4.5 Undo Log 与长事务的危害

```
长事务导致的问题：

1. Undo Log 空间膨胀
   ┌──────────────────────────────────────────────┐
   │ 正常: undo log可及时回收，空间稳定           │
   │ 长事务: undo log堆积，占用大量空间           │
   └──────────────────────────────────────────────┘

2. 版本链过长
   查询需要遍历更长的版本链，性能下降

3. 锁等待时间延长
   长事务持有锁时间更长

解决方案：
- 设置 innodb_rollback_segments 增加并发
- 监控 information_schema.innodb_trx
- 设置 set global max_execution_time
- 使用 pt-kill 杀掉长事务
```

---

## 五、两阶段提交深度解析

### 5.1 为什么需要两阶段提交？

```
问题场景：假设事务提交，redo已写，binlog未写，然后崩溃

【单阶段提交的问题】
时间线:
t1: 事务执行，写redo log并刷盘
t2: 写binlog（此时宕机）
t3: 数据库崩溃

恢复后:
- redo log存在 → 数据已持久化
- binlog不存在 → 从库无法同步
结果: 主从数据不一致！

【两阶段提交解决】
时间线:
t1: 事务执行
t2: 写redo log (prepare状态)
t3: 写binlog并刷盘
t4: 提交redo log (commit状态)

崩溃恢复判断:
- redo处于prepare且binlog存在 → 提交
- redo处于prepare但binlog不存在 → 回滚
```

### 5.2 两阶段提交时序图

```
MySQL客户端                        InnoDB                Binlog
   │                                │                     │
   │  BEGIN                         │                     │
   │───────────────────────────────>│                     │
   │                                │                     │
   │  INSERT ...                    │                     │
   │───────────────────────────────>│                     │
   │  (写Undo, 写Buffer Pool)       │                     │
   │                                │                     │
   │  COMMIT                        │                     │
   │───────────────────────────────>│                     │
   │                                │                     │
   │              ┌─────────────────┤                     │
   │              │  prepare        │                     │
   │              │  写redo log     │                     │
   │              │  (prepare状态)  │                     │
   │              └─────────────────┤                     │
   │                                │                     │
   │              ┌─────────────────────────────────────┤ │
   │              │  写binlog                             │ │
   │              └─────────────────────────────────────┤ │
   │                                │                     │
   │              ┌─────────────────┤                     │
   │              │  commit        │                     │
   │              │  redo改为commit │                     │
   │              └─────────────────┤                     │
   │                                │                     │
   │  OK                            │                     │
   │<───────────────────────────────│                     │
```

### 5.3 崩溃恢复逻辑

```c
// MySQL源码简化逻辑
void recover_crash() {
    // 1. 扫描redo log，按LSN顺序重做已提交的事务
    scan_redo_log_and_apply();

    // 2. 处理prepare状态的事务
    for each trx in prepare_state {
        if (trx_exists_in_binlog(trx_id)) {
            // binlog存在，说明事务应该提交
            commit_transaction(trx);
        } else {
            // binlog不存在，说明事务应该回滚
            rollback_transaction(trx, using_undo_log);
        }
    }
}
```

### 5.4 组提交（Group Commit）

```
问题：每个事务都fsync一次binlog，IO压力大

组提交机制：
┌──────────────────────────────────────────────────────────┐
│  阶段一: Flush阶段，将binlog cache写入binlog文件          │
│  └── 多个事务的binlog顺序写入                            │
│                                                          │
│  阶段二: Sync阶段，fsync binlog文件                      │
│  └── 多个事务共享一次fsync                               │
│                                                          │
│  阶段三: Commit阶段，提交redo log                         │
│  └── 批量将redo改为commit状态                            │
└──────────────────────────────────────────────────────────┘

参数：
binlog_group_commit_sync_delay = 10   # 微秒，延迟等待更多事务
binlog_group_commit_sync_no_delay_count = 5  # 至少5个事务才等待
```

---

## 六、三大日志的协同工作示例

```
完整的UPDATE执行流程：

1. 执行器调用InnoDB引擎
2. InnoDB查找数据页（Buffer Pool或磁盘）
3. 记录Undo Log（记录修改前的值）
4. 修改Buffer Pool中的数据页
5. 记录Redo Log（Buffer）
6. 事务提交：
   a. Redo Log 写入磁盘（prepare状态）
   b. Binlog 写入磁盘
   c. Redo Log 标记为commit状态
7. 返回成功

崩溃恢复：
- redo重做：恢复未刷盘的数据页
- undo回滚：回滚未提交事务的修改
- binlog同步：主从复制使用
```

---

## 七、性能调优参数总结

### 7.1 Binlog调优

| 参数 | 推荐值 | 说明 |
|------|--------|------|
| binlog_format | ROW | 确保主从一致 |
| sync_binlog | 1（安全）/ 100（性能） | 控制刷盘频率 |
| binlog_cache_size | 1M-4M | 大事务需要增大 |
| binlog_group_commit_sync_delay | 10-100 | 组提交延迟 |
| expire_logs_days | 7 | 自动清理天数 |

### 7.2 Redo Log调优

| 参数 | 推荐值 | 说明 |
|------|--------|------|
| innodb_log_file_size | 1G-2G | 单个redo文件大小 |
| innodb_log_files_in_group | 2 | redo文件数量 |
| innodb_flush_log_at_trx_commit | 1（安全）/ 2（性能） | 刷盘策略 |
| innodb_log_buffer_size | 16M-64M | redo buffer大小 |

### 7.3 Undo Log调优

| 参数 | 推荐值 | 说明 |
|------|--------|------|
| innodb_undo_tablespaces | 2+ | 独立undo表空间数量 |
| innodb_max_undo_log_size | 1G | 单个undo表空间上限 |
| innodb_rollback_segments | 128 | 回滚段数量（默认足够） |
| innodb_purge_threads | 4 | purge线程数 |

---

## 八、高频面试题（深入版）

**Q1: 如果误删除了数据，如何恢复？**

```sql
A: 恢复步骤：
1. 立即停止数据库，防止新事务覆盖旧数据
2. 恢复最近的全量备份
3. 使用mysqlbinlog工具提取binlog：
   mysqlbinlog --start-position=154 --stop-position=500 \
   --database=dbname bin.000003 > recovery.sql
4. 找到误删除的position，跳过该语句
5. 应用后续binlog完成恢复

或者使用闪回功能（binlog_format=ROW）：
mysqlbinlog --flashback --start-position=500 --stop-position=600 \
bin.000003 | mysql -u root -p
```

**Q2: Redo Log写满会怎样？**

```
A: Redo Log写满的处理流程：

1. 停止所有更新操作（停库！）
2. 触发checkpoint，将脏页刷盘
3. 推进checkpoint位置，腾出空间
4. 恢复更新操作

这就是为什么innodb_log_file_size不能设置太小的原因：
- 太小 → 频繁checkpoint → 刷盘频繁 → 性能下降
- 太大 → 崩溃恢复时间长

建议：根据脏页生成速度设置，一般1G-2G
```

**Q3: MVCC能否解决幻读？**

```
A: MVCC解决的是快照读的幻读，不能解决当前读的幻读：

快照读（SELECT）：
- 使用Read View读取历史版本
- 不加锁，通过版本链实现可重复读

当前读（SELECT ... FOR UPDATE / UPDATE / DELETE）：
- 必须读取最新版本
- 需要通过Gap Lock（间隙锁）+ Next-Key Lock解决幻读

这就是为什么RR级别下：
- 普通SELECT不会幻读（MVCC）
- SELECT ... FOR UPDATE仍可能锁等待（Next-Key Lock）
```

**Q4: Binlog和Redo Log都能恢复数据，为什么都需要？**

```
A:
           Binlog                    Redo Log
         ┌─────────┐               ┌─────────┐
定位:    Server层                   InnoDB引擎
类型:    逻辑日志                   物理日志
恢复粒度:   按SQL/行               按数据页
恢复范围:   全库任意时间           只恢复InnoDB表
用途:    主从复制、跨引擎          崩溃恢复、性能优化

组合使用：
1. 数据恢复 = 全量备份 + Binlog增量
2. 崩溃恢复 = Redo Log（InnoDB引擎）
3. 主从复制 = Binlog
4. 数据一致性 = Binlog + Redo Log两阶段提交
```

**Q5: 为什么RR级别下默认不开启binlog时，会退化成RC？**

```
A: 这是一个经典陷阱！

实际上MySQL的RR级别实现依赖于MVCC，与binlog是否开启无关。
但开启binlog后，为了保证binlog与实际执行顺序一致，MySQL会
限制某些优化，可能导致不同的执行计划。

更准确地说：
- 开启binlog + statement格式：RR可能受影响
- 开启binlog + row格式：RR不受影响

所以生产环境建议：
binlog_format=ROW + 隔离级别=READ-COMMITTED
或
binlog_format=ROW + 隔离级别=REPEATABLE-READ
```

**Q6: Doublewrite Buffer是什么？解决了什么问题？**

```
A: Doublewrite Buffer是InnoDB的额外机制，解决页断裂问题：

问题：16KB的数据页写入时，如果只写了4KB就宕机，导致页面不完整

解决：
1. 先写入doublewrite buffer（连续区域）
2. 再写入实际位置

恢复：
- 如果实际页面损坏，从doublewrite buffer恢复

参数：
innodb_doublewrite = 1  # 开启
innodb_doublewrite_files = 2
innodb_doublewrite_pages = 32
```

**Q7: 什么是脏页？什么时候刷脏页？**

```
A: 脏页：Buffer Pool中已修改但未刷盘的数据页

刷脏页的时机：
1. redo log空间不足时（checkpoint）
2. Buffer Pool空间不足时（LRU淘汰）
3. MySQL空闲时（后台线程）
4. MySQL正常关闭时

相关参数：
innodb_max_dirty_pages_pct = 75  # 脏页比例阈值
innodb_io_capacity = 2000         # IO能力限制
innodb_adaptive_flushing = 1      # 自适应刷脏页
```

**Q8: WAL为什么能提升性能？**

```
A: WAL的核心是将随机IO转化为顺序IO：

传统方式：
- 修改数据页（随机位置）
- 立即写回磁盘（随机IO）
- 性能：~100-200 IOPS

WAL方式：
- 写redo log（顺序追加）
- 异步写数据页（批量）
- 性能：~10000+ IOPS

核心优势：
1. 顺序写比随机写快100倍+
2. 小日志合并为大页面写入
3. 可以批量刷盘，减少IO次数
```

---

## 九、核心总结

```
三大日志的记忆框架：

┌────────────────────────────────────────────────────────────┐
│                      Binlog (逻辑)                          │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐     │
│  │  主从复制     │  │  数据恢复     │  │   审计追踪    │     │
│  └──────────────┘  └──────────────┘  └──────────────┘     │
│        向外传            向后追            记历史             │
└────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌────────────────────────────────────────────────────────────┐
│                    Redo Log (物理)                          │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐     │
│  │  崩溃恢复     │  │  WAL优化      │  │   异步刷盘    │     │
│  └──────────────┘  └──────────────┘  └──────────────┘     │
│        向前做            提性能            减压力             │
└────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌────────────────────────────────────────────────────────────┐
│                    Undo Log (逻辑)                          │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐     │
│  │  事务回滚     │  │   MVCC        │  │  原子性保证   │     │
│  └──────────────┘  └──────────────┘  └──────────────┘     │
│        向后退            快照读            要么全做          │
└────────────────────────────────────────────────────────────┘
```

**一句话总结**：
- **Binlog** = Server层的"录像机"，记录发生了什么（给复制和恢复用）
- **Redo Log** = InnoDB的"草稿本"，先记下来慢慢誊写（提升性能）
- **Undo Log** = InnoDB的"时光机"，可以回到修改前的状态（事务和MVCC）

---

## 十、参考链接

- [MySQL 8.0 Reference Manual - The Binary Log](https://dev.mysql.com/doc/refman/8.0/en/binary-log.html)
- [MySQL 8.0 Reference Manual - InnoDB Redo Log](https://dev.mysql.com/doc/refman/8.0/en/innodb-redo-log.html)
- [MySQL 8.0 Reference Manual - Undo Log](https://dev.mysql.com/doc/refman/8.0/en/innodb-undo-log.html)
