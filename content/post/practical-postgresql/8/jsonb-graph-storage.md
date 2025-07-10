+++
title = "第八章 使用 JSON + SQL 模拟图模型 - 第一节：使用 JSONB 存储节点与边关系"
date = 2025-07-10
lastmod = 2025-07-10T01:00:00+08:00
tags = ["PostgreSQL", "practical", "book", "graph", "jsonb", "sql"]
categories = ["PostgreSQL", "practical", "book"]
draft = false
author = "b40yd"
+++

# 第三部分：NoSQL 能力扩展实战
## 第八章 使用 JSON + SQL 模拟图模型
### 第一节 使用 JSONB 存储节点与边关系

> **目标**：学习一种不依赖 `ltree` 或 Apache AGE 等专门扩展，仅使用 PostgreSQL 内核功能（特别是 `JSONB` 和递归查询）来模拟图数据模型的方法。

虽然 Apache AGE 提供了完整的图数据库体验，但在某些场景下，我们可能因为以下原因无法使用它：
-   **环境限制**：无法在生产环境中安装第三方扩展。
-   **轻量级需求**：图模型相对简单，引入一个完整的图数据库扩展显得“杀鸡用牛刀”。
-   **数据一致性**：希望将图的边信息与节点自身的数据更紧密地耦合在一起。

在这种情况下，我们可以利用 PostgreSQL 强大的 `JSONB` 数据类型和 SQL 功能，来巧妙地模拟一个图数据库。这种方法的核心思想是：**用表来存节点，用 `JSONB` 字段来存边**。

---

### 数据建模思路

我们将创建一个 `nodes` 表来存储所有的图节点（人、公司等）。表中的每一行代表一个节点。

-   **节点属性**：使用 `JSONB` 类型的 `properties` 字段来存储节点的各种属性（如姓名、年龄），这提供了极大的灵活性。
-   **边关系**：同样使用 `JSONB` 类型的 `edges` 字段来存储该节点出发的所有**出向边（Outgoing Edges）**。

---

## 🏛️ 第一步：设计数据表

```sql
CREATE TABLE graph_nodes (
    id BIGSERIAL PRIMARY KEY,
    label TEXT NOT NULL, -- 节点的标签，如 'Person', 'Company'
    properties JSONB,
    edges JSONB
);

-- 为了加速节点查找，在标签和特定属性上创建索引
CREATE INDEX idx_nodes_label ON graph_nodes(label);
CREATE INDEX idx_nodes_properties_gin ON graph_nodes USING GIN (properties);

-- 为了加速边的查找，在 edges 字段上创建 GIN 索引
CREATE INDEX idx_nodes_edges_gin ON graph_nodes USING GIN (edges);
```

**`edges` 字段的 `JSONB` 结构设计：**
我们把 `edges` 设计成一个 JSON 数组，数组中的每个对象代表一条边。

```json
[
    {
        "type": "FRIENDS_WITH",
        "target_id": 2,
        "properties": { "since": "2021-05-20" }
    },
    {
        "type": "WORKS_AT",
        "target_id": 3,
        "properties": { "role": "Developer" }
    }
]
```
-   `type`: 边的类型。
-   `target_id`: 边指向的目标节点的 `id`。
-   `properties`: 边自身的属性。

---

## ✍️ 第二步：插入图数据

让我们用这个模型来构建之前章节中使用的社交网络。

```sql
-- 插入节点
INSERT INTO graph_nodes (label, properties) VALUES
('Person',  '{"name": "Alice"}'),   -- id: 1
('Person',  '{"name": "Bob"}'),     -- id: 2
('Person',  '{"name": "Charlie"}'), -- id: 3
('Company', '{"name": "GraphDB Inc."}'); -- id: 4

-- 为 Alice 添加边关系
UPDATE graph_nodes
SET edges = '[
    {"type": "FRIENDS_WITH", "target_id": 2},
    {"type": "WORKS_AT", "target_id": 4}
]'
WHERE id = 1;

-- 为 Bob 添加边关系
UPDATE graph_nodes
SET edges = '[
    {"type": "FRIENDS_WITH", "target_id": 3}
]'
WHERE id = 2;
```

---

### 优缺点分析

**优点：**
1.  **无外部依赖**：只使用 PostgreSQL 核心功能，部署和维护简单。
2.  **数据聚合**：一个节点及其所有出向边的信息都存储在同一行中，对于某些“以节点为中心”的查询非常方便。
3.  **高度灵活**：`JSONB` 的无模式特性使得添加新的节点和边属性变得异常简单。

**缺点：**
1.  **查询复杂**：图遍历需要编写复杂的递归 SQL 查询，远不如 Cypher 直观。
2.  **反向边查询低效**：要查找指向某个节点的所有**入向边（Incoming Edges）**，需要扫描整个表的 `edges` 字段，这是一个昂贵的操作。
3.  **数据冗余与一致性**：边信息存储在源节点中，如果目标节点被删除，需要应用层逻辑来清理悬挂的边，以保证数据一致性。

---

## 📌 小结

使用 `JSONB` 和一张表来模拟图，是一种在特定约束条件下的“权变之策”。它将图的结构信息“编码”到关系型表的字段中，通过 `JSONB` 的灵活性弥补了关系模型的不足。

这种方法最适合以下场景：
-   图的规模不大。
-   查询模式主要是从已知节点出发进行遍历。
-   对入向边的查询需求较少。
-   无法或不愿引入专门的图数据库扩展。

在下一节，我们将直面这种模型的最大挑战：如何使用递归 SQL 来实现图的遍历和查询。
