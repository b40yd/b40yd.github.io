+++
title = "PostgreSQL实战指南：多领域数据库应用与实践 - 第2章：SQL高级查询与数据操作"
date = 2025-07-04
lastmod = 2025-07-04T03:00:00+08:00
tags = ["PostgreSQL", "practical", "guide", "book", "query", "data", "operations"]
categories = ["PostgreSQL", "practical", "guide", "book", "query", "data", "operations"]
draft = false
author = "b40yd"
+++

## PostgreSQL实战指南：多领域数据库应用与实践

-----

### 第一部分：关系型数据库深度探索

-----

#### 第2章：SQL高级查询与数据操作

在本章中，我们将深入探索PostgreSQL强大的SQL查询能力，学习如何从复杂的数据集中提取、组合和转换信息。掌握本章内容，你将能够编写出高效且精确的SQL语句，满足各种业务分析和数据处理需求。我们将涵盖从基础的`SELECT`语句到高级的联接（**JOIN**）、聚合函数、子查询、通用表表达式（**CTE**）以及窗口函数，并通过一系列实际案例来巩固你的理解。

##### 2.1 数据查询基础：SELECT语句

`SELECT`语句是SQL中最常用的命令，用于从数据库中检索数据。

**使用场景：** 获取特定表中的所有数据或部分数据。

**基本语法：**

```sql
SELECT column1, column2, ...
FROM table_name
WHERE condition;
```

  * `SELECT *`：选择所有列。
  * `SELECT column1, column2`：选择指定的列。
  * `FROM table_name`：指定要查询的表。
  * `WHERE condition`：可选，用于筛选满足特定条件的行。

**实战举例：**

假设我们要从`users`表中查询所有用户信息，或者只查询特定用户。

```sql
-- 查询所有用户的所有信息
SELECT *
FROM users;

-- 查询所有用户的用户名和邮箱
SELECT username, email
FROM users;

-- 查询注册日期在2023年1月1日之后的用户
SELECT username, registration_date
FROM users
WHERE registration_date > '2023-01-01';

-- 查询用户名以'a'开头的用户 (不区分大小写)
SELECT username, email
FROM users
WHERE username ILIKE 'a%'; -- ILIKE 是 PostgreSQL 特有的，用于不区分大小写的模式匹配
```

##### 2.2 数据排序与限制：ORDER BY 和 LIMIT

  * **`ORDER BY`**：用于对查询结果进行排序。
      * `ASC`：升序（默认）。
      * `DESC`：降序。
  * **`LIMIT`**：限制返回的行数。
  * **`OFFSET`**：跳过指定的行数。

**使用场景：** 获取排序后的数据，或进行分页查询。

**实战举例：**

```sql
-- 按照商品价格从高到低排序所有商品
SELECT product_name, price
FROM products
ORDER BY price DESC;

-- 按照注册日期从早到晚排序用户，并获取前2个用户
SELECT username, registration_date
FROM users
ORDER BY registration_date ASC
LIMIT 2;

-- 获取订单表中总金额最高的3个订单
SELECT order_id, total_amount
FROM orders
ORDER BY total_amount DESC
LIMIT 3;

-- 分页查询：获取第2页的商品数据，每页3个 (跳过前3个，再取3个)
SELECT product_name, price
FROM products
ORDER BY product_id
LIMIT 3 OFFSET 3;
```

##### 2.3 数据聚合：分组与聚合函数

聚合函数用于对一组行进行计算，并返回单个值。常见的聚合函数包括：`COUNT()`、`SUM()`、`AVG()`、`MAX()`、`MIN()`。

  * **`GROUP BY`**：将具有相同值的行分组。聚合函数通常与`GROUP BY`一起使用。
  * **`HAVING`**：在`GROUP BY`之后过滤分组。`WHERE`子句在分组前过滤行，而`HAVING`在分组后过滤分组。

**使用场景：** 计算总数、平均值、最大最小值、统计分组数据等。

**实战举例：**

```sql
-- 统计用户总数
SELECT COUNT(*) AS total_users
FROM users;

-- 计算所有订单的总金额
SELECT SUM(total_amount) AS total_sales
FROM orders;

-- 计算商品的平均价格
SELECT AVG(price) AS average_product_price
FROM products;

-- 按照用户ID分组，统计每个用户的订单数量和总消费金额
SELECT
    user_id,
    COUNT(order_id) AS order_count,
    SUM(total_amount) AS total_spent
FROM
    orders
GROUP BY
    user_id;

-- 统计每个订单状态的订单数量
SELECT
    status,
    COUNT(order_id) AS count_by_status
FROM
    orders
GROUP BY
    status;

-- 查找总订单金额超过 1000 的用户 (先按用户分组，再过滤分组)
SELECT
    user_id,
    SUM(total_amount) AS total_spent
FROM
    orders
GROUP BY
    user_id
HAVING
    SUM(total_amount) > 1000;
```

