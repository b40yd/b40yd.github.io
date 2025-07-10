+++
title = "第五章 表分区与继承 - 第二节：原生分区 vs. 继承分区"
date = 2025-07-10
lastmod = 2025-07-10T01:00:00+08:00
tags = ["PostgreSQL", "practical", "book", "partitioning", "inheritance"]
categories = ["PostgreSQL", "practical", "book"]
draft = false
author = "b40yd"
+++

## 第二节 原生分区 vs. 继承分区

> **目标**：理解 PostgreSQL 中实现分区的两种历史方式——表继承（Table Inheritance）和原生分区（Declarative Partitioning），并明确为什么在现代版本（PostgreSQL 10+）中应始终首选原生分区。

在 PostgreSQL 10 版本发布之前，实现表分区的唯一方法是利用**表继承**机制，并配合触发器和 `CHECK` 约束来手动“模拟”分区。这个过程相当繁琐且容易出错。自 PostgreSQL 10 引入了原生的声明式分区后，分区实现变得前所未有的简单和高效。

本节将带你回顾传统继承分区是如何工作的，并与现代原生分区进行全方位对比。

---

## 📜 一、传统方式：使用表继承实现分区

让我们看看在“石器时代”（PostgreSQL 9.6 及更早版本），要实现一个按月分区的日志表需要多少步骤。

### 步骤 1：创建主表（父表）
主表是一个普通的表，它本身不存储任何数据。
```sql
CREATE TABLE access_logs_parent (
    log_id      BIGSERIAL,
    ip_address  INET NOT NULL,
    log_time    TIMESTAMP WITH TIME ZONE NOT NULL,
    url         TEXT NOT NULL
);
```

### 步骤 2：创建子表（分区）
每个子表都继承自主表，并使用 `CHECK` 约束来定义它所能存储的数据范围。
```sql
-- 2025年1月份的分区
CREATE TABLE access_logs_y2025m01 (
    CHECK ( log_time >= '2025-01-01' AND log_time < '2025-02-01' )
) INHERITS (access_logs_parent);

-- 2025年2月份的分区
CREATE TABLE access_logs_y2025m02 (
    CHECK ( log_time >= '2025-02-01' AND log_time < '2025-03-01' )
) INHERITS (access_logs_parent);
```

### 步骤 3：创建触发器函数以路由数据
由于数据直接插入父表 `access_logs_parent` 不会自动路由到子表，我们必须创建一个触发器函数来手动完成这项工作。

```sql
CREATE OR REPLACE FUNCTION access_logs_insert_trigger()
RETURNS TRIGGER AS $$
BEGIN
    IF ( NEW.log_time >= '2025-01-01' AND NEW.log_time < '2025-02-01' ) THEN
        INSERT INTO access_logs_y2025m01 VALUES (NEW.*);
    ELSIF ( NEW.log_time >= '2025-02-01' AND NEW.log_time < '2025-03-01' ) THEN
        INSERT INTO access_logs_y2025m02 VALUES (NEW.*);
    ELSE
        RAISE EXCEPTION 'Date out of range.  Fix the access_logs_insert_trigger() function!';
    END IF;
    RETURN NULL; -- 阻止向父表中插入数据
END;
$$
LANGUAGE plpgsql;
```

### 步骤 4：在主表上创建触发器
```sql
CREATE TRIGGER insert_access_logs_trigger
    BEFORE INSERT ON access_logs_parent
    FOR EACH ROW EXECUTE FUNCTION access_logs_insert_trigger();
```

**小结**：整个过程非常复杂，涉及手动创建约束、编写和维护触发器，每当增加新分区时，都需要修改触发器函数，极易出错。

---

## ✨ 二、现代方式：原生声明式分区

现在，让我们回顾一下上一节中原生分区的实现方式，对比之下，其简洁性一目了然。

```sql
-- 1. 创建分区主表
CREATE TABLE access_logs (
    log_id      BIGSERIAL,
    ip_address  INET NOT NULL,
    log_time    TIMESTAMP WITH TIME ZONE NOT NULL,
    url         TEXT NOT NULL
) PARTITION BY RANGE (log_time);

-- 2. 创建分区
CREATE TABLE access_logs_y2025m01 PARTITION OF access_logs
    FOR VALUES FROM ('2025-01-01') TO ('2025-02-01');

CREATE TABLE access_logs_y2025m02 PARTITION OF access_logs
    FOR VALUES FROM ('2025-02-01') TO ('2025-03-01');
```
**完成！** 不需要任何触发器，不需要手动写 `CHECK` 约束（数据库会自动创建）。数据插入 `access_logs` 后会自动、高效地路由到正确的分区。

---

## 🆚 三、全方位对比：原生分区 vs. 继承分区

| 特性 | 原生分区 (Declarative Partitioning) | 继承分区 (Inheritance-based) | 优势方 |
| :--- | :--- | :--- | :--- |
| **实现复杂度** | **极低**。只需 `PARTITION BY` 和 `PARTITION OF`。 | **非常高**。需要手动管理 `INHERITS`, `CHECK` 约束和触发器。 | **原生分区** |
| **数据插入性能** | **高**。由数据库内核直接路由，开销极小。 | **较低**。每次插入都需要执行触发器函数，有额外开销。 | **原生分区** |
| **查询性能** | **更高**。查询优化器有更多关于分区结构的信息，能生成更优的执行计划。 | **较好**。依赖 `constraint_exclusion` 参数进行分区裁剪，但优化能力有限。 | **原生分区** |
| **分区管理** | **简单**。使用 `ATTACH/DETACH PARTITION` 命令，操作是原子性的且速度极快。 | **复杂**。需要手动 `ALTER TABLE ... INHERIT` 和 `NO INHERIT`，并管理约束。 | **原生分区** |
| **数据一致性** | **强保证**。数据库保证数据只能进入其所属的分区。 | **弱保证**。依赖于 `CHECK` 约束和触发器逻辑的正确性，容易出错。 | **原生分区** |
| **功能支持** | 支持 `UPDATE` 跨分区移动行；支持 `FOREIGN KEY`；更好的索引支持。 | 不支持 `UPDATE` 跨分区移动；外键支持有限。 | **原生分区** |
| **灵活性** | 结构固定，不支持一个分区有额外列。 | 更灵活，子表可以有自己独特的列和约束。 | 继承分区 |

### 继承分区的唯一“优势”？

继承分区唯一的优势在于其灵活性——子表可以拥有父表没有的列。但这在绝大多数分区场景下并非必要，反而可能破坏数据模型的统一性。对于需要这种异构数据模型的场景，通常有更好的设计模式可以替代。

---

## 📌 结论

**对于 PostgreSQL 10 及以上版本，请始终使用原生声明式分区。**

原生分区在性能、易用性、可维护性和数据安全性上全面超越了基于继承的老方法。它将分区从一个需要数据库管理员（DBA）精细手动操作的“高级技巧”，变为了一个内建的、可靠的、易于使用的核心功能。

了解继承分区的工作原理有助于理解 PostgreSQL 的发展历史，以及在维护旧系统时可能遇到的情况，但对于所有新项目，它都应该被视为已废弃的技术。
