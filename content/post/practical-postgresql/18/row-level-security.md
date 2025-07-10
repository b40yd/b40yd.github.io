+++
title = "第十八章 多租户架构设计 - 第一节：行级安全策略（RLS）"
date = 2025-07-10
lastmod = 2025-07-10T01:00:00+08:00
tags = ["PostgreSQL", "practical", "book", "multi-tenancy", "rls", "security"]
categories = ["PostgreSQL", "practical", "book"]
draft = false
author = "b40yd"
+++

## 第十八章 多租户架构设计
### 第一节 行级安全策略（RLS）与 Row-Level Security

> **目标**：理解多租户架构中数据隔离的核心挑战，并掌握 PostgreSQL 提供的强大功能——行级安全策略（Row-Level Security, RLS），以在数据库层面实现精细、安全的数据访问控制。

**多租户（Multi-Tenancy）**是一种软件架构模式，它允许单个应用实例服务于多个不同的客户（租户）。例如，一个 SaaS 平台，每个注册的公司都是一个独立的租户。

在多租户架构中，最大的挑战之一就是**数据隔离**：如何确保租户 A 绝对无法访问到租户 B 的数据？

有多种实现数据隔离的策略（如独立数据库、独立 Schema），但最常用、最具扩展性的是**共享数据库、共享 Schema，通过一个 `tenant_id` 列来区分数据**的模式。然而，这种模式完全依赖于应用层代码来保证数据隔离。如果应用代码出现一个 bug（例如，在 `WHERE` 子句中忘记添加 `WHERE tenant_id = ...`），就可能导致灾难性的数据泄露。

**行级安全策略（Row-Level Security, RLS）** 将数据隔离的责任从应用层下沉到了**数据库层**，提供了一种更强大、更可靠的安全保障。

---

### RLS 的工作原理

RLS 允许你为一个表定义一个**安全策略（Policy）**。这个策略本质上就是一个会返回布尔值的 SQL 表达式。当一个用户尝试对该表进行 `SELECT`, `INSERT`, `UPDATE`, `DELETE` 操作时，PostgreSQL 会**自动地、强制地**将这个策略表达式作为 `WHERE` 子句（对于读操作）或 `WITH CHECK` 选项（对于写操作）应用到用户的查询上。

如果策略表达式返回 `true`，则允许访问该行；如果返回 `false` 或 `NULL`，则该行对用户**完全不可见**，就像它不存在一样。

---

### 实战：为 SaaS 应用的 `documents` 表启用 RLS

#### 场景

我们有一个 `documents` 表，存储了所有租户的文档。我们需要确保普通用户只能看到和操作自己租户的文档。

#### 第一步：准备表和数据

```sql
CREATE TABLE documents (
    id SERIAL PRIMARY KEY,
    tenant_id TEXT NOT NULL,
    owner_user_id TEXT NOT NULL,
    content TEXT
);

-- 插入属于不同租户和用户的数据
INSERT INTO documents (tenant_id, owner_user_id, content) VALUES
('acme', 'alice@acme.com', 'ACME Corp confidential report.'),
('acme', 'bob@acme.com', 'Bob''s personal notes.'),
('globex', 'charlie@globex.com', 'Globex Corp sales projection.');
```

#### 第二步：创建数据库用户

为了演示，我们创建两个代表不同应用用户的数据库角色。
```sql
CREATE ROLE app_user_alice LOGIN PASSWORD 'password';
CREATE ROLE app_user_charlie LOGIN PASSWORD 'password';

-- 授予他们对表的访问权限
GRANT SELECT, INSERT, UPDATE, DELETE ON documents TO app_user_alice, app_user_charlie;
```
此时，如果 `app_user_alice` 连接到数据库，她可以 `SELECT * FROM documents` 并看到**所有**租户的数据，这是不安全的。

#### 第三步：启用 RLS 并定义策略

**1. 在表上启用 RLS**
```sql
ALTER TABLE documents ENABLE ROW LEVEL SECURITY;
```
**重要**：一旦启用 RLS，默认情况下**所有**访问都会被拒绝（默认拒绝策略），直到你创建至少一个允许访问的策略。此时，即便是表的所有者也无法看到任何数据。

**2. 创建一个策略**
策略的核心是利用 PostgreSQL 的**会话配置参数**。我们可以在应用连接到数据库后，设置一个代表当前租户 ID 和用户 ID 的参数。

```sql
CREATE POLICY tenant_isolation_policy ON documents
-- 这个策略对所有操作都生效
FOR ALL
-- 适用于所有用户
TO PUBLIC
-- 核心的策略表达式
USING (tenant_id = current_setting('app.current_tenant_id'))
-- 对于写操作，强制新写入的行也必须满足此条件
WITH CHECK (tenant_id = current_setting('app.current_tenant_id'));
```
-   `USING (...)`: 定义了用于读操作（`SELECT`）的过滤条件。
-   `WITH CHECK (...)`: 定义了用于写操作（`INSERT`, `UPDATE`）的检查条件。

#### 第四步：验证 RLS 效果

现在，让我们模拟应用用户的操作。

**模拟 Alice (租户 'acme') 的会话：**
```sql
-- 以 Alice 的身份连接
-- psql -U app_user_alice -d your_db

-- 设置当前会话的租户 ID
SET app.current_tenant_id = 'acme';

-- 执行查询
SELECT * FROM documents;
```
**结果：**
只会返回 'acme' 租户的两条文档，Globex 的文档对她完全不可见。
```
 id | tenant_id |  owner_user_id   |             content
----+-----------+------------------+----------------------------------
  1 | acme      | alice@acme.com   | ACME Corp confidential report.
  2 | acme      | bob@acme.com     | Bob's personal notes.
```

**尝试插入非法数据：**
```sql
-- Alice 尝试为其他租户创建文档
INSERT INTO documents (tenant_id, owner_user_id, content)
VALUES ('other_corp', 'alice@acme.com', 'some data');
```
**结果：**
查询会失败，并报错 `new row violates WITH CHECK OPTION for "documents"`，因为 `WITH CHECK` 条件阻止了这个操作。

---

### 更精细的控制

RLS 策略可以非常灵活。例如，我们可以增加一个策略，允许用户只能修改**自己拥有**的文档。

```sql
-- 创建一个只针对 UPDATE 的、更严格的策略
CREATE POLICY user_own_documents_policy ON documents
FOR UPDATE
USING (owner_user_id = current_setting('app.current_user_id'));
```
*（假设我们也设置了 `app.current_user_id` 参数）*

当多个策略存在时，它们会用 `OR` 逻辑组合起来。

---

## 📌 小结

-   **RLS 是实现数据库层多租户数据隔离的黄金标准**。它提供了一种强大而可靠的安全机制，可以防止因应用层代码漏洞导致的数据泄露。
-   **核心原理**：通过 `CREATE POLICY` 定义访问规则，PostgreSQL 会自动将这些规则强制应用到所有查询上。
-   **实现方式**：通常结合**会话配置参数**（如 `current_setting('app.current_tenant_id')`）来动态地、安全地将当前用户的身份信息传递给策略表达式。
-   **默认拒绝**：启用 RLS 后，必须显式创建 `ALLOW` 策略，否则所有访问都会被拒绝。

将安全策略下沉到离数据最近的地方，是构建健壮、可信的 SaaS 应用的关键一步。RLS 让 PostgreSQL 在多租户架构设计中，拥有了与许多专业数据库相媲美的安全能力。