##### 2.4 数据联接：JOIN操作

**JOIN**用于将两个或多个表中的行基于相关列组合起来。理解不同类型的JOIN对于构建复杂查询至关重要。

  * **`INNER JOIN`**：只返回两个表中都匹配的行。
  * **`LEFT JOIN` (或 `LEFT OUTER JOIN`)**：返回左表的所有行，以及右表中匹配的行。如果右表中没有匹配，则右表的列显示为NULL。
  * **`RIGHT JOIN` (或 `RIGHT OUTER JOIN`)**：返回右表的所有行，以及左表中匹配的行。如果左表中没有匹配，则左表的列显示为NULL。
  * **`FULL JOIN` (或 `FULL OUTER JOIN`)**：返回两个表中的所有行。如果某行在一个表中没有匹配，则对应表的列显示为NULL。
  * **`CROSS JOIN`**：返回两个表的笛卡尔积，即左表的每一行与右表的每一行组合。通常用于特殊目的，不常用。

**使用场景：** 组合来自不同表的相关数据。

**实战举例：**

```sql
-- INNER JOIN: 查找所有有订单的用户的订单信息
SELECT
    u.username,
    o.order_id,
    o.order_date,
    o.total_amount
FROM
    users u
INNER JOIN
    orders o ON u.user_id = o.user_id;

-- LEFT JOIN: 查找所有用户及其订单信息，包括没有下过订单的用户
SELECT
    u.username,
    o.order_id,
    o.order_date
FROM
    users u
LEFT JOIN
    orders o ON u.user_id = o.user_id;

-- 查找 Alice 的所有订单详情，包括商品信息
SELECT
    o.order_id,
    o.order_date,
    p.product_name,
    oi.quantity,
    oi.unit_price
FROM
    orders o
JOIN
    order_items oi ON o.order_id = oi.order_id
JOIN
    products p ON oi.product_id = p.product_id
WHERE
    o.user_id = (SELECT user_id FROM users WHERE username = 'alice');

-- 查找购买了 "Laptop Pro" 的所有订单信息和用户
SELECT
    u.username,
    o.order_id,
    o.order_date,
    oi.quantity
FROM
    users u
JOIN
    orders o ON u.user_id = o.user_id
JOIN
    order_items oi ON o.order_id = oi.order_id
JOIN
    products p ON oi.product_id = p.product_id
WHERE
    p.product_name = 'Laptop Pro';
```

##### 2.5 子查询与通用表表达式（CTE）

  * **子查询（Subquery）**：一个嵌套在另一个SQL查询内部的查询。子查询可以出现在`SELECT`、`FROM`、`WHERE`或`HAVING`子句中。

  * **通用表表达式（Common Table Expressions, CTE）**：使用`WITH`子句定义的临时命名结果集。CTE提高了查询的可读性和模块化，尤其是在处理复杂查询时。

**使用场景：** 逐步构建复杂查询，提高可读性，解决分步计算问题。

**实战举例：**

**子查询示例：**

```sql
-- 查找购买数量超过平均购买数量的订单项
SELECT
    order_item_id,
    quantity
FROM
    order_items
WHERE
    quantity > (SELECT AVG(quantity) FROM order_items);

-- 查找所有订单总金额高于平均订单总金额的订单
SELECT
    order_id,
    total_amount
FROM
    orders
WHERE
    total_amount > (SELECT AVG(total_amount) FROM orders);
```

**CTE示例：**

```sql
-- 使用 CTE 查找每个用户的订单总金额，并找出总金额最高的3个用户
WITH UserTotalOrders AS (
    SELECT
        user_id,
        SUM(total_amount) AS total_spent
    FROM
        orders
    GROUP BY
        user_id
)
SELECT
    u.username,
    uto.total_spent
FROM
    users u
JOIN
    UserTotalOrders uto ON u.user_id = uto.user_id
ORDER BY
    uto.total_spent DESC
LIMIT 3;

-- 使用 CTE 计算每个商品的销售总量和总收入
WITH ProductSales AS (
    SELECT
        product_id,
        SUM(quantity) AS total_quantity_sold,
        SUM(quantity * unit_price) AS total_revenue
    FROM
        order_items
    GROUP BY
        product_id
)
SELECT
    p.product_name,
    ps.total_quantity_sold,
    ps.total_revenue
FROM
    products p
JOIN
    ProductSales ps ON p.product_id = ps.product_id
ORDER BY
    ps.total_revenue DESC;
```

