+++
title = "第二十章 安全机制与合规性 - 第二节：行级安全与列级权限控制"
date = 2025-07-10
lastmod = 2025-07-10T01:00:00+08:00
tags = ["PostgreSQL", "practical", "book", "security", "rls", "acl"]
categories = ["PostgreSQL", "practical", "book"]
draft = false
author = "b40yd"
+++

## 第二节 行级安全与列级权限控制

> **目标**：掌握 PostgreSQL 中两种最核心的、用于精细化数据访问控制的机制——行级安全（Row-Level Security, RLS）和列级权限（Column-Level Privileges），并理解如何将它们结合使用以实现复杂的安全矩阵。

标准的 `GRANT` 语句可以控制用户对**整张表**的 `SELECT`, `INSERT`, `UPDATE`, `DELETE` 权限。但很多时候，我们需要更精细的控制：
-   “用户 A 只能看到**他自己所在部门**的员工记录。” (按行过滤)
-   “所有员工都能看到 `employees` 表，但只有 HR 部门的用户才能看到 `salary` 这一**列**。” (按列过滤)

PostgreSQL 提供了强大的功能来满足这两种需求。

---

### 一、行级安全 (Row-Level Security, RLS) 回顾

我们在第十八章已经深入探讨过 RLS。这里我们快速回顾其核心思想：
-   **目的**：控制用户能够访问**哪些行**。
-   **机制**：通过 `CREATE POLICY` 创建一个策略，该策略会被**自动、强制**地应用为查询的 `WHERE` 子句。
-   **核心**：将访问控制逻辑从应用层下沉到数据库层，提供更根本的安全保障。

**示例：用户只能看到自己的待办事项**
```sql
-- 假设已有一个 tasks 表，包含 user_id 列
-- 假设已通过 SET aoo.current_user_id = '...' 设置了当前用户

ALTER TABLE tasks ENABLE ROW LEVEL SECURITY;

CREATE POLICY user_tasks_policy ON tasks
    FOR ALL TO app_users -- 假设 app_users 是一个用户组角色
    USING (user_id = current_setting('app.current_user_id'));
```
这个策略确保了任何隶属于 `app_users` 角色的用户，在查询 `tasks` 表时，其 SQL 都会被隐式地加上 `WHERE user_id = '当前登录的用户ID'` 这个条件。

---

### 二、列级权限 (Column-Level Privileges)

列级权限是标准 `GRANT` 语句的一个扩展，它允许你将权限授予到**表的特定列**上，而不是整张表。

**目的**：控制用户能够访问**哪些列**。

**语法：**
`GRANT <privilege> (column_name, ...) ON table_name TO role_name;`

#### 实战：保护员工的薪水信息

**场景**：我们有一个 `employees` 表，包含 `id`, `name`, `department`, `salary` 等列。
-   所有普通员工（`staff` 角色）都可以查看员工名录（`id`, `name`, `department`）。
-   只有人力资源部门的用户（`hr_user` 角色）才能查看和修改 `salary` 列。

**第一步：准备表和角色**
```sql
CREATE TABLE employees (
    id SERIAL PRIMARY KEY,
    name TEXT NOT NULL,
    department TEXT,
    salary NUMERIC(10, 2)
);

CREATE ROLE staff;
CREATE ROLE hr_user;

INSERT INTO employees (name, department, salary) VALUES
('Alice', 'Engineering', 80000),
('Bob', 'HR', 75000);
```

**第二步：授予权限**

1.  **首先，撤销所有角色对整张表的默认权限。**
    这是一个好习惯，确保我们从一个最小权限状态开始。
    ```sql
    REVOKE ALL ON employees FROM PUBLIC;
    ```

2.  **为 `staff` 角色授予对公共列的 `SELECT` 权限。**
    ```sql
    GRANT SELECT (id, name, department) ON employees TO staff;
    ```

3.  **为 `hr_user` 角色授予对所有列的权限。**
    ```sql
    -- HR 可以查看所有列
    GRANT SELECT ON employees TO hr_user;
    -- HR 可以更新薪水
    GRANT UPDATE (salary) ON employees TO hr_user;
    ```

**第三步：验证权限**

**以普通员工身份查询：**
```sql
-- 模拟一个 staff 成员
SET ROLE staff;

-- 这个查询会成功
SELECT id, name, department FROM employees;

-- 这个查询会失败，因为没有 salary 列的权限
SELECT salary FROM employees;
-- ERROR:  permission denied for table employees
```

**以 HR 员工身份查询：**
```sql
SET ROLE hr_user;

-- 这个查询会成功
SELECT id, name, salary FROM employees;

-- 尝试更新非授权列，会失败
UPDATE employees SET name = 'Robert' WHERE id = 2;
-- ERROR:  permission denied for table employees

-- 尝试更新薪水，会成功
UPDATE employees SET salary = 78000 WHERE id = 2;
```

---

### 三、结合 RLS 和列级权限

这两种机制可以完美地结合在一起，实现一个强大的安全矩阵。

**场景**：一个部门经理（`manager` 角色）应该：
1.  **只能看到**自己所在部门的员工记录 (RLS)。
2.  **不能看到**这些员工的 `salary` 列 (列级权限)。

**实现：**
1.  为 `employees` 表创建一个 RLS 策略，`USING (department = current_setting('app.user_department'))`。
2.  像上面一样，只将 `(id, name, department)` 的 `SELECT` 权限授予 `manager` 角色。

当一个经理查询时，PostgreSQL 会首先应用 RLS 策略，将结果集过滤为只包含他自己部门的员工；然后，再应用列级权限，确保他只能看到被授权的列。

---

## 📌 小结

-   **行级安全 (RLS)** 控制**“能看哪些行”**，是实现多租户隔离和按用户身份过滤数据的理想工具。
-   **列级权限** 控制**“能看哪些列”**，是保护敏感数据字段（如薪水、个人身份信息 PII）的直接手段。

在设计数据库安全模型时，应将这两种机制视为你工具箱中的核心工具。通过将它们与传统的基于角色的访问控制（RBAC）相结合，你可以在数据库层面构建起一个纵深防御体系，无论应用层代码如何变化，都能为你的核心数据资产提供坚实的保护。
