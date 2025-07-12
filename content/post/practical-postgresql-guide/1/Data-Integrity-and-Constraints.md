+++
title = "PostgreSQL实战指南：多领域数据库应用与实践 - 第5章：数据完整性与约束"
date = 2025-07-12
lastmod = 2025-07-12T16:00:00+08:00
tags = ["PostgreSQL", "practical", "guide", "book", "data integrity", "constraints"]
categories = ["PostgreSQL", "practical", "guide", "book", "data integrity", "constraints"]
draft = false
author = "b40yd"
+++

-----

## PostgreSQL实战指南：多领域数据库应用与实践

-----

### 第一部分：关系型数据库深度探索

-----

#### 第5章：数据完整性与约束

在任何数据驱动的应用程序中，数据的**准确性、有效性和一致性**至关重要。设想一下，如果用户的邮箱地址格式错误，或者订单的总金额为负数，这些“脏数据”将严重影响业务决策和系统稳定性。PostgreSQL通过强大的\*\*约束（Constraints）\*\*机制，帮助我们在数据库层面强制实施这些数据完整性规则，从而在数据写入时就进行严格把关，避免无效或不一致的数据进入系统。

本章将深入探讨PostgreSQL支持的各种数据完整性约束，包括**主键约束（Primary Key）**、**外键约束（Foreign Key）**、**唯一性约束（Unique）**、**非空约束（NOT NULL）和检查约束（CHECK）**。我们将通过实际场景的举例，演示如何有效地应用这些约束，并理解它们在维护数据质量和简化应用逻辑方面的重要性。

##### 5.1 数据完整性概述

**数据完整性**是指数据库中数据的准确性和一致性。它是数据库设计和管理中一个非常重要的概念。数据完整性通常可以分为以下几类：

  * **实体完整性（Entity Integrity）**：确保表中的每一行都是唯一的，并且可以被唯一标识。这主要通过**主键约束**实现。
  * **参照完整性（Referential Integrity）**：维护表与表之间关系的有效性。这主要通过**外键约束**实现，确保引用关系指向有效的数据。
  * **域完整性（Domain Integrity）**：确保列中的数据符合预定义的类型、格式和取值范围。这通过**数据类型**、**NOT NULL**约束和**CHECK**约束实现。
  * **用户定义完整性（User-Defined Integrity）**：根据特定的业务规则，由用户自定义的完整性约束。这通常也通过**CHECK**约束、\*\*触发器（Triggers）**或**存储过程（Stored Procedures）\*\*实现。

##### 5.2 主键约束（PRIMARY KEY）

**主键**是用于唯一标识表中每一行的列或列的组合。它有以下特点：

  * **唯一性（Unique）**：主键列的值不能重复。
  * **非空性（NOT NULL）**：主键列的值不能为NULL。
  * **强制性**：每个表通常都应该有一个主键。

在PostgreSQL中，`PRIMARY KEY`约束会自动创建一个**唯一索引**。

**使用场景：** 唯一标识用户、商品、订单等实体。

**实战举例：**

在之前的电子商务订单系统中，我们已经为 `users`、`products`、`orders` 和 `order_items` 表定义了主键。

```sql
-- users 表的 user_id 列作为主键，自动生成且唯一
CREATE TABLE users (
    user_id SERIAL PRIMARY KEY, -- SERIAL 会自动递增，并隐式添加 NOT NULL 和 UNIQUE
    username VARCHAR(50) UNIQUE NOT NULL,
    email VARCHAR(100) UNIQUE NOT NULL,
    registration_date TIMESTAMPTZ DEFAULT NOW()
);

-- products 表的 product_id 列作为主键
CREATE TABLE products (
    product_id SERIAL PRIMARY KEY,
    product_name VARCHAR(255) NOT NULL,
    description TEXT,
    price NUMERIC(10, 2) NOT NULL CHECK (price >= 0),
    stock_quantity INTEGER NOT NULL CHECK (stock_quantity >= 0)
);

-- 订单项表可以使用复合主键，确保一个订单中同一商品只能出现一次
-- 虽然我们之前用了 SERIAL 和 UNIQUE(order_id, product_id)
-- 另一种设计可以是直接使用复合主键：
/*
CREATE TABLE order_items (
    order_id INTEGER NOT NULL REFERENCES orders(order_id) ON DELETE CASCADE,
    product_id INTEGER NOT NULL REFERENCES products(product_id) ON DELETE RESTRICT,
    quantity INTEGER NOT NULL CHECK (quantity > 0),
    unit_price NUMERIC(10, 2) NOT NULL CHECK (unit_price >= 0),
    PRIMARY KEY (order_id, product_id) -- 将 order_id 和 product_id 组合作为主键
);
*/
```

