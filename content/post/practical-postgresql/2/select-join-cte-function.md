+++
title = "PostgreSQL 数据库实战指南 - SELECT、JOIN、CTE、窗口函数详解"
date = 2025-07-04
lastmod = 2025-07-04T01:00:00+08:00
tags = ["PostgreSQL", "practical", "book", "function"]
categories = ["PostgreSQL", "practical", "book", "function"]
draft = false
author = "b40yd"
+++

# 第二章 SQL 语法与高级查询  
## 第一节 SELECT、JOIN、CTE、窗口函数详解

> **目标**：掌握 PostgreSQL 中 `SELECT` 查询语句的核心结构，理解多种类型的 `JOIN` 操作，熟练使用 CTE（Common Table Expressions）和窗口函数进行复杂数据分析，并结合 Northwind 示例数据集进行实操练习。

SQL 是关系型数据库中最核心的语言。虽然语法看似简单，但要真正发挥其在复杂业务场景下的强大能力，需要深入理解 `SELECT`、`JOIN`、`CTE` 和 `窗口函数` 等高级用法。

本节将围绕以下重点展开：

- `SELECT` 基础与进阶用法
- 多种 JOIN 类型的对比与适用场景
- 使用 CTE 提升查询可读性与性能
- 窗口函数的原理与典型应用场景

---

## 🧩 一、SELECT 查询基础回顾

`SELECT` 是用于从一个或多个表中检索数据的核心命令。基本语法如下：

```sql
SELECT [DISTINCT] column1, column2, ...
FROM table_name
[WHERE condition]
[GROUP BY column(s)]
[HAVING condition]
[ORDER BY column(s) [ASC|DESC]]
[LIMIT n];
```

### 示例：从 Northwind 查询客户信息

```sql
SELECT customer_id, company_name, contact_name, country
FROM customers
WHERE country = 'USA'
ORDER BY company_name;
```

#### 常用子句说明：

| 子句 | 功能 |
|------|------|
| `DISTINCT` | 去重返回结果 |
| `WHERE` | 过滤行数据 |
| `GROUP BY` | 聚合分组 |
| `HAVING` | 对聚合结果进行过滤 |
| `ORDER BY` | 排序 |
| `LIMIT` | 限制返回记录数 |

---

## 🔗 二、JOIN 操作详解

JOIN 是 SQL 中最强大的功能之一，用于连接多个表并根据关联条件提取数据。PostgreSQL 支持多种 JOIN 类型。

### 1. INNER JOIN（内连接）

只返回两个表中匹配的行。

```sql
SELECT o.order_id, c.company_name
FROM orders o
INNER JOIN customers c ON o.customer_id = c.customer_id;
```

### 2. LEFT JOIN（左外连接）

返回左表所有行，即使右表无匹配项。

```sql
SELECT c.company_name, o.order_id
FROM customers c
LEFT JOIN orders o ON c.customer_id = o.customer_id;
```

### 3. RIGHT JOIN（右外连接）

返回右表所有行，即使左表无匹配项（不常用）。

```sql
SELECT e.first_name, o.order_id
FROM employees e
RIGHT JOIN orders o ON e.employee_id = o.employee_id;
```

### 4. FULL OUTER JOIN（全外连接）

返回两个表中所有行，不管是否匹配。

```sql
SELECT p.product_name, o.order_id
FROM products p
FULL OUTER JOIN order_details o ON p.product_id = o.product_id;
```

### 5. CROSS JOIN（交叉连接）

返回两个表的笛卡尔积（慎用）。

```sql
SELECT * FROM categories CROSS JOIN suppliers;
```

### ✅ 实战建议：

- 尽量使用 `LEFT JOIN` 替代 `NOT IN` 或 `NOT EXISTS`
- 避免不必要的 `CROSS JOIN`，容易造成性能问题
- 在多表连接时使用别名简化 SQL 并提升可读性

---

## 📦 三、CTE（Common Table Expression）公共表表达式

CTE 是一种临时命名的结果集，可以在后续查询中被多次引用。相比嵌套子查询，CTE 更具可读性和可维护性。

### 基本语法：

```sql
WITH cte_name AS (
    SELECT ...
)
SELECT * FROM cte_name;
```

### 示例：计算每个客户的订单数量

```sql
WITH order_counts AS (
    SELECT customer_id, COUNT(*) AS total_orders
    FROM orders
    GROUP BY customer_id
)
SELECT c.company_name, oc.total_orders
FROM customers c
LEFT JOIN order_counts oc ON c.customer_id = oc.customer_id
ORDER BY oc.total_orders DESC NULLS LAST;
```

