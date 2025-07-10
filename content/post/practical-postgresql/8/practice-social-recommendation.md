+++
title = "第八章 使用 JSON + SQL 模拟图模型 - 第三节 实战：社交网络中好友推荐算法模拟"
date = 2025-07-10
lastmod = 2025-07-10T01:00:00+08:00
tags = ["PostgreSQL", "practical", "book", "graph", "jsonb", "sql", "recommendation"]
categories = ["PostgreSQL", "practical", "book"]
draft = false
author = "b40yd"
+++

## 第三节 实战：社交网络中好友推荐算法模拟

> **目标**：在一个基于 `JSONB` 模拟的社交网络图上，实现一个经典的好友推荐算法——“朋友的朋友”，并按共同好友数量进行排序。

好友推荐是社交应用的基石功能之一。最简单也最有效的推荐逻辑是：**你朋友的朋友，也很有可能成为你的朋友**。如果你们之间有越多的共同好友，那么你们成为朋友的可能性就越大。

本实战将基于前两节构建的 `graph_nodes` 表，为用户 'Alice' 推荐新朋友。

**推荐逻辑：**
1.  找到 Alice 的所有直接朋友（一度关系）。
2.  找到这些朋友的所有朋友（二度关系）。
3.  从二度关系中排除 Alice 自己以及她已经是朋友的人。
4.  计算剩余的“潜在朋友”与 Alice 的共同好友数量。
5.  按共同好友数量降序排名，得出推荐列表。

---

### 准备工作：扩展我们的社交网络

为了让推荐结果更有趣，我们先向图中加入更多用户和关系。

```sql
-- 插入新节点
INSERT INTO graph_nodes (label, properties) VALUES
('Person', '{"name": "David"}'),  -- id: 5
('Person', '{"name": "Emily"}');  -- id: 6

-- 更新边关系
-- Alice 是 Bob 和 Emily 的朋友
UPDATE graph_nodes SET edges = '[
    {"type": "FRIENDS_WITH", "target_id": 2},
    {"type": "FRIENDS_WITH", "target_id": 6}
]' WHERE properties->>'name' = 'Alice';

-- Bob 是 Alice 和 Charlie 的朋友
UPDATE graph_nodes SET edges = '[
    {"type": "FRIENDS_WITH", "target_id": 1},
    {"type": "FRIENDS_WITH", "target_id": 3}
]' WHERE properties->>'name' = 'Bob';

-- Charlie 是 Bob 和 David 的朋友
UPDATE graph_nodes SET edges = '[
    {"type": "FRIENDS_WITH", "target_id": 2},
    {"type": "FRIENDS_WITH", "target_id": 5}
]' WHERE properties->>'name' = 'Charlie';

-- David 是 Charlie 和 Emily 的朋友
UPDATE graph_nodes SET edges = '[
    {"type": "FRIENDS_WITH", "target_id": 3},
    {"type": "FRIENDS_WITH", "target_id": 6}
]' WHERE properties->>'name' = 'David';

-- Emily 是 Alice 和 David 的朋友
UPDATE graph_nodes SET edges = '[
    {"type": "FRIENDS_WITH", "target_id": 1},
    {"type": "FRIENDS_WITH", "target_id": 5}
]' WHERE properties->>'name' = 'Emily';
```
**当前社交网络图示:**
-   Alice 认识 Bob, Emily
-   Bob 认识 Alice, Charlie
-   Charlie 认识 Bob, David
-   David 认识 Charlie, Emily
-   Emily 认识 Alice, David

---

## 🚀 实现好友推荐查询

我们将使用多个 CTE（公用表表达式）来分步实现这个逻辑，使查询更具可读性。

