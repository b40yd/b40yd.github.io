+++
title = "PostgreSQL 数据库实战指南 - 索引类型（B-tree, Hash, GiST, SP-GiST, BRIN, Bloom）"
date = 2025-07-04
lastmod = 2025-07-04T01:00:00+08:00
tags = ["PostgreSQL", "practical", "book", "index"]
categories = ["PostgreSQL", "practical", "book", "index"]
draft = false
author = "b40yd"
+++

# 第二章 SQL 语法与高级查询  
## 第二节 索引类型（B-tree, Hash, GiST, SP-GiST, BRIN, Bloom）

> **目标**：掌握 PostgreSQL 中常见的索引类型及其适用场景，能够根据数据分布和查询模式选择合适的索引，提升数据库查询性能并优化系统响应时间。

在处理大规模数据时，索引是提高查询效率的关键。PostgreSQL 提供了多种索引类型，适用于不同的数据结构和查询需求。本节将详细介绍以下六种常见索引类型：

- B-tree
- Hash
- GiST
- SP-GiST
- BRIN
- Bloom

我们将通过原理讲解、使用场景分析以及实战示例，帮助你理解每种索引的优缺点，并能够在实际项目中做出合理的选择。

---

## 🔍 一、索引的基本概念

索引是一种用于加速数据库查询的数据结构，它为表中的一个或多个列建立“查找路径”，从而避免全表扫描。

### 创建索引基本语法：

```sql
CREATE INDEX index_name ON table_name (column_name);
```

也可以创建多列复合索引：

```sql
CREATE INDEX idx_orders_customer_date ON orders (customer_id, order_date);
```

---

## 📚 二、常用索引类型详解

### 1. B-tree（平衡树）

#### ✅ 特点：

- 支持等值查询（=）、范围查询（<, >, BETWEEN）
- 支持排序（ORDER BY）、唯一性约束（UNIQUE）
- 是默认的索引类型

#### 📌 使用场景：

- 主键或唯一键字段
- 常见的数值、日期、字符串字段
- 查询条件涉及 `WHERE`, `JOIN`, `ORDER BY` 的列

#### 示例：

```sql
-- 在订单编号上创建 B-tree 索引
CREATE INDEX idx_order_id ON orders (order_id);

-- 在客户ID和订单日期上创建复合索引
CREATE INDEX idx_customer_order_date ON orders (customer_id, order_date);
```

---

### 2. Hash（哈希索引）

#### ✅ 特点：

- 只支持等值查询（=）
- 不支持范围查询或排序
- 性能略高于 B-tree（但在 v10+ 后性能差距不大）

#### 📌 使用场景：

- 枚举型字段（如状态码）
- 快速查找固定值（如登录名、身份证号）

#### 示例：

```sql
-- 创建 hash 索引
CREATE INDEX idx_users_email_hash ON users USING HASH (email);
```

⚠️ 注意：从 PostgreSQL v10 开始，Hash 索引支持 WAL 日志和复制，因此可以在生产环境中安全使用。

---

### 3. GiST（Generalized Search Tree）

#### ✅ 特点：

- 支持非传统数据类型的复杂查询（如全文检索、地理空间、JSON、数组等）
- 支持多维数据
- 可扩展性强（可通过插件定义自定义操作符类）

#### 📌 使用场景：

- 地理位置数据（如 PostGIS 扩展）
- 全文搜索（tsvector 类型）
- JSONB 字段的 GIN + GiST 混合索引

#### 示例：

```sql
-- 在地理位置字段上创建 GiST 索引（需安装 PostGIS）
CREATE INDEX idx_locations_gist ON locations USING GIST (geom);

-- 在 tsvector 字段上创建 GiST 索引
CREATE INDEX idx_documents_fts ON documents USING GIST (to_tsvector('english', content));
```

---

### 4. SP-GiST（Space-Partitioned GiST）

#### ✅ 特点：

- 专为非平衡树结构设计，适合稀疏数据
- 支持高效存储和查询，如 IP 地址、电话前缀等
- 通常比 GiST 更节省空间

#### 📌 使用场景：

- 分级数据（如目录树、分类）
- IP 地址匹配（如 CIDR 或 inet 类型）
- 电话号码、邮政编码等具有层级结构的数据