##### 5.3 外键约束（FOREIGN KEY）

**外键**用于建立和维护两个表之间的参照完整性。它引用另一个表（父表）的**主键**或**唯一键**。外键确保了子表中的数据引用父表中存在的有效数据。

**使用场景：** 确保订单的`user_id`对应一个真实存在的用户，订单项的`product_id`对应一个真实存在的商品。

**核心概念：**

  * **`REFERENCES parent_table(parent_column)`**：指定外键引用的父表和父列。
  * **`ON DELETE action`**：当父表中的被引用行被删除时，子表的行为。
      * `NO ACTION`（默认）：如果存在依赖行，不允许删除父行。
      * `RESTRICT`：与`NO ACTION`类似，但在检查时机上略有不同。
      * `CASCADE`：级联删除。删除父行时，自动删除子表中所有引用该父行的行。
      * `SET NULL`：将子表中引用该父行的外键列设置为NULL（要求该外键列允许为NULL）。
      * `SET DEFAULT`：将子表中引用该父行的外键列设置为其默认值（要求该外键列有默认值）。
  * **`ON UPDATE action`**：当父表中的被引用行的主键（或唯一键）被更新时，子表的行为。选项与`ON DELETE`类似。

**实战举例：**

在`orders`表中，`user_id`是外键，引用`users`表的`user_id`。
在`order_items`表中，`order_id`是外键，引用`orders`表的`order_id`；`product_id`是外键，引用`products`表的`product_id`。

```sql
-- orders 表的 user_id 引用 users 表的 user_id
CREATE TABLE orders (
    order_id SERIAL PRIMARY KEY,
    user_id INTEGER NOT NULL REFERENCES users(user_id) ON DELETE CASCADE, -- 当用户被删除时，其所有订单也删除
    order_date TIMESTAMPTZ DEFAULT NOW(),
    total_amount NUMERIC(10, 2) NOT NULL CHECK (total_amount >= 0),
    status order_status DEFAULT 'pending'
);

-- order_items 表的 order_id 引用 orders 表的 order_id，product_id 引用 products 表的 product_id
CREATE TABLE order_items (
    order_item_id SERIAL PRIMARY KEY,
    order_id INTEGER NOT NULL REFERENCES orders(order_id) ON DELETE CASCADE, -- 当订单被删除时，其订单项也删除
    product_id INTEGER NOT NULL REFERENCES products(product_id) ON DELETE RESTRICT, -- 当商品有订单项关联时，不允许删除商品
    quantity INTEGER NOT NULL CHECK (quantity > 0),
    unit_price NUMERIC(10, 2) NOT NULL CHECK (unit_price >= 0),
    UNIQUE (order_id, product_id)
);

-- 尝试插入一个不存在的用户ID的订单 (会报错)
-- INSERT INTO orders (user_id, total_amount, status) VALUES (999, 50.00, 'pending');
-- ERROR:  insert or update on table "orders" violates foreign key constraint "orders_user_id_fkey"

-- 尝试删除一个有订单项关联的商品 (会报错，因为 ON DELETE RESTRICT)
-- DELETE FROM products WHERE product_id = 1;
-- ERROR:  update or delete on table "products" violates foreign key constraint "order_items_product_id_fkey" on table "order_items"
```

##### 5.4 唯一性约束（UNIQUE）

