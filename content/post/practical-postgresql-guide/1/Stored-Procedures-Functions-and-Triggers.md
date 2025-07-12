+++
title = "PostgreSQL实战指南：多领域数据库应用与实践 - 第7章：存储过程、函数与触发器"
date = 2025-07-12
lastmod = 2025-07-12T16:00:00+08:00
tags = ["PostgreSQL", "practical", "guide", "book", "stored procedures", "functions", "triggers"]
categories = ["PostgreSQL", "practical", "guide", "book", "stored procedures", "functions", "triggers"]
draft = false
author = "b40yd"
+++

-----

## PostgreSQL实战指南：多领域数据库应用与实践

-----

### 第一部分：关系型数据库深度探索

-----

#### 第7章：存储过程、函数与触发器

数据库不仅仅是存储数据的仓库，它还可以承载和执行复杂的业务逻辑。PostgreSQL通过提供**存储过程（Stored Procedures）**、**函数（Functions）和触发器（Triggers）**，极大地扩展了其在数据库层面的编程能力。这些特性允许你将业务规则、数据转换和自动化任务直接嵌入到数据库中，从而提高数据处理效率、增强数据完整性，并简化应用程序的开发和维护。

本章将深入探讨存储过程、函数和触发器的区别、使用场景、创建语法以及如何在实际应用中发挥它们的最大作用。我们将通过具体的电子商务场景，演示如何利用这些特性来自动化库存管理、日志记录和数据校验等任务。

##### 7.1 函数（Functions）

在PostgreSQL中，**函数**是一段可执行的SQL或过程语言（如PL/pgSQL）代码，它接受零个或多个输入参数，执行一系列操作，并**返回一个值**。函数可以用于各种目的，从简单的计算到复杂的数据查询和转换。

**函数的主要特点：**

  * **返回一个值**：这是函数最主要的特征，可以是任何数据类型，甚至是一个表（表函数）。
  * **可以用于`SELECT`、`WHERE`、`HAVING`、`FROM`子句**：可以在查询的任何部分调用。
  * **可以包含事务控制语句（DCL）**：但本身不是一个事务块，如果需要事务控制，需要在函数内部明确定义。
  * **隔离级别限制**：函数默认运行在调用者的事务中，其隔离级别受限于调用事务。

**PostgreSQL支持的函数语言：**

  * **SQL语言**：最简单，直接执行SQL语句。
  * **PL/pgSQL**：PostgreSQL自带的过程语言，功能强大，支持变量、控制流（IF/ELSE, LOOP）、异常处理等，是编写复杂函数和存储过程的首选。
  * **其他语言**：如PL/Python、PL/Perl、PL/Tcl等，允许你用熟悉的编程语言编写数据库逻辑。

**创建函数的语法：**

```sql
CREATE [OR REPLACE] FUNCTION function_name (parameter_list)
RETURNS data_type [AS $$]
LANGUAGE lang_name
[CALLED ON NULL INPUT | RETURNS NULL ON NULL INPUT | STRICT]
[VOLATILE | STABLE | IMMUTABLE]
AS $$
    -- 函数体（SQL语句或PL/pgSQL代码）
$$;
```

  * `parameter_list`：参数列表，如 `param1 type1, param2 type2`。
  * `RETURNS data_type`：函数返回的数据类型。
  * `LANGUAGE lang_name`：指定函数使用的语言，如 `SQL` 或 `PLPGSQL`。
  * `VOLATILE | STABLE | IMMUTABLE`：函数的**易变性类别**，影响优化器对函数的处理：
      * `VOLATILE`（默认）：每次调用函数时，结果都可能不同（如 `NOW()`）。
      * `STABLE`：在单个语句执行期间，给定相同参数，函数将返回相同结果（如查询表数据的函数）。
      * `IMMUTABLE`：给定相同参数，函数总是返回相同结果，且不访问数据库或仅访问只读表（如数学函数 `SQRT()`）。优化器可以安全地优化和缓存这类函数的结果。

**实战举例：计算订单总金额**

