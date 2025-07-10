+++
title = "第二十章 安全机制与合规性 - 第四节 实战：金融系统中的访问控制与审计"
date = 2025-07-10
lastmod = 2025-07-10T01:00:00+08:00
tags = ["PostgreSQL", "practical", "book", "security", "rls", "acl", "audit", "financial"]
categories = ["PostgreSQL", "practical", "book"]
draft = false
author = "b40yd"
+++

## 第四节 实战：金融系统中的访问控制与审计

> **目标**：为一个模拟的金融交易系统，设计并实现一个多层次、纵深防御的安全模型，综合运用基于角色的访问控制（RBAC）、列级权限、行级安全（RLS）和 `pg_audit` 审计。

### 场景描述

我们正在为一个银行设计一个客户交易记录系统。该系统需要满足极高的安全和合规要求。

**核心实体与角色：**
-   **`accounts` 表**：存储客户账户信息，包括 `account_id`, `customer_id`, `branch_id`, `balance`。
-   **`transactions` 表**：存储交易流水，包括 `transaction_id`, `account_id`, `amount`, `transaction_time`。
-   **角色**：
    -   **`customer_service_rep` (客服代表)**：可以查看客户的基本信息，但不能看到其余额。只能看到自己所在分行的客户。
    -   **`branch_manager` (分行经理)**：可以查看自己分行所有客户的完整信息，包括余额。
    -   **`auditor` (审计员)**：可以查看所有数据，但所有操作都必须被严格记录。

---

## 🏛️ 第一步：表结构与角色定义

```sql
CREATE TABLE accounts (
    account_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    customer_id BIGINT NOT NULL,
    branch_id TEXT NOT NULL, -- e.g., 'branch-nyc', 'branch-london'
    balance NUMERIC(15, 2) NOT NULL
);

-- (transactions 表结构略)

-- 创建角色
CREATE ROLE customer_service_rep;
CREATE ROLE branch_manager;
CREATE ROLE auditor;
```

---

## 🔐 第二步：配置列级权限 (ACLs)

我们首先定义每个角色能“看到”哪些列。

```sql
-- 默认拒绝所有公开访问
REVOKE ALL ON accounts FROM PUBLIC;

-- 客服代表只能看非敏感列
GRANT SELECT (account_id, customer_id, branch_id) ON accounts TO customer_service_rep;

-- 分行经理可以看所有列
GRANT SELECT ON accounts TO branch_manager;

-- 审计员可以看所有列
GRANT SELECT ON accounts TO auditor;
```

---

## 🛡️ 第三步：配置行级安全 (RLS)

接下来，我们定义每个角色能“看到”哪些行。

```sql
-- 在 accounts 表上启用 RLS
ALTER TABLE accounts ENABLE ROW LEVEL SECURITY;

-- 创建策略，限制客服和经理只能访问自己分行的数据
-- 假设应用连接后会执行: SET app.current_branch_id = 'branch-nyc';
CREATE POLICY branch_isolation_policy ON accounts
    FOR SELECT
    TO customer_service_rep, branch_manager
    USING (branch_id = current_setting('app.current_branch_id'));

-- 审计员需要看到所有数据，所以我们为他们创建一个“全通”策略
-- 注意：如果没有这个策略，审计员也会被默认的拒绝策略挡住
CREATE POLICY auditor_full_access_policy ON accounts
    FOR SELECT
    TO auditor
    USING (true); -- 表达式为 true，意味着允许访问所有行
```

---

## ✍️ 第四步：配置审计 (`pg_audit`)

对于金融系统，所有的数据访问都必须被记录。

**1. 在 `postgresql.conf` 中配置 `pg_audit`**
```ini
pgaudit.log = 'read, write, ddl, role'
pgaudit.log_relation = on
# ... 其他 pgaudit 配置
```
*(需要重启 PostgreSQL)*

**2. 创建扩展**
```sql
CREATE EXTENSION IF NOT EXISTS pgaudit;
```
现在，任何对 `accounts` 表的查询，无论是来自客服、经理还是审计员，都会被 `pg_audit` 详细地记录到 PostgreSQL 的日志文件中。

---

## ✅ 第五步：验证多层次安全模型

让我们模拟不同角色的登录和查询。

#### 模拟 1：纽约分行的客服代表

```sql
-- 模拟应用为该用户设置会话参数
SET app.current_branch_id = 'branch-nyc';
SET ROLE customer_service_rep;

-- 执行查询
SELECT * FROM accounts;
-- 错误！因为他没有对 balance 列的 SELECT 权限。
-- ERROR: permission denied for table accounts

-- 执行合法的查询
SELECT account_id, customer_id, branch_id FROM accounts;
-- 成功！结果集中只包含 branch_id = 'branch-nyc' 的行 (RLS 生效)。
-- balance 列的数据在数据库层面就被屏蔽了 (列级权限生效)。
```

#### 模拟 2：伦敦分行的分行经理

```sql
SET app.current_branch_id = 'branch-london';
SET ROLE branch_manager;

-- 执行查询
SELECT account_id, customer_id, branch_id, balance FROM accounts;
-- 成功！结果集中只包含 branch_id = 'branch-london' 的行 (RLS 生效)。
-- 他可以看到 balance 列 (列级权限允许)。
```

#### 模拟 3：审计员

```sql
RESET app.current_branch_id; -- 审计员不受分行限制
SET ROLE auditor;

-- 执行查询
SELECT * FROM accounts;
-- 成功！返回所有分行的所有账户信息，包括余额 (RLS 策略允许)。
```

**同时，在 PostgreSQL 的日志文件中：**
上述所有 `SELECT` 操作，无论成功还是失败，都会被 `pg_audit` 记录下来，形成一条不可篡改的审计轨迹。

---

## 📌 小结

本实战构建了一个符合金融级别安全要求的纵深防御体系：
1.  **第一层防御 (ACLs)**：首先通过**列级权限**，限制了不同角色能接触到的数据字段的“宽度”，保护了核心敏感信息（如 `balance`）。
2.  **第二层防御 (RLS)**：然后通过**行级安全策略**，限制了用户能访问的数据行的“深度”，实现了严格的业务隔离（如按分行隔离）。
3.  **第三层防御 (Auditing)**：最后通过 **`pg_audit`**，为所有操作提供了完整的、可追溯的审计日志，满足了合规性要求。

这套组合拳展示了 PostgreSQL 在数据安全方面的强大能力。通过在数据库层面强制执行这些复杂的安全规则，我们可以极大地降低应用层代码漏洞带来的风险，构建出真正安全、可靠的关键业务系统。
