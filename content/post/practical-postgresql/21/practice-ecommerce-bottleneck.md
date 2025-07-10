+++
title = "第二十一章 性能调优实战 - 第三节 实战：电商平台高峰期性能瓶颈排查"
date = 2025-07-10
lastmod = 2025-07-10T01:00:00+08:00
tags = ["PostgreSQL", "practical", "book", "performance", "tuning", "troubleshooting", "ecommerce"]
categories = ["PostgreSQL", "practical", "book"]
draft = false
author = "b40yd"
+++

## 第三节 实战：电商平台高峰期性能瓶颈排查

> **目标**：通过模拟一次电商平台在促销活动期间遇到的性能瓶颈，实践一套系统化的慢查询定位、分析和优化流程，综合运用 `pg_stat_statements`, `EXPLAIN`, 索引优化和查询重写等核心技能。

### 场景描述

我们的电商平台正在进行“黑色星期五”大促，网站流量激增。运维团队收到大量告警，用户反馈商品搜索页面加载极慢，甚至出现超时。数据库服务器的 CPU 使用率飙升至 100%。我们的任务是快速定位并解决性能瓶颈。

---

### 第一步：初步诊断 (Triage)

当系统出现紧急性能问题时，首先要快速了解“现在正在发生什么”。

**1. 查看当前活动的查询**
连接到数据库，使用 `pg_stat_activity` 视图来查看当前正在运行的查询。
```sql
SELECT
    pid,
    now() - xact_start AS duration,
    state,
    query
FROM pg_stat_activity
WHERE state != 'idle'
ORDER BY duration DESC
LIMIT 10;
```
我们可能会发现有几条相同的、执行时间非常长的（`duration` 达到几十秒）的搜索查询处于 `active` 状态，这为我们提供了第一个线索。

**2. 使用 `pg_stat_statements` 定位“元凶”**
`pg_stat_activity` 只显示当前快照，而 `pg_stat_statements` 扩展能聚合统计一段时间内所有查询的性能数据，是定位“问题查询”的最有力工具。

*(需要预先在 `postgresql.conf` 中配置 `shared_preload_libraries = 'pg_stat_statements'` 并 `CREATE EXTENSION pg_stat_statements;`)*

```sql
SELECT
    total_exec_time,
    mean_exec_time,
    calls,
    query
FROM pg_stat_statements
ORDER BY mean_exec_time DESC
LIMIT 5;
```
通过这个查询，我们几乎可以肯定地找到那个平均执行时间（`mean_exec_time`）最长的查询。假设我们找到了如下这条“罪魁祸首”：

**问题查询：**
```sql
SELECT *
FROM products
WHERE
    category_id = 15
    AND in_stock = true
    AND lower(description) LIKE '%special edition%'
ORDER BY created_at DESC
LIMIT 20;
```

---

### 第二步：分析执行计划 (`EXPLAIN ANALYZE`)

我们把这条慢查询拿出来，执行 `EXPLAIN ANALYZE` 来深入分析它的执行计划。

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT *
FROM products
WHERE
    category_id = 15
    AND in_stock = true
    AND lower(description) LIKE '%special edition%'
ORDER BY created_at DESC
LIMIT 20;
```

**糟糕的执行计划（示例）：**
```
Limit  (cost=51321.55..51321.57 rows=20 width=1024) (actual time=4531.123..4531.125 rows=20 loops=1)
  Buffers: shared hit=12345 read=54321
  ->  Sort  (cost=51321.55..51321.60 rows=100 width=1024) (actual time=4531.121..4531.122 rows=20 loops=1)
        Sort Key: created_at DESC
        Sort Method: external merge Disk: 512MB  -- 警告：使用了磁盘排序！
        Buffers: shared hit=12345 read=54321, temp read=65536 written=65536
        ->  Seq Scan on products  (cost=0.00..51319.50 rows=100 width=1024) (actual time=0.055..4520.555 rows=100 loops=1)
              Filter: (in_stock AND (category_id = 15) AND (lower(description) ~~ '%special edition%'))
              Rows Removed by Filter: 999900
              Buffers: shared hit=12345 read=54321
