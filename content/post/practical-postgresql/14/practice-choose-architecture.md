+++
title = "第十四章 PostgreSQL 的分布式方案概览 - 第三节 实战：选择适合业务场景的分布式架构"
date = 2025-07-10
lastmod = 2025-07-10T01:00:00+08:00
tags = ["PostgreSQL", "practical", "book", "distributed", "citus", "architecture"]
categories = ["PostgreSQL", "practical", "book"]
draft = false
author = "b40yd"
+++

## 第三节 实战：选择适合业务场景的分布式架构

> **目标**：通过分析几个典型的业务场景，学习如何做出关键的架构决策：何时需要从单机 PostgreSQL 转向分布式，以及在分布式场景下如何选择最合适的分布列（Distribution Column）。

选择正确的数据库架构是系统设计的基石。过早地选择分布式会带来不必要的复杂性，而过晚地转向分布式则可能导致业务因性能问题而停滞。

---

### 决策点一：真的需要分布式吗？

在考虑 Citus 之前，请先确认是否已将**单机 PostgreSQL** 的潜力挖掘到极致。

**检查清单：**
1.  **垂直扩展（Scale-Up）**：当前服务器的 CPU、内存、磁盘 IOPS 是否还有提升空间？升级到更强大的硬件通常是最简单、最经济的解决方案。
2.  **性能调优**：是否已经对慢查询进行了彻底的分析和优化？是否建立了合适的索引？`postgresql.conf` 的核心参数（如 `shared_buffers`, `work_mem`）是否已合理配置？
3.  **读写分离**：如果瓶颈在于大量的读请求，是否可以先尝试使用 PostgreSQL 内置的流复制（Streaming Replication）来搭建一个或多个只读副本，从而分散读取压力？

**结论**：只有当**数据量（Storage）**本身大到单机无法存储，或者**写入吞吐量（Write Throughput）**和**并行计算需求（CPU-bound queries）**超过了最强单机的处理能力时，才真正需要考虑像 Citus 这样的分布式方案。

---

### 决策点二：如何选择分布列？

一旦决定使用 Citus，选择**分布列**是**最重要**的一个决策，它直接决定了集群的性能和扩展能力。

选择分布列的核心原则是：**让最频繁的查询能够被路由到单个工作节点上，同时让数据尽可能均匀地分布在所有工作节点之间。**

---

### 场景分析

#### 场景一：B2B 多租户 SaaS 平台

-   **业务描述**：一个为企业客户提供服务的 SaaS 应用（如 CRM, ERP）。每个企业（租户）拥有自己的一套数据（用户、订单、产品等），应用的大部分查询都包含 `tenant_id` 或 `company_id`。
-   **数据特点**：租户间数据隔离，查询高度本地化。
-   **最佳分布列**：`tenant_id` (或 `company_id`)。
-   **表设计**：应用中**所有**的表，都应该包含 `tenant_id` 这一列，并将其作为分布列。

```sql
-- 用户表
CREATE TABLE users (
    tenant_id TEXT NOT NULL,
    user_id BIGSERIAL,
    email TEXT,
    PRIMARY KEY (tenant_id, user_id)
);
SELECT create_distributed_table('users', 'tenant_id');

-- 订单表
CREATE TABLE orders (
    tenant_id TEXT NOT NULL,
    order_id BIGSERIAL,
    user_id BIGINT,
    amount NUMERIC,
    PRIMARY KEY (tenant_id, order_id)
);
SELECT create_distributed_table('orders', 'tenant_id');
```

-   **查询示例**：`SELECT * FROM orders WHERE tenant_id = 'acme' AND user_id = 123;`
-   **优势**：
    -   此查询会被直接路由到存储着 'acme' 公司数据的那个工作节点，延迟极低。
    -   `users` 表和 `orders` 表在同一个租户下的数据会被**共置（Co-located）**在同一个工作节点上，这意味着在这两张表之间进行 `JOIN` 操作时，可以直接在工作节点内部完成，无需跨节点网络通信，速度飞快。

#### 场景二：用户行为分析平台

-   **业务描述**：一个记录网站或 App 用户点击流、浏览历史等事件的系统。需要对海量事件进行实时聚合分析，例如计算日活跃用户（DAU）、分析用户转化漏斗等。
-   **数据特点**：数据量极大，写入频繁，查询多为大型聚合。
-   **最佳分布列**：`user_id`。
-   **表设计**：

```sql
CREATE TABLE events (
    user_id BIGINT NOT NULL,
    event_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    event_type TEXT,
    event_time TIMESTAMPTZ DEFAULT NOW(),
    properties JSONB
);
SELECT create_distributed_table('events', 'user_id');
```

-   **查询示例**：`SELECT event_type, count(*) FROM events WHERE event_time > NOW() - INTERVAL '1 day' GROUP BY event_type;`
-   **优势**：
    -   **写入扩展**：数据写入可以均匀地分散到所有工作节点，避免了单点写入瓶颈。
    -   **查询并行化**：上面的 `GROUP BY` 查询会被下推到所有工作节点上**并行执行**。每个 worker 计算一小部分数据的聚合结果，然后协调器只需对这些部分结果进行最终合并，极大地缩短了大型分析查询的响应时间。

#### 场景三：物联网（IoT）时间序列数据

-   **业务描述**：大量设备持续不断地上报时序数据。
-   **数据特点**：兼具“实时分析”和“多租户”（多设备）的特点。
-   **最佳实践**：**Citus + TimescaleDB**。
-   **架构**：
    1.  首先使用 TimescaleDB 将数据表转换为 Hypertable，实现按 `time` 的自动分区。
    2.  然后使用 Citus 将这个 Hypertable 进一步按 `device_id` 进行分布式。
-   **优势**：这种组合拳可以同时获得两种扩展的好处：TimescaleDB 提供了极致的时序数据管理能力（压缩、保留策略、时序函数），而 Citus 提供了集群的水平扩展能力。

---

## 📌 小结

| 场景 | 核心瓶颈 | 推荐方案 | 关键分布列 |
| :--- | :--- | :--- | :--- |
| **读多写少** | 读取压力 | 单机 + 只读副本 | (不适用) |
| **多租户 SaaS** | 租户数量/数据隔离 | Citus | `tenant_id` |
| **实时分析** | 写入吞吐/聚合查询速度 | Citus | `user_id` / `entity_id` |
| **物联网/时序** | 海量写入/时序分析 | Citus + TimescaleDB | `device_id` |

做出正确的架构决策需要深入理解业务的查询模式和数据的分布特点。在选择分布式方案时，**分布列的选择是重中之重**，它决定了你的分布式集群能否真正发挥出强大的威力。
