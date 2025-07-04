+++
title = "PostgreSQL 数据库实战指南 - 锁机制与死锁检测"
date = 2025-07-04
lastmod = 2025-07-04T01:00:00+08:00
tags = ["PostgreSQL", "practical", "book", "locked"]
categories = ["PostgreSQL", "practical", "book", "locked"]
draft = false
author = "b40yd"
+++

# 第三章 事务控制与并发机制  
## 第三节 锁机制与死锁检测

> **目标**：掌握 PostgreSQL 中的锁机制类型、加锁规则以及如何通过系统视图和日志分析锁等待和死锁问题，能够在高并发场景下合理设计事务逻辑以避免锁冲突和死锁的发生。

在数据库系统中，**锁（Locking）** 是实现数据一致性和并发控制的重要手段。虽然 PostgreSQL 使用 **MVCC（多版本并发控制）** 来减少读写之间的阻塞，但在某些操作（如更新、删除、DDL）中仍需要使用锁来保证一致性。

本节将深入讲解：

- PostgreSQL 支持的锁类型
- 行级锁与表级锁的区别与应用场景
- 死锁的成因与检测机制
- 如何查看当前锁信息
- 实战：模拟并解决死锁问题

---

## 🔒 一、PostgreSQL 中的锁机制概述

### 1. 锁的作用

锁用于防止多个事务同时修改相同的数据，从而确保数据的一致性。例如：

```sql
BEGIN;
SELECT * FROM accounts WHERE id = 1 FOR UPDATE; -- 显式加锁
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
COMMIT;
```

上面的例子中，`FOR UPDATE` 会显式加锁，阻止其他事务修改该行。

### 2. 锁的分类

| 类型 | 描述 |
|------|------|
| **表级锁（Table-level Locks）** | 控制对整个表的访问，适用于 DDL 和大批量操作 |
| **行级锁（Row-level Locks）** | 控制对单个行的访问，适用于 OLTP 场景 |
| **页级锁（Page-level Locks）** | 控制对页面级别的访问，较少直接使用 |

---

## 📦 二、常见的锁模式及其兼容性

PostgreSQL 提供了多种锁模式，不同模式之间有不同的兼容性规则。

### 常见锁模式列表：

| 锁模式 | 描述 | 兼容性说明 |
|--------|------|-------------|
| `ACCESS SHARE` | 最低级别锁，用于 SELECT 查询 | 可与其他所有锁共存 |
| `ROW SHARE` | 用于 SELECT FOR SHARE | 不兼容 EXCLUSIVE 和 ACCESS EXCLUSIVE |
| `ROW EXCLUSIVE` | 用于 INSERT、UPDATE、DELETE | 不兼容 SHARE、SHARE ROW EXCLUSIVE、EXCLUSIVE、ACCESS EXCLUSIVE |
| `SHARE UPDATE EXCLUSIVE` | 用于 VACUUM（非 FULL）、CREATE INDEX CONCURRENTLY 等 | 不兼容 SHARE UPDATE EXCLUSIVE、SHARE ROW EXCLUSIVE、EXCLUSIVE、ACCESS EXCLUSIVE |
| `SHARE` | 用于 CREATE INDEX（不带 CONCURRENTLY） | 不兼容 ROW EXCLUSIVE、SHARE ROW EXCLUSIVE、EXCLUSIVE、ACCESS EXCLUSIVE |
| `SHARE ROW EXCLUSIVE` | 更严格的共享锁 | 不兼容大多数锁 |
| `EXCLUSIVE` | 最严格的锁之一 | 仅允许读操作 |
| `ACCESS EXCLUSIVE` | 最高级别锁，如 DROP TABLE | 所有其他锁都不可共存 |

📌 **兼容性矩阵**：

