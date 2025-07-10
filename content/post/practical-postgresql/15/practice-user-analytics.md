+++
title = "第十五章 Citus 扩展实战 - 第四节 实战：用户行为数据的分布式统计分析"
date = 2025-07-10
lastmod = 2025-07-10T01:00:00+08:00
tags = ["PostgreSQL", "practical", "book", "distributed", "citus", "analytics"]
categories = ["PostgreSQL", "practical", "book"]
draft = false
author = "b40yd"
+++

## 第四节 实战：用户行为数据的分布式统计分析

> **目标**：设计一个用于存储和分析海量用户行为事件的分布式数据仓库，综合运用 Citus 的哈希分片、引用表、共置和并行聚合能力，解决真实世界中的大数据分析挑战。

### 场景描述

我们正在为一个大型电商网站构建一个用户行为分析平台。该平台需要记录用户的每一次点击、浏览、加购、购买等事件，并支持对这些数据进行快速的统计分析，以生成业务报表。

**数据与查询需求：**
1.  **海量事件数据**：每天产生数十亿条事件记录。
2.  **用户信息**：需要关联用户的注册信息（如所在城市）。
3.  **商品信息**：需要关联商品的基本信息（如品类）。
4.  **核心查询**：
    -   按城市统计每日的独立访客数（DAU）。
    -   按商品品类统计每周的销售总额。
    -   下钻分析特定用户的完整行为路径。

---

## 🏛️ 第一步：设计分布式数据模型

根据业务需求，我们设计三张核心表。

**1. `events` 表 (哈希分布式)**
这是最大的事实表，存储所有用户行为事件。选择 `user_id` 作为分布列是最佳选择，因为它可以让同一个用户的所有事件都存储在同一个节点上，便于路径分析，同时也能让大型聚合查询均匀地分布到所有节点。

```sql
CREATE TABLE events (
    event_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id BIGINT NOT NULL,
    event_type TEXT NOT NULL,
    product_id BIGINT,
    event_time TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    properties JSONB
);

-- 使用 user_id 作为分布列
SELECT create_distributed_table('events', 'user_id');
```

**2. `users` 表 (哈希分布式 & 共置)**
存储用户信息。为了能与 `events` 表进行高效的 `JOIN`，我们也必须使用 `user_id` 作为分布列，以实现**共置**。

```sql
CREATE TABLE users (
    user_id BIGINT PRIMARY KEY,
    registration_date DATE,
    city TEXT
);

-- 同样使用 user_id 作为分布列
SELECT create_distributed_table('users', 'user_id');
```

**3. `products` 表 (引用表)**
商品信息表相对较小，且不经常变动，但需要频繁与 `events` 表进行 `JOIN`。因此，将其设为**引用表**是理想的选择。

```sql
CREATE TABLE products (
    product_id BIGINT PRIMARY KEY,
    category TEXT,
    name TEXT
);

-- 声明为引用表
SELECT create_reference_table('products');
```

---

## ✍️ 第二步：模拟数据加载

```sql
-- 插入用户和商品
INSERT INTO users (user_id, city) SELECT g, 'city-' || (g % 10) FROM generate_series(1, 1000) g;
INSERT INTO products (product_id, category) SELECT g, 'category-' || (g % 20) FROM generate_series(1, 10000) g;

-- 插入大量事件
INSERT INTO events (user_id, event_type, product_id)
SELECT
    (random() * 999 + 1)::int,
    'view',
    (random() * 9999 + 1)::int
FROM generate_series(1, 1000000);
```

---

## 🔍 第三步：执行分布式分析查询

#### 查询 1：按城市统计日活跃用户 (DAU)

这个查询需要 `JOIN` 共置的 `events` 和 `users` 表。Citus 会将 `JOIN` 和聚合操作都下推到工作节点并行执行。

```sql
EXPLAIN VERBOSE
SELECT
    u.city,
    count(DISTINCT e.user_id) AS dau
FROM events e
JOIN users u ON e.user_id = u.user_id
WHERE e.event_time >= date_trunc('day', NOW())
GROUP BY u.city
ORDER BY dau DESC
LIMIT 10;
```
`EXPLAIN` 输出会显示一个复杂的分布式计划，但核心是 `JOIN` 和 `GROUP BY` 都在工作节点上完成。

#### 查询 2：按商品品类统计销售额

这个查询需要 `JOIN` 分布式表 `events` 和引用表 `products`。`JOIN` 同样可以在工作节点上高效完成。

```sql
-- 假设 'purchase' 事件的 properties 字段包含价格
-- INSERT INTO events ... VALUES (..., 'purchase', ..., '{"price": 99.99}');

EXPLAIN VERBOSE
SELECT
    p.category,
    sum((e.properties->>'price')::numeric) AS total_revenue
FROM events e
JOIN products p ON e.product_id = p.product_id
WHERE e.event_type = 'purchase'
GROUP BY p.category
ORDER BY total_revenue DESC
LIMIT 10;
```

#### 查询 3：下钻分析单个用户的行为路径

由于 `events` 表是按 `user_id` 分布的，查询单个用户的行为会非常快，因为它只会被路由到单个工作节点。

```sql
SELECT event_type, product_id, event_time
FROM events
WHERE user_id = 123
ORDER BY event_time;
```
`EXPLAIN` 输出会显示这是一个单分片查询（`Task Count: 1`）。

---

## 📌 小结

本实战案例展示了如何为一个典型的大数据分析场景设计 Citus 分布式数据模型。
1.  **识别核心实体**：识别出最大的事实表（`events`）和需要关联的维度表（`users`, `products`）。
2.  **选择分布策略**：
    -   对最大的事实表（`events`）和需要与之共置的表（`users`），选择一个能最大化查询下推和并行化的**分布列**（`user_id`）。
    -   对需要频繁 `JOIN` 的小表（`products`），使用**引用表**来消除跨节点网络开销。
3.  **验证查询计划**：通过 `EXPLAIN` 确认 `JOIN` 和聚合操作是否如预期一样被高效地并行执行。

通过这种精心设计，我们可以利用 Citus 将 PostgreSQL 集群的能力发挥到极致，以极高的性价比构建一个能够处理海量数据、支持快速分析的强大数据平台。
