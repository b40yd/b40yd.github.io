+++
title = "第十五章 Citus 扩展实战 - 第二节：分片策略与表共置"
date = 2025-07-10
lastmod = 2025-07-10T01:00:00+08:00
tags = ["PostgreSQL", "practical", "book", "distributed", "citus", "sharding", "co-location"]
categories = ["PostgreSQL", "practical", "book"]
draft = false
author = "b40yd"
+++

## 第二节 分片策略（hash, range, append）与表共置

> **目标**：深入理解 Citus 中不同类型的分布式表，特别是哈希（hash-distributed）和引用（reference）表，并掌握实现高性能分布式 JOIN 的关键概念——共置（Co-location）。

在上一节，我们使用 `create_distributed_table` 创建了一个分布式表，Citus 默认使用了**哈希分片（Hash-based Sharding）**。然而，Citus 还提供了其他类型的表，以适应不同的查询场景。

---

### 一、分布式表的类型

#### 1. 哈希分布式表 (Hash-Distributed Tables)

这是最常用、最核心的表类型。
-   **工作原理**：Citus 对指定的**分布列**的值计算一个哈希，然后根据这个哈希值将每一行数据路由到一个特定的分片（Shard）中。
-   **适用场景**：需要水平扩展的大型事实表，如多租户应用中的 `users`, `orders` 表，或实时分析中的 `events` 表。
-   **创建方法**：
    ```sql
    -- 这是默认行为
    SELECT create_distributed_table('table_name', 'distribution_column');
    ```

#### 2. 引用表 (Reference Tables)

-   **工作原理**：引用表的数据被**完整地复制**到**每一个工作节点**上。
-   **适用场景**：通常是那些**较小的、不经常变动、但需要频繁与大型分布式表进行 JOIN 的维度表**。例如，存储国家代码的 `countries` 表，存储商品类别的 `categories` 表。
-   **创建方法**：
    ```sql
    -- 创建一个普通的表
    CREATE TABLE categories (
        category_id INT PRIMARY KEY,
        category_name TEXT
    );
    -- 将其声明为引用表
    SELECT create_reference_table('categories');
    ```
-   **优势**：当一个大型分布式表需要与一个引用表进行 JOIN 时，这个 JOIN 操作可以**完全在工作节点内部完成**，因为每个工作节点上都有一份引用表数据的完整拷贝。这避免了昂贵的跨节点网络通信。

#### 3. 本地表 (Local Tables)

-   **工作原理**：这些是只存在于**协调器节点**上的普通 PostgreSQL 表。它们不参与分布式查询。
-   **适用场景**：用于存储集群管理、用户认证等与分布式数据无关的管理型数据。

---

### 二、共置 (Co-location)：高性能分布式 JOIN 的基石

**共置**是 Citus 中一个极其重要的概念，也是实现高性能分布式 `JOIN` 的关键。

**定义**：如果两个（或多个）哈希分布式表使用**相同类型**的分布列，并且它们的**分片数量相同**，那么 Citus 会确保**具有相同分布列值的数据行，总是被存储在同一个工作节点上**。这种情况就称为“表是共置的”。

#### 实战示例：多租户应用

在一个多租户 SaaS 应用中，我们有 `companies` 表和 `products` 表，它们都以 `company_id` 作为分布列。

```sql
-- 公司表
CREATE TABLE companies (
    company_id BIGINT PRIMARY KEY,
    name TEXT
);
SELECT create_distributed_table('companies', 'company_id');

-- 产品表
CREATE TABLE products (
    product_id BIGSERIAL,
    company_id BIGINT, -- 与 companies 表的分布列类型相同
    name TEXT,
    PRIMARY KEY (product_id, company_id)
);
SELECT create_distributed_table('products', 'company_id');
```

因为 `companies` 和 `products` 表都是用 `company_id` 分布的，Citus 会保证：
-   ID 为 5 的公司信息，和所有 `company_id = 5` 的产品信息，**一定会被存储在同一个工作节点上**。

**这带来了巨大的好处：**
当我们执行一个 `JOIN` 查询时：
```sql
SELECT c.name, p.name
FROM companies c
JOIN products p ON c.company_id = p.company_id
WHERE c.company_id = 5;
```
Citus 足够智能，它知道这个查询需要的所有数据都在一个节点上。因此，它会将整个 `JOIN` 操作**下推（push down）**到那个工作节点去执行。查询完全在节点内部完成，避免了任何跨节点的数据传输，其性能几乎与在单机上执行 `JOIN` 一样快。

**不共置的后果：**
如果 `products` 表是按 `product_id` 分布的，那么执行上述 `JOIN` 时，Citus 将不得不：
1.  从一个节点获取 `company_id = 5` 的公司数据。
2.  从**所有**其他节点捞取 `products` 数据。
3.  将这些数据全部传输回协调器。
4.  在协调器上完成最终的 `JOIN`。
这个过程涉及大量的网络 I/O，性能会急剧下降。

---

## 📌 小结

| 表类型 | 数据分布 | 适用场景 |
| :--- | :--- | :--- |
| **哈希分布式** | 按分布列的哈希值分散到各节点 | 大型事实表，需要水平扩展 |
| **引用表** | 完整复制到每个工作节点 | 较小的、不常变动的维度表 |
| **本地表** | 只存在于协调器 | 集群管理、非业务数据 |

-   **共置（Co-location）**是 Citus 性能优化的核心。在设计 Schema 时，应尽可能地让需要相互 `JOIN` 的大型分布式表使用相同的分布列，以实现共置。
-   使用**引用表**来存储需要与大型分布式表 `JOIN` 的小表，是另一种实现高效 `JOIN` 的重要策略。

理解并善用这些分片策略和共置原则，是构建高性能 Citus 分布式数据库集群的关键。
