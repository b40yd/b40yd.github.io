+++
title = "第九章 Apache AGE 集成与图数据库实战 - 第四节 实战：知识图谱的构建与查询优化"
date = 2025-07-10
lastmod = 2025-07-10T01:00:00+08:00
tags = ["PostgreSQL", "practical", "book", "graph", "apache age", "knowledge graph", "cypher"]
categories = ["PostgreSQL", "practical", "book"]
draft = false
author = "b40yd"
+++

## 第四节 实战：知识图谱的构建与查询优化

> **目标**：通过构建一个小型电影知识图谱，综合运用 Apache AGE 的建模、查询和优化能力，体验图数据库在处理复杂关联数据时的威力。

**知识图谱（Knowledge Graph）** 是图数据库最经典的应用之一。它用图的方式来描述现实世界中的实体（如人、电影、公司）以及它们之间的复杂关系。本实战将带领你从零开始，构建一个关于电影、演员和导演的知识图谱。

**我们将实现以下查询：**
1.  查找某位演员出演过的所有电影。
2.  查找某部电影的所有演员和导演。
3.  推荐“合作过的演员的演员”（类似于“六度分隔”理论的应用）。
4.  对查询进行分析和优化。

---

### 第一步：初始化图并建模

我们创建的实体（节点）有 `Person` 和 `Movie`。它们之间的关系（边）有 `ACTED_IN` 和 `DIRECTED`。

```sql
-- 初始化图
SELECT create_graph('movie_kg');
SET search_path = movie_kg, public;
```

---

### 第二步：构建图谱 (Cypher)

我们来添加一些关于经典电影《黑客帝国》系列的数据。

```sql
SELECT * FROM cypher('movie_kg', $$
    -- 创建电影节点
    CREATE (m1:Movie {title: 'The Matrix', released: 1999}),
           (m2:Movie {title: 'The Matrix Reloaded', released: 2003}),
           (m3:Movie {title: 'The Matrix Revolutions', released: 2003}),

    -- 创建人物节点
           (p1:Person {name: 'Keanu Reeves'}),
           (p2:Person {name: 'Laurence Fishburne'}),
           (p3:Person {name: 'Carrie-Anne Moss'}),
           (p4:Person {name: 'Lana Wachowski', role: 'Director'}),
           (p5:Person {name: 'Lilly Wachowski', role: 'Director'}),

    -- 创建关系
           (p1)-[:ACTED_IN {role: 'Neo'}]->(m1),
           (p1)-[:ACTED_IN {role: 'Neo'}]->(m2),
           (p1)-[:ACTED_IN {role: 'Neo'}]->(m3),

           (p2)-[:ACTED_IN {role: 'Morpheus'}]->(m1),
           (p2)-[:ACTED_IN {role: 'Morpheus'}]->(m2),
           (p2)-[:ACTED_IN {role: 'Morpheus'}]->(m3),

           (p3)-[:ACTED_IN {role: 'Trinity'}]->(m1),
           (p3)-[:ACTED_IN {role: 'Trinity'}]->(m2),
           (p3)-[:ACTED_IN {role: 'Trinity'}]->(m3),

           (p4)-[:DIRECTED]->(m1), (p4)-[:DIRECTED]->(m2), (p4)-[:DIRECTED]->(m3),
           (p5)-[:DIRECTED]->(m1), (p5)-[:DIRECTED]->(m2), (p5)-[:DIRECTED]->(m3)
$$) AS (r agtype);
```

---

### 第三步：执行图查询

#### 查询 1：查找 Keanu Reeves 出演过的所有电影

```sql
SELECT * FROM cypher('movie_kg', $$
    MATCH (p:Person {name: 'Keanu Reeves'})-[:ACTED_IN]->(m:Movie)
    RETURN m.title, m.released
$$) AS (title agtype, released agtype);
```

#### 查询 2：查找电影《The Matrix》的所有演员和导演

```sql
SELECT * FROM cypher('movie_kg', $$
    MATCH (p:Person)-[r:ACTED_IN|:DIRECTED]->(m:Movie {title: 'The Matrix'})
    RETURN p.name, type(r) AS relationship
$$) AS (person_name agtype, relationship agtype);
```
这里我们使用 `|` 来同时匹配 `ACTED_IN` 和 `DIRECTED` 两种关系。

