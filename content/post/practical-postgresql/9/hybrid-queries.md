+++
title = "第九章 Apache AGE 集成与图数据库实战 - 第三节：图数据库与关系数据混合查询"
date = 2025-07-10
lastmod = 2025-07-10T01:00:00+08:00
tags = ["PostgreSQL", "practical", "book", "graph", "apache age", "hybrid query", "sql", "cypher"]
categories = ["PostgreSQL", "practical", "book"]
draft = false
author = "b40yd"
+++

## 第三节 图数据库与关系数据混合查询

> **目标**：掌握在 Apache AGE 中实现图数据和关系数据混合查询的两种核心方法，充分发挥 PostgreSQL 作为多模型数据库的强大能力。

如果只是孤立地使用 Cypher 查询图，或者使用 SQL 查询关系表，那么 AGE 的价值就只体现了一半。AGE 真正的威力在于，它允许你在**同一个查询**中无缝地结合这两种模式，让图数据和关系数据能够“对话”。

这种混合查询的能力，对于需要将丰富的图状关系（如社交网络、知识图谱）与大量的结构化业务数据（如订单记录、用户日志）进行关联分析的场景，是至关重要的。

AGE 提供了两种主要的混合查询方式。

---

## 方式一：从 SQL 到 Cypher (SQL -> Cypher)

这是最常见的方式。我们先用 SQL 从关系表中筛选出一些数据，然后将这些数据作为参数，传递给 `cypher()` 函数来执行图查询。

### 场景

假设我们有一个关系表 `user_events`，记录了用户的登录事件。我们希望找到**最近 24 小时内登录过的用户**，并在社交网络图中查找这些用户的**朋友**。

**1. 准备关系表**
```sql
CREATE TABLE user_events (
    event_id SERIAL PRIMARY KEY,
    username TEXT NOT NULL,
    event_type TEXT NOT NULL,
    event_time TIMESTAMPTZ DEFAULT NOW()
);

INSERT INTO user_events (username, event_type) VALUES
('Alice', 'login'),
('Bob', 'logout');
```

**2. 准备图数据 (Cypher)**
```sql
-- 假设图中已有 Alice, Bob, Charlie 等节点
-- 并且 Alice 是 Bob 的朋友，Bob 是 Charlie 的朋友
SELECT * FROM cypher('social_graph', $$
    MERGE (a:Person {name: 'Alice'})
    MERGE (b:Person {name: 'Bob'})
    MERGE (c:Person {name: 'Charlie'})
    MERGE (a)-[:FRIENDS_WITH]->(b)
    MERGE (b)-[:FRIENDS_WITH]->(c)
$$) AS (r agtype);
```
*(`MERGE` 类似于 `CREATE`，但如果节点或关系已存在，则不会重复创建)*

**3. 执行混合查询**
```sql
SELECT
    ue.username AS active_user,
    friends.friend_name
FROM
    -- 步骤1: 用 SQL 筛选出活跃用户
    (SELECT DISTINCT username
     FROM user_events
     WHERE event_type = 'login' AND event_time > NOW() - INTERVAL '1 day'
    ) AS ue,
    -- 步骤2: 将活跃用户名传入 cypher() 函数
    LATERAL (
        SELECT * FROM cypher('social_graph', $$
            MATCH (p:Person {name: $user_name})-[:FRIENDS_WITH]->(friend:Person)
            RETURN friend.name
        $$, jsonb_build_object('user_name', ue.username)) -- 关键：传递参数
        AS (friend_name agtype)
    ) AS friends;
```

### 查询解析

-   **`jsonb_build_object('user_name', ue.username)`**: 这是将 SQL 数据传递给 Cypher 的关键。`cypher()` 函数的第三个参数是一个 `jsonb` 对象，它包含了要传递的参数。
-   **`$user_name`**: 在 Cypher 查询字符串中，使用 `$` 符号来引用传入的参数。
-   **`LATERAL` JOIN**: `LATERAL` 关键字允许右侧的子查询（`cypher()` 函数）引用左侧的表（`ue`）中的列。

这个查询优雅地实现了“先用 SQL 从海量日志中高效筛选，再用 Cypher 在图中进行关系遍历”的逻辑。

---

## 方式二：从 Cypher 到 SQL (Cypher -> SQL)

这种方式反其道而行之：先用 Cypher 从图中查询出一些节点或边，然后将结果与关系表进行 JOIN。

### 场景

我们想在社交网络中找到 'Alice' 的所有朋友，并从 `user_events` 表中查询出他们**最近的一次活动记录**。

```sql
SELECT
    p.person_name,
    latest_events.event_type,
    latest_events.event_time
FROM
    -- 步骤1: 用 Cypher 找出 Alice 的朋友
    (SELECT * FROM cypher('social_graph', $$
        MATCH (:Person {name: 'Alice'})-[:FRIENDS_WITH]->(friend:Person)
        RETURN friend.name
    $$) AS (person_name agtype)) AS p
-- 步骤2: 将图查询结果与关系表进行 JOIN
LEFT JOIN LATERAL (
    SELECT ue.event_type, ue.event_time
    FROM user_events ue
    WHERE ue.username = (p.person_name#>>'{}')::text -- 关键：从 agtype 中提取文本
    ORDER BY ue.event_time DESC
    LIMIT 1
) AS latest_events ON true;
```

### 查询解析

-   **`(p.person_name#>>'{}')::text`**: 这是从 `agtype` 类型中提取纯文本值的标准方法。`agtype` 本质上是一种特殊的 JSON，`#>>'{}'` 可以将其解包成 `text`。
-   **`LEFT JOIN LATERAL ... ON true`**: 我们使用 `LEFT JOIN` 来确保即使某个朋友在 `user_events` 中没有任何记录，他仍然会出现在最终结果中。

这个查询展示了如何“先用 Cypher 进行复杂的图关系发现，再用 SQL 补充详细的业务数据”。

---

## 📌 小结

| 方式 | 流程 | 适用场景 |
| :--- | :--- | :--- |
| **SQL -> Cypher** | `SQL WHERE ...` -> `cypher(..., $param)` | 先对海量关系数据进行高效过滤，然后将少量结果作为图查询的起点。 |
| **Cypher -> SQL** | `cypher(...)` -> `JOIN sql_table` | 先进行复杂的图遍历或路径查找，然后用关系表中的数据来丰富或补充图节点的信息。 |

混合查询是 Apache AGE 最具吸引力的特性之一。它打破了关系模型和图模型之间的壁垒，让开发者可以根据问题的性质，在同一个查询中灵活地选择最合适的工具，从而以最高效、最优雅的方式解决复杂的业务问题。