```sql
-- 场景：我们希望有一个函数，给定订单ID，能够计算出该订单所有订单项的总金额。

-- 1. 创建一个简单的 SQL 函数来计算订单总金额
CREATE OR REPLACE FUNCTION calculate_order_total(p_order_id INTEGER)
RETURNS NUMERIC(10, 2)
LANGUAGE SQL
STABLE -- 订单总金额在一次查询中是稳定的，不会随意变化
AS $$
    SELECT SUM(quantity * unit_price)
    FROM order_items
    WHERE order_id = p_order_id;
$$;

-- 调用函数
SELECT calculate_order_total(1); -- 应该返回 1280.00
SELECT o.order_id, o.total_amount, calculate_order_total(o.order_id) AS recalculated_total
FROM orders o;

-- 2. 创建一个 PL/pgSQL 函数来获取用户下过的最高金额订单
CREATE OR REPLACE FUNCTION get_user_highest_order(p_user_id INTEGER)
RETURNS TABLE (order_id INTEGER, total_amount NUMERIC(10, 2), order_date TIMESTAMPTZ) -- 返回一个表
LANGUAGE PLPGSQL
STABLE
AS $$
DECLARE
    v_order_id INTEGER;
    v_total_amount NUMERIC(10, 2);
    v_order_date TIMESTAMPTZ;
BEGIN
    SELECT order_id, total_amount, order_date
    INTO v_order_id, v_total_amount, v_order_date
    FROM orders
    WHERE user_id = p_user_id
    ORDER BY total_amount DESC
    LIMIT 1;

    RETURN QUERY SELECT v_order_id, v_total_amount, v_order_date; -- 返回查询结果集
END;
$$;

-- 调用表函数
SELECT * FROM get_user_highest_order(1); -- 查询 Alice 的最高金额订单
```

##### 7.2 存储过程（Stored Procedures）

从PostgreSQL 11开始，引入了对SQL标准**存储过程**的支持。与函数不同，存储过程不强制返回一个值，但更重要的是，它们可以包含**事务控制语句**（如`COMMIT`和`ROLLBACK`），这使得它们更适合执行一系列复杂、需要事务完整性的操作。

**存储过程的主要特点：**

  * **不强制返回一个值**：可以返回零个或多个结果集（通过 `OUT` 参数），也可以不返回任何值。
  * **支持事务控制**：可以在过程内部使用 `COMMIT` 和 `ROLLBACK`，这对于复杂业务流程（例如批处理、数据迁移）至关重要。
  * **不能直接在`SELECT`、`WHERE`等子句中调用**：只能通过 `CALL` 语句调用。
  * **独立的事务上下文**：存储过程可以在其内部管理事务。

**创建存储过程的语法：**

```sql
CREATE [OR REPLACE] PROCEDURE procedure_name (parameter_list)
LANGUAGE lang_name
AS $$
    -- 过程体（PL/pgSQL代码）
$$;
```

**调用存储过程的语法：**

```sql
CALL procedure_name (argument_list);
```

**实战举例：处理批量订单库存扣减**