| 当前持有锁 \ 请求锁 | ACCESS SHARE | ROW SHARE | ROW EXCLUSIVE | SHARE UPDATE EXCLUSIVE | SHARE | SHARE ROW EXCLUSIVE | EXCLUSIVE | ACCESS EXCLUSIVE |
|---------------------|--------------|-----------|---------------|------------------------|-------|----------------------|-----------|--------------------|
| ACCESS SHARE        | ✅           | ✅        | ✅            | ✅                     | ✅    | ✅                   | ✅        | ❌                 |
| ROW SHARE           | ✅           | ✅        | ✅            | ✅                     | ✅    | ✅                   | ✅        | ❌                 |
| ROW EXCLUSIVE       | ✅           | ✅        | ✅            | ✅                     | ❌    | ❌                   | ❌        | ❌                 |
| SHARE UPDATE EXCLUSIVE | ✅         | ✅        | ✅            | ✅                     | ❌    | ❌                   | ❌        | ❌                 |
| SHARE               | ✅           | ✅        | ❌            | ❌                     | ✅    | ❌                   | ❌        | ❌                 |
| SHARE ROW EXCLUSIVE | ✅           | ✅        | ❌            | ❌                     | ❌    | ✅                   | ❌        | ❌                 |
| EXCLUSIVE           | ✅           | ✅        | ❌            | ❌                     | ❌    | ❌                   | ❌        | ❌                 |
| ACCESS EXCLUSIVE    | ❌           | ❌        | ❌            | ❌                     | ❌    | ❌                   | ❌        | ❌                 |

---

## 🧠 三、行级锁与表级锁详解

### 1. 行级锁（Row-level Locks）

#### （1）FOR UPDATE / FOR NO KEY UPDATE / FOR SHARE / FOR KEY SHARE

这些子句用于在 SELECT 查询时显式锁定行，防止其他事务修改。

##### 示例：

```sql
-- 用户 A 获取行锁
BEGIN;
SELECT * FROM accounts WHERE id = 1 FOR UPDATE;

-- 用户 B 尝试更新同一行
BEGIN;
UPDATE accounts SET balance = 900 WHERE id = 1; -- 阻塞
```

#### （2）行级锁类型对比

| 锁类型 | 是否允许其他事务更新？ | 是否允许其他事务删除？ | 是否允许其他事务加锁？ |
|--------|-------------------------|-------------------------|--------------------------|
| FOR UPDATE              | ❌                      | ❌                      | ❌                       |
| FOR NO KEY UPDATE       | ❌                      | ❌                      | ✅                       |
| FOR SHARE               | ✅                      | ✅                      | ❌                       |
| FOR KEY SHARE           | ✅                      | ✅                      | ✅                       |

---

### 2. 表级锁（Table-level Locks）

通常由 DDL 操作自动获取，也可手动使用 `LOCK TABLE` 命令。

#### 示例：

```sql
LOCK TABLE orders IN EXCLUSIVE MODE;
```

常见用法包括：

- 大批量数据导入导出
- 创建索引或维护操作
- 避免结构变更影响正在进行的操作

---

## 💀 四、死锁（Deadlock）原理与检测机制

### 1. 死锁的定义

当两个或多个事务相互等待对方持有的锁释放时，就会发生死锁。

#### 示例：

```sql
-- 用户 A
BEGIN;
UPDATE accounts SET balance = 900 WHERE id = 1;
UPDATE customers SET name = 'Alice' WHERE id = 100;

-- 用户 B
BEGIN;
UPDATE customers SET name = 'Bob' WHERE id = 100;
UPDATE accounts SET balance = 800 WHERE id = 1; -- 死锁发生
```

此时两个事务都在等待对方释放锁，形成循环依赖。

### 2. PostgreSQL 如何检测死锁？

PostgreSQL 内部有一个 **死锁检测器（Deadlock Detector）**，它会周期性地检查锁等待图是否存在环路。如果发现死锁，会选择一个事务进行回滚（通常是代价最小的事务）。

#### 日志输出示例：

```
ERROR:  deadlock detected
DETAIL:  Process 1234 waits for ShareLock on transaction 5678; blocked by process 5678.
HINT:  See server log for query details.
CONTEXT:  while updating tuple (0,1) in relation "accounts"
STATEMENT:  UPDATE accounts SET balance = 800 WHERE id = 1;
```

