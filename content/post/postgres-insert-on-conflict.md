+++
title = "Postgres Insert On Conflict"
date = 2025-07-01
lastmod = 2025-07-02T16:49:15+08:00
tags = ["postgres", "insert", "conflict"]
categories = ["postgres", "insert", "conflict"]
draft = false
author = "B40yd"
+++

## 场景 {#场景}

将物料批量导出后，修改后批量导入数据操作。


### 数据表 {#数据表}

```sql
-- public.demo definition

-- Drop table

-- DROP TABLE public.demo;

CREATE TABLE public.demo (
    "name" varchar NULL,
    id bigserial NOT NULL,
    age int8 NULL,
    s1 varchar NULL,
    s2 varchar NULL,
    CONSTRAINT demo_pk PRIMARY KEY (id)
);
```


### 重复冲突 {#重复冲突}

如果插入数据时，id存在则更新数据，如果数据没有变化则不更新。

```sql
with upserted AS (
    INSERT INTO demo (id, name, age, s1, s2)
    VALUES
        (3, 'b', 25, 's1_val1', 's2_val1'),
        (4, 'c', 30, 's1_val2', 's2_val2'),
        (5, 'd', 5, 's1_val31', 's2_val3')
    ON CONFLICT (id) DO UPDATE SET
        name = EXCLUDED.name,
        age = EXCLUDED.age,
        s1 = EXCLUDED.s1,
        s2 = EXCLUDED.s2
    WHERE
        demo.name IS DISTINCT FROM EXCLUDED.name OR
        demo.age IS DISTINCT FROM EXCLUDED.age or

        demo.s1 IS DISTINCT FROM EXCLUDED.s1 OR
        demo.s2 IS DISTINCT FROM EXCLUDED.s2
    RETURNING
        (xmax = 0) AS is_insert,
        (xmax != 0) AS is_update
)
SELECT
    coalesce(SUM(CASE WHEN is_insert THEN 1 ELSE 0 END), 0) AS inserted_rows,
    coalesce(SUM(CASE WHEN is_update THEN 1 ELSE 0 END), 0) AS updated_rows
FROM upserted;

```

-   不要遗漏 WHERE demo.name IS DISTINCT FROM EXCLUDED.name ：
    1.  避免无意义更新（即使字段值没有变化也触发更新）
    2.  否则会导致不必要的 xmax 设置，影响 MVCC 行为和性能
-   RETURNING xmax 只能用于判断单行状态，不能直接统计总数


### 使用约束条件 {#使用约束条件}

```sql
with upserted AS (
    INSERT INTO demo (id, name, age, s1, s2)
    VALUES
        (3, 'b', 25, 's1_val1', 's2_val1'),
        (4, 'c', 30, 's1_val2', 's2_val2'),
        (5, 'd', 5, 's1_val31', 's2_val3')
    ON CONFLICT ON CONSTRAINT demo_pk DO UPDATE SET
        name = EXCLUDED.name,
        age = EXCLUDED.age,
        s1 = EXCLUDED.s1,
        s2 = EXCLUDED.s2
    WHERE
        demo.name IS DISTINCT FROM EXCLUDED.name OR
        demo.age IS DISTINCT FROM EXCLUDED.age or

        demo.s1 IS DISTINCT FROM EXCLUDED.s1 OR
        demo.s2 IS DISTINCT FROM EXCLUDED.s2
    RETURNING
        (xmax = 0) AS is_insert,
        (xmax != 0) AS is_update
)
SELECT
    coalesce(SUM(CASE WHEN is_insert THEN 1 ELSE 0 END), 0) AS inserted_rows,
    coalesce(SUM(CASE WHEN is_update THEN 1 ELSE 0 END), 0) AS updated_rows
FROM upserted;
```


### 处理指定字段重复，不更新 {#处理指定字段重复-不更新}

```sql
WITH input_data(id, name, age, s1, s2) AS (
    VALUES
        (3, 'b', 25, 's1_val1', 's2_val1'),
        (4, 'c', 30, 's1_val2', 's2_val2'),
        (5, 'd', 15, 's1_val31', 's2_val3')
),
existing_ages AS (
    SELECT age
    FROM demo
    WHERE age IN (SELECT age FROM input_data)
),
upserted AS (
    INSERT INTO demo (id, name, age, s1, s2)
    SELECT id, name, age, s1, s2
    FROM input_data
    WHERE age NOT IN (SELECT age FROM existing_ages)
    ON CONFLICT (id) DO UPDATE SET
        name = EXCLUDED.name,
        age = EXCLUDED.age,
        s1 = EXCLUDED.s1,
        s2 = EXCLUDED.s2
    WHERE
        demo.name IS DISTINCT FROM EXCLUDED.name OR
        demo.age IS DISTINCT FROM EXCLUDED.age OR
        demo.s1 IS DISTINCT FROM EXCLUDED.s1 OR
        demo.s2 IS DISTINCT FROM EXCLUDED.s2
    RETURNING
        (xmax = 0) AS is_insert,
        (xmax != 0) AS is_update
)
SELECT
    COALESCE(SUM(CASE WHEN is_insert THEN 1 ELSE 0 END), 0) AS inserted_rows,
    COALESCE(SUM(CASE WHEN is_update THEN 1 ELSE 0 END), 0) AS updated_rows,
    (SELECT COUNT(*) FROM existing_ages) AS duplicate_rows
FROM upserted;
```