#### 示例：

```sql
-- 在 IP 地址字段上创建 SP-GiST 索引
CREATE INDEX idx_ip_addresses_spgist ON access_logs USING SPGIST (ip_address);
```

---

### 5. BRIN（Block Range Index）

#### ✅ 特点：

- 非常轻量级，占用空间极小
- 适用于按物理顺序存储的大表
- 只能提供粗略过滤，依赖数据的自然聚集性

#### 📌 使用场景：

- 时间序列数据（如日志、传感器记录）
- 按插入顺序存储的大型历史表
- 对查询速度要求不高但对存储开销敏感的场景

#### 示例：

```sql
-- 在订单日期字段上创建 BRIN 索引
CREATE INDEX idx_orders_order_date_brin ON orders USING BRIN (order_date);
```

---

### 6. Bloom（布隆索引）

#### ✅ 特点：

- 适用于多列等值查询
- 占用空间小，构建速度快
- 存在误判可能（即可能返回“假阳性”结果）

#### 📌 使用场景：

- 大宽表的多字段等值过滤
- 联邦查询、ETL 加载后的快速筛选
- 适合大数据量下减少磁盘 I/O 的场景

#### 示例：

```sql
-- 安装 bloom 扩展
CREATE EXTENSION bloom;

-- 创建 bloom 索引（最多 32 列）
CREATE INDEX idx_user_lookup_bloom ON users USING bloom (user_id, email, phone);
```

---

## ⚙️ 三、索引管理与维护建议

### 1. 查看现有索引信息

```sql
-- 查看某张表的所有索引
\d+ table_name

-- 查询所有索引及其大小
SELECT relname AS table_name, indexrelname AS index_name, pg_size_pretty(pg_relation_size(indexrelid)) AS size
FROM pg_stat_user_indexes
WHERE schemaname = 'public';
```

### 2. 删除不必要的索引

```sql
DROP INDEX IF EXISTS idx_orders_customer_date;
```

### 3. 统计索引使用情况

```sql
SELECT * FROM pg_stat_user_indexes WHERE schemaname = 'public';
```

### 4. 重建索引（修复碎片）

```sql
REINDEX INDEX idx_orders_order_date_brin;
```

---

## 🧪 四、实战演练：为 Northwind 表添加合适索引

### 场景描述：

Northwind 数据库中存在一张 `orders` 表，包含大量订单数据。你需要为该表添加合适的索引以优化以下查询：

1. 根据客户 ID 查询订单列表
2. 根据订单日期范围筛选订单
3. 多条件联合查询（客户 ID + 订单日期）

### 步骤如下：

```sql
-- 1. 为客户 ID 添加 B-tree 索引
CREATE INDEX idx_orders_customer_id ON orders (customer_id);

-- 2. 为订单日期添加 BRIN 索引（假设数据按时间递增插入）
CREATE INDEX idx_orders_order_date_brin ON orders USING BRIN (order_date);

-- 3. 创建客户 ID 和订单日期的复合索引
CREATE INDEX idx_orders_customer_date ON orders (customer_id, order_date);
```

执行查询验证索引是否生效：

```sql
EXPLAIN ANALYZE
SELECT * FROM orders
WHERE customer_id = 'VINET'
  AND order_date BETWEEN '1996-07-01' AND '1996-08-01';
```

查看执行计划中是否使用了索引扫描（Index Scan / Bitmap Heap Scan）。

---

## 📌 小结

| 索引类型 | 适用场景 | 是否支持范围查询 | 是否支持排序 |
|----------|----------|------------------|--------------|
| B-tree   | 通用查询、主键、唯一约束 | ✅              | ✅           |
| Hash     | 等值查询                | ❌              | ❌           |
| GiST     | 多维数据、全文检索      | ✅              | ✅（部分）   |
| SP-GiST  | 层级数据、IP地址        | ✅              | ❌           |
| BRIN     | 时间序列、大表          | ✅（效果有限）  | ❌           |
| Bloom    | 多列等值查询            | ✅（有误判）    | ❌           |

选择合适的索引类型可以显著提升查询性能，但也需要注意其适用范围和潜在开销。下一节我们将深入讲解“查询优化技巧与执行计划解读”，帮助你更好地理解和调优 SQL 查询。
