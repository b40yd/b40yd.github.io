+++
title = "PostgreSQL 数据库实战指南 - PL/pgSQL 编写存储过程详解"
date = 2025-07-04
lastmod = 2025-07-04T01:00:00+08:00
tags = ["PostgreSQL", "practical", "book", "function", "procedure", "trigger"]
categories = ["PostgreSQL", "practical", "book", "function", "procedure", "trigger"]
draft = false
author = "b40yd"
+++

# 第四章 函数、触发器与过程语言  
## 第一节 PL/pgSQL 编写存储过程详解

> **目标**：深入掌握 PostgreSQL 中使用 PL/pgSQL 编写存储过程的方法，理解事务控制、异常处理、游标和循环等高级结构，并能够结合业务场景设计高性能的数据库逻辑层。

随着业务复杂度的提升，将部分逻辑从应用层下沉到数据库层变得越来越常见。PostgreSQL 支持丰富的存储过程功能，尤其在 v11 引入对存储过程（Procedure）的支持后，使得数据库具备了完整的事务控制能力。

本节将围绕以下核心内容展开：

- 存储过程与函数的区别
- PL/pgSQL 基础语法结构
- 事务控制（COMMIT / ROLLBACK）
- 游标（Cursor）与循环结构
- 异常处理机制
- 实战案例：批量订单状态更新的存储过程实现

---

## 📌 一、存储过程 vs 函数

| 特性 | 函数（Function） | 存储过程（Procedure） |
|------|------------------|------------------------|
| 是否有返回值 | ✅ 有 | ❌ 无（可使用 OUT 参数） |
| 可否调用 COMMIT/ROLLBACK | ❌ 不支持 | ✅ 支持 |
| 调用方式 | SELECT 或 PERFORM（在 PL/pgSQL 中） | CALL |
| 是否可在 SQL 中直接调用 | ✅ | ❌ |

### 示例对比：

#### 函数调用：

```sql
SELECT calculate_total_sales('2024-01-01', '2024-12-31');
```

#### 存储过程调用：

```sql
CALL update_order_status_batch('shipped');
```

---

## 🧱 二、PL/pgSQL 基础语法结构

PL/pgSQL 是 PostgreSQL 内置的过程语言，其基本语法结构如下：

```sql
CREATE OR REPLACE PROCEDURE procedure_name(param1 type, param2 type, ...)
LANGUAGE plpgsql
AS $$
DECLARE
    -- 变量声明区
BEGIN
    -- 执行体
EXCEPTION
    -- 异常处理
END;
$$;
```

---

## 🔁 三、变量声明与赋值

### 1. 声明变量

```sql
DECLARE
    counter INT := 0;
    customer_name TEXT;
    total_amount NUMERIC(10,2) DEFAULT 0;
```

### 2. 赋值操作

```sql
counter := counter + 1;

SELECT name INTO customer_name FROM customers WHERE id = 1;

total_amount := calculate_total(customer_id);
```

---

## 💡 四、流程控制语句

### 1. IF 判断

```sql
IF total_amount > 1000 THEN
    RAISE NOTICE '大额订单';
ELSIF total_amount BETWEEN 500 AND 1000 THEN
    RAISE NOTICE '中等订单';
ELSE
    RAISE NOTICE '小额订单';
END IF;
```

### 2. LOOP 循环

```sql
LOOP
    counter := counter + 1;
    EXIT WHEN counter >= 10;
END LOOP;
```

### 3. FOR 循环（遍历结果集）

```sql
FOR rec IN SELECT * FROM orders WHERE status = 'pending' LOOP
    UPDATE orders SET status = 'processing' WHERE order_id = rec.order_id;
END LOOP;
```

---

## 🔄 五、事务控制（COMMIT / ROLLBACK）

存储过程支持显式事务控制，这在执行批量操作或关键数据变更时非常有用。

### 示例：插入用户并提交事务

```sql
CREATE OR REPLACE PROCEDURE create_user_and_commit(username TEXT)
LANGUAGE plpgsql
AS $$
BEGIN
    INSERT INTO users (username) VALUES ($1);
    COMMIT;
END;
$$;
```

### 示例：出现错误时回滚事务

```sql
CREATE OR REPLACE PROCEDURE transfer_funds(from_id INT, to_id INT, amount NUMERIC)
LANGUAGE plpgsql
AS $$
BEGIN
    UPDATE accounts SET balance = balance - amount WHERE id = from_id;
    UPDATE accounts SET balance = balance + amount WHERE id = to_id;

    IF (SELECT balance < 0 FROM accounts WHERE id = from_id) THEN
        RAISE EXCEPTION '余额不足';
    END IF;

    COMMIT;
EXCEPTION
    WHEN OTHERS THEN
        ROLLBACK;
        RAISE NOTICE '转账失败：% %', SQLERRM, SQLSTATE;
END;
$$;
```