### 递归 CTE（Recursive CTE）

适用于树形结构查询，如组织架构、目录层级等。

```sql
WITH RECURSIVE employee_hierarchy AS (
    SELECT employee_id, first_name || ' ' || last_name AS name, reports_to, 1 AS level
    FROM employees
    WHERE reports_to IS NULL  -- 根节点（CEO）

    UNION ALL

    SELECT e.employee_id, e.first_name || ' ' || e.last_name, e.reports_to, eh.level + 1
    FROM employees e
    INNER JOIN employee_hierarchy eh ON e.reports_to = eh.employee_id
)
SELECT * FROM employee_hierarchy ORDER BY level;
```

---

## 📈 四、窗口函数（Window Functions）

窗口函数是 PostgreSQL 强大的分析工具，它允许你在保留原始行的同时，对一组相关行执行计算（如排名、累计、移动平均等）。

### 基本语法：

```sql
function_name (expression) OVER (
    [PARTITION BY partition_expression]
    [ORDER BY sort_expression]
    [frame_clause]
)
```

### 常用窗口函数：

| 函数 | 描述 |
|------|------|
| `ROW_NUMBER()` | 行号（唯一） |
| `RANK()` | 排名（跳过重复） |
| `DENSE_RANK()` | 排名（不跳过重复） |
| `LAG(col, n)` | 获取前一行某列值 |
| `LEAD(col, n)` | 获取后一行某列值 |
| `SUM()`, `AVG()` 等 | 聚合函数作为窗口函数使用 |

### 示例 1：按销售额排序并显示排名

```sql
SELECT p.product_name,
       SUM(od.quantity * od.unit_price) AS total_sales,
       RANK() OVER (ORDER BY SUM(od.quantity * od.unit_price) DESC) AS sales_rank
FROM products p
JOIN order_details od ON p.product_id = od.product_id
GROUP BY p.product_name
ORDER BY sales_rank;
```

### 示例 2：计算累计销售额

```sql
SELECT order_id, order_date,
       total_amount,
       SUM(total_amount) OVER (ORDER BY order_date ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS running_total
FROM orders
ORDER BY order_date;
```

### 实战技巧：

- 使用 `PARTITION BY` 可以实现“分组统计”
- 控制 `ROWS BETWEEN ...` 可定义窗口范围（如滑动窗口）
- 窗口函数非常适合做时间序列分析、趋势预测等场景

---

## 🧪 五、实战演练：使用 Northwind 分析销售数据

### 场景描述：

你需要分析 Northwind 中的销售数据，找出最畅销的产品、高价值客户以及年度销售趋势。

### 步骤 1：创建产品销售额视图

```sql
CREATE VIEW product_sales AS
SELECT p.product_id, p.product_name,
       SUM(od.quantity * od.unit_price) AS total_sales
FROM products p
JOIN order_details od ON p.product_id = od.product_id
GROUP BY p.product_id, p.product_name;
```

### 步骤 2：列出前 10 名畅销产品

```sql
SELECT product_name, total_sales
FROM product_sales
ORDER BY total_sales DESC
LIMIT 10;
```

### 步骤 3：计算每个客户的总订单数与平均订单金额

```sql
SELECT c.company_name,
       COUNT(o.order_id) AS total_orders,
       ROUND(AVG(o.freight + SUM(od.quantity * od.unit_price)), 2) AS avg_order_value
FROM customers c
JOIN orders o ON c.customer_id = o.customer_id
JOIN order_details od ON o.order_id = od.order_id
GROUP BY c.company_name
ORDER BY total_orders DESC
LIMIT 10;
```

---

## 📌 小结

| 技术 | 特点 | 推荐使用场景 |
|------|------|--------------|
| `SELECT` | 基础查询语言 | 所有数据检索操作 |
| `JOIN` | 多表关联 | 关联多个实体对象 |
| `CTE` | 可复用的中间结果 | 复杂逻辑拆解、递归查询 |
| `窗口函数` | 高级分析能力 | 排名、累计、趋势分析 |

通过本节的学习，你应该已经掌握了 PostgreSQL 中高级 SQL 查询的核心技能，并能够灵活运用 `SELECT`、`JOIN`、`CTE` 和 `窗口函数` 来处理复杂的业务需求。下一节我们将进入“查询优化技巧与执行计划解读”，帮助你进一步提升查询性能与系统调优能力。