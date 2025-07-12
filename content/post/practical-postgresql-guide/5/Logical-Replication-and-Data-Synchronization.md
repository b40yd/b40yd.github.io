+++
title = "PostgreSQL实战指南：多领域数据库应用与实践 - 第20章：逻辑复制与数据同步"
date = 2025-07-12
lastmod = 2025-07-12T10:55:00+08:00
tags = ["PostgreSQL", "practical", "guide", "book", "distributed", "replication", "synchronization"]
categories = ["PostgreSQL", "practical", "guide", "book", "distributed"]
draft = false
author = "b40yd"
+++

## PostgreSQL实战指南：多领域数据库应用与实践

-----

### 第五部分：PostgreSQL与分布式数据库

虽然 PostgreSQL 是一个单机数据库的典范，但其强大的生态系统提供了多种构建分布式解决方案的途径。本部分将探讨如何通过逻辑复制、外部数据封装器（FDW）以及 CitusData、Greenplum 等专业扩展，将 PostgreSQL 从单机扩展到分布式集群，以应对海量数据和高并发的挑战。

-----

#### 第20章：逻辑复制与数据同步

逻辑复制是 PostgreSQL 10 引入的一项强大功能，它允许将数据更改从一个数据库（发布方）流式传输到另一个数据库（订阅方）。与基于 WAL（预写日志）物理块级别差异的物理复制不同，逻辑复制是基于数据行的逻辑更改（INSERT, UPDATE, DELETE），这为数据同步和集成提供了极大的灵活性。

##### 20.1 逻辑复制 vs 物理复制

| 特性 | 物理复制 (Streaming Replication) | 逻辑复制 (Logical Replication) |
| :--- | :--- | :--- |
| **复制单位** | 整个数据库集群的物理块 | 逻辑数据更改（行级别） |
| **数据格式** | 二进制，与发布方完全相同 | 逻辑表示，可以被订阅方以不同方式解释 |
| **选择性** | 只能复制整个集群 | 可以选择性地复制某些表，甚至某些行或列 |
| **版本/平台** | 要求主备服务器的 PostgreSQL 主版本、操作系统和架构完全相同 | 不要求，订阅方可以是不同主版本、不同操作系统，甚至可以是其他数据库 |
| **灵活性** | 低，备库是主库的只读镜像 | 高，订阅方是可写的，可以有自己的数据和索引 |
| **主要用途** | 高可用性（HA）、灾难恢复（DR）、读扩展 | 数据集成、零停机升级、构建读写分离 |

##### 20.2 发布/订阅模型

逻辑复制的工作方式基于发布/订阅模型。

- **发布 (Publication)**: 在源数据库（发布方）上创建一个 `PUBLICATION` 对象，它定义了要发布哪些表的哪些更改（`INSERT`, `UPDATE`, `DELETE`）。
- **订阅 (Subscription)**: 在目标数据库（订阅方）上创建一个 `SUBSCRIPTION` 对象，它定义了如何连接到发布方以及要订阅哪个 `PUBLICATION`。

**配置要求:**

1.  在发布方的 `postgresql.conf` 中设置 `wal_level = logical`。
2.  在发布方的 `pg_hba.conf` 中允许来自订阅方的复制连接。
3.  发布方和订阅方上的表结构必须兼容（至少订阅方需要包含所有被发布的列）。

##### 20.3 场景实战：构建读写分离与数据聚合

**业务场景描述:**

我们有一个高并发的在线交易处理系统（OLTP），数据库压力很大。为了分担压力，我们希望将报表和分析查询（OLAP）迁移到一个独立的报表数据库上。我们需要将主库（OLTP）的 `orders` 和 `order_items` 表的更改实时同步到报表数据库。

**步骤1：在发布方（主库）上操作**

```sql
-- 1. 检查 wal_level (需要重启才能生效)
SHOW wal_level; -- 应该是 logical

-- 2. 创建一个 PUBLICATION，发布 orders 和 order_items 表的所有更改
CREATE PUBLICATION report_publication FOR TABLE orders, order_items;

-- 如果只想发布 INSERT 操作
-- CREATE PUBLICATION insert_only_pub FOR TABLE orders WITH (publish = 'insert');
```

**步骤2：在订阅方（报表库）上操作**

```sql
-- 1. 确保表结构存在且兼容
-- 通常，你需要先将主库的表结构 dump 出来，在报表库上恢复
CREATE TABLE orders (
    order_id SERIAL PRIMARY KEY,
    user_id INTEGER NOT NULL,
    order_date TIMESTAMPTZ DEFAULT NOW(),
    total_amount NUMERIC(10, 2) NOT NULL,
    status TEXT -- 在报表库，我们可以使用更简单的 TEXT 类型
);

CREATE TABLE order_items (
    order_item_id SERIAL PRIMARY KEY,
    order_id INTEGER NOT NULL,
    product_id INTEGER NOT NULL,
    quantity INTEGER NOT NULL,
    unit_price NUMERIC(10, 2) NOT NULL
);

-- 2. 创建一个 SUBSCRIPTION，连接到主库并订阅 report_publication
CREATE SUBSCRIPTION report_subscription
    CONNECTION 'host=primary_host port=5432 user=replicator password=secret dbname=oltp_db'
    PUBLICATION report_publication;
```

**验证与管理:**

- 在主库上对 `orders` 或 `order_items` 表进行增删改操作。
- 稍等片刻，在报表库上查询，你会发现数据已经被同步过来。
- 在订阅方，你可以为这些表创建额外的索引、物化视图，甚至添加不属于发布内容的列，而不会影响复制。

```sql
-- 在报表库上，为分析查询创建额外的索引
CREATE INDEX idx_orders_order_date ON orders (order_date);
```

**管理订阅:**

```sql
-- 暂停订阅
ALTER SUBSCRIPTION report_subscription DISABLE;

-- 恢复订阅
ALTER SUBSCRIPTION report_subscription ENABLE;

-- 查看订阅状态
SELECT * FROM pg_stat_subscription;
```

##### 20.4 逻辑复制的其他应用场景

- **零停机时间的主版本升级**:
    1.  在新的 PostgreSQL 版本上建立一个新服务器。
    2.  设置从旧服务器到新服务器的逻辑复制。
    3.  等待数据完全同步。
    4.  在短暂的维护窗口内，将应用程序的连接切换到新服务器。
- **数据集成**: 将多个源数据库的数据聚合到一个中央数据仓库中。每个源数据库都可以是一个发布方，中央仓库是订阅方。
- **微服务架构**: 在微服务架构中，每个服务可能都有自己的数据库。逻辑复制可以用于在不同服务之间可靠地同步数据事件。

##### 20.5 总结

本章我们深入学习了 PostgreSQL 的逻辑复制功能。我们理解了它与物理复制的根本区别，掌握了其核心的发布/订阅模型，并通过一个构建读写分离报表库的实战案例，完整地实践了从配置、创建到管理的全过程。逻辑复制的灵活性使其成为实现数据库零停机升级、数据集成和构建复杂分布式架构的基石。

在下一章，我们将学习另一种强大的数据集成技术——外部数据封装器（FDW），它能让你像访问本地表一样去查询远程的、甚至是异构的数据库。
-----
