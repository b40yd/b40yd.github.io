+++
title = "第七章 PostgreSQL 中的图数据库能力 - 第三节：图遍历与路径查找"
date = 2025-07-10
lastmod = 2025-07-10T01:00:00+08:00
tags = ["PostgreSQL", "practical", "book", "graph", "apache age", "traversal", "pathfinding"]
categories = ["PostgreSQL", "practical", "book"]
draft = false
author = "b40yd"
+++

## 第三节 图遍历与路径查找

> **目标**：掌握图数据库中最核心的操作——图遍历（Graph Traversal）和路径查找（Pathfinding），并学会使用 openCypher 语法来描述和执行这些复杂查询。

图数据库的威力，正体现在它能够轻松地回答那些对于关系型数据库来说极其困难的问题，例如：“A 和 D 是如何关联的？”、“在社交网络中，我和他之间隔了几个人？”、“从城市 A 到城市 B 的最短路径是什么？”。这些问题的本质都是**图遍历**和**路径查找**。

本节我们将使用 Apache AGE 和 openCypher 语言，深入探索这些核心图操作。

---

### 准备工作：构建一个更复杂的社交网络

在执行查询前，我们先用 Cypher 构建一个包含更多节点和关系的社交网络图。

```sql
-- 确保已创建图并设置 search_path
-- SELECT create_graph('social_network');
-- SET search_path = social_network, public;

-- 创建节点和关系
SELECT * FROM cypher('social_network', $$
    CREATE (alice:Person {name: 'Alice'}),
           (bob:Person {name: 'Bob'}),
           (charlie:Person {name: 'Charlie'}),
           (david:Person {name: 'David'}),
           (emily:Person {name: 'Emily'}),
           (alice)-[:FRIENDS_WITH]->(bob),
           (bob)-[:FRIENDS_WITH]->(charlie),
           (charlie)-[:FRIENDS_WITH]->(david),
           (alice)-[:WORKS_AT]->(:Company {name: 'GraphDB Inc.'}),
           (david)-[:WORKS_AT]->(:Company {name: 'GraphDB Inc.'})
$$) AS (result agtype);
```
这个图包含了“朋友的朋友”这种间接关系，以及通过“同在一家公司工作”建立的联系。

---

## 🚶 一、固定长度路径遍历

这是最简单的遍历，我们指定要跨越的边的确切数量。

**查询：查找 Alice 的“朋友的朋友”**
这相当于查找从 Alice 出发，经过两条 `:FRIENDS_WITH` 边的路径。

```sql
SELECT * FROM cypher('social_network', $$
    MATCH (p1:Person {name: 'Alice'})-[:FRIENDS_WITH*2]->(p2:Person)
    RETURN p1.name, p2.name
$$) AS (person1 agtype, person2 agtype);
```
-   `-[:FRIENDS_WITH*2]->` 这个语法表示沿着 `:FRIENDS_WITH` 类型的边，连续走两步。
-   **结果**：会返回 `('Alice', 'Charlie')`。

---

## 🌐 二、可变长度路径遍历

当不知道两个节点之间隔了多少层关系时，就需要使用可变长度路径遍历。这在查找所有间接联系时非常有用。

**查询：查找 Alice 通过“朋友关系”能触达的所有人**

```sql
SELECT * FROM cypher('social_network', $$
    MATCH (p1:Person {name: 'Alice'})-[:FRIENDS_WITH*1..]->(p2:Person)
    RETURN p1.name, p2.name
$$) AS (person1 agtype, person2 agtype);
```
-   `*1..` 表示匹配长度从 1 开始，到无穷远的路径。
-   `*1..5` 则表示匹配长度在 1 到 5 之间的路径。
-   **结果**：会返回 Alice 的所有直接和间接朋友 (Bob, Charlie, David)。

---

## 🗺️ 三、最短路径查找 (Shortest Path)

最短路径是图算法中的一个经典问题，在导航、网络路由、关系发现等领域有广泛应用。Cypher 提供了 `shortestPath()` 函数来解决这个问题。

**查询：查找 Alice 和 David 之间的最短联系路径**

```sql
SELECT * FROM cypher('social_network', $$
    MATCH path = shortestPath(
        (p1:Person {name: 'Alice'})-[*]-(p2:Person {name: 'David'})
    )
    RETURN path
$$) AS (shortest_path agtype);
```
-   `shortestPath(...)` 函数会找出符合条件的节点之间的最短路径。
-   `-[*]-` 表示匹配任意类型、任意方向的边。
-   返回的 `path` 是一个包含路径上所有节点和边的完整对象。

**分析结果**：
这个查询可能会返回两条长度相同的路径：
1.  `Alice -> Bob -> Charlie -> David` (通过朋友关系)
2.  `Alice -> GraphDB Inc. <- David` (通过同公司关系)
`shortestPath()` 会返回它找到的第一条最短路径。如果你想找到所有最短路径，可以使用 `allShortestPaths()` 函数。

---

## 🔄 四、混合查询：结合关系数据

AGE 的一大优势是可以将图查询与标准的 SQL 查询结合。

**场景**：假设我们有一个普通的 SQL 表 `user_activity` 记录了用户活跃度，我们想找到 Alice 的朋友中，活跃度最高的人。

```sql
-- 假设存在一个 SQL 表
CREATE TABLE user_activity (username TEXT, activity_score INT);
INSERT INTO user_activity VALUES ('Bob', 95), ('Charlie', 80);

-- 混合查询
SELECT friend.name, act.activity_score
FROM cypher('social_network', $$
    MATCH (:Person {name: 'Alice'})-[:FRIENDS_WITH]->(friend:Person)
    RETURN friend
$$) AS (friend agtype),
LATERAL (
    SELECT activity_score
    FROM user_activity
    WHERE username = (friend.properties->>'name')::text
) act
ORDER BY act.activity_score DESC
LIMIT 1;
```
-   我们首先用 `cypher()` 函数找出 Alice 的所有朋友。
-   然后，使用 `LATERAL` JOIN 将图查询的结果（`friend` 节点）与 `user_activity` 表进行关联。
-   `friend.properties->>'name'` 用于从 `agtype` 类型的节点中提取属性值。
-   最后用标准 SQL 的 `ORDER BY` 和 `LIMIT` 找出最活跃的朋友。

---

## 📌 小结

图遍历和路径查找是释放图数据潜力的钥匙。通过 openCypher，我们可以用声明式、可读性强的方式来描述复杂的图查询：
-   **固定长度遍历 (`*n`)** 用于查找特定深度的关系。
-   **可变长度遍历 (`*m..n`)** 用于探索网络中的可达性。
-   **`shortestPath()`** 用于高效地发现节点间的最短联系。

更重要的是，AGE 允许我们将这些强大的图查询能力与 PostgreSQL 成熟的 SQL 功能无缝集成，为解决复杂的数据关联问题提供了前所未有的灵活性。