Planning Time: 0.250 ms
Execution Time: 4531.500 ms -- 耗时 4.5 秒！
```

**问题分析：**
1.  **`Seq Scan on products`**: 查询规划器选择了全表扫描，扫描了近百万行数据，这是最主要的问题。
2.  **`Filter: (lower(description) ...)`**: 在 `description` 列上使用 `lower()` 函数和前导通配符的 `LIKE`，导致无法使用任何标准 B-Tree 索引。
3.  **`Sort Method: external merge Disk: 512MB`**: `ORDER BY` 操作因为工作内存不足（`work_mem` 太小），不得不使用磁盘进行外部排序，这极大地拖慢了速度。

---

### 第三步：实施优化

#### 1. 创建合适的索引
-   **复合索引**：为 `WHERE` 子句中的精确匹配和范围匹配列创建复合索引。
    ```sql
    CREATE INDEX idx_products_category_stock ON products (category_id, in_stock);
    ```
-   **文本搜索索引**：使用 `pg_trgm` 扩展来处理模糊 `LIKE` 查询。
    ```sql
    CREATE EXTENSION IF NOT EXISTS pg_trgm;
    CREATE INDEX idx_products_desc_trgm ON products USING GIN (description gin_trgm_ops);
    ```
-   **排序索引**：为 `ORDER BY` 的列创建索引。
    ```sql
    CREATE INDEX idx_products_created_at_desc ON products (created_at DESC);
    ```

#### 2. 重写查询
修改查询以利用新创建的 `pg_trgm` 索引。注意，`pg_trgm` 默认是大小写不敏感的，所以可以去掉 `lower()` 函数。
```sql
-- 优化后的查询
SELECT *
FROM products
WHERE
    category_id = 15
    AND in_stock = true
    AND description % 'special edition' -- 使用 % 操作符来触发 GIN 索引
ORDER BY created_at DESC
LIMIT 20;
```

#### 3. 调整配置 (可选)
如果磁盘排序问题依然存在，可以考虑适度增加 `work_mem`。但通常在创建了合适的 `ORDER BY` 索引后，排序本身的开销会大大减少。

---

### 第四步：验证优化效果

再次对**优化后**的查询执行 `EXPLAIN ANALYZE`。

**理想的执行计划：**
```
Limit  (cost=0.56..150.58 rows=20 width=1024) (actual time=0.150..0.250 rows=20 loops=1)
  Buffers: shared hit=256
  ->  Index Scan using idx_products_created_at_desc on products  (cost=0.56..750.12 rows=100 width=1024) (actual time=0.148..0.245 rows=20 loops=1)
        Index Cond: ((category_id = 15) AND (in_stock) AND (description % 'special edition'))
        Buffers: shared hit=256
Planning Time: 0.500 ms
Execution Time: 0.300 ms -- 耗时 0.3 毫秒！
```

**效果分析：**
-   **`Index Scan`**: 查询规划器现在使用了最高效的索引扫描。
-   **没有 `Sort` 节点**: 因为 `ORDER BY` 的顺序与 `idx_products_created_at_desc` 索引的顺序一致，PostgreSQL 可以直接从索引中按顺序读取数据，完全避免了排序操作。
-   **执行时间**：从 4.5 秒骤降至 0.3 毫秒，性能提升了上万倍！

---

## 📌 小结

本次实战演练了一套完整、系统化的性能瓶颈排查流程：
1.  **监控与定位**：使用 `pg_stat_activity` 和 `pg_stat_statements` 快速找到消耗资源最多的问题查询。
2.  **分析与诊断**：使用 `EXPLAIN ANALYZE` 深入剖析查询的执行计划，找出 `Seq Scan`、磁盘排序等性能瓶颈。
3.  **优化与实施**：针对性地创建**复合索引**、**函数索引**（如 `pg_trgm`）和**排序索引**，并重写查询以利用这些索引。
4.  **验证与对比**：再次执行 `EXPLAIN ANALYZE`，确认优化措施生效，并量化性能提升的效果。

这套方法论是每个数据库管理员和后端开发者都应掌握的核心技能，它能帮助你在系统面临性能压力时，有条不紊地解决问题，保障服务的稳定。
