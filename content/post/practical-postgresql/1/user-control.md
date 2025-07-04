+++
title = "PostgreSQL 数据库实战指南 - 用户权限管理基础"
date = 2025-07-04
lastmod = 2025-07-04T04:00:00+08:00
tags = ["PostgreSQL", "practical", "book", "RBAC"]
categories = ["PostgreSQL", "practical", "book", "RBAC"]
draft = false
author = "b40yd"
+++

# 第一章 PostgreSQL 基础与安装配置
## 第四节 用户权限管理基础

> **目标**：掌握 PostgreSQL 中用户和角色的基本概念、创建与管理方法，并能够根据最小权限原则合理分配数据库访问权限，保障数据库安全性。

在任何生产级数据库系统中，合理的用户权限管理是安全性的核心。PostgreSQL 提供了灵活且强大的权限控制机制，支持基于角色（Role-based Access Control, RBAC）的权限管理模型。

本节将介绍：

- PostgreSQL 中的角色（Role）与用户的区别
- 如何创建、修改和删除角色
- 权限的授予与回收（GRANT / REVOKE）
- 最小权限原则与安全实践

---

## 🧑‍💼 一、角色（Role）与用户（User）

在 PostgreSQL 中，“角色”是一个更通用的概念，既可以代表一个实际的数据库用户，也可以作为组角色（Group Role），用于权限的统一管理。

### 1. 角色分类

| 类型 | 特点 |
|------|------|
| 普通角色（Normal Role） | 默认无登录权限，通常用于权限继承或授权 |
| 登录角色（Login Role） | 可以连接数据库，相当于“用户” |
| 管理员角色（Superuser） | 具有所有权限，可执行任何操作（谨慎使用） |
| 组角色（Group Role） | 本身不能登录，但可以包含多个成员角色 |

### 2. 查看现有角色

```sql
SELECT rolname, rolsuper, rolcanlogin, rolconnlimit FROM pg_roles;
```

或者使用 `psql` 快捷命令：

```bash
\du
```

---

## 🔐 二、创建与管理角色

### 1. 创建角色

#### 使用 SQL 创建角色：

```sql
CREATE ROLE developer LOGIN PASSWORD 'securepass';
```

- `LOGIN` 表示该角色可以登录数据库。
- `PASSWORD` 设置登录密码（明文或加密形式均可）。

#### 创建超级用户角色：

```sql
CREATE ROLE admin SUPERUSER LOGIN PASSWORD 'adminpass';
```

> ⚠️ 超级用户拥有完全控制权，请确保仅在必要时创建，并设置强密码。

#### 创建只读角色：

```sql
CREATE ROLE readonly_user LOGIN PASSWORD 'readonlypass';
GRANT CONNECT ON DATABASE your_db TO readonly_user;
GRANT USAGE ON SCHEMA public TO readonly_user;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO readonly_user;
```

### 2. 修改角色属性

```sql
ALTER ROLE developer WITH PASSWORD 'newpassword';
ALTER ROLE developer RENAME TO dev_user;
ALTER ROLE dev_user WITH NOLOGIN; -- 禁止登录
```

### 3. 删除角色

```sql
DROP ROLE IF EXISTS dev_user;
```

⚠️ 注意：如果角色拥有对象（如表、函数等），需先转移所有权或删除对象，否则会报错。

---

## 📌 三、权限管理：GRANT 与 REVOKE

PostgreSQL 支持细粒度的权限控制，包括对数据库、模式、表、列、函数等对象的访问控制。

### 1. 数据库级别权限

```sql
-- 授予某角色连接数据库的权限
GRANT CONNECT ON DATABASE mydb TO dev_user;

-- 回收连接权限
REVOKE CONNECT ON DATABASE mydb FROM dev_user;
```

### 2. 模式（Schema）级别权限

```sql
-- 授予使用 schema 的权限
GRANT USAGE ON SCHEMA public TO dev_user;

-- 授予创建对象的权限
GRANT CREATE ON SCHEMA public TO dev_user;
```