**唯一性约束**确保指定列（或列的组合）中的所有值都是唯一的。与主键不同的是：

  * 一个表可以有多个唯一性约束。
  * 被唯一性约束的列可以包含一个NULL值（除非同时指定了`NOT NULL`），因为NULL在SQL中不等于任何其他值，包括另一个NULL。

**使用场景：** 确保用户名、邮箱地址、商品SKU（库存单位）等在系统中是唯一的。

**实战举例：**

在`users`表中，`username`和`email`都设置为`UNIQUE`。

```sql
CREATE TABLE users (
    user_id SERIAL PRIMARY KEY,
    username VARCHAR(50) UNIQUE NOT NULL, -- 用户名必须唯一且非空
    email VARCHAR(100) UNIQUE NOT NULL,   -- 邮箱必须唯一且非空
    registration_date TIMESTAMPTZ DEFAULT NOW()
);

-- 尝试插入重复的用户名或邮箱 (会报错)
-- INSERT INTO users (username, email) VALUES ('alice', 'alice@example.com');
-- ERROR:  duplicate key value violates unique constraint "users_username_key"
-- ERROR:  duplicate key value violates unique constraint "users_email_key"
```

##### 5.5 非空约束（NOT NULL）

**非空约束**确保列不能包含NULL值。如果尝试插入或更新NULL值到带有`NOT NULL`约束的列中，将导致错误。

**使用场景：** 强制某些关键信息（如商品名称、订单总金额）必须存在。

**实战举例：**

在我们的表中，许多关键列都使用了`NOT NULL`。

```sql
-- products 表的 product_name、price、stock_quantity 都定义为 NOT NULL
CREATE TABLE products (
    product_id SERIAL PRIMARY KEY,
    product_name VARCHAR(255) NOT NULL, -- 商品名称不能为空
    description TEXT,
    price NUMERIC(10, 2) NOT NULL CHECK (price >= 0),
    stock_quantity INTEGER NOT NULL CHECK (stock_quantity >= 0)
);

-- 尝试插入一个没有 product_name 的商品 (会报错)
-- INSERT INTO products (description, price, stock_quantity) VALUES ('Test description', 10.00, 100);
-- ERROR:  null value in column "product_name" of relation "products" violates not-null constraint
```

##### 5.6 检查约束（CHECK）

**检查约束**允许你定义一个布尔表达式，表中插入或更新的每一行都必须满足这个表达式。这提供了最灵活的数据完整性控制。

**使用场景：**

  * 限制数值列的范围（例如，年龄必须大于0，价格不能为负）。
  * 限制字符串列的格式（例如，邮政编码必须是5位数字）。
  * 强制枚举类型的值。
  * 复杂业务逻辑的验证。

**实战举例：**

在`products`表中，`price`和`stock_quantity`都使用了`CHECK`约束。

```sql
-- products 表中价格和库存的 CHECK 约束
CREATE TABLE products (
    product_id SERIAL PRIMARY KEY,
    product_name VARCHAR(255) NOT NULL,
    description TEXT,
    price NUMERIC(10, 2) NOT NULL CHECK (price >= 0),          -- 价格必须大于等于0
    stock_quantity INTEGER NOT NULL CHECK (stock_quantity >= 0) -- 库存数量必须大于等于0
);

-- orders 表中总金额的 CHECK 约束
CREATE TABLE orders (
    order_id SERIAL PRIMARY KEY,
    user_id INTEGER NOT NULL REFERENCES users(user_id) ON DELETE CASCADE,
    order_date TIMESTAMPTZ DEFAULT NOW(),
    total_amount NUMERIC(10, 2) NOT NULL CHECK (total_amount >= 0), -- 总金额必须大于等于0
    status order_status DEFAULT 'pending'
);

-- order_items 表中购买数量的 CHECK 约束
CREATE TABLE order_items (
    order_item_id SERIAL PRIMARY KEY,
    order_id INTEGER NOT NULL REFERENCES orders(order_id) ON DELETE CASCADE,
    product_id INTEGER NOT NULL REFERENCES products(product_id) ON DELETE RESTRICT,
    quantity INTEGER NOT NULL CHECK (quantity > 0), -- 购买数量必须大于0
    unit_price NUMERIC(10, 2) NOT NULL CHECK (unit_price >= 0),
    UNIQUE (order_id, product_id)
);

-- 尝试插入不符合 CHECK 约束的数据 (会报错)
-- INSERT INTO products (product_name, price, stock_quantity) VALUES ('Negative Price Item', -10.00, 100);
-- ERROR:  new row for relation "products" violates check constraint "products_price_check"

-- INSERT INTO order_items (order_id, product_id, quantity, unit_price) VALUES (1, 1, 0, 1200.00);
-- ERROR:  new row for relation "order_items" violates check constraint "order_items_quantity_check"
```