##### 2.6 窗口函数

窗口函数（Window Functions）对与当前行相关的“窗口”内的一组行执行计算。与聚合函数不同，窗口函数不会将行折叠成单个输出行，而是为查询的每一行返回一个值。它们常用于排名、移动平均、累计求和等场景。

**核心概念：**

  * **`OVER`子句**：定义窗口的范围。
      * `PARTITION BY`：将行分成不同的组（分区）。
      * `ORDER BY`：在每个分区内对行进行排序。
      * `ROWS BETWEEN` / `RANGE BETWEEN`：定义窗口内具体包含哪些行。

**常见窗口函数：**

  * **排名函数**：`ROW_NUMBER()`、`RANK()`、`DENSE_RANK()`、`NTILE(n)`
  * **分析函数**：`LAG()`、`LEAD()`、`FIRST_VALUE()`、`LAST_VALUE()`
  * **聚合函数作为窗口函数**：`SUM() OVER()`, `AVG() OVER()`

**使用场景：** 复杂的排名、趋势分析、前后数据比较。

**实战举例：**

```sql
-- 查找每个用户订单的排名（按订单总金额降序）
SELECT
    user_id,
    order_id,
    total_amount,
    RANK() OVER (PARTITION BY user_id ORDER BY total_amount DESC) as rank_in_user_orders
FROM
    orders;

-- 查找每个用户最近的两次订单
WITH RankedOrders AS (
    SELECT
        user_id,
        order_id,
        order_date,
        total_amount,
        ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY order_date DESC) as rn
    FROM
        orders
)
SELECT
    user_id,
    order_id,
    order_date,
    total_amount
FROM
    RankedOrders
WHERE
    rn <= 2;

-- 计算每个订单项在其所属订单中的累积购买数量
SELECT
    order_id,
    product_id,
    quantity,
    SUM(quantity) OVER (PARTITION BY order_id ORDER BY order_item_id) AS running_total_quantity
FROM
    order_items
ORDER BY
    order_id, order_item_id;

-- 比较每个订单与上一个订单的总金额 (LAG)
SELECT
    order_id,
    order_date,
    total_amount,
    LAG(total_amount, 1, 0) OVER (ORDER BY order_date) AS previous_order_amount,
    total_amount - LAG(total_amount, 1, 0) OVER (ORDER BY order_date) AS amount_difference
FROM
    orders
ORDER BY
    order_date;
```

##### 2.7 数据修改：INSERT, UPDATE, DELETE

除了查询数据，SQL还提供了用于修改数据的命令。

  * **`INSERT INTO`**：向表中添加新行。
  * **`UPDATE`**：修改表中现有行的列值。
  * **`DELETE FROM`**：删除表中的行。

**使用场景：** 增、改、删数据。

**实战举例：**

```sql
-- 插入新用户
INSERT INTO users (username, email) VALUES
('charlie', 'charlie@example.com');

-- 更新商品的库存数量
UPDATE products
SET stock_quantity = 60
WHERE product_name = 'Laptop Pro';

-- 更新 Alice 的第一个订单状态为 'shipped'
UPDATE orders
SET status = 'shipped'
WHERE order_id = (SELECT order_id FROM users u JOIN orders o ON u.user_id = o.user_id WHERE u.username = 'alice' LIMIT 1);

-- 删除特定商品的所有订单项（谨慎操作）
-- 通常不直接删除，而是更新状态或使用事务进行保护
DELETE FROM order_items
WHERE product_id = (SELECT product_id FROM products WHERE product_name = 'Wireless Mouse');

-- 删除用户 'charlie' （因为有 ON DELETE CASCADE，如果 charlie 有订单，订单也会被删除）
DELETE FROM users
WHERE username = 'charlie';
```


##### 2.8 UPSERT 操作：`INSERT ... ON CONFLICT`

在许多场景下，我们需要“如果记录不存在则插入，如果已存在则更新”的逻辑，这通常被称为 UPSERT (Update or Insert)。PostgreSQL 通过 `INSERT ... ON CONFLICT` 语句原生支持此功能。

**语法:**
`INSERT INTO table_name (...) VALUES (...) ON CONFLICT (conflict_target) DO UPDATE SET ...`
`INSERT INTO table_name (...) VALUES (...) ON CONFLICT (conflict_target) DO NOTHING`

- **`conflict_target`**: 指定一个或多个具有唯一约束（UNIQUE a或 PRIMARY KEY）的列，用于判断冲突。
- **`DO UPDATE`**: 当冲突发生时执行更新操作。在 `SET` 子句中，你可以使用特殊的 `EXCLUDED` 表来引用试图插入但失败的行中的值。
- **`DO NOTHING`**: 当冲突发生时，静默地忽略插入操作。