### 3. 表级别权限

```sql
-- 授予 SELECT 权限
GRANT SELECT ON employees TO dev_user;

-- 授予 INSERT, UPDATE, DELETE 权限
GRANT INSERT, UPDATE, DELETE ON employees TO dev_user;

-- 回收某个权限
REVOKE UPDATE ON employees FROM dev_user;
```

### 4. 列级别权限（适用于敏感字段）

```sql
-- 授予对 salary 字段的 SELECT 权限
GRANT SELECT (salary) ON employees TO dev_user;

-- 授予对 salary 字段的 UPDATE 权限
GRANT UPDATE (salary) ON employees TO dev_user;
```

### 5. 函数/过程权限

```sql
-- 授予执行函数的权限
GRANT EXECUTE ON FUNCTION calculate_salary(integer) TO dev_user;
```

---

## 🧱 四、权限继承与角色嵌套

PostgreSQL 支持角色之间的成员关系，允许将权限集中管理。

### 示例：创建组角色并添加成员

```sql
-- 创建开发组角色
CREATE ROLE developers;

-- 将已有角色加入组角色
GRANT developers TO dev_user1, dev_user2;

-- 授予 developers 组权限
GRANT SELECT ON ALL TABLES IN SCHEMA public TO developers;
```

这样，`dev_user1` 和 `dev_user2` 自动继承了 `developers` 组的权限。

---

## 🛡️ 五、安全最佳实践

| 实践 | 描述 |
|------|------|
| 最小权限原则 | 只授予完成任务所需的最小权限 |
| 分离职责 | 不同业务模块使用不同角色，避免权限重叠 |
| 定期审计权限 | 定期检查角色权限，清理过期或不必要的权限 |
| 密码策略 | 强制使用复杂密码，启用密码过期策略 |
| 启用 SSL 连接 | 在 `pg_hba.conf` 中限制 SSL 访问 |
| 使用行级安全（RLS） | 对多租户或数据隔离场景非常有用 |
| 日志审计 | 开启日志记录，监控异常行为 |

---

## 🧪 六、实战演练：为应用创建专用用户并授予权限

### 场景描述：

你正在部署一个 Web 应用，需要为应用创建专用数据库用户，只能访问特定数据库和表，并具备基本的 CRUD 权限。

### 步骤如下：

```sql
-- 1. 创建应用用户
CREATE ROLE webapp_user LOGIN PASSWORD 'webapppass';

-- 2. 授予连接数据库权限
GRANT CONNECT ON DATABASE appdb TO webapp_user;

-- 3. 授予对 public 模式的使用权限
GRANT USAGE ON SCHEMA public TO webapp_user;

-- 4. 授予对 users 表的 CRUD 权限
GRANT SELECT, INSERT, UPDATE, DELETE ON users TO webapp_user;

-- 5. 授予序列使用权限（如果表中有 SERIAL 主键）
GRANT USAGE, SELECT ON SEQUENCE users_id_seq TO webapp_user;
```

现在，`webapp_user` 可以正常连接并操作 `users` 表，但无法访问其他表或执行高危操作。

---

## 📌 小结

| 操作 | 命令示例 |
|------|----------|
| 创建角色 | `CREATE ROLE dev_user LOGIN PASSWORD 'pass';` |
| 修改角色 | `ALTER ROLE dev_user WITH PASSWORD 'newpass';` |
| 授予权限 | `GRANT SELECT ON table_name TO role_name;` |
| 回收权限 | `REVOKE SELECT ON table_name FROM role_name;` |
| 查看权限 | `\du`, `\z`, 查询 `pg_roles`, `information_schema.table_privileges` |

通过本节的学习，你应该已经掌握了 PostgreSQL 中用户和角色的基本管理方法，以及如何根据业务需求进行权限分配。下一节我们将进入实战环节：**实战：搭建本地开发环境并导入样例数据集**，帮助你将前面所学知识综合运用起来。
