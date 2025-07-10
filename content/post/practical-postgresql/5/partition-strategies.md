+++
title = "第五章 表分区与继承 - 第一节：原生分区策略 (范围、列表、哈希)"
date = 2025-07-10
lastmod = 2025-07-10T01:00:00+08:00
tags = ["PostgreSQL", "practical", "book", "partitioning", "range", "list", "hash"]
categories = ["PostgreSQL", "practical", "book"]
draft = false
author = "b40yd"
+++

# 第五章 表分区与继承
## 第一节 原生分区策略 (范围、列表、哈希)

> **目标**：理解表分区的核心概念与优势，并熟练掌握 PostgreSQL 17 提供的三种原生分区策略：范围（Range）、列表（List）和哈希（Hash）。

当一张表的数据量增长到数千万甚至数十亿行时，查询和维护性能会急剧下降。索引会变得臃肿，`VACUUM` 操作耗时漫长，简单的查询也可能需要扫描大量无关数据。表分区（Table Partitioning）正是解决这一问题的关键技术。

**表分区**是将一个逻辑上的大表，根据特定规则（分区键），物理上分割成多个更小、更易于管理的子表（分区）的过程。对于应用程序来说，它访问的仍然是主表（分区表），数据库会自动将查询路由到正确的分区上。

### 分区的主要优势：
- **查询性能提升**：查询优化器可以根据 `WHERE` 子句中的条件，只扫描相关的分区（这个过程称为“分区裁剪”，Partition Pruning），避免全表扫描。
- **维护效率更高**：可以对单个分区进行 `VACUUM`、`REINDEX` 或备份，操作时间大大缩短。对于历史数据，可以直接附加（Attach）或分离（Detach）分区，实现秒级数据归档或加载。
- **批量加载/删除更快**：通过 `DROP` 整个分区来删除大量数据，远比 `DELETE` 命令快得多。

PostgreSQL 提供了三种原生的分区方法，下面我们逐一详解。

---

## 📈 1. 范围分区 (Range Partitioning)

范围分区是最常见的分区类型，它根据分区键的连续范围将数据分配到不同的分区。非常适合处理具有时间序列特性的数据，如日志、交易记录、物联网传感器数据等。

### 语法结构

```sql
CREATE TABLE measurement (
    city_id         int not null,
    logdate         date not null,
    peaktemp        int,
    unitsales       int
) PARTITION BY RANGE (logdate);
```

### 实战示例：按月分区的日志表

假设我们有一个 `access_logs` 表，需要按月进行分区。

**第一步：创建分区主表**
```sql
CREATE TABLE access_logs (
    log_id      BIGSERIAL,
    ip_address  INET NOT NULL,
    log_time    TIMESTAMP WITH TIME ZONE NOT NULL,
    url         TEXT NOT NULL,
    PRIMARY KEY (log_id, log_time) -- 分区键必须是主键或唯一约束的一部分
) PARTITION BY RANGE (log_time);
```

**第二步：创建具体的分区**
每个分区都存储特定时间范围的数据。
```sql
-- 2025年1月份的分区
CREATE TABLE access_logs_y2025m01 PARTITION OF access_logs
    FOR VALUES FROM ('2025-01-01') TO ('2025-02-01');

-- 2025年2月份的分区
CREATE TABLE access_logs_y2025m02 PARTITION OF access_logs
    FOR VALUES FROM ('2025-02-01') TO ('2025-03-01');
```

**第三步：插入和查询**
数据插入主表后，会自动路由到正确的分区。
```sql
INSERT INTO access_logs (ip_address, log_time, url)
VALUES ('192.168.1.100', '2025-01-15 10:00:00+08', '/home');

-- 这个查询将只会扫描 access_logs_y2025m01 分区
EXPLAIN SELECT * FROM access_logs WHERE log_time >= '2025-01-10' AND log_time < '2025-01-20';
```

---

## 🗂️ 2. 列表分区 (List Partitioning)

列表分区根据分区键的离散值列表来组织数据。它适用于分区键的值是有限且固定的场景，如国家代码、产品类别、城市等。

### 语法结构

```sql
CREATE TABLE sales (
    sale_id     BIGSERIAL,
    country     TEXT NOT NULL,
    amount      NUMERIC
) PARTITION BY LIST (country);
```

### 实战示例：按国家分区的销售数据表

**第一步：创建分区主表**
```sql
CREATE TABLE sales_by_country (
    sale_id     BIGSERIAL,
    country     TEXT NOT NULL,
    amount      NUMERIC,
    sale_date   DATE,
    PRIMARY KEY (sale_id, country)
) PARTITION BY LIST (country);
```

**第二步：创建具体的分区**
每个分区可以包含一个或多个国家的数据。
```sql
-- 亚洲国家分区
CREATE TABLE sales_asia PARTITION OF sales_by_country
    FOR VALUES IN ('China', 'Japan', 'India');

-- 欧洲国家分区
CREATE TABLE sales_europe PARTITION OF sales_by_country
    FOR VALUES IN ('Germany', 'France', 'UK');

-- 北美国家分区
CREATE TABLE sales_north_america PARTITION OF sales_by_country
    FOR VALUES IN ('USA', 'Canada');

-- 为其他所有情况创建一个默认分区
CREATE TABLE sales_others PARTITION OF sales_by_country DEFAULT;
```
**注意**：`DEFAULT` 分区非常有用，可以防止因出现未预期的分区键值而导致插入失败。

---

## #️⃣ 3. 哈希分区 (Hash Partitioning)

当分区键没有自然的范围或列表归属，但你希望将数据均匀地分布到固定数量的分区中以实现负载均衡时，哈希分区是最佳选择。它通过对分区键计算哈希值，然后根据哈希值的模数来决定数据应存入哪个分区。

### 语法结构

```sql
CREATE TABLE user_profiles (
    user_id     BIGINT,
    username    TEXT,
    email       TEXT
) PARTITION BY HASH (user_id);
```

### 实战示例：按用户ID分区的用户表

**第一步：创建分区主表**
```sql
CREATE TABLE users (
    user_id     BIGINT,
    username    TEXT,
    signup_date DATE,
    PRIMARY KEY (user_id) -- 哈希分区键通常就是主键
) PARTITION BY HASH (user_id);
```

**第二步：创建固定数量的分区**
假设我们想把用户数据均匀地分到 4 个分区中。
```sql
CREATE TABLE users_p0 PARTITION OF users FOR VALUES WITH (MODULUS 4, REMAINDER 0);
CREATE TABLE users_p1 PARTITION OF users FOR VALUES WITH (MODULUS 4, REMAINDER 1);
CREATE TABLE users_p2 PARTITION OF users FOR VALUES WITH (MODULUS 4, REMAINDER 2);
CREATE TABLE users_p3 PARTITION OF users FOR VALUES WITH (MODULUS 4, REMAINDER 3);
```
- `MODULUS`：指定总的分区数量。
- `REMAINDER`：指定当前分区接收的哈希余数。

数据插入时，PostgreSQL 会自动计算 `user_id` 的哈希值，并根据 `hash(user_id) % 4` 的结果将数据放入对应的分区。

---

## 📌 小结

| 分区策略 | 适用场景 | 优点 | 缺点 |
| :--- | :--- | :--- | :--- |
| **范围 (Range)** | 时间序列数据、连续变化的数值 | 便于按时间归档和清理 | 可能出现数据倾斜（如节假日数据暴增） |
| **列表 (List)** | 分类数据、状态码、地理位置 | 分区逻辑清晰，符合业务直觉 | 分区键的值必须是可枚举的 |
| **哈希 (Hash)** | 希望均匀分布数据，无明显业务分组 | 数据分布均匀，避免热点 | 失去业务上的可读性，不便于按业务归档 |

选择正确的分区策略是设计可扩展数据库架构的第一步。在下一节中，我们将对比现代原生分区与传统的继承分区方法，让你更深刻地理解原生分区的优势。