**场景实战：统计网页的每日访问量**

```sql
CREATE TABLE page_views (
    page_url TEXT,
    view_date DATE,
    view_count INTEGER,
    PRIMARY KEY (page_url, view_date)
);

-- 模拟一次访问
INSERT INTO page_views (page_url, view_date, view_count)
VALUES ('/home', '2025-07-12', 1)
ON CONFLICT (page_url, view_date)
DO UPDATE SET view_count = page_views.view_count + 1;

-- 再次访问
INSERT INTO page_views (page_url, view_date, view_count)
VALUES ('/home', '2025-07-12', 1)
ON CONFLICT (page_url, view_date)
DO UPDATE SET view_count = page_views.view_count + EXCLUDED.view_count;
-- 使用 EXCLUDED.view_count (值为1) 比硬编码的 +1 更通用
```

##### 2.9 MERGE 语句：复杂的同步逻辑 (PostgreSQL 15+)

`MERGE` 语句是 SQL:2003 标准的一部分，在 PostgreSQL 15 中被引入。它提供了一个统一的、强大的语法来根据源表同步目标表的数据，可以同时处理匹配、不匹配等多种情况。

**语法:**
```sql
MERGE INTO target_table AS t
USING source_table AS s
ON t.match_column = s.match_column
WHEN MATCHED [AND condition] THEN
    UPDATE SET ... | DELETE
WHEN NOT MATCHED [AND condition] THEN
    INSERT ...
```

**`MERGE` vs `INSERT ... ON CONFLICT`:**
- `ON CONFLICT` 只能处理基于唯一约束的冲突，并且只能执行一种操作（`UPDATE` 或 `NOTHING`）。
- `MERGE` 基于任意的连接条件，并且可以根据不同的条件执行不同的操作，包括 `UPDATE`, `DELETE` 和 `INSERT`，更加灵活和强大。

**场景实战：同步商品库存信息**

假设我们有一个 `warehouse_stock` (仓库库存，目标表) 和一个 `daily_shipments` (每日到货，源表)，需要同步库存。

```sql
-- 目标表
CREATE TABLE warehouse_stock (
    product_id INT PRIMARY KEY,
    stock_quantity INT
);
INSERT INTO warehouse_stock VALUES (1, 100), (2, 50);

-- 源表
CREATE TABLE daily_shipments (
    product_id INT PRIMARY KEY,
    shipment_quantity INT
);
INSERT INTO daily_shipments VALUES (1, 20), (3, 30); -- 商品1到货20件，商品3是新品到货30件

-- 执行 MERGE 操作
MERGE INTO warehouse_stock AS w
USING daily_shipments AS s
ON w.product_id = s.product_id
WHEN MATCHED THEN -- 对于已存在的商品，增加库存
    UPDATE SET stock_quantity = w.stock_quantity + s.shipment_quantity
WHEN NOT MATCHED THEN -- 对于新商品，直接插入
    INSERT (product_id, stock_quantity) VALUES (s.product_id, s.shipment_quantity);

-- 查看结果
SELECT * FROM warehouse_stock;
-- product_id | stock_quantity
-- -----------|----------------
--          2 |             50
--          1 |            120  (100 + 20)
--          3 |             30  (newly inserted)
```

**更复杂的 `MERGE` 场景：**
`MERGE` 还可以处理更复杂的逻辑，例如：
- `WHEN MATCHED AND w.stock_quantity < 10 THEN DELETE`: 当匹配到且库存小于10时，删除该商品。
- `WHEN NOT MATCHED BY SOURCE THEN UPDATE SET ...`: 当目标表中的行在源表中不存在时，对其进行更新（例如，标记为“未盘点”）。


##### 2.10 总结

本章我们深入学习了PostgreSQL中SQL的高级查询和数据操作功能。从基础的`SELECT`语句到复杂的`JOIN`操作，再到强大的聚合函数、子查询、CTE和窗口函数，我们掌握了如何灵活地组合这些工具来应对各种数据分析需求。而 `INSERT ... ON CONFLICT` 和 `MERGE` 语句则为处理数据同步和冲突提供了强大而高效的原生解决方案，使得过去需要复杂应用层逻辑或存储过程才能完成的任务，现在可以用一条简洁的 SQL 语句来完成。同时，我们也复习了数据修改命令`INSERT`、`UPDATE`和`DELETE`。

理解并熟练运用本章内容是成为PostgreSQL专家的基石。在下一章中，我们将聚焦于数据库的可靠性和并发性，深入探讨PostgreSQL的**事务管理与并发控制**机制，确保数据在多用户环境下的正确性和一致性。

-----