#### 查询 3：为 Keanu Reeves 推荐合作演员

逻辑是：找到与 Keanu Reeves 合作过的演员（即出演过同一部电影），再找到这些演员合作过的其他演员。

```sql
SELECT * FROM cypher('movie_kg', $$
    MATCH (p1:Person {name: 'Keanu Reeves'})-[:ACTED_IN]->(m:Movie)<-[:ACTED_IN]-(p2:Person)
    WITH p2 // 将 p2 作为下一轮匹配的起点
    MATCH (p2)-[:ACTED_IN]->(m2:Movie)<-[:ACTED_IN]-(p3:Person)
    WHERE p1 <> p3 AND NOT (p1)-[:ACTED_IN]->()<-[:ACTED_IN]-(p3)
    RETURN DISTINCT p3.name
$$) AS (recommended_actor agtype);
```
**查询解析：**
-   `MATCH (p1)-[:ACTED_IN]->(m)<-[:ACTED_IN]-(p2)`: 找到所有与 Keanu Reeves (p1) 合作过的演员 (p2)。
-   `WITH p2`: Cypher 中的 `WITH` 类似于管道，将前一个 `MATCH` 的结果传递给下一个 `MATCH`。
-   `MATCH (p2)-...->(p3)`: 找到 p2 的合作演员 p3。
-   `WHERE p1 <> p3`: 排除 Keanu Reeves 本人。
-   `WHERE NOT ...`: 排除那些已经和 Keanu Reeves 直接合作过的演员。
-   `RETURN DISTINCT p3.name`: 返回去重后的推荐演员姓名。

---

### 第四步：查询优化

图查询的性能同样至关重要。AGE 利用 PostgreSQL 的索引机制来加速查询。

**关键优化点：在节点属性上创建索引。**

当我们执行 `MATCH (p:Person {name: 'Keanu Reeves'})` 时，AGE 需要快速找到这个起始节点。如果没有索引，它将不得不扫描所有的 `Person` 节点。

**为 `Person` 的 `name` 属性和 `Movie` 的 `title` 属性创建索引：**
AGE 的索引创建语法与标准 SQL 略有不同。

```sql
-- 为 Person 标签的 name 属性创建索引
CREATE INDEX ON movie_kg."Person" (properties->'name');

-- 为 Movie 标签的 title 属性创建索引
CREATE INDEX ON movie_kg."Movie" (properties->'title');
```
**注意**：表名需要用双引号括起来，因为它是在 `movie_kg` 这个 schema 下的。

创建索引后，所有通过 `name` 或 `title` 查找起始节点的查询，其性能都会得到数量级的提升。你可以使用 `EXPLAIN` (在 `cypher()` 函数外层) 来分析查询计划，观察索引是否被有效利用。

```sql
EXPLAIN (ANALYZE, VERBOSE)
SELECT * FROM cypher('movie_kg', $$
    MATCH (p:Person {name: 'Keanu Reeves'})-[:ACTED_IN]->(m:Movie)
    RETURN m.title
$$) AS (title agtype);
```
在输出的执行计划中，你应该能看到类似 `Index Scan` 的操作，证明索引生效了。

---

## 📌 小结

本实战完整地展示了使用 Apache AGE 构建知识图谱的全过程：
1.  **图建模**：将现实世界的实体和关系映射为节点和边。
2.  **图构建**：使用 `CREATE` 语句填充图谱数据。
3.  **图查询**：利用 `MATCH` 和多步遍历来回答复杂的关联性问题。
4.  **查询优化**：通过在节点的核心属性上创建索引，大幅提升查询性能。

知识图谱是释放数据内在关联价值的强大工具。Apache AGE 与 PostgreSQL 的结合，为构建和管理大规模知识图谱提供了一个高性能、高可靠且经济高效的平台。第二部分的学习到此结束，我们已经掌握了在 PostgreSQL 中处理图数据的各种核心技术。
