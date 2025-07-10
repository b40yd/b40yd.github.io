+++
title = "第九章 Apache AGE 集成与图数据库实战 - 第二节：Cypher 查询语言支持"
date = 2025-07-10
lastmod = 2025-07-10T01:00:00+08:00
tags = ["PostgreSQL", "practical", "book", "graph", "apache age", "cypher"]
categories = ["PostgreSQL", "practical", "book"]
draft = false
author = "b40yd"
+++

## 第二节 Cypher 查询语言支持

> **目标**：学习 openCypher 查询语言的核心语法和模式匹配思想，掌握如何使用它在 Apache AGE 中进行图的增、删、改、查（CRUD）操作。

**openCypher** 是一种声明式的、受 SQL 启发的图查询语言。它的设计哲学是“ASCII-Art”，即用简单的文本符号来“画”出你想要查询的图模式。这种直观的表达方式使得 Cypher 成为目前最流行、最易于学习的图查询语言。

Apache AGE 实现了 openCypher 规范，让我们可以通过 `cypher()` 函数在 PostgreSQL 中执行 Cypher 查询。

---

### Cypher 语法核心

Cypher 的一切都围绕着**模式匹配（Pattern Matching）**。你首先描述一个你感兴趣的图模式，然后告诉数据库如何处理匹配到的数据。

#### 1. 节点 (Nodes)

节点用一对圆括号 `()` 表示。
-   `()`: 一个匿名的、任意类型的节点。
-   `(v)`: 为节点命名一个变量 `v`，以便后续引用。
-   `(:Person)`: 一个带有 `Person` 标签的节点。
-   `(p:Person)`: 一个带有 `Person` 标签，并命名为 `p` 的节点。
-   `(p:Person {name: 'Alice'})`: 带有标签和属性的节点。

#### 2. 边 (Edges / Relationships)

边用一对中括号 `[]` 表示，并用箭头指示方向。
-   `-->` 或 `<-`: 一条有方向的、匿名的边。
-   `-[r]-`: 一条无方向（或双向）的、名为 `r` 的边。
-   `-[:FRIENDS_WITH]->`: 一条带有 `FRIENDS_WITH` 类型的边。
-   `-[r:FRIENDS_WITH {since: 2021}]->`: 带有类型、属性并命名的边。

#### 3. 模式 (Patterns)

将节点和边组合起来，就构成了模式。
-   `(a)-[:KNOWS]->(b)`: 变量 `a` 通过 `KNOWS` 关系连接到变量 `b`。
-   `(a:Person {name: 'Alice'})-[:FRIENDS_WITH]->(friend:Person)`: 查找名为 'Alice' 的人，以及她所有的朋友。

---

### Cypher CRUD 操作

所有 Cypher 查询都通过 AGE 的 `cypher()` 函数执行。

**基本结构：**
```sql
SELECT * FROM cypher('graph_name', $$
    -- Cypher query goes here
$$) AS (result_alias agtype);
```

#### 1. CREATE：创建节点和边

`CREATE` 子句用于在图中创建新的节点和关系。

```sql
SELECT * FROM cypher('my_graph', $$
    CREATE (p:Person {name: 'John Doe', age: 35, city: 'New York'}),
           (c:City {name: 'New York'}),
           (p)-[:LIVES_IN]->(c)
$$) AS (result agtype);
```
这个查询创建了一个 `Person` 节点和一个 `City` 节点，并用一条 `LIVES_IN` 的边将它们连接起来。

#### 2. MATCH & RETURN：查询和返回数据

`MATCH` 是 Cypher 的核心，用于指定要查找的图模式。`RETURN` 用于指定返回哪些数据。

**查询：查找所有在纽约生活的人**
```sql
SELECT * FROM cypher('my_graph', $$
    MATCH (p:Person)-[:LIVES_IN]->(c:City {name: 'New York'})
    RETURN p.name, p.age
$$) AS (name agtype, age agtype);
```

#### 3. WHERE：添加过滤条件

`WHERE` 子句用于在 `MATCH` 的基础上添加更复杂的过滤条件。

**查询：查找所有年龄大于 30 岁的人**
```sql
SELECT * FROM cypher('my_graph', $$
    MATCH (p:Person)
    WHERE p.age > 30
    RETURN p.name, p.age
$$) AS (name agtype, age agtype);
```

#### 4. SET & REMOVE：更新节点和边

`SET` 用于添加或更新属性，`REMOVE` 用于移除属性或标签。

**更新 John 的年龄并添加一个新属性：**
```sql
SELECT * FROM cypher('my_graph', $$
    MATCH (p:Person {name: 'John Doe'})
    SET p.age = 36, p.verified = true
$$) AS (result agtype);
```

**移除 John 的 `verified` 属性：**
```sql
SELECT * FROM cypher('my_graph', $$
    MATCH (p:Person {name: 'John Doe'})
    REMOVE p.verified
$$) AS (result agtype);
```

#### 5. DELETE & DETACH DELETE：删除节点和边

-   `DELETE`: 用于删除**没有边**的节点或独立的边。
-   `DETACH DELETE`: 用于删除一个节点**及其所有关联的边**。这是一个非常方便但也很危险的操作。

**删除一条边：**
```sql
SELECT * FROM cypher('my_graph', $$
    MATCH (p:Person)-[r:LIVES_IN]->(c:City)
    WHERE p.name = 'John Doe' AND c.name = 'New York'
    DELETE r
$$) AS (result agtype);
```

**删除一个节点及其所有关系：**
```sql
SELECT * FROM cypher('my_graph', $$
    MATCH (p:Person {name: 'John Doe'})
    DETACH DELETE p
$$) AS (result agtype);
```

---

## 📌 小结

Cypher 提供了一种与传统 SQL 完全不同但又极其强大的数据操作范式。
-   **核心是模式匹配**：你“画”出你想要的结构，数据库负责找到它。
-   **可读性强**：`()-->()` 的语法非常直观。
-   **功能完备**：支持完整的增删改查（CRUD）以及更高级的图算法。

掌握 Cypher 的基本语法是使用 Apache AGE 的前提。在刚开始接触时，它的思维方式可能需要一些适应，但一旦你习惯了用“模式”来思考问题，你将能以惊人的效率解决复杂的数据关联问题。在下一节，我们将探讨如何将 Cypher 查询与传统 SQL 查询结合起来。