---

## 🔍 五、查询当前锁信息

你可以通过以下方式查看当前数据库中的锁状态：

### 1. 查询当前锁等待情况

```sql
SELECT
    blocked_locks.pid AS blocked_pid,
    blocked_activity.usename AS blocked_user,
    blocking_locks.pid AS blocking_pid,
    blocking_activity.usename AS blocking_user,
    blocked_activity.query AS blocked_query,
    blocking_activity.query AS blocking_query
FROM pg_catalog.pg_locks blocked_locks
JOIN pg_catalog.pg_stat_all_tables blocked_relations ON (
    blocked_locks.relation = blocked_relations.relid
)
JOIN pg_catalog.pg_stat_statements blocked_activity ON (
    blocked_locks.pid = blocked_activity.pid
)
JOIN pg_catalog.pg_locks blocking_locks ON (
    blocking_locks.relation = blocked_locks.relation AND
    blocking_locks.mode = blocked_locks.mode AND
    blocking_locks.granted
)
JOIN pg_catalog.pg_stat_statements blocking_activity ON (
    blocking_locks.pid = blocking_activity.pid
)
WHERE NOT blocked_locks.granted;
```

### 2. 查看所有当前持有的锁

```sql
SELECT * FROM pg_locks;
```

常用字段：

- `pid`: 客户端进程 ID
- `mode`: 锁模式
- `granted`: 是否已获得锁
- `relation`: 表 OID
- `transactionid`: 事务 ID

---

## 🧪 六、实战演练：模拟并解决死锁问题

### 场景描述：

用户 A 和用户 B 同时尝试更新两个不同的记录，但顺序相反，导致死锁。

#### 步骤 1：创建测试表

```sql
CREATE TABLE test_lock (
    id INT PRIMARY KEY,
    value TEXT
);

INSERT INTO test_lock VALUES (1, 'A'), (2, 'B');
```

#### 步骤 2：用户 A 开始事务

```sql
BEGIN;
UPDATE test_lock SET value = 'X' WHERE id = 1;
```

#### 步骤 3：用户 B 开始事务

```sql
BEGIN;
UPDATE test_lock SET value = 'Y' WHERE id = 2;
```

#### 步骤 4：用户 A 更新第二条记录

```sql
UPDATE test_lock SET value = 'Z' WHERE id = 2; -- 正常执行
```

#### 步骤 5：用户 B 更新第一条记录

```sql
UPDATE test_lock SET value = 'W' WHERE id = 1; -- 死锁触发
```

PostgreSQL 自动检测到死锁，并回滚其中一个事务。

---

## 🛠️ 七、避免死锁的最佳实践

| 实践 | 描述 |
|------|------|
| **统一访问顺序** | 所有事务按固定顺序访问资源 |
| **减少事务粒度** | 缩短事务时间，尽早提交 |
| **使用 NOWAIT 或 SKIP LOCKED** | 在无法获取锁时立即报错或跳过 |
| **监控锁等待** | 使用 pg_locks、pg_stat_statements 监控锁等待情况 |
| **设置合理的超时** | 使用 `statement_timeout` 和 `lock_timeout` 避免长时间阻塞 |

---

## 📌 小结

| 技术 | 功能 | 推荐使用场景 |
|------|------|--------------|
| `FOR UPDATE` | 显式行锁 | 高并发写入场景 |
| `pg_locks` | 查看锁信息 | 故障排查、性能调优 |
| 死锁检测 | 自动回滚 | 避免系统挂起 |
| 锁模式兼容性 | 控制并发行为 | DDL、批量操作等 |
| 锁等待超时 | 防止长时间阻塞 | 生产环境推荐配置 |

通过本节的学习，你应该已经掌握了 PostgreSQL 中的锁机制类型、锁兼容性规则、死锁的成因与检测机制，并能够通过系统视图和日志分析锁等待问题，在实际项目中合理应用锁策略以提升系统稳定性与并发能力。