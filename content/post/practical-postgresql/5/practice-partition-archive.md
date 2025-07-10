+++
title = "第五章 表分区与继承 - 第四节 实战：按年份分区的历史数据归档系统"
date = 2025-07-10
lastmod = 2025-07-10T01:00:00+08:00
tags = ["PostgreSQL", "practical", "book", "partitioning", "archive", "detach"]
categories = ["PostgreSQL", "practical", "book"]
draft = false
author = "b40yd"
+++

## 第四节 实战：按年份分区的历史数据归档系统

> **目标**：综合运用本章所学的分区知识，设计并实现一个对大型历史事件表（`historical_events`）进行按年分区管理的系统，并演示如何高效地归档（分离）旧数据。

### 场景描述

假设我们有一个 `historical_events` 表，用于记录全球范围内的历史事件。随着时间推移，这张表将变得异常庞大。为了便于管理和查询，我们决定采用范围分区策略，按**年份**对数据进行分区。

我们的目标是：
1.  创建一个按年分区的 `historical_events` 表。
2.  为最近几年创建分区。
3.  演示数据如何自动路由到正确的分区。
4.  展示分区裁剪（Partition Pruning）带来的查询性能优势。
5.  执行核心操作：将一个旧的年份分区**分离（Detach）**出来，以进行归档。

---

## 🏛️ 第一步：创建分区主表

我们将使用 `event_date` 作为分区键，并按 `RANGE` 进行分区。

```sql
CREATE TABLE historical_events (
    event_id    BIGSERIAL,
    event_date  DATE NOT NULL,
    country     TEXT NOT NULL,
    description TEXT,
    PRIMARY KEY (event_id, event_date) -- 分区键必须包含在主键中
) PARTITION BY RANGE (event_date);

COMMENT ON TABLE historical_events IS '按年份分区的全球历史事件记录表';
```

---

## 🗓️ 第二步：创建分区并插入数据

我们为2022年、2023年和2024年创建分区。

```sql
-- 创建2022年的分区
CREATE TABLE historical_events_y2022 PARTITION OF historical_events
    FOR VALUES FROM ('2022-01-01') TO ('2023-01-01');

-- 创建2023年的分区
CREATE TABLE historical_events_y2023 PARTITION OF historical_events
    FOR VALUES FROM ('2023-01-01') TO ('2024-01-01');

-- 创建2024年的分区
CREATE TABLE historical_events_y2024 PARTITION OF historical_events
    FOR VALUES FROM ('2024-01-01') TO ('2025-01-01');

-- 插入一些样本数据
INSERT INTO historical_events (event_date, country, description) VALUES
('2022-11-30', 'USA', 'OpenAI releases ChatGPT'),
('2023-06-12', 'USA', 'Apple Vision Pro announced'),
('2023-09-23', 'China', 'Hangzhou Asian Games opening ceremony'),
('2024-01-10', 'Argentina', 'Lionel Messi wins another award');
```

你可以使用 `psql` 的 `\d+ historical_events` 命令查看分区信息，会清晰地列出所有分区及其范围。

---

## ⚡ 第三步：验证查询性能（分区裁剪）

分区最大的优势之一就是查询时可以跳过不相关的分区。我们用 `EXPLAIN` 来验证这一点。

**查询特定年份的事件：**
```sql
EXPLAIN (COSTS OFF)
SELECT * FROM historical_events WHERE event_date >= '2023-01-01' AND event_date < '2024-01-01';
```

**预期输出：**
```
                          QUERY PLAN
---------------------------------------------------------------
 Append
   ->  Seq Scan on historical_events_y2023
         Filter: ((event_date >= '2023-01-01'::date) AND (event_date < '2024-01-01'::date))
```
从执行计划中可以清晰地看到，PostgreSQL 只扫描了 `historical_events_y2023` 这一个分区，完全跳过了 2022 和 2024 年的分区，极大地提升了查询效率。

---

## 📦 第四步：归档旧数据（分离分区）

随着时间的推移，2022年的数据可能不再需要频繁访问，但又不能直接删除。最好的办法是将其从分区表中**分离**出来，变成一个独立的普通表。这样既能让主表“瘦身”，又能完整保留历史数据以备将来分析。

`DETACH PARTITION` 命令可以瞬间完成这个操作，因为它只修改元数据，不涉及数据移动。

```sql
-- 将2022年的分区从主表中分离
ALTER TABLE historical_events DETACH PARTITION historical_events_y2022;
```

**验证分离结果：**
1.  再次查看主表的分区列表，`historical_events_y2022` 已经不在其中。
    ```sql
    \d+ historical_events
    ```
2.  `historical_events_y2022` 现在是一个完全独立的、可以独立查询的普通表。
    ```sql
    SELECT * FROM historical_events_y2022;
    ```
3.  此时，如果查询主表中2022年的数据，将一无所获。
    ```sql
    SELECT * FROM historical_events WHERE event_date < '2023-01-01'; -- 返回空
    ```

**处理分离出的表：**
对于分离出的 `historical_events_y2022` 表，我们可以：
- **备份到外部存储**：使用 `pg_dump` 将其导出。
  ```bash
  pg_dump -U your_user -t historical_events_y2022 your_db > y2022_archive.sql
  ```
- **迁移到冷数据存储**：如数据仓库或对象存储。
- **直接删除**：如果确认不再需要，`DROP TABLE historical_events_y2022;`。

---

## 🔄 第五步：（可选）重新附加分区

如果将来需要重新分析已归档的数据，也可以使用 `ATTACH PARTITION` 命令将其重新附加回主表。

```sql
-- 重新附加2022年的分区
ALTER TABLE historical_events ATTACH PARTITION historical_events_y2022
    FOR VALUES FROM ('2022-01-01') TO ('2023-01-01');
```
这个操作同样是秒级完成。

---

## 📌 小结

本实战完整地演示了分区表的全生命周期管理：
1.  **设计与创建**：根据业务需求选择分区键和策略。
2.  **数据路由**：插入数据时，数据库自动完成路由。
3.  **查询优化**：通过分区裁剪实现高效查询。
4.  **维护与归档**：使用 `DETACH PARTITION` 实现零成本、高性能的数据归档。

掌握分区表的 `ATTACH` 和 `DETACH` 操作是管理超大型数据表（VLDB, Very Large Databases）的核心技能之一。它提供了一种比 `DELETE` 或 `INSERT` 高效成千上万倍的数据管理方式。