```sql
-- 场景：实现一个存储过程，用于原子性地处理多个订单项的库存扣减。
-- 如果任何一个商品库存不足，整个批处理操作都应该回滚。

CREATE OR REPLACE PROCEDURE process_order_items_stock_deduction(
    p_order_id INTEGER
)
LANGUAGE PLPGSQL
AS $$
DECLARE
    v_product_id INTEGER;
    v_quantity INTEGER;
    v_current_stock INTEGER;
BEGIN
    -- 遍历订单中的每个订单项
    FOR v_product_id, v_quantity IN
        SELECT product_id, quantity
        FROM order_items
        WHERE order_id = p_order_id
    LOOP
        -- 检查当前库存（使用 FOR UPDATE 锁定行，防止并发问题）
        SELECT stock_quantity INTO v_current_stock
        FROM products
        WHERE product_id = v_product_id
        FOR UPDATE; -- 关键：获取排他行锁

        -- 检查库存是否足够
        IF v_current_stock < v_quantity THEN
            RAISE EXCEPTION 'Product ID % has insufficient stock. Required: %, Available: %',
                            v_product_id, v_quantity, v_current_stock;
        END IF;

        -- 扣减库存
        UPDATE products
        SET stock_quantity = stock_quantity - v_quantity
        WHERE product_id = v_product_id;
    END LOOP;

    -- 如果所有库存扣减都成功，则自动提交外部事务（如果外部有 BEGIN）
    -- 否则，存储过程内部的 RAISE EXCEPTION 会导致事务回滚
    RAISE NOTICE 'Order % stock deduction completed successfully.', p_order_id;

EXCEPTION
    WHEN OTHERS THEN
        RAISE NOTICE 'An error occurred during stock deduction for Order %: %', p_order_id, SQLERRM;
        -- 在存储过程中，通常会记录错误并让调用者决定是否回滚
        -- 如果外部没有 BEGIN，可以在这里直接 ROLLBACK;
        -- 但在大多数情况下，是外部事务来控制
END;
$$;

-- 模拟调用：
-- 假设 product_id 1 (Laptop Pro) 库存 50，product_id 2 (Keyboard) 库存 200
-- 现有订单 1，包含 Laptop Pro (1个) 和 Keyboard (1个)

-- 成功情况：
BEGIN;
CALL process_order_items_stock_deduction(1);
-- COMMIT; -- 或者 ROLLBACK;
SELECT product_id, stock_quantity FROM products WHERE product_id IN (1,2);
-- 预期：Laptop Pro 49, Keyboard 199
COMMIT;

-- 失败情况（模拟库存不足）:
-- 假设我们尝试扣减 Keyboard 250个，但库存只有199
UPDATE order_items SET quantity = 250 WHERE order_id = 1 AND product_id = 2; -- 临时修改订单项数量
BEGIN;
CALL process_order_items_stock_deduction(1); -- 会抛出异常并提示库存不足
-- ROLLBACK; -- 外部事务回滚
SELECT product_id, stock_quantity FROM products WHERE product_id IN (1,2);
-- 预期：库存保持不变，因为事务被回滚
```

##### 7.3 触发器（Triggers）

**触发器**是一种特殊的数据库对象，它在特定数据库事件（如`INSERT`、`UPDATE`、`DELETE`）发生时自动执行预定义的函数或存储过程。触发器是实现业务规则、审计日志、数据同步或复杂数据完整性检查的强大工具。

**触发器的主要特点：**

  * **事件驱动**：当`INSERT`、`UPDATE`或`DELETE`操作在指定表上发生时被激活。
  * **时机**：可以在事件发生\*\*之前（BEFORE）**或**之后（AFTER）\*\*触发。
  * **粒度**：可以为\*\*每行（FOR EACH ROW）**触发，也可以为**每个语句（FOR EACH STATEMENT）\*\*触发。
      * `FOR EACH ROW`：对受事件影响的每一行执行一次触发器。
      * `FOR EACH STATEMENT`：无论影响多少行，对整个语句只执行一次触发器。
  * **条件（WHEN）**：可以指定一个条件，只有当条件满足时才执行触发器函数。
  * **特殊的 `OLD` 和 `NEW` 伪行变量**：在`FOR EACH ROW`触发器中，可以访问正在被操作行的旧值（`OLD`）和新值（`NEW`）。

**创建触发器的语法：**

```sql
CREATE [CONSTRAINT] TRIGGER trigger_name
{ BEFORE | AFTER | INSTEAD OF } { INSERT | UPDATE [ OF column_name [, ... ] ] | DELETE | TRUNCATE }
ON table_name
[ FROM referenced_table_name ]
[ NOT DEFERRABLE | [ DEFERRABLE ] { INITIALLY IMMEDIATE | INITIALLY DEFERRED } ]
[ FOR EACH ROW | FOR EACH STATEMENT ]
[ WHEN ( condition ) ]
EXECUTE FUNCTION function_name ( arguments );
```

