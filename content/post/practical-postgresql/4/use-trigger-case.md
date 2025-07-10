+++
title = "PostgreSQL 数据库实战指南 - 触发器的定义与使用场景"
date = 2025-07-04
lastmod = 2025-07-04T01:00:00+08:00
tags = ["PostgreSQL", "practical", "book", "function", "procedure", "trigger"]
categories = ["PostgreSQL", "practical", "book", "function", "procedure", "trigger"]
draft = false
author = "b40yd"
+++

# 第四章 函数、触发器与过程语言  
## 第二节 触发器的定义与使用场景

> **目标**：掌握 PostgreSQL 中触发器的高级使用技巧，包括行级与语句级触发器的区别、条件性触发、性能优化策略以及如何避免常见的性能瓶颈和死锁问题，并通过实战案例展示其在复杂业务逻辑中的应用价值。

触发器是 PostgreSQL 中实现数据变更自动响应的重要机制。它可以在表发生 `INSERT`、`UPDATE` 或 `DELETE` 操作时自动执行一段逻辑代码（通常是 PL/pgSQL 编写的函数），从而实现日志记录、数据校验、审计跟踪、状态同步等功能。

本节将围绕以下核心内容展开：

- 行级触发器与语句级触发器的区别
- 条件性触发器设计
- 触发器性能优化策略
- 死锁与递归触发问题
- 实战：构建订单状态变更审计系统

---

## 🧠 一、行级触发器 vs 语句级触发器

### 1. 行级触发器（FOR EACH ROW）

- 每一行受影响都会触发一次函数
- 适用于需要对每条记录进行处理的场景
- 可访问 `NEW` 和 `OLD` 伪记录

#### 示例：

```sql
CREATE TRIGGER trg_order_update_row
AFTER UPDATE ON orders
FOR EACH ROW
EXECUTE FUNCTION log_order_change();
```

---

### 2. 语句级触发器（FOR EACH STATEMENT）

- 整个 SQL 语句只触发一次
- 不区分具体修改的记录数
- 不可访问 `NEW` / `OLD`，仅能获取上下文信息

#### 示例：

```sql
CREATE TRIGGER trg_order_update_statement
AFTER UPDATE ON orders
FOR EACH STATEMENT
EXECUTE FUNCTION update_order_counter();
```

---

## 📌 二、条件性触发器设计

并不是所有的 DML 操作都需要触发逻辑。我们可以通过添加条件判断来控制触发器是否执行。

### 示例：仅当订单状态发生变化时才记录日志

```sql
CREATE OR REPLACE FUNCTION log_status_change()
RETURNS TRIGGER AS $$
BEGIN
    IF OLD.status IS DISTINCT FROM NEW.status THEN
        INSERT INTO order_log (order_id, old_status, new_status)
        VALUES (NEW.order_id, OLD.status, NEW.status);
    END IF;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_order_status_change
AFTER UPDATE ON orders
FOR EACH ROW
WHEN (OLD.status IS DISTINCT FROM NEW.status)
EXECUTE FUNCTION log_status_change();
```

📌 使用 `WHEN (...)` 子句可以减少不必要的触发次数，提升性能。

---

## ⚙️ 三、触发器性能优化策略

虽然触发器非常强大，但如果使用不当，可能会引入性能瓶颈甚至死锁。以下是几个优化建议：

### 1. 尽量避免在触发器中执行耗时操作

如：

- 复杂计算
- 远程调用（HTTP、外部服务）
- 大量写入或更新其他表

👉 建议：将这些操作异步化，例如写入消息队列或日志表，由后台任务异步处理。

---

### 2. 避免触发器之间的递归调用

如果一个触发器修改了另一个会触发它的表，则可能形成递归调用链。

#### 示例：

```sql
-- A表触发器修改 B表 → B表触发器又修改 A表 → 形成循环
```

👉 解决方案：

- 明确设计触发顺序，避免交叉依赖
- 使用 `pg_trigger_depth()` 函数检测嵌套深度，防止无限循环

```sql
IF pg_trigger_depth() > 1 THEN
    RETURN NULL; -- 跳过后续逻辑
END IF;
```

---

### 3. 合理选择触发时机（BEFORE / AFTER）

