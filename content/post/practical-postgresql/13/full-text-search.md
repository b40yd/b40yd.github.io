+++
title = "第十三章 全文搜索与向量相似度匹配 - 第一节：tsvector 与 GIN 索引全文搜索"
date = 2025-07-10
lastmod = 2025-07-10T01:00:00+08:00
tags = ["PostgreSQL", "practical", "book", "full text search", "tsvector", "tsquery", "gin"]
categories = ["PostgreSQL", "practical", "book"]
draft = false
author = "b40yd"
+++

## 第十三章 全文搜索与向量相似度匹配
### 第一节 `tsvector` 与 GIN 索引全文搜索

> **目标**：理解 PostgreSQL 内置全文搜索引擎的工作原理，掌握 `tsvector` 和 `tsquery` 两种核心数据类型，并学会使用 GIN 索引来加速文本搜索查询。

在 `pg_trgm` 提供了高效的模糊搜索（相似度搜索）能力之外，PostgreSQL 还拥有一个功能更全面、更符合传统搜索引擎逻辑的**全文搜索引擎（Full Text Search, FTS）**。

与 `LIKE` 或 `pg_trgm` 不同，FTS 引擎能够理解**自然语言的结构**。它知道如何：
-   **分词（Parsing）**：将文本分解成独立的词元（Tokens）。
-   **词形还原（Stemming）**：将不同形式的同一个词（如 "running", "ran", "runs"）还原为其基本形式（"run"）。
-   **忽略停用词（Stop Words）**：自动忽略像 "a", "the", "is" 这样没有实际意义的常见词。
-   **处理复杂查询**：支持 `AND`, `OR`, `NOT` 以及短语搜索。

---

### 全文搜索的核心组件

PostgreSQL 的 FTS 功能主要由两种数据类型和一种操作符构成。

#### 1. `tsvector` (Text Search Vector)

`tsvector` 是一种特殊的数据类型，用于存储**预处理过**的文档。它将一个文本文档转换为一个由**词位（Lexemes）**组成的、排好序的、去重的列表。词位就是经过词形还原后的词。

**`to_tsvector()` 函数：**
这个函数负责将普通文本转换为 `tsvector`。
```sql
SELECT to_tsvector('english', 'A quick brown fox jumps over the lazy dog');
```
**结果：**
```
'brown':3 'dog':9 'fox':4 'jump':5 'lazi':8 'quick':2
```
-   `'english'` 是指定的文本搜索配置，它决定了使用哪种语言的停用词列表和词形还原规则。
-   停用词 "A", "over", "the" 被移除了。
-   "jumps" 被还原为 "jump"，"lazy" 被还原为 "lazi"。
-   每个词位后面都跟着它在原文中出现的位置，这对于后续的相关度排序很重要。

#### 2. `tsquery` (Text Search Query)

`tsquery` 用于存储**预处理过**的查询条件。它同样会将查询字符串进行词形还原。

**`to_tsquery()` 函数：**
```sql
SELECT to_tsquery('english', 'jumping & dogs');
```
**结果：**
```
'jump' & 'dog'
```
-   `&` 代表 `AND`。
-   `|` 代表 `OR`。
-   `!` 代表 `NOT`。
-   `<->` 代表短语搜索中的 `FOLLOWED BY`。

#### 3. `@@` 匹配操作符

`@@` 操作符用于判断一个 `tsvector` 是否匹配一个 `tsquery`。这是 FTS 的核心查询操作符。

```sql
SELECT to_tsvector('A quick brown fox') @@ to_tsquery('foxes');
-- 结果: true (因为 'foxes' 会被还原为 'fox')
```

---

### 实战：构建一个文章搜索引擎

#### 第一步：准备数据表

```sql
CREATE TABLE articles (
    id SERIAL PRIMARY KEY,
    title TEXT,
    body TEXT,
    -- 我们将创建一个专门的 tsvector 列来存储预处理过的内容
    search_vector TSVECTOR
);
```
将 `tsvector` 单独存为一列是最佳实践，因为 `to_tsvector` 是一个计算密集型函数，我们不希望在每次查询时都重新计算它。

#### 第二步：自动更新 `tsvector` 列

我们可以使用一个触发器，在 `articles` 表的 `title` 或 `body` 发生变化时，自动更新 `search_vector` 列。

```sql
-- 将 title 和 body 合并，并设置权重。'A' > 'B' > 'C' > 'D'
CREATE OR REPLACE FUNCTION update_search_vector()
RETURNS TRIGGER AS $$
BEGIN
    NEW.search_vector :=
        setweight(to_tsvector('pg_catalog.english', coalesce(NEW.title,'')), 'A') ||
        setweight(to_tsvector('pg_catalog.english', coalesce(NEW.body,'')), 'B');
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER articles_search_update
BEFORE INSERT OR UPDATE ON articles
FOR EACH ROW EXECUTE FUNCTION update_search_vector();
```
-   `setweight()` 函数可以为文档的不同部分（如标题、正文）赋予不同的重要性权重，这在相关度排序时非常有用。

#### 第三步：创建 GIN 索引

为了让 `@@` 操作符能够高效查询，我们必须在 `tsvector` 列上创建一个 GIN 索引。

```sql
CREATE INDEX idx_articles_search_gin ON articles USING GIN (search_vector);
```

#### 第四步：执行搜索和排序

```sql
-- 插入一些数据
INSERT INTO articles (title, body) VALUES
('PostgreSQL Performance', 'Tuning PostgreSQL queries is crucial for performance.'),
('Learning SQL', 'SQL is a powerful language for databases.');

-- 执行搜索
SELECT title, body FROM articles
WHERE search_vector @@ to_tsquery('english', 'query & tuning');

-- 按相关度排序
-- ts_rank_cd 函数根据词频和权重计算一个相关度分数
SELECT title, ts_rank_cd(search_vector, to_tsquery('english', 'performance | sql')) AS rank
FROM articles
WHERE search_vector @@ to_tsquery('english', 'performance | sql')
ORDER BY rank DESC;
```

---

## 📌 小结

PostgreSQL 内置的全文搜索引擎是一个功能极其完备的系统：
1.  **语言感知**：通过 `tsvector` 和 `tsquery`，它能理解词形变化和停用词，提供比 `LIKE` 或 `pg_trgm` 更精准的搜索结果。
2.  **高性能**：配合 GIN 索引，`@@` 操作符可以对海量文本数据进行毫秒级的查询。
3.  **相关度排序**：通过 `ts_rank_cd` 等函数，可以像专业搜索引擎一样，将最相关的结果排在最前面。
4.  **高级查询**：支持布尔操作符和短语搜索，能满足复杂的搜索需求。

对于任何需要在应用中构建文本搜索功能（如站内搜索、文档检索）的场景，PostgreSQL 的 FTS 都是一个不容忽视的、开箱即用的强大解决方案。在下一节，我们将探索一个更前沿的领域：向量搜索。
