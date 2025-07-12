+++
title = "PostgreSQL实战指南：多领域数据库应用与实践 - 第11章：社交网络与推荐系统实践"
date = 2025-07-12
lastmod = 2025-07-12T10:10:00+08:00
tags = ["PostgreSQL", "practical", "guide", "book", "graph", "social-network", "recommendation-system", "age"]
categories = ["PostgreSQL", "practical", "guide", "book", "graph", "database"]
draft = false
author = "b40yd"
+++

## PostgreSQL实战指南：多领域数据库应用与实践

-----

### 第二部分：PostgreSQL与图数据库

PostgreSQL 通过 `Apache AGE` 和 `pgRouting` 等强大扩展，原生支持图数据结构和分析，使其成为一个功能完备的图数据库。本部分将深入探讨如何利用 PostgreSQL 对图数据进行建模、存储、查询和分析，并结合实际场景，展示其在社交网络、路径规划和推荐系统等领域的强大应用。

-----

#### 第11章：社交网络与推荐系统实践

在本章中，我们将综合运用前两章学到的图建模和查询知识，构建一个迷你的社交网络应用。我们将设计一个包含用户、帖子和兴趣标签的图模型，并在此基础上实现一些核心的社交功能，如好友推荐和内容推荐。这个实践项目将帮助你深入理解图数据库在真实世界应用中的价值。

##### 11.1 社交网络图模型设计

一个社交网络的核心是“人”以及“人”之间的关系和互动。

**实体与关系分析:**

- **实体 (节点)**:
    - `User`: 代表系统中的用户。
    - `Post`: 代表用户发布的帖子。
    - `Tag`: 代表帖子的兴趣标签，如 #PostgreSQL, #GraphDB。
- **关系 (边)**:
    - `FOLLOWS`: 一个用户关注另一个用户。
    - `POSTED`: 一个用户发表了一个帖子。
    - `LIKES`: 一个用户喜欢一个帖子。
    - `HAS_TAG`: 一个帖子拥有一个标签。

**图模型可视化:**

`(User A) -[:FOLLOWS]-> (User B)`
`(User A) -[:POSTED]-> (Post 1)`
`(User B) -[:LIKES]-> (Post 1)`
`(Post 1) -[:HAS_TAG]-> (Tag: #PostgreSQL)`

##### 11.2 场景实战：构建迷你社交网络与推荐系统

我们将使用 `Apache AGE` 来实现这个社交网络。

**1. 准备环境和图**

```sql
-- 确保 AGE 已加载并设置好搜索路径
LOAD 'age';
SET search_path = ag_catalog, "$user", public;

-- 创建一个新的图
SELECT create_graph('mini_social');
```

**2. 创建节点和关系**

```sql
-- 创建用户、帖子和标签
SELECT * FROM cypher('mini_social', $$
    CREATE (u1:User {name: 'Alice'}),
           (u2:User {name: 'Bob'}),
           (u3:User {name: 'Charlie'}),
           (u4:User {name: 'David'}),
           (p1:Post {title: 'Intro to Graph DBs'}),
           (p2:Post {title: 'PostgreSQL Performance Tuning'}),
           (p3:Post {title: 'Learning Cypher'}),
           (t1:Tag {name: 'GraphDB'}),
           (t2:Tag {name: 'PostgreSQL'}),
           (t3:Tag {name: 'Cypher'})
$$) AS (v agtype);

-- 创建关系
SELECT * FROM cypher('mini_social', $$
    MATCH (u1:User {name: 'Alice'}), (u2:User {name: 'Bob'}), (u3:User {name: 'Charlie'}), (u4:User {name: 'David'})
    CREATE (u1)-[:FOLLOWS]->(u2),
           (u1)-[:FOLLOWS]->(u3),
           (u2)-[:FOLLOWS]->(u1),
           (u2)-[:FOLLOWS]->(u3),
           (u3)-[:FOLLOWS]->(u4)
$$) AS (e agtype);

SELECT * FROM cypher('mini_social', $$
    MATCH (u1:User {name: 'Alice'}), (u2:User {name: 'Bob'}), (u3:User {name: 'Charlie'}),
          (p1:Post {title: 'Intro to Graph DBs'}), (p2:Post {title: 'PostgreSQL Performance Tuning'}), (p3:Post {title: 'Learning Cypher'})
    CREATE (u1)-[:POSTED]->(p1),
           (u2)-[:POSTED]->(p2),
           (u3)-[:POSTED]->(p3),
           (u1)-[:LIKES]->(p3),
           (u2)-[:LIKES]->(p1),
           (u3)-[:LIKES]->(p2),
           (u4:User {name: 'David'})-[:LIKES]->(p1)
$$) AS (e agtype);

SELECT * FROM cypher('mini_social', $$
    MATCH (p1:Post {title: 'Intro to Graph DBs'}), (p2:Post {title: 'PostgreSQL Performance Tuning'}), (p3:Post {title: 'Learning Cypher'}),
          (t1:Tag {name: 'GraphDB'}), (t2:Tag {name: 'PostgreSQL'}), (t3:Tag {name: 'Cypher'})
    CREATE (p1)-[:HAS_TAG]->(t1),
           (p1)-[:HAS_TAG]->(t3),
           (p2)-[:HAS_TAG]->(t2),
           (p3)-[:HAS_TAG]->(t3)
$$) AS (e agtype);
```