##### 5.7 延迟约束（Deferrable Constraints）

PostgreSQL还支持**延迟约束检查**。通常，约束会在每个语句执行后立即检查。但对于某些复杂的场景，例如需要在一个事务中交换两个唯一值时，立即检查可能会导致临时违规。通过将约束定义为`DEFERRABLE`，你可以在事务结束时（提交时）才进行检查。

**语法：**

```sql
CREATE TABLE my_table (
    id1 INTEGER UNIQUE DEFERRABLE INITIALLY DEFERRED,
    id2 INTEGER UNIQUE DEFERRABLE INITIALLY IMMEDIATE
);

-- 在事务中设置约束检查时机
SET CONSTRAINTS ALL DEFERRED;
SET CONSTRAINTS ALL IMMEDIATE;
```

  * `INITIALLY IMMEDIATE`（默认）：约束在每个语句结束后立即检查。
  * `INITIALLY DEFERRED`：约束在事务提交时检查。

**使用场景：** 在单个事务中进行复杂的数据交换或批量操作，其中中间步骤可能会暂时违反约束，但在事务结束时会恢复一致。

**实战举例：交换两个用户的邮箱地址（唯一性约束）**

```sql
-- 假设 users 表的 email 字段有 UNIQUE NOT NULL 约束
-- 尝试直接交换 alice 和 bob 的邮箱地址会失败，因为会临时出现重复
-- UPDATE users SET email = 'bob@example.com' WHERE username = 'alice';
-- UPDATE users SET email = 'alice@example.com' WHERE username = 'bob'; -- 此处会报错

-- 使用 DEFERRABLE 约束
ALTER TABLE users DROP CONSTRAINT users_email_key; -- 先删除旧约束
ALTER TABLE users ADD CONSTRAINT users_email_key UNIQUE (email) DEFERRABLE INITIALLY DEFERRED;

BEGIN;
SET CONSTRAINTS ALL DEFERRED; -- 在当前事务中延迟所有可延迟的约束检查

-- 交换操作
UPDATE users SET email = 'temp_bob@example.com' WHERE username = 'bob';
UPDATE users SET email = 'bob@example.com' WHERE username = 'alice';
UPDATE users SET email = 'alice@example.com' WHERE username = 'bob';

COMMIT; -- 此时才会检查唯一性约束
```

##### 5.8 总结

本章我们深入探讨了PostgreSQL中维护数据完整性的关键机制——**约束**。我们学习了主键、外键、唯一性、非空和检查约束各自的作用、使用场景以及如何通过SQL语句进行定义。理解并合理应用这些约束，是确保数据库数据质量、减少应用程序端数据校验逻辑、以及构建健壮可靠系统的基础。

通过本章的学习，你现在应该能够：

  * 理解数据完整性的不同类型和重要性。
  * 熟练运用`PRIMARY KEY`、`FOREIGN KEY`、`UNIQUE`、`NOT NULL`和`CHECK`约束。
  * 根据业务需求选择合适的约束类型和行为（如`ON DELETE`）。
  * 了解并能在特定场景下使用`DEFERRABLE`约束。

在下一章中，我们将把目光转向数据库查询结果的灵活呈现——**视图与物化视图的应用**，学习如何通过它们简化复杂查询、提高数据访问效率和安全性。

-----