---

## 🧭 六、游标（Cursor）与大数据处理

当需要逐条处理大量数据时，可以使用游标来减少内存占用。

### 1. 显式游标示例

```sql
CREATE OR REPLACE PROCEDURE process_large_dataset()
LANGUAGE plpgsql
AS $$
DECLARE
    cur CURSOR FOR SELECT * FROM orders WHERE processed = FALSE;
    rec RECORD;
BEGIN
    OPEN cur;
    LOOP
        FETCH cur INTO rec;
        EXIT WHEN NOT FOUND;

        -- 处理单条记录
        UPDATE orders SET processed = TRUE WHERE order_id = rec.order_id;
    END LOOP;
    CLOSE cur;
END;
$$;
```

### 2. 使用 FOR 循环简化游标

```sql
FOR rec IN SELECT * FROM orders WHERE processed = FALSE LOOP
    UPDATE orders SET processed = TRUE WHERE order_id = rec.order_id;
END LOOP;
```

---

## ⚠️ 七、异常处理机制

PostgreSQL 支持强大的异常处理机制，可以在出错时优雅地进行日志记录或回滚操作。

### 示例：捕获特定异常类型

```sql
BEGIN
    -- 尝试删除一个被引用的记录
    DELETE FROM products WHERE product_id = 1;
EXCEPTION
    WHEN foreign_key_violation THEN
        RAISE NOTICE '该商品已被引用，无法删除';
    WHEN others THEN
        RAISE NOTICE '发生未知错误：% %', SQLERRM, SQLSTATE;
END;
```

---

## 🧪 八、实战演练：批量更新订单状态的存储过程

### 场景描述：

你需要编写一个存储过程，将所有“已付款”状态的订单统一改为“已发货”，并在过程中添加日志记录和异常处理。

### 步骤如下：

#### 1. 创建订单表（如尚未存在）

```sql
CREATE TABLE orders (
    order_id SERIAL PRIMARY KEY,
    user_id INT,
    status TEXT DEFAULT 'pending',
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

#### 2. 创建日志表

```sql
CREATE TABLE order_update_log (
    log_id SERIAL PRIMARY KEY,
    old_status TEXT,
    new_status TEXT,
    update_time TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

#### 3. 编写存储过程

```sql
CREATE OR REPLACE PROCEDURE batch_update_order_status(new_status TEXT)
LANGUAGE plpgsql
AS $$
DECLARE
    rec RECORD;
BEGIN
    FOR rec IN SELECT * FROM orders WHERE status = 'paid' LOOP
        UPDATE orders SET status = new_status, updated_at = NOW()
        WHERE order_id = rec.order_id;

        INSERT INTO order_update_log (old_status, new_status)
        VALUES ('paid', new_status);
    END LOOP;

    COMMIT;
EXCEPTION
    WHEN OTHERS THEN
        ROLLBACK;
        RAISE NOTICE '更新失败：% %', SQLERRM, SQLSTATE;
END;
$$;
```

#### 4. 调用存储过程

```sql
CALL batch_update_order_status('shipped');
```

---

## 📈 九、性能优化建议

| 技巧 | 描述 |
|------|------|
| **避免逐行处理** | 使用集合操作替代游标，提高效率 |
| **减少事务粒度** | 对于大批量操作，适当分批提交 |
| **启用自动提交** | 对只读操作可设置 `SET LOCAL autocommit = on` |
| **使用临时表缓存中间结果** | 减少重复计算 |
| **合理使用索引** | 确保查询条件字段有合适的索引支持 |

---

## 📌 小结

| 技术 | 功能 | 推荐使用场景 |
|------|------|--------------|
| 存储过程 | 支持事务控制 | 批量数据处理、关键业务逻辑 |
| PL/pgSQL | PostgreSQL 原生语言 | 通用业务封装 |
| 游标 | 逐条处理大数据 | 日志处理、ETL |
| 异常处理 | 捕获错误并恢复 | 金融交易、支付系统 |
| 事务控制 | 提交或回滚操作 | 数据一致性要求高场景 |

通过本节的学习，你应该已经掌握了如何使用 PL/pgSQL 编写 PostgreSQL 存储过程，包括变量、流程控制、游标、事务管理和异常处理等高级特性，并能够在实际项目中设计高效的数据库逻辑模块。