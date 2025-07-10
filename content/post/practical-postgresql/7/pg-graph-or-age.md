+++
title = "第七章 PostgreSQL 中的图数据库能力 - 第二节：图数据库扩展 Apache AGE"
date = 2025-07-10
lastmod = 2025-07-10T01:00:00+08:00
tags = ["PostgreSQL", "practical", "book", "graph", "apache age", "cypher"]
categories = ["PostgreSQL", "practical", "book"]
draft = false
author = "b40yd"
+++

## 第二节 图数据库扩展 Apache AGE

> **目标**：了解 PostgreSQL 如何通过扩展变身为一个功能完备的图数据库，重点介绍 Apache AGE (A Graph Extension) 的核心概念、优势，以及它如何将 openCypher 查询语言引入 PostgreSQL。

虽然 `ltree` 能高效处理树形结构，但对于更复杂的图（例如，多对多关系、网络结构），`ltree` 就显得力不从心了。为了在 PostgreSQL 中获得原生的图数据库体验，社区开发了专门的扩展，其中最引人注目的是 **Apache AGE**。

**Apache AGE (A Graph Extension)** 是一个由 Apache 基金会孵化的顶级项目，它将 PostgreSQL 扩展为一个多模型数据库，使其能够同时处理关系数据和图数据。

> **历史注记**：在 AGE 出现之前，曾有 `pgGraph` 等项目尝试做类似的事情，但它们大多已经停止更新或功能有限。目前，Apache AGE 是在 PostgreSQL 上实践图数据库功能的最主流和最有前途的选择。

---

### Apache AGE 的核心优势

1.  **统一存储**：关系数据（行和列）与图数据（节点和边）可以存储在同一个 PostgreSQL 数据库中，甚至在同一个事务中进行操作，保证了数据的一致性。
2.  **支持 openCypher**：支持业界标准的图查询语言 openCypher，让熟悉 Neo4j 等原生图数据库的开发者可以无缝迁移。
3.  **利用 PostgreSQL 生态**：继承了 PostgreSQL 成熟的备份、恢复、安全和高可用特性。
4.  **高性能**：AGE 为图的存储和查询进行了专门的优化，性能远超基于纯 SQL 的模拟方案。

---

### 图数据库基本概念

在进入 AGE 的世界前，我们先快速回顾一下图数据库的两个核心概念：

-   **节点（Vertex / Node）**: 代表图中的实体，如一个人、一个商品、一个地点。节点可以有**标签（Label）**来定义其类型（如 `:Person`），也可以有**属性（Properties）**来存储其信息（如 `{name: 'Alice', age: 30}`）。

-   **边（Edge / Relationship）**: 代表节点之间的关系，如“朋友”、“购买”、“位于”。边是**有向的**，可以有**类型（Type）**（如 `:FRIENDS_WITH`），也可以有**属性**（如 `{since: '2020-01-15'}`）。

**可视化表示：**
`(Alice:Person)`-`[:FRIENDS_WITH {since: '2020'}]`->`(Bob:Person)`

---

### Apache AGE 的工作方式

AGE 在 PostgreSQL 内部创建了一套新的机制来存储和查询图数据。

1.  **创建图**：每个图在 PostgreSQL 中都对应一个 schema。你需要使用 `create_graph()` 函数来初始化一个图。
    ```sql
    SELECT create_graph('social_network');
    ```
    这会创建一个名为 `social_network` 的 schema。

2.  **设置图路径**：在执行 Cypher 查询之前，需要告诉 AGE 当前会话要操作哪个图。
    ```sql
    SET search_path = social_network, public;
    ```

3.  **执行 Cypher 查询**：所有 Cypher 查询都通过一个特殊的 `cypher()` 函数来执行。

    ```sql
    SELECT * FROM cypher('social_network', $$
        -- 这里是 openCypher 查询语句
        CREATE (a:Person {name: 'Alice', age: 30})
    $$) AS (result agtype);
    ```
    -   第一个参数是图的名称。
    -   第二个参数是 Cypher 查询字符串。
    -   `agtype` 是 AGE 引入的一种新的数据类型，类似于 `JSONB`，专门用于存储和表示图数据（节点、边、路径等）。

---

### 一个简单的 Cypher 查询示例

让我们创建一个简单的社交关系图。

**1. 创建两个节点和一条边：**
```sql
SELECT * FROM cypher('social_network', $$
    CREATE (a:Person {name: 'Alice', age: 30}),
           (b:Person {name: 'Bob', age: 32}),
           (a)-[r:FRIENDS_WITH {since: '2021-05-20'}]->(b)
    RETURN a, r, b
$$) AS (a agtype, r agtype, b agtype);
```

**2. 查询朋友关系：**
```sql
SELECT * FROM cypher('social_network', $$
    MATCH (p1:Person)-[:FRIENDS_WITH]->(p2:Person)
    WHERE p1.name = 'Alice'
    RETURN p2.name
$$) AS (friend_name agtype);
```
这个查询会找到名为 'Alice' 的 `Person` 节点，并沿着 `:FRIENDS_WITH` 类型的边，找到她所有的朋友，最后返回这些朋友的姓名。

---

## 📌 小结

Apache AGE 成功地将强大的图数据库能力嫁接到成熟的 PostgreSQL 内核之上，为我们带来了“两全其美”的解决方案：
-   我们不再需要在关系型数据库和图数据库之间做痛苦的“二选一”。
-   可以在一个统一的平台中，对关系型数据使用 SQL，对图状数据使用 Cypher，甚至将它们混合查询。

在接下来的章节中，我们将更深入地探讨图的遍历、路径查找，并最终通过实战来巩固这些知识。AGE 为 PostgreSQL 打开了一扇通往图数据世界的大门。