```sql
WITH
-- 步骤 1: 定义目标用户
target_user AS (
    SELECT id FROM graph_nodes WHERE properties->>'name' = 'Alice'
),
-- 步骤 2: 找到目标用户的所有直接朋友 (一度关系)
friends_of_target AS (
    SELECT (edge.value->>'target_id')::BIGINT AS friend_id
    FROM graph_nodes, jsonb_array_elements(edges) AS edge, target_user
    WHERE id = target_user.id AND edge.value->>'type' = 'FRIENDS_WITH'
),
-- 步骤 3: 找到“朋友的朋友” (二度关系)
friends_of_friends AS (
    SELECT
        (edge.value->>'target_id')::BIGINT AS fof_id,
        f.friend_id AS mutual_friend_id -- 记录下这位共同好友
    FROM graph_nodes n
    JOIN friends_of_target f ON n.id = f.friend_id
    , jsonb_array_elements(n.edges) AS edge
    WHERE edge.value->>'type' = 'FRIENDS_WITH'
),
-- 步骤 4: 筛选并计算共同好友数量
recommendations AS (
    SELECT
        fof.fof_id,
        count(DISTINCT fof.mutual_friend_id) AS mutual_friends_count
    FROM friends_of_friends fof
    WHERE
        -- 排除目标用户自己
        fof.fof_id != (SELECT id FROM target_user)
        -- 排除已经是朋友的人
        AND fof.fof_id NOT IN (SELECT friend_id FROM friends_of_target)
    GROUP BY fof.fof_id
)
-- 最终步骤: 输出推荐列表，并附上姓名
SELECT
    (SELECT properties->>'name' FROM graph_nodes WHERE id = r.fof_id) AS recommended_friend,
    r.mutual_friends_count
FROM recommendations r
ORDER BY r.mutual_friends_count DESC;
```

### 查询解析

1.  **`target_user`**: 锁定我们要为之推荐的用户 'Alice' 的 ID。
2.  **`friends_of_target`**: 从 'Alice' 的 `edges` 字段中，抽取出所有 `FRIENDS_WITH` 关系的 `target_id`，得到她的直接好友列表 (Bob, Emily)。
3.  **`friends_of_friends`**: 遍历 Alice 的直接好友，并从他们的 `edges` 字段中，再次抽取出好友列表。这一步会得到 Alice 的二度关系，并记录下是通过哪个共同好友连接的。
4.  **`recommendations`**: 这是核心聚合步骤。
    *   `WHERE` 子句排除了 Alice 自己和她已经是朋友的人。
    *   `GROUP BY fof_id` 按潜在朋友进行分组。
    *   `count(DISTINCT fof.mutual_friend_id)` 计算每个潜在朋友与 Alice 的共同好友数量。
5.  **最终 `SELECT`**: 格式化输出，将推荐的 ID 转换回姓名，并按共同好友数排序。

**预期结果：**
对于 Alice 来说：
-   她的朋友是 Bob 和 Emily。
-   Bob 的朋友是 Charlie。
-   Emily 的朋友是 David。
-   Charlie 和 David 都不是 Alice 的朋友。
-   Alice 和 Charlie 的共同好友是 Bob (1个)。
-   Alice 和 David 的共同好友是 Emily (1个)。

所以，最终的推荐列表会是 Charlie 和 David，共同好友数都是 1。
```
 recommended_friend | mutual_friends_count
--------------------+----------------------
 Charlie            |                    1
 David              |                    1
```

---

## 📌 小结

本实战展示了即便不使用专门的图扩展，我们依然可以利用 PostgreSQL 强大的 SQL 和 `JSONB` 功能来解决复杂的图分析问题。
-   **CTE 的威力**：通过将复杂问题分解为一系列逻辑清晰的 CTE 步骤，我们可以构建出可读性和可维护性都较好的 SQL 查询。
-   **`JSONB` 的灵活性**：`jsonb_array_elements` 和 `->>` 操作符是解析 `JSONB` 边数据的关键。

当然，我们也应再次认识到，这个 SQL 查询的复杂性与等效的 Cypher 查询相比仍然很高。一个等效的 Cypher 查询可能只需要几行 `MATCH` 语句。这再次印证了我们的结论：**JSONB 模拟图是一种能力，而 Apache AGE 是一种生产力**。选择哪种方案，取决于你的项目约束和对开发效率的追求。
