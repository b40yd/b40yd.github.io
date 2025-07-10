+++
title = "第八章 使用 JSON + SQL 模拟图模型 - 第二节：构建图结构并进行递归查询"
date = 2025-07-10
lastmod = 2025-07-10T01:00:00+08:00
tags = ["PostgreSQL", "practical", "book", "graph", "jsonb", "sql", "recursive ctes"]
categories = ["PostgreSQL", "practical", "book"]
draft = false
author = "b40yd"
+++

## 第二节 构建图结构并进行递归查询

> **目标**：掌握使用 SQL 递归公用表表达式（Recursive Common Table Expressions, or CTEs）在 `JSONB` 图模型上执行遍历查询的核心技术。

上一节我们设计了用 `JSONB` 存储图的模型，现在我们将攻克其最核心、也最复杂的部分：如何用纯 SQL 查询这个图。这里的关键武器是**递归查询**。

一个递归 CTE 通常包含三个部分：
1.  **初始成员（Anchor Member）**：定义遍历的起点，它只被执行一次。
2.  **递归成员（Recursive Member）**：引用 CTE 自身，反复执行直到没有新数据产生。它定义了如何从当前节点“走”到下一个节点。
3.  **`UNION ALL`**：连接初始成员和递归成员。

---

### 准备工作：`jsonb_to_recordset` 函数

在编写查询前，我们需要一个能将 `edges` 字段（一个 JSON 数组）展开成多行记录的工具。`jsonb_to_recordset` 函数完美地满足了这个需求。

**示例：**
```sql
SELECT *
FROM jsonb_to_recordset('[
    {"type": "FRIENDS_WITH", "target_id": 2},
    {"type": "WORKS_AT", "target_id": 4}
]') AS x(type TEXT, target_id BIGINT);
```
**输出：**
```
      type      | target_id
----------------+-----------
 FRIENDS_WITH   |         2
 WORKS_AT       |         4
```

---

## 🚶‍♂️ 一、实现图遍历查询

我们的目标是：**从 'Alice' 出发，找出她通过 'FRIENDS_WITH' 关系能到达的所有人**。

这需要一个递归查询，记录下已经访问过的节点以避免无限循环，并跟踪从起点到当前节点的路径。

```sql
WITH RECURSIVE graph_traversal (
    start_node_id,
    target_node_id,
    path,
    visited
) AS (
    -- 1. 初始成员: 找到起点 'Alice'
    SELECT
        id,
        id,
        ARRAY[id] AS path,
        ARRAY[id] AS visited
    FROM graph_nodes
    WHERE properties->>'name' = 'Alice'

    UNION ALL

    -- 2. 递归成员: 从当前节点出发，寻找下一个节点
    SELECT
        gt.start_node_id,
        edge.target_id,
        gt.path || edge.target_id, -- 将新节点追加到路径中
        gt.visited || edge.target_id -- 将新节点加入已访问列表
    FROM
        graph_traversal gt
    JOIN
        graph_nodes n ON n.id = gt.target_node_id
    -- 将 edges 字段展开为多行，以便 JOIN
    CROSS JOIN LATERAL
        jsonb_to_recordset(n.edges) AS edge(type TEXT, target_id BIGINT)
    WHERE
        edge.type = 'FRIENDS_WITH'
        AND NOT (edge.target_id = ANY(gt.visited)) -- 关键：避免访问已访问过的节点
)
-- 3. 最终查询: 从遍历结果中提取信息
SELECT
    (SELECT properties->>'name' FROM graph_nodes WHERE id = t.start_node_id) AS start_node,
    (SELECT properties->>'name' FROM graph_nodes WHERE id = t.target_node_id) AS friend_of_friend,
    t.path
FROM graph_traversal t
WHERE t.start_node_id != t.target_node_id; -- 不显示起点自身
```

### 查询解析

1.  **`WITH RECURSIVE graph_traversal(...) AS (...)`**: 定义一个名为 `graph_traversal` 的递归 CTE，它有四个字段来跟踪遍历状态：
    *   `start_node_id`: 遍历的起始节点 ID，在整个递归过程中保持不变。
    *   `target_node_id`: 当前遍历到的节点 ID。
    *   `path`: 从起点到当前节点所经过的节点 ID 数组。
    *   `visited`: 一个包含所有已访问节点的 ID 数组，用于防止循环。

2.  **初始成员**: `SELECT ... WHERE properties->>'name' = 'Alice'`，它初始化了遍历，找到了 Alice 的 ID，并将其作为路径和已访问列表的第一个元素。

3.  **递归成员**:
    *   `JOIN graph_nodes n ON n.id = gt.target_node_id`: 关联到当前节点，以获取其 `edges` 信息。
    *   `CROSS JOIN LATERAL jsonb_to_recordset(n.edges) AS edge(...)`: 这是核心步骤，它将当前节点的 `edges` JSON 数组展开成一个虚拟的“边”表。
    *   `WHERE edge.type = 'FRIENDS_WITH'`: 筛选出我们感兴趣的关系类型。
    *   `WHERE NOT (edge.target_id = ANY(gt.visited))`: 这是防止无限循环的关键！在“走”到下一步之前，检查目标节点是否已经存在于 `visited` 数组中。

4.  **最终查询**: 从 `graph_traversal` 的结果中整理出可读的输出。

**结果：**
```
 start_node | friend_of_friend |   path
------------+------------------+---------
 Alice      | Bob              | {1,2}
 Alice      | Charlie          | {1,2,3}
```

---

## 📌 小结

通过这个复杂的例子，我们可以清晰地看到使用纯 SQL 模拟图查询的特点：

-   **功能强大**：递归 CTE 理论上可以实现几乎所有图遍历算法。
-   **语法复杂**：查询语句冗长，难以编写和调试。与 Cypher 的 `MATCH (a)-[*]->(b)` 相比，可读性相差甚远。
-   **性能考量**：对于大型图和深度遍历，递归查询的性能可能会成为瓶颈。每一步递归都需要进行 JOIN 和复杂的条件判断。

**结论**：
使用 `JSONB` + 递归 CTE 的方法，是在没有专用图扩展时的一个可行方案。它充分展示了 PostgreSQL 的灵活性和 SQL 语言的强大表达能力。然而，这种方法的复杂性也凸显了 Apache AGE 这类扩展的价值——它们将复杂的遍历逻辑封装在简洁的 Cypher 查询之后，让开发者能更专注于业务问题本身，而不是底层的实现细节。

在下一节，我们将基于这个模型，完成一个社交网络好友推荐的实战案例。
