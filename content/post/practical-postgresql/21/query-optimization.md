+++
title = "第二十一章 性能调优实战 - 第一节：查询优化技巧（索引、重写、缓存）"
date = 2025-07-10
lastmod = 2025-07-10T01:00:00+08:00
tags = ["PostgreSQL", "practical", "book", "performance", "tuning", "query optimization", "index"]
categories = ["PostgreSQL", "practical", "book"]
draft = false
author = "b40yd"
+++

## 第二十一章 性能调优实战
### 第一节 查询优化技巧（索引、重写、缓存）

> **目标**：掌握 PostgreSQL 查询优化的核心思想和三大基本技术：创建正确的索引、重写低效的 SQL 以及理解和利用缓存，为解决慢查询问题打下坚实基础。

数据库性能调优是一个系统工程，但其核心往往落在**查询优化**上。一个糟糕的查询可能会让一台顶配服务器的资源耗尽，而一个经过优化的查询则可能在毫秒间完成。本节将介绍最基本、也最有效的查询优化技巧。

---

### 一、索引 (Indexing)：最有效的优化手段

**索引是提高查询性能最重要、最直接的工具。** 它的作用是让数据库在查找数据时，不必扫描整张表（Seq Scan），而是像查字典一样，通过一个排好序的结构快速定位到目标数据。

#### 1. 何时创建索引？
-   **`WHERE` 子句中的列**：最常见的场景。`WHERE user_id = 123` 或 `WHERE created_at > '2025-01-01'`。
-   **`JOIN` 操作的连接键**：在 `ON a.id = b.a_id` 中的 `a.id` 和 `b.a_id` 列上都应该有索引。
-   **`ORDER BY` 子句中的列**：为排序列创建索引，可以避免昂贵的“文件排序”（Filesort）操作。
-   **`GROUP BY` 子句中的列**：同样有助于优化器快速找到分组数据。

#### 2. 索引并非越多越好
-   **写操作开销**：每次 `INSERT`, `UPDATE`, `DELETE`，数据库都需要额外地更新索引，这会降低写入性能。
-   **存储开销**：索引本身也占用磁盘空间。

**原则**：只为那些能显著提升**读查询**性能的列创建索引。

#### 3. 复合索引 (Composite Indexes)
当查询条件经常涉及多个列时，应该创建一个包含这些列的复合索引。
```sql
-- 经常执行这样的查询:
-- SELECT * FROM users WHERE city = 'New York' AND status = 'active';

-- 应该创建一个复合索引
CREATE INDEX idx_users_city_status ON users (city, status);
```
**重要**：复合索引中列的顺序很重要。通常，应该将**选择性最高**（区分度最大）的列放在前面。

#### 4. `EXPLAIN`：你的优化导航仪
`EXPLAIN` 命令是查询优化的起点。它会显示 PostgreSQL 的查询规划器为你的 SQL 选择的执行计划。
```sql
EXPLAIN ANALYZE SELECT * FROM users WHERE email = 'test@example.com';
```
-   **`Seq Scan` (Sequential Scan)**：全表扫描。如果在一个大表上看到它，通常意味着缺少索引。
-   **`Index Scan` / `Bitmap Heap Scan`**: 索引扫描。这是我们希望看到的。
-   **`Sort`**: 排序操作。如果 `ORDER BY` 的列没有索引，就会出现。
-   **`Nested Loop`, `Hash Join`, `Merge Join`**: 不同的 `JOIN` 算法。

---

### 二、查询重写 (Query Rewriting)

有时，即使有索引，SQL 本身的写法也可能导致性能低下。重写查询，使其对查询规划器更“友好”，是另一种重要的优化技巧。

#### 1. 避免在 `WHERE` 子句的列上使用函数
**反例（慢）：**
```sql
SELECT * FROM logs WHERE date_trunc('day', log_time) = '2025-07-10';
```
这会导致 `log_time` 列上的索引失效，因为数据库必须为**每一行**计算 `date_trunc` 的结果。

**正例（快）：**
```sql
SELECT * FROM logs
WHERE log_time >= '2025-07-10' AND log_time < '2025-07-11';
```
这个查询将函数操作移到了常量一侧，使得查询可以直接利用 `log_time` 列上的 B-Tree 索引进行高效的范围扫描。

#### 2. 使用 `EXISTS` 代替 `IN`
当子查询返回大量数据时，`EXISTS` 通常比 `IN` 更高效。
**反例（可能慢）：**
```sql
SELECT * FROM products WHERE category_id IN (SELECT id FROM categories WHERE active = true);
```
**正例（通常更快）：**
```sql
SELECT * FROM products p
WHERE EXISTS (
    SELECT 1 FROM categories c WHERE c.id = p.category_id AND c.active = true
);
```
`EXISTS` 只要找到第一个匹配项就会停止，而 `IN` 通常需要处理整个子查询的结果集。

#### 3. 使用 `UNION ALL` 代替 `OR`
对于某些复杂的 `OR` 条件，查询规划器可能难以优化。将其拆分为两个独立的查询并用 `UNION ALL` 连接，有时能让每个子查询都用上最优的索引。
**反例（可能慢）：**
```sql
SELECT * FROM users WHERE city = 'New York' OR status = 'pending';
```
**正例（可能更快）：**
```sql
SELECT * FROM users WHERE city = 'New York'
UNION ALL
SELECT * FROM users WHERE status = 'pending' AND city <> 'New York';
```
*(注意第二个查询需要排除第一个查询已包含的情况)*

---

### 三、理解和利用缓存

PostgreSQL 有多层缓存，理解它们有助于解释为什么一个查询第二次运行时会比第一次快得多。
1.  **PostgreSQL 共享缓冲区 (`shared_buffers`)**: 这是 PostgreSQL 在内存中缓存数据页（表和索引）的地方。如果查询所需的数据页都在 `shared_buffers` 中，就可以避免从磁盘读取，速度极快。
2.  **操作系统文件系统缓存**: 操作系统自身也会缓存最近访问过的文件。
3.  **查询缓存**: PostgreSQL 会缓存查询计划，但**不会**缓存查询结果。

**我们能做什么？**
-   **预热（Warm-up）**：在应用高峰期到来之前，可以主动执行一些常用查询，将热点数据和索引加载到缓存中。
-   **调整 `shared_buffers`**: 这是最重要的内存参数之一。合理配置它的大小（通常是系统内存的 25%）可以显著提升性能。

---

## 📌 小结

-   **索引是第一道防线**：面对慢查询，首先检查 `WHERE`, `JOIN`, `ORDER BY` 的列上是否有合适的索引。
-   **`EXPLAIN` 是诊断工具**：学会阅读 `EXPLAIN` 的输出，找出性能瓶颈（如 `Seq Scan`, `Sort`）。
-   **重写是良药**：避免在列上使用函数，优先使用对索引友好的查询模式。
-   **缓存是加速器**：理解缓存的存在，并合理配置 `shared_buffers`。

掌握这些基础技巧，你就能够解决 80% 以上的数据库慢查询问题。在下一节，我们将深入探讨如何通过调整 PostgreSQL 的核心配置参数来进行全局性能调优。