##### 11.3 实现核心推荐功能

**功能1：好友推荐 (基于共同关注)**

推荐逻辑：“我关注的人”也关注了谁，而我尚未关注他/她。

```sql
-- 为 Alice 推荐好友
SELECT * FROM cypher('mini_social', $$
    -- 1. 找到 Alice
    MATCH (alice:User {name: 'Alice'})
    -- 2. 找到 Alice 关注的人 (friend)
    MATCH (alice)-[:FOLLOWS]->(friend)
    -- 3. 找到 friend 关注的人 (fof), 且这个人不是 Alice 自己
    MATCH (friend)-[:FOLLOWS]->(fof) WHERE fof.name <> alice.name
    -- 4. 确保 Alice 尚未关注 fof
    WHERE NOT EXISTS((alice)-[:FOLLOWS]->(fof))
    -- 5. 按共同好友数量排名
    RETURN fof.name AS recommendation, count(friend) AS mutual_friends
    ORDER BY mutual_friends DESC
$$) AS (recommendation agtype, mutual_friends agtype);
```
*预期结果*: `David` 会被推荐给 `Alice`，因为他们都关注了 `Charlie`。

**功能2：内容推荐 (基于关注的人喜欢的帖子)**

推荐逻辑：为我推荐“我关注的人”喜欢过，而我尚未看过的帖子。

```sql
-- 为 Alice 推荐她可能感兴趣的帖子
SELECT * FROM cypher('mini_social', $$
    -- 1. 找到 Alice
    MATCH (alice:User {name: 'Alice'})
    -- 2. 找到 Alice 关注的人 (friend)
    MATCH (alice)-[:FOLLOWS]->(friend)
    -- 3. 找到 friend 喜欢过的帖子 (post)
    MATCH (friend)-[:LIKES]->(post)
    -- 4. 确保 Alice 没有发表过或喜欢过这个帖子
    WHERE NOT EXISTS((alice)-[:POSTED]->(post)) AND NOT EXISTS((alice)-[:LIKES]->(post))
    -- 5. 按喜欢该帖子的好友数量排名
    RETURN post.title AS recommendation, count(DISTINCT friend) AS liked_by_friends
    ORDER BY liked_by_friends DESC
$$) AS (recommendation agtype, liked_by_friends agtype);
```
*预期结果*: `Intro to Graph DBs` 和 `PostgreSQL Performance Tuning` 会被推荐给 `Alice`。

**功能3：内容推荐 (基于共同兴趣标签)**

推荐逻辑：为我推荐一些帖子，这些帖子的标签与我喜欢过的帖子的标签相同。

```sql
-- 为 Alice 推荐与她兴趣相关的帖子
SELECT * FROM cypher('mini_social', $$
    -- 1. 找到 Alice 和她喜欢过的帖子
    MATCH (alice:User {name: 'Alice'})-[:LIKES]->(liked_post)
    -- 2. 找到这些帖子的标签 (tag)
    MATCH (liked_post)-[:HAS_TAG]->(tag)
    -- 3. 找到其他拥有相同标签的帖子 (recommended_post)
    MATCH (recommended_post)-[:HAS_TAG]->(tag)
    -- 4. 确保推荐的帖子不是 Alice 已经喜欢或发表过的
    WHERE NOT EXISTS((alice)-[:LIKES]->(recommended_post)) AND NOT EXISTS((alice)-[:POSTED]->(recommended_post))
    -- 5. 按共同标签数量排名
    RETURN recommended_post.title AS recommendation, count(DISTINCT tag) AS common_tags
    ORDER BY common_tags DESC
$$) AS (recommendation agtype, common_tags agtype);
```
*预期结果*: `Intro to Graph DBs` 会被推荐给 `Alice`，因为她喜欢过 `Learning Cypher`，而这两个帖子都有 `Cypher` 或 `GraphDB` 相关的标签。

##### 11.4 总结

在本章中，我们从零开始设计并实现了一个基于图的迷你社交网络。通过这个实践项目，我们看到了图数据库在处理复杂关系和实现推荐系统等功能时的直观性和强大能力。`Cypher` 查询语言使得描述“朋友的朋友”、“共同兴趣”等复杂关系模式变得异常简单。

至此，我们完成了对 PostgreSQL 图数据库能力的探索。在接下来的部分，我们将转向 PostgreSQL 的另一个强大领域：它对 NoSQL 特性的支持。
-----
