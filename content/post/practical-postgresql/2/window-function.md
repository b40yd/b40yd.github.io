+++
title = "PostgreSQL 数据库实战指南 - 实战：使用窗口函数分析销售数据趋势"
date = 2025-07-04
lastmod = 2025-07-04T01:00:00+08:00
tags = ["PostgreSQL", "practical", "book", "window"]
categories = ["PostgreSQL", "practical", "book", "window"]
draft = false
author = "b40yd"
+++

# 第二章 SQL 语法与高级查询  
## 第四节 实战：使用窗口函数分析销售数据趋势

> **目标**：通过结合 Northwind 示例数据库中的订单和产品数据，掌握如何使用 PostgreSQL 的窗口函数进行销售趋势分析，包括累计销售额、月环比增长、排名分析等实用场景。

窗口函数（Window Function）是 PostgreSQL 中用于执行复杂数据分析的强大工具。它可以在不减少行数的前提下对一组相关行进行计算，非常适合用于时间序列分析、排名统计、趋势预测等业务场景。

本节将以 Northwind 示例数据库为基础，演示如何使用窗口函数对销售数据进行深入分析，并生成可视化友好的趋势报表。

---

## 📊 一、准备环境与数据集

### 1. 确保已导入 Northwind 数据集

如果你尚未导入 Northwind 数据集，请参考第一章第五节完成安装与导入。

主要涉及的表如下：

- `orders`：订单表，包含订单 ID、客户 ID、订单日期等字段
- `order_details`：订单详情表，包含订单 ID、产品 ID、单价、数量等字段
- `products`：产品表，包含产品 ID、产品名称等信息

### 2. 创建视图简化查询逻辑

```sql
CREATE OR REPLACE VIEW sales_data AS
SELECT
    o.order_id,
    o.order_date,
    od.product_id,
    p.product_name,
    od.quantity,
    od.unit_price,
    (od.quantity * od.unit_price) AS total_amount
FROM orders o
JOIN order_details od ON o.order_id = od.order_id
JOIN products p ON od.product_id = p.product_id;
```

---

## 🧮 二、窗口函数实战案例

### 1. 计算每日销售额并生成累计值（Running Total）

```sql
SELECT
    order_date,
    SUM(total_amount) AS daily_sales,
    SUM(SUM(total_amount)) OVER (ORDER BY order_date ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS running_total
FROM sales_data
GROUP BY order_date
ORDER BY order_date;
```

📌 **说明**：
- `SUM(SUM(...)) OVER (...)` 表示按日期顺序计算累计销售额。
- `ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW` 定义窗口范围为从第一行到当前行。

---

### 2. 按月汇总销售额并计算环比增长率（Month-over-Month Growth）

```sql
WITH monthly_sales AS (
    SELECT
        DATE_TRUNC('month', order_date) AS month,
        SUM(total_amount) AS total_sales
    FROM sales_data
    GROUP BY DATE_TRUNC('month', order_date)
)
SELECT
    month,
    total_sales,
    LAG(total_sales, 1) OVER (ORDER BY month) AS prev_month_sales,
    ROUND(
        ((total_sales - LAG(total_sales, 1) OVER (ORDER BY month)) / LAG(total_sales, 1) OVER (ORDER BY month)) * 100,
        2
    ) AS growth_rate_percent
FROM monthly_sales
ORDER BY month;
```

📌 **说明**：
- `DATE_TRUNC('month', order_date)` 将日期按月分组。
- `LAG()` 函数获取上个月的销售额。
- 最后一行计算环比增长率（百分比形式）。

---

### 3. 对每个客户的订单金额进行排名（Ranking）

```sql
SELECT
    customer_id,
    company_name,
    total_spent,
    RANK() OVER (ORDER BY total_spent DESC) AS customer_rank
FROM (
    SELECT
        c.customer_id,
        c.company_name,
        SUM(od.quantity * od.unit_price) AS total_spent
    FROM orders o
    JOIN customers c ON o.customer_id = c.customer_id
    JOIN order_details od ON o.order_id = od.order_id
    GROUP BY c.customer_id, c.company_name
) AS customer_spend
ORDER BY total_spent DESC;
```

📌 **说明**：
- 内层子查询计算每位客户的总消费额。
- 外层使用 `RANK()` 对客户进行排名。
- 可替换为 `DENSE_RANK()` 或 `ROW_NUMBER()` 根据需求选择排名方式。

---

### 4. 分析每个产品的销售额占比（Percentage Contribution）

```sql
SELECT
    product_name,
    SUM(total_amount) AS product_sales,
    ROUND(
        (SUM(total_amount) / SUM(SUM(total_amount)) OVER ()) * 100,
        2
    ) AS percentage_of_total
FROM sales_data
GROUP BY product_name
ORDER BY product_sales DESC;
```

📌 **说明**：
- 子句 `SUM(SUM(...)) OVER ()` 表示全表总销售额。
- 每个产品的销售额除以总销售额得到其占比。

---

### 5. 滑动平均（Moving Average）分析季度销售趋势

```sql
SELECT
    order_date,
    AVG(daily_sales) OVER (ORDER BY order_date ROWS BETWEEN 2 PRECEDING AND CURRENT ROW) AS moving_avg_3days
FROM (
    SELECT
        order_date,
        SUM(total_amount) AS daily_sales
    FROM sales_data
    GROUP BY order_date
) AS daily_sales_summary
ORDER BY order_date;
```

📌 **说明**：
- 这里使用了一个滑动窗口（`ROWS BETWEEN 2 PRECEDING AND CURRENT ROW`），即每次取前三天的平均值。
- 可扩展为 7 天、30 天等周期。

---

## 📈 三、结果可视化建议（可选）

虽然 PostgreSQL 本身不是可视化工具，但你可以将查询结果导出后使用以下工具进行图表展示：

| 工具 | 特点 |
|------|------|
| **Grafana** | 支持 PostgreSQL 数据源，适合实时监控 |
| **Metabase** | 开源 BI 工具，支持拖拽式图表 |
| **Power BI / Tableau** | 商业级 BI 工具，可视化能力强 |
| **Python（Matplotlib/Pandas）** | 适用于本地数据分析与报告生成 |

---

## 🧪 四、实战总结与拓展练习

### ✅ 推荐练习题目：

1. 查询“最畅销的 Top 10 产品”，并显示它们的累计销量曲线。
2. 使用递归 CTE + 窗口函数构建客户生命周期价值分析模型。
3. 结合 `PARTITION BY` 和 `ORDER BY`，实现不同类别产品的销售排名对比。
4. 使用 `LEAD()` 函数预测下一个月的销售额（基于线性回归趋势模拟）。

---

## 📌 小结

| 技术 | 功能 | 应用场景 |
|------|------|----------|
| `SUM(...) OVER (...)` | 累计求和 | 销售总额趋势分析 |
| `LAG()` / `LEAD()` | 获取前/后行数据 | 环比、同比分析 |
| `RANK()` / `DENSE_RANK()` | 排名分析 | 客户、产品排名 |
| `AVG(...) OVER (...)` | 移动平均 | 时间序列平滑处理 |
| `PARTITION BY` | 分组统计 | 多维度对比分析 |

通过本节的学习，你应该已经掌握了如何在 PostgreSQL 中使用窗口函数进行复杂的销售数据分析，并能够根据实际业务需求灵活构造查询语句。下一节我们将进入第三章：**事务控制与并发机制**，深入讲解 PostgreSQL 的 ACID 实现机制与 MVCC 并发控制原理。