+++
title = "第十五章 Citus 扩展实战 - 第三节：分布式 JOIN 与聚合查询"
date = 2025-07-10
lastmod = 2025-07-10T01:00:00+08:00
tags = ["PostgreSQL", "practical", "book", "distributed", "citus", "join", "aggregate"]
categories = ["PostgreSQL", "practical", "book"]
draft = false
author = "b40yd"
+++

## 第三节 分布式 JOIN 与聚合查询

> **目标**：理解 Citus 如何处理和优化分布式环境下的 `JOIN` 和聚合查询，并学会通过 `EXPLAIN` 命令来分析查询计划，判断查询是在工作节点上并行执行还是在协调器上执行。

在分布式数据库中，`JOIN` 和聚合（`GROUP BY`）是最能体现其并行计算能力的查询类型，但也是最考验其查询规划器智能性的地方。Citus 的查询规划器会根据表的类型、分布方式和查询条件，来决定最高效的执行策略。

---

### 一、分布式 JOIN 的执行策略

Citus 处理 `JOIN` 的策略，很大程度上取决于被连接的表是否**共置（Co-located）**。

#### 策略 1：下推执行 (Push-Down Execution) - 最高效

-   **触发条件**：
    1.  两个哈希分布式表是**共置**的，并且 `JOIN` 条件中包含了它们的**分布列**。
    2.  一个哈希分布式表与一个**引用表**进行 `JOIN`。
-   **执行过程**：Citus 将整个 `JOIN` 查询下推到所有相关的工作节点上**并行执行**。每个工作节点在本地完成 `JOIN`，然后将结果返回给协调器。
-   **优势**：避免了跨节点的数据传输，性能极高。

**示例（共置表 JOIN）：**
```sql
-- companies 和 products 都按 company_id 分布
EXPLAIN SELECT c.name, p.name
FROM companies c
JOIN products p ON c.company_id = p.company_id
WHERE c.company_id = 5;
```
`EXPLAIN` 的输出会显示一个 `Distributed Query`，并且其子任务是下推到单个工作节点上执行的。

**示例（与引用表 JOIN）：**
```sql
-- products 是分布式表，categories 是引用表
EXPLAIN SELECT p.name, c.category_name
FROM products p
JOIN categories c ON p.category_id = c.category_id;
```
`EXPLAIN` 的输出会显示一个并行执行的计划，`JOIN` 操作在每个工作节点上独立完成。

#### 策略 2：重新分区查询 (Repartition Query)

-   **触发条件**：两个哈希分布式表**不共置**，或者 `JOIN` 条件中不包含分布列。
-   **执行过程**：这是 Citus 最智能但也最复杂的操作之一。
    1.  协调器会选择其中一个（通常是较小的）表。
    2.  动态地将这个表的数据**重新分区（Repartition）**，并将其发送到需要的其他工作节点上，使其临时“共置”。
    3.  在工作节点上执行 `JOIN`。
-   **代价**：涉及大量的跨节点网络数据传输，性能远低于“下推执行”。

**示例（非共置表 JOIN）：**
```sql
-- orders 按 customer_id 分布, products 按 product_id 分布
EXPLAIN SELECT *
FROM orders o
JOIN products p ON o.product_id = p.product_id;
```
`EXPLAIN` 的输出会明确显示一个 `Repartition` 步骤。

---

### 二、分布式聚合查询

Citus 的并行计算能力在大型聚合查询中体现得淋漓尽致。

#### 场景：计算每个商品类别的总销售额

```sql
-- events 是一个大型分布式表，按 user_id 分布
EXPLAIN SELECT
    properties->>'category' AS category,
    count(*) AS num_events,
    sum((properties->>'price')::numeric) AS total_revenue
FROM events
GROUP BY category;
```

**执行过程：**
1.  **Map 阶段**：协调器将 `GROUP BY` 查询下推到**所有**工作节点。
2.  每个工作节点并行地扫描自己的分片，计算出一个**部分聚合结果**（例如，worker1 算出 'Electronics' 类别的部分总和是 1000，worker2 算出是 1500）。
3.  **Reduce 阶段**：所有工作节点将它们的部分结果返回给协调器。
4.  协调器对这些部分结果进行最终的合并（例如，`1000 + 1500 = 2500`），得出全局的最终结果。

这种 **Map-Reduce** 风格的并行执行，使得对海量数据的复杂聚合分析，能够从数小时缩短到数分钟甚至数秒。

---

### 三、使用 `EXPLAIN` 分析分布式查询

`EXPLAIN` 命令是优化 Citus 查询的必备工具。

**如何解读 `EXPLAIN` 输出：**
-   **`Distributed Query`**: 表明这是一个分布式查询计划。
-   **`Task Count`**: 显示查询被分解成了多少个任务，下推到了多少个工作节点。
-   **`-> Seq Scan on table_name ...`**: 如果直接出现在顶层，表明这是一个在协调器上执行的查询。
-   **`-> Custom Scan (Citus)`**: 表明这是一个被下推到工作节点上执行的任务。
-   **`Repartition` / `Broadcast`**: 留意这些关键词，它们意味着发生了跨节点的数据传输，可能是潜在的性能瓶颈。

**示例：**
```sql
EXPLAIN VERBOSE SELECT * FROM products WHERE product_id = 1;
```
输出会显示这是一个单分片查询（`Task Count: 1`），并且被路由到了一个特定的工作节点上。

---

## 📌 小结

-   Citus 处理 `JOIN` 的性能，关键在于**共置**。应尽力设计你的 Schema 以实现共置，从而启用最高效的**下推执行**。
-   对于无法下推的 `JOIN`，Citus 会尝试进行**重新分区查询**，但这会带来显著的网络开销。
-   Citus 通过 **Map-Reduce** 风格的并行执行，极大地加速了大型聚合查询。
-   **`EXPLAIN`** 是你最好的朋友。通过分析查询计划，你可以洞察 Citus 的执行策略，并找到优化的方向。

理解 Citus 的查询处理机制，可以帮助你编写出对分布式环境更“友好”的 SQL，从而最大限度地发挥集群的并行计算能力。