**实战举例：库存变动日志和订单总金额自动更新**

**场景一：库存变动日志**

我们希望每次商品库存数量发生变化时，都能自动记录到`stock_logs`表中，方便追溯和审计。

```sql
-- 1. 创建库存日志表
CREATE TABLE stock_logs (
    log_id SERIAL PRIMARY KEY,
    product_id INTEGER NOT NULL,
    old_stock INTEGER,
    new_stock INTEGER,
    change_amount INTEGER,
    change_time TIMESTAMPTZ DEFAULT NOW(),
    operation_type VARCHAR(10) -- 'INSERT', 'UPDATE', 'DELETE' (虽然我们只关注 UPDATE)
);

-- 2. 创建一个触发器函数，用于记录库存日志
CREATE OR REPLACE FUNCTION log_stock_changes()
RETURNS TRIGGER
LANGUAGE PLPGSQL
AS $$
BEGIN
    IF TG_OP = 'UPDATE' THEN -- 只处理更新操作
        IF NEW.stock_quantity IS DISTINCT FROM OLD.stock_quantity THEN -- 只有当库存真正改变时才记录
            INSERT INTO stock_logs (product_id, old_stock, new_stock, change_amount, operation_type)
            VALUES (OLD.product_id, OLD.stock_quantity, NEW.stock_quantity, NEW.stock_quantity - OLD.stock_quantity, 'UPDATE');
        END IF;
    END IF;
    RETURN NEW; -- 对于 AFTER 触发器，通常返回 NEW 或 OLD，对于 BEFORE 触发器，返回 NEW 或 NULL
END;
$$;

-- 3. 在 products 表上创建触发器
CREATE TRIGGER trg_log_product_stock
AFTER UPDATE OF stock_quantity ON products -- 当 products 表的 stock_quantity 列更新后
FOR EACH ROW -- 对每一行受影响的行执行
EXECUTE FUNCTION log_stock_changes();

-- 演示触发器效果
UPDATE products SET stock_quantity = 48 WHERE product_id = 1; -- 之前是49
UPDATE products SET stock_quantity = 200 WHERE product_id = 2; -- 之前是199
UPDATE products SET description = 'Updated Description' WHERE product_id = 3; -- 不会触发，因为没有更新 stock_quantity

SELECT * FROM stock_logs;
```

**场景二：订单总金额自动更新**

我们希望`orders`表的`total_amount`字段能根据`order_items`表的变动自动更新。

```sql
-- 1. 创建一个触发器函数，用于更新订单总金额
CREATE OR REPLACE FUNCTION update_order_total_amount()
RETURNS TRIGGER
LANGUAGE PLPGSQL
AS $$
BEGIN
    IF TG_OP = 'INSERT' OR TG_OP = 'UPDATE' THEN
        -- 对于插入或更新操作，更新 NEW.order_id 对应的订单
        UPDATE orders
        SET total_amount = calculate_order_total(NEW.order_id)
        WHERE order_id = NEW.order_id;
        RETURN NEW;
    ELSIF TG_OP = 'DELETE' THEN
        -- 对于删除操作，更新 OLD.order_id 对应的订单
        UPDATE orders
        SET total_amount = calculate_order_total(OLD.order_id)
        WHERE order_id = OLD.order_id;
        RETURN OLD;
    END IF;
    RETURN NULL; -- 对于 statement-level 触发器，通常返回 NULL
END;
$$;

-- 2. 在 order_items 表上创建触发器
CREATE TRIGGER trg_update_order_total
AFTER INSERT OR UPDATE OR DELETE ON order_items -- 在 order_items 插入、更新或删除后
FOR EACH ROW -- 对每一行受影响的行执行
EXECUTE FUNCTION update_order_total_amount();

-- 演示触发器效果
-- 检查订单1的总金额
SELECT order_id, total_amount FROM orders WHERE order_id = 1; -- 假设当前是 1280.00

-- 向订单1中添加一个新商品项 (Wireless Mouse, 2个)
INSERT INTO order_items (order_id, product_id, quantity, unit_price) VALUES (1, 3, 2, 25.00);

-- 再次检查订单1的总金额，应该自动更新 (1280 + 2*25 = 1330.00)
SELECT order_id, total_amount FROM orders WHERE order_id = 1;

-- 更新订单项数量
UPDATE order_items SET quantity = 3 WHERE order_id = 1 AND product_id = 3; -- 之前是2个，现在是3个

-- 再次检查订单1的总金额，应该自动更新 (1280 + 3*25 = 1355.00)
SELECT order_id, total_amount FROM orders WHERE order_id = 1;

-- 删除一个订单项
DELETE FROM order_items WHERE order_id = 1 AND product_id = 3;

-- 再次检查订单1的总金额，应该自动回到 1280.00
SELECT order_id, total_amount FROM orders WHERE order_id = 1;
```

