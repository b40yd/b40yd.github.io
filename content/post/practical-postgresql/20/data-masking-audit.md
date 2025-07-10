+++
title = "第二十章 安全机制与合规性 - 第三节：数据脱敏与审计日志"
date = 2025-07-10
lastmod = 2025-07-10T01:00:00+08:00
tags = ["PostgreSQL", "practical", "book", "security", "masking", "audit", "pg_audit"]
categories = ["PostgreSQL", "practical", "book"]
draft = false
author = "b40yd"
+++

## 第三节 数据脱敏与审计日志

> **目标**：学习两种高级的安全与合规技术：1) 数据脱敏（Data Masking），用于在不改变原始数据的情况下向特定用户展示脱敏后的数据；2) 数据库审计（Auditing），用于记录谁在何时对哪些数据执行了什么操作。

除了访问控制，一个完整的安全策略还需要考虑如何处理“合法用户”对敏感数据的访问，以及如何记录所有重要操作以备后续审计。

---

### 一、数据脱敏 (Data Masking)

**数据脱敏**是一种在不影响数据可用性的前提下，对敏感数据进行模糊化处理，以保护隐私的技术。例如，向客服人员展示用户的手机号时，只显示 `138****1234`。

虽然 PostgreSQL 16 之前的版本没有内置的数据脱敏功能，但我们可以通过**安全视图（Security Barrier Views）**和**函数**来巧妙地实现它。从 PostgreSQL 16 开始，引入了更强大的 `pg_mask` 扩展，但视图方法更具通用性。

#### 使用安全视图实现数据脱敏

**场景**：我们有一个 `customers` 表，包含敏感的 `email` 和 `phone` 信息。我们希望创建一个视图 `customers_for_support`，供客服人员使用，该视图会自动对敏感信息进行脱敏。

**第一步：创建脱敏函数**
```sql
CREATE OR REPLACE FUNCTION mask_email(email TEXT)
RETURNS TEXT AS $$
BEGIN
    RETURN regexp_replace(email, '(?<=.).*?(?=@)', '****');
END;
$$ LANGUAGE plpgsql IMMUTABLE;

CREATE OR REPLACE FUNCTION mask_phone(phone TEXT)
RETURNS TEXT AS $$
BEGIN
    RETURN regexp_replace(phone, '(\d{3})\d{4}(\d{4})', '\1****\2');
END;
$$ LANGUAGE plpgsql IMMUTABLE;
```

**第二步：创建安全视图**
```sql
CREATE VIEW customers_for_support AS
SELECT
    customer_id,
    name,
    mask_email(email) AS email,
    mask_phone(phone) AS phone
FROM customers;
```

**第三步：授予权限**
现在，我们不直接授予客服角色对 `customers` 表的访问权限，而是授予他们对这个**视图**的访问权限。
```sql
GRANT SELECT ON customers_for_support TO support_role;
```
当客服查询 `customers_for_support` 视图时，他们将只能看到脱敏后的数据，而原始的敏感数据在数据库层面就被保护起来了。

---

### 二、数据库审计日志 (Audit Logging)

**审计**是安全合规的基石。它要求系统能够记录下所有重要的数据库操作，以便在发生安全事件时进行追溯、分析和问责。

虽然可以手动使用触发器来记录 `INSERT`, `UPDATE`, `DELETE` 操作，但这无法覆盖 `SELECT` 操作，且性能和维护成本较高。最专业的解决方案是使用专门的审计扩展，其中最著名的是 **`pg_audit`**。

#### 使用 `pg_audit` 扩展

`pg_audit` 是一个开源的 PostgreSQL 扩展，它提供了非常详细和可配置的会话和对象级别的审计日志记录功能。

**1. 安装和配置**
`pg_audit` 需要被添加到 `shared_preload_libraries`。

**`postgresql.conf`:**
```ini
shared_preload_libraries = 'pgaudit'

# --- pgaudit settings ---
# 记录哪些类别的操作
pgaudit.log = 'read, write, ddl, role'
# 'read': SELECT, COPY WHEN a relation is the source
# 'write': INSERT, UPDATE, DELETE, TRUNCATE, COPY WHEN a relation is the destination
# 'ddl': 所有 DDL
# 'role': GRANT, REVOKE, CREATE/ALTER/DROP ROLE

# 日志格式
pgaudit.log_relation = on # 在日志中记录被引用的表
pgaudit.log_parameter = on # 记录查询的参数
```
修改配置后需要**重启 PostgreSQL**。

**2. 在数据库中创建扩展**
```sql
CREATE EXTENSION pgaudit;
```

**3. 查看审计日志**
`pg_audit` 将其日志输出到标准的 PostgreSQL 日志文件中。一条典型的审计日志如下所示：
```log
AUDIT: SESSION,1,1,READ,SELECT,TABLE,public.employees,"SELECT * FROM employees WHERE id = 1",<not logged>
```
这条日志清晰地记录了：
-   **类型**: `AUDIT`
-   **操作**: `READ` / `SELECT`
-   **对象**: `TABLE` `public.employees`
-   **完整语句**: `"SELECT * FROM employees WHERE id = 1"`

**对象级别审计**
`pg_audit` 还允许你只针对特定的表进行审计。例如，只审计对 `employees` 表的 `SELECT` 和 `UPDATE` 操作。
```sql
-- 这需要超级用户权限
SELECT audit.role('hr_audit_role', 'read, write');
ALTER TABLE employees ADD ROLE hr_audit_role;
```

---

## 📌 小结

-   **数据脱敏**是一种重要的隐私保护技术。在 PostgreSQL 中，可以通过创建**安全视图**和自定义函数，向不同角色的用户展示不同视图（原始或脱敏）的数据。
-   **审计日志**对于安全合规和事件追溯至关重要。使用像 **`pg_audit`** 这样的专业扩展，是实现全面、可靠审计的最佳实践。
-   `pg_audit` 提供了丰富的配置选项，可以让你精确地控制需要记录的操作类型（读、写、DDL等），并将详细的审计信息记录到 PostgreSQL 的标准日志流中。

将访问控制（ACLs）、行级安全（RLS）、列级权限、数据脱敏和审计日志这些安全机制结合起来，你可以为你的 PostgreSQL 数据库构建一个多层次、纵深防御的安全体系，满足最严格的行业合规要求。
