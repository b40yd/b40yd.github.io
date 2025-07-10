+++
title = "第六章 扩展与插件生态 - 第三节 实战：使用 pg_trgm 提升模糊搜索效率"
date = 2025-07-10
lastmod = 2025-07-10T01:00:00+08:00
tags = ["PostgreSQL", "practical", "book", "extensions", "pg_trgm", "fuzzy search", "index"]
categories = ["PostgreSQL", "practical", "book"]
draft = false
author = "b40yd"
+++

## 第三节 实战：使用 `pg_trgm` 提升模糊搜索效率

> **目标**：解决传统 `LIKE` 查询在处理模糊搜索时的性能瓶颈和功能局限，通过实战掌握 `pg_trgm` 扩展，实现高效、支持相似度匹配的模糊搜索功能。

### 场景描述

假设我们有一个 `users` 表，存储了数百万用户的姓名。业务需求是提供一个搜索框，允许用户输入部分姓名来查找匹配的用户，并且需要能容忍一些微小的拼写错误。

---

### 传统 `LIKE` 查询的困境

最直接的想法是使用 `LIKE` 操作符：
```sql
SELECT * FROM users WHERE username LIKE '%john%';
```

这种方式存在两个致命问题：
1.  **性能极差**：以通配符 `%` 开头的 `LIKE` 查询 (`LIKE '%...'`) 无法使用标准的 B-Tree 索引，导致数据库必须对全表进行扫描（Full Table Scan）。在百万级数据量的表上，这个查询可能会耗时数秒甚至更久。
2.  **功能有限**：`LIKE` 只能进行模式匹配，无法处理拼写错误。例如，搜索 `'jon'` 无法匹配到 `'john'`。

---

## 🚀 `pg_trgm` 登场

`pg_trgm`（PostgreSQL Trigram）是一个官方 `contrib` 扩展，它通过将文本拆分为“三元组（Trigram）”的序列来工作。一个三元组是文本中任意三个连续字符的组合。

例如，单词 `'postgres'` 的三元组是：
`"  p"`, `" po"`, `"pos"`, `"ost"`, `"stg"`, `"tgr"`, `"gre"`, `"res"`, `"es "`, `"s  "`
（前后会补充空格）

通过比较两个字符串共享的三元组数量，`pg_trgm` 可以快速计算它们的**相似度**。

---

## 🛠️ 第一步：安装并启用扩展

```sql
-- 如果尚未安装 contrib 包，请先在操作系统层面安装
-- sudo apt-get install postgresql-contrib-17

-- 在数据库中启用扩展
CREATE EXTENSION IF NOT EXISTS "pg_trgm";
```

---

## ✍️ 第二步：准备数据表

我们创建一个 `users` 表并插入一些样本数据。

```sql
CREATE TABLE users (
    user_id SERIAL PRIMARY KEY,
    username TEXT NOT NULL
);

-- 插入大量随机数据以模拟真实场景
INSERT INTO users (username) SELECT 'user_' || substr(md5(random()::text), 0, 20) FROM generate_series(1, 1000000);

-- 插入一些我们想搜索的目标数据
INSERT INTO users (username) VALUES
('Johnathan Smith'),
('Jonathan Livingston'),
('Jon Snow'),
('Peter Johnson');
```

---

## ⚡ 第三步：使用 `pg_trgm` 进行搜索

`pg_trgm` 提供了 `similarity()` 函数和 `%` 操作符。

- `similarity(text1, text2)`: 返回一个 0 到 1 之间的浮点数，表示两个字符串的相似度。
- `text1 % text2`: 如果 `text1` 和 `text2` 的相似度大于默认阈值（默认为 0.3），则返回 `true`。

**示例：**
```sql
-- 计算相似度
SELECT similarity('postgres', 'postman'); -- 返回一个数值，如 0.5

-- 使用相似度操作符进行搜索
SELECT * FROM users WHERE username % 'Jonathan';
```

**但是，此时的查询性能依然很差！** 因为我们还没有为 `pg_trgm` 操作创建合适的索引。

---

## 📈 第四步：创建 GiST/GIN 索引以实现性能飞跃

为了让 `pg_trgm` 的操作符能够利用索引，我们必须创建一个 **GiST** 或 **GIN** 类型的索引。

- **GiST (Generalized Search Tree)**: 索引创建速度较快，占用空间较小，但查询速度略慢。
- **GIN (Generalized Inverted Index)**: 索引创建速度较慢，占用空间较大，但查询速度非常快。

对于读多写少的场景，**GIN 索引通常是更好的选择**。

```sql
CREATE INDEX idx_users_username_trgm ON users USING GIN (username gin_trgm_ops);
```
**注意**：`gin_trgm_ops` 是 `pg_trgm` 提供的特殊操作符类，必须指定它才能创建正确的索引。

---

## ✅ 第五步：验证索引效果

现在，让我们再次执行相同的查询，并用 `EXPLAIN ANALYZE` 查看其执行计划。

```sql
EXPLAIN ANALYZE
SELECT * FROM users WHERE username % 'Jonathan';
```

**创建索引后的预期输出（节选）：**
```
                                                     QUERY PLAN
--------------------------------------------------------------------------------------------------------------------
 Bitmap Heap Scan on users  (cost=... rows=... width=...) (actual time=... rows=... loops=...)
   Recheck Cond: (username % 'Jonathan'::text)
   ->  Bitmap Index Scan on idx_users_username_trgm  (cost=... rows=...) (actual time=... loops=...)
         Index Cond: (username % 'Jonathan'::text)
```
关键在于 `Bitmap Index Scan`，它表明查询成功地使用了我们创建的 `idx_users_username_trgm` 索引。在百万级数据量的表上，查询时间将从数秒锐减到几十毫秒，性能提升成百上千倍。

---

### 调整相似度阈值

默认的 0.3 阈值可能过于宽松。我们可以通过 `pg_trgm.similarity_threshold` 参数来调整它，或者直接在查询中使用 `similarity()` 函数。

**设置会话级别的阈值：**
```sql
SET pg_trgm.similarity_threshold = 0.5;
```

**在查询中指定阈值：**
```sql
SELECT *, similarity(username, 'Jonathan') AS sml
FROM users
WHERE similarity(username, 'Jonathan') > 0.4
ORDER BY sml DESC;
```
这种写法更灵活，并且可以利用 `ORDER BY` 将最匹配的结果排在最前面。

---

## 📌 小结

通过本实战，我们完成了从低效的 `LIKE` 查询到高性能、功能强大的 `pg_trgm` 模糊搜索的转变：
1.  **识别瓶颈**：认识到 `LIKE '%...'` 无法使用 B-Tree 索引的问题。
2.  **引入新工具**：启用 `pg_trgm` 扩展。
3.  **建立加速结构**：创建 GIN 索引 (`USING GIN (column gin_trgm_ops)`)，这是实现性能飞跃的**关键步骤**。
4.  **灵活运用**：使用 `%` 操作符进行快速筛选，或结合 `similarity()` 函数实现更精细的控制。

`pg_trgm` 是 PostgreSQL 在文本搜索领域的一大利器，对于任何需要模糊搜索、拼写纠错或相似度匹配功能的应用来说，它都是一个不容错过的强大扩展。
