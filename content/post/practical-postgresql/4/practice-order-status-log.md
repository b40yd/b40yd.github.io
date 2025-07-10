+++
title = "第四章 函数、触发器与过程语言 - 第四节 实战：订单状态变更自动记录日志"
date = 2025-07-10
lastmod = 2025-07-10T01:00:00+08:00
tags = ["PostgreSQL", "practical", "book", "trigger", "audit", "log"]
categories = ["PostgreSQL", "practical", "book"]
draft = false
author = "b40yd"
+++

## 第四节 实战：订单状态变更自动记录日志

> **目标**：设计并实现一个健壮的触发器，用于自动、详细地记录 `orders` 表中状态字段（`status`）的每一次变更，为业务审计和问题追溯提供可靠依据。

在许多业务系统（如电商、物流、工单处理）中，核心对象的状态流转是关键业务逻辑。记录每一次状态变更不仅是合规性要求，也是排查问题、分析业务流程的重要手段。本节将通过一个完整的实战案例，展示如何利用触发器实现这一功能。

---

### 场景分析

我们需要一个 `order_status_audit` 表，专门用来存放 `orders` 表的状态变更历史。当一个订单的状态从 `pending` 变为 `paid`，再到 `shipped` 时，每一条变更都应该被记录下来。

审计日志应包含以下关键信息：
- 关联的订单ID (`order_id`)
- 旧状态 (`old_status`)
- 新状态 (`new_status`)
- 变更发生的时间 (`changed_at`)
- 执行变更的操作员 (`changed_by`)

---

## 🛠️ 第一步：创建数据表结构

首先，我们需要 `orders` 表和用于审计的 `order_status_audit` 表。

```sql
-- 订单表
CREATE TABLE orders (
    order_id SERIAL PRIMARY KEY,
    customer_name TEXT NOT NULL,
    total_amount NUMERIC(10, 2) NOT NULL,
    status TEXT NOT NULL DEFAULT 'pending', -- 订单状态
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

-- 订单状态审计日志表
CREATE TABLE order_status_audit (
    audit_id SERIAL PRIMARY KEY,
    order_id INT NOT NULL,
    old_status TEXT,
    new_status TEXT NOT NULL,
    changed_by NAME DEFAULT CURRENT_USER, -- 记录执行操作的数据库用户
    changed_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    CONSTRAINT fk_order
        FOREIGN KEY(order_id) 
        REFERENCES orders(order_id)
        ON DELETE CASCADE -- 如果订单被删除，关联的日志也一并删除
);

-- 为审计表的 order_id 创建索引，加速查询
CREATE INDEX idx_order_status_audit_order_id ON order_status_audit(order_id);
```

---

## ✍️ 第二步：编写触发器函数

触发器函数是核心逻辑所在。它需要在 `orders` 表发生 `INSERT` 或 `UPDATE` 时被调用，并判断 `status` 字段是否发生变化。

```sql
CREATE OR REPLACE FUNCTION log_order_status_change()
RETURNS TRIGGER
LANGUAGE plpgsql
AS $$
BEGIN
    -- 场景1: 新插入订单 (TG_OP = 'INSERT')
    -- 直接记录初始状态
    IF (TG_OP = 'INSERT') THEN
        INSERT INTO order_status_audit (order_id, old_status, new_status)
        VALUES (NEW.order_id, NULL, NEW.status);

    -- 场景2: 更新订单 (TG_OP = 'UPDATE')
    -- 仅在 status 字段实际发生变化时记录
    ELSIF (TG_OP = 'UPDATE' AND NEW.status IS DISTINCT FROM OLD.status) THEN
        INSERT INTO order_status_audit (order_id, old_status, new_status)
        VALUES (NEW.order_id, OLD.status, NEW.status);
    END IF;

    -- 更新主表的 updated_at 时间戳
    NEW.updated_at := NOW();

    -- 返回 NEW 以便 DML 操作继续执行
    RETURN NEW;
END;
$$;
```
**代码解析**：
- `TG_OP` 是一个特殊变量，值为 `INSERT`, `UPDATE`, `DELETE` 等，表示触发事件的类型。
- `NEW.status IS DISTINCT FROM OLD.status` 是一种严谨的比较方式，可以正确处理 `NULL` 值。
- 我们顺便在触发器中更新了 `updated_at` 字段，这是触发器的常见用法之一。

---

## 🔗 第三步：创建并绑定触发器

现在，我们将这个函数绑定到 `orders` 表。我们希望在每次插入或更新行之前（`BEFORE`）执行此逻辑。

```sql
CREATE TRIGGER trigger_orders_status_change
BEFORE INSERT OR UPDATE ON orders
FOR EACH ROW
EXECUTE FUNCTION log_order_status_change();
```
**为什么用 `BEFORE`？**
- 可以在数据写入磁盘前，统一修改 `NEW` 记录，如更新 `updated_at` 字段。
- 如果触发器逻辑中发现错误，可以通过 `RAISE EXCEPTION` 来阻止不合法的 `INSERT` 或 `UPDATE` 操作。

---

## ✅ 第四步：验证触发器功能

让我们模拟一次完整的订单生命周期，来验证触发器是否按预期工作。

```sql
-- 1. 创建一个新订单
INSERT INTO orders (customer_name, total_amount, status)
VALUES ('张三', 199.99, 'pending');
-- 此时应在 audit 表插入一条 (NULL -> 'pending') 的记录

-- 2. 模拟支付，更新状态
UPDATE orders SET status = 'paid' WHERE customer_name = '张三';
-- 此时应在 audit 表插入一条 ('pending' -> 'paid') 的记录

-- 3. 更新订单的其他信息，但不改变状态
UPDATE orders SET total_amount = 205.99 WHERE customer_name = '张三';
-- 此时 audit 表不应有新记录，但 orders.updated_at 应被更新

-- 4. 模拟发货，更新状态
UPDATE orders SET status = 'shipped' WHERE customer_name = '张三';
-- 此时应在 audit 表插入一条 ('paid' -> 'shipped') 的记录
```

**查询验证结果：**

```sql
-- 查看订单最终状态
SELECT order_id, status, updated_at FROM orders WHERE customer_name = '张三';

-- 查看完整的状态变更历史
SELECT order_id, old_status, new_status, changed_at
FROM order_status_audit
WHERE order_id = (SELECT order_id FROM orders WHERE customer_name = '张三')
ORDER BY changed_at;
```

**预期输出：**
`order_status_audit` 表中应有三条按时间排序的记录，清晰地展示了订单从创建到支付再到发货的完整状态流转路径。

---

## 📌 小结

通过本实战，我们成功构建了一个自动化的订单状态审计系统。这个模式不仅限于订单，可以轻松扩展到任何需要追踪字段变更历史的场景，例如：
- 记录用户权限的变更。
- 审计产品价格的调整。
- 追踪敏感配置的修改。

触发器是实现此类自动化、强制性数据策略的强大工具，但务必注意保持其逻辑的简洁和高效，以避免对数据库性能造成不必要的影响。
