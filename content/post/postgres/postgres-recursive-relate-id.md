---
title: "在PostgreSQL中实现一个递归无限归类查询"
date: 2024-11-22T16:44:10+08:00
tags: ["postgres", "recursive"]
categories: ["postgres", "recursive"]
draft: false
author: "b40yd"
---

## 背景

由于某些特殊业务，允许唯一id更新，通常这类业务的id是动态计算生成的。 

比如，根据请求日志分析某个应用的`api`有那些，那些请求存在安全隐患等问题，那么会根据这个应用的`schema`，`host`，`port`计算一个sha512的值出来。

那么以上的计算仅是一个普通常见的例子，在这样的需求上会存在，多个子域名同属一个应用的情况下，那么计算id的host可能是通过`*.example.com`计算，也可能是其中几个子域名一起计算得出的。

有了以上的需求，那么就可能存在业务调整，域名变更等等。

导致最终id的变化，这个时候，期望原历史数据能正确访问，分析，计算等。

## 解决方案

通过学习并实践，有两种方式来解决。

1. 直接修改关联数据ID，会有大量修改操作，一致性问题，而且复杂易出错。

直接修改ID面临会各种问题，比如常见的数据一致性问题，修改后，在数据同步和最新的数据之间会有时差，导致数据迷失造成不可访问的垃圾数据，还有冷数据访问等问题，如果数据量大，几乎不可能修改冷数据。

2. 关系映射，查询复杂，优点不易出错。

## 举例实践

如何通过postgres实现一个无限递归的关联关系查询？

定义两张表：

- `original_table`表记录基础信息，id一旦记录就永远不变化。
- `id_mapping`表记录，id变化的关系。

```sql
CREATE TABLE original_table (
    id SERIAL PRIMARY KEY,
    name VARCHAR(255) NOT NULL
);


CREATE TABLE id_mapping (
    old_id INT PRIMARY KEY,
    new_id INT NOT NULL
);

INSERT INTO original_table (name) VALUES ('Alice'), ('Bob'), ('Charlie'), ('David');

-- 假设我们想将 id 1 和 2 归类到新的 id 10
INSERT INTO id_mapping (old_id, new_id) VALUES (1, 10), (2, 10);

-- 假设我们想将 id 10 归类到新的 id 100
INSERT INTO id_mapping (old_id, new_id) VALUES (10, 100);

-- 假设我们想将 id 3 和 4 归类到新的 id 20
INSERT INTO id_mapping (old_id, new_id) VALUES (3, 20), (4, 20);

-- 假设我们想将 id 20 归类到新的 id 200
INSERT INTO id_mapping (old_id, new_id) VALUES (20, 200);

-- 假设改变了新id
INSERT INTO id_mapping (old_id, new_id) VALUES (200, 400);
```

通过`CTE`实现递归查询所有数据的关系。

```sql
WITH RECURSIVE id_hierarchy AS (
    -- 基础查询，选择所有原始表中的 ID 和名称
    SELECT 
        o.id AS original_id,
        o.name,
        o.id AS current_id
    FROM 
        original_table o
    UNION ALL
    -- 递归查询，遍历 id_mapping 表
    SELECT 
        ih.original_id,
        ih.name,
        im.new_id AS current_id
    FROM 
        id_hierarchy ih
    JOIN 
        id_mapping im
    ON 
        ih.current_id = im.old_id
)
SELECT 
    original_id,
    name,
    current_id AS final_id
FROM 
    id_hierarchy
ORDER BY 
    original_id;
```

获取最新id并根据关联id分组。

```sql
WITH RECURSIVE id_hierarchy AS (
    -- 基础查询，选择所有 id_mapping 表中的旧 ID 和新 ID
    SELECT 
        old_id,
        new_id
    FROM 
        id_mapping
    UNION ALL
    -- 递归查询，遍历 id_mapping 表
    SELECT 
        ih.old_id,
        im.new_id
    FROM 
        id_hierarchy ih
    JOIN 
        id_mapping im
    ON 
        ih.new_id = im.old_id
),
latest_ids AS (
    -- 找到所有没有被其他 ID 指向的新 ID
    SELECT 
        new_id
    FROM 
        id_hierarchy
    WHERE 
        new_id NOT IN (SELECT old_id FROM id_mapping)
)
SELECT 
    li.new_id AS latest_id,
    ih.old_id
FROM 
    latest_ids li
JOIN 
    id_hierarchy ih
ON 
    li.new_id = ih.new_id
group by latest_id, old_id;
ORDER BY 
    latest_id, old_id;
```

根据最新id聚合关联id，获取历史使用的id。
   
```sql
WITH RECURSIVE id_hierarchy AS (
    SELECT old_id, new_id
    FROM id_mapping
    UNION ALL
    SELECT im.old_id, ih.new_id
    FROM id_mapping im
    JOIN id_hierarchy ih ON im.new_id = ih.old_id
),
latest_ids AS (
	-- 优化掉 NOT IN，NOT IN 子查询在某些情况下可能会导致性能问题，尤其是在数据量很大的情况下。可以考虑使用 LEFT JOIN 和 IS NULL 替代。
    -- SELECT new_id
    -- FROM id_hierarchy
    -- WHERE new_id NOT IN (SELECT old_id FROM id_hierarchy)
    SELECT ih.new_id
    FROM id_hierarchy ih
    LEFT JOIN id_hierarchy ih2 ON ih.new_id = ih2.old_id
    WHERE ih2.old_id IS NULL
)
SELECT
    li.new_id AS latest_id,
    array_agg(DISTINCT ih.old_id) AS old_ids
FROM
    latest_ids li
JOIN
    id_hierarchy ih ON li.new_id = ih.new_id
GROUP BY
    li.new_id
ORDER BY
    li.new_id;
```