##### 7.4 存储过程、函数与触发器的选择与注意事项

| 特性           | 函数（Function）                       | 存储过程（Stored Procedure）           | 触发器（Trigger）                      |
| :------------- | :------------------------------------- | :------------------------------------- | :------------------------------------- |
| **返回类型** | 必须返回一个值（或表）                 | 不强制返回，或通过`OUT`参数返回        | 不返回值，但触发器函数本身必须返回`TRIGGER`类型 |
| **事务控制** | 不允许直接使用`COMMIT/ROLLBACK`（除非是子事务） | **允许**使用`COMMIT/ROLLBACK`         | 不允许直接使用`COMMIT/ROLLBACK`       |
| **调用方式** | `SELECT function_name()`, `WHERE`, `JOIN`等 | `CALL procedure_name()`                 | 自动执行，无需手动调用                  |
| **执行时机** | 应用程序或SQL查询显式调用              | 应用程序或SQL脚本显式调用              | 在数据修改事件发生时自动执行            |
| **适用场景** | 计算、数据转换、复杂查询封装、表函数   | 复杂批处理、数据迁移、需要事务原子性的多步操作 | 自动化业务规则、审计日志、数据同步、复杂完整性检查 |
| **开销** | 相对较低                               | 较高（如果包含复杂逻辑和事务控制）    | 每次数据修改都会有额外开销，可能影响写性能 |

**注意事项：**

  * **性能影响**：函数和触发器虽然强大，但滥用或设计不当可能导致性能问题。尤其是`FOR EACH ROW`触发器，如果操作复杂，在批量数据修改时会显著增加开销。
  * **可维护性**：复杂的业务逻辑应优先考虑在应用程序层面实现。只有那些与数据紧密耦合、且数据库层面实现更高效或更安全（如强制数据完整性）的逻辑才考虑使用。
  * **调试**：数据库层面的代码调试通常比应用程序代码更复杂。
  * **幂等性**：设计存储过程和函数时考虑幂等性，即多次执行相同操作不会产生额外副作用。
  * **错误处理**：在PL/pgSQL代码中，应使用`EXCEPTION`块进行适当的错误处理，以提高健壮性。

##### 7.5 总结

本章我们深入学习了PostgreSQL的**存储过程、函数和触发器**，掌握了如何利用它们来扩展数据库的功能，实现更复杂的业务逻辑和自动化数据操作。我们理解了函数和存储过程在返回类型和事务控制上的关键区别，并学会了如何根据不同的业务需求选择合适的工具。此外，我们还通过触发器的实战，看到了它们在自动化审计和数据同步方面的巨大潜力。

通过本章的学习，你现在应该能够：

  * 区分PostgreSQL函数和存储过程，并根据场景进行选择。
  * 熟练使用PL/pgSQL编写函数和存储过程，处理变量、控制流和异常。
  * 创建和管理触发器，实现数据修改的自动化响应。
  * 在实际应用中，合理利用这些高级特性来提高数据完整性和系统效率。

在下一章中，我们将聚焦于如何高效管理大规模数据集，深入探讨**分区表与大规模数据管理**，这是处理巨量数据和优化查询性能的重要策略。

-----