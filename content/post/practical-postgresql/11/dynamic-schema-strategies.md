+++
title = "第十一章 文档型数据库风格操作 - 第二节：动态 Schema 设计与更新策略"
date = 2025-07-10
lastmod = 2025-07-10T01:00:00+08:00
tags = ["PostgreSQL", "practical", "book", "nosql", "jsonb", "schema design"]
categories = ["PostgreSQL", "practical", "book"]
draft = false
author = "b40yd"
+++

## 第二节 动态 Schema 设计与更新策略

> **目标**：理解在使用 `JSONB` 进行“无模式”（Schema-less）开发时所面临的挑战，并学习如何设计健壮的数据结构和更新策略，以在灵活性和数据一致性之间取得平衡。

`JSONB` 最大的吸引力之一就是它的**灵活性**。你可以随时向 JSON 文档中添加、修改或删除字段，而无需执行 `ALTER TABLE` 这样成本高昂的操作。这种“动态 Schema”或“无模式”的特性，极大地加快了应用的迭代速度。

然而，**“无模式”不等于“无设计”**。完全随意的 `JSONB` 结构会很快导致数据混乱、难以查询和维护。本节将探讨在享受 `JSONB` 灵活性的同时，如何保持数据的规整和一致。

---

### 挑战：当 Schema 发生变化

假设我们有一个 `products` 表，其中 `details` 字段是一个 `JSONB`。

**版本 1 的 `details` 结构：**
```json
{"price": 100, "weight_kg": 1.5}
```

随着业务发展，我们决定将价格拆分为金额和货币，并将重量单位改为克。

**版本 2 的 `details` 结构：**
```json
{"price": {"amount": 100, "currency": "USD"}, "weight_g": 1500}
```

现在，我们的 `products` 表中同时存在两种不同结构的 `JSONB` 数据。这将导致一系列问题：
-   **查询复杂性**：查询价格时，你需要同时处理 `details->'price'` 和 `details->'price'->'amount'` 两种情况。
-   **应用层逻辑混乱**：应用代码需要编写额外的逻辑来兼容新旧两种数据结构。
-   **数据分析困难**：进行数据统计时，需要先将数据清洗和统一，增加了 ETL 的复杂度。

---

### 策略一：混合模式（Hybrid Schema）

这是最推荐、最能发挥 PostgreSQL优势的策略。它的核心思想是：**将稳定、通用、需要频繁过滤的字段作为表的普通列，将易变的、非结构化的字段放入 `JSONB` 列。**

**示例：**
```sql
CREATE TABLE products_hybrid (
    product_id SERIAL PRIMARY KEY,
    name TEXT NOT NULL,
    -- 核心、稳定的字段作为普通列
    price NUMERIC(10, 2) NOT NULL,
    currency CHAR(3) NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    -- 易变的、可选的属性放入 JSONB
    attributes JSONB
);

-- 在核心字段上创建 B-Tree 索引
CREATE INDEX ON products_hybrid(price);

-- 在 JSONB 字段上创建 GIN 索引
CREATE INDEX ON products_hybrid USING GIN (attributes);
```

**插入数据：**
```sql
INSERT INTO products_hybrid (name, price, currency, attributes) VALUES
('Laptop', 1299.99, 'USD', '{"color": "Silver", "ports": ["USB-C", "HDMI"]}'),
('Book', 29.99, 'USD', '{"author": "John Doe", "pages": 350}');
```

**优点：**
-   **两全其美**：同时享受关系模型的严格约束、高性能查询和 `JSONB` 的灵活性。
-   **结构清晰**：核心业务数据一目了然，易于理解和维护。
-   **性能最优**：对价格、时间等字段的范围查询可以使用高效的 B-Tree 索引。

---

### 策略二：版本化 Schema (Schema Versioning)

如果无法使用混合模式，或者 `JSONB` 内部结构确实需要演进，可以引入一个**版本号**。

```sql
CREATE TABLE products_versioned (
    product_id SERIAL PRIMARY KEY,
    details JSONB
);

-- 版本 1 数据
INSERT INTO products_versioned (details) VALUES
('{"schema_version": 1, "price": 100, "weight_kg": 1.5}');

-- 版本 2 数据
INSERT INTO products_versioned (details) VALUES
('{"schema_version": 2, "price": {"amount": 100, "currency": "USD"}, "weight_g": 1500}');
```

**查询时：**
应用层代码或 SQL 查询可以根据 `schema_version` 字段的值，来决定如何解析 `details` 字段。

```sql
SELECT
    CASE
        WHEN (details->>'schema_version')::int = 1 THEN details->'price'
        WHEN (details->>'schema_version')::int = 2 THEN details->'price'->'amount'
    END AS price
FROM products_versioned;
```

**优点：**
-   明确地记录了每一次结构变更，使数据演进过程有迹可循。
-   查询逻辑虽然变复杂，但至少是清晰和可控的。

---

### 策略三：后台数据迁移 (Background Migration)

对于已经存在大量旧结构数据的表，最好的办法是执行一次性的后台数据迁移，将所有旧数据更新为新结构。

**迁移 `v1` 到 `v2` 的 `UPDATE` 语句：**
```sql
UPDATE products_versioned
SET details = jsonb_build_object(
        'schema_version', 2,
        'price', jsonb_build_object(
            'amount', details->'price',
            'currency', 'USD' -- 假设默认货币为 USD
        ),
        'weight_g', (details->>'weight_kg')::numeric * 1000
    )
WHERE (details->>'schema_version')::int = 1;
```
这个 `UPDATE` 操作可能会锁定大量行并消耗大量 I/O，因此应该在业务低峰期分批执行。

---

## 📌 小结

`JSONB` 的动态 Schema 是一把双刃剑。为了善用其灵活性，同时避免陷入混乱，请遵循以下最佳实践：
1.  **首选混合模式**：将你的数据模型看作一个光谱，一端是严格结构化的，另一端是完全动态的。把字段放在光谱上最适合它的位置（普通列或 `JSONB`）。
2.  **引入版本号**：当 `JSONB` 内部结构必须演进时，在文档内部增加一个 `schema_version` 字段，让你的应用能够兼容处理不同版本的数据。
3.  **及时迁移**：不要让旧版本的数据结构永远存在。制定计划，通过后台任务将旧数据逐步迁移到新结构，以简化未来的查询和维护工作。

好的设计是在约束和灵活性之间找到最佳平衡点。通过这些策略，你可以在享受 `JSONB` 带来的敏捷开发的同时，构建出健壮、可维护的长期应用。