| 类型 | 描述 | 推荐使用场景 |
|------|------|--------------|
| BEFORE | 在操作前执行，可用于修改 NEW 值 | 字段默认值设置、字段转换、验证逻辑 |
| AFTER | 在操作后执行，不可修改 NEW 值 | 日志记录、通知、级联操作 |

---

### 4. 避免锁竞争与死锁

触发器中执行的函数通常也会参与事务，因此也可能导致锁等待或死锁。

#### 优化建议：

- 短小精悍的触发逻辑
- 避免在触发器中执行长时间事务
- 使用 `NOWAIT` 避免阻塞

---

## 💀 四、触发器与死锁问题分析

### 1. 死锁成因

触发器函数内部执行了涉及多个表的操作，而不同事务以不同的顺序修改这些表，就可能发生死锁。

#### 示例：

```sql
-- 用户 A 更新 orders → 触发器修改 inventory
-- 用户 B 更新 inventory → 触发器修改 orders
```

此时两个事务相互等待对方释放锁，造成死锁。

### 2. 如何排查？

查看 PostgreSQL 日志：

```
ERROR:  deadlock detected
DETAIL:  Process 1234 waits for ShareLock on transaction 5678; blocked by process 5678.
```

也可以查询 `pg_locks` 表：

```sql
SELECT * FROM pg_locks WHERE NOT granted;
```

---

## 🧪 五、实战演练：构建订单状态变更审计系统

### 场景描述：

每当订单状态发生变化时，系统应自动记录变更前后的内容，并支持按时间维度进行审计。

### 步骤如下：

#### 1. 创建订单表（orders）

```sql
CREATE TABLE orders (
    order_id SERIAL PRIMARY KEY,
    user_id INT,
    status TEXT DEFAULT 'pending',
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

#### 2. 创建审计日志表（order_audit）

```sql
CREATE TABLE order_audit (
    audit_id SERIAL PRIMARY KEY,
    order_id INT,
    old_status TEXT,
    new_status TEXT,
    change_time TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

#### 3. 创建触发函数

```sql
CREATE OR REPLACE FUNCTION log_order_status_change()
RETURNS TRIGGER AS $$
BEGIN
    IF TG_OP = 'UPDATE' AND OLD.status IS DISTINCT FROM NEW.status THEN
        INSERT INTO order_audit (order_id, old_status, new_status)
        VALUES (NEW.order_id, OLD.status, NEW.status);
    END IF;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;
```

#### 4. 创建触发器

```sql
CREATE TRIGGER trg_order_status_change
AFTER UPDATE OF status ON orders
FOR EACH ROW
WHEN (OLD.status IS DISTINCT FROM NEW.status)
EXECUTE FUNCTION log_order_status_change();
```

#### 5. 测试触发器行为

```sql
UPDATE orders SET status = 'shipped' WHERE order_id = 1;
SELECT * FROM order_audit;
```

你应该能看到一条新的审计记录。

---

## 📈 六、扩展建议：触发器 + 异步处理架构

为了进一步解耦数据库与业务逻辑，你可以结合以下技术构建更健壮的触发机制：

| 技术 | 描述 |
|------|------|
| **LISTEN / NOTIFY** | 触发器内发送通知，外部监听器消费事件 |
| **pgmq / RabbitMQ / Kafka** | 将触发器产生的事件推送到消息队列异步处理 |
| **Materialized Views / Triggers + Redis** | 实现实时缓存更新 |
| **Temporal Tables / System Versioning** | 利用触发器实现历史版本记录 |

---

## 📌 小结

| 技术 | 功能 | 推荐使用场景 |
|------|------|--------------|
| 行级触发器 | 每条记录变更都触发 | 审计、日志记录 |
| 语句级触发器 | 整体操作触发 | 批量统计、计数 |
| 条件触发器 | 控制触发时机 | 仅特定字段变化时触发 |
| BEFORE/AFTER | 控制触发时机 | 验证、修改数据、日志记录 |
| 死锁预防 | 避免资源竞争 | 高并发系统 |
| 异步触发设计 | 解耦逻辑 | 消息队列集成、日志异步处理 |

通过本节的学习，你应该已经掌握了 PostgreSQL 中触发器的高级使用方法，理解了行级与语句级触发器的区别、条件性触发的设计方式、常见性能瓶颈及优化策略，并能够在实际项目中合理应用触发器完成自动化数据处理和审计功能。