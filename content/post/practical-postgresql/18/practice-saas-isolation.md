+++
title = "第十八章 多租户架构设计 - 第三节 实战：SaaS 应用中的多租户数据隔离"
date = 2025-07-10
lastmod = 2025-07-10T01:00:00+08:00
tags = ["PostgreSQL", "practical", "book", "multi-tenancy", "rls", "schema"]
categories = ["PostgreSQL", "practical", "book"]
draft = false
author = "b40yd"
+++

## 第三节 实战：SaaS 应用中的多租户数据隔离

> **目标**：通过为一个项目管理 SaaS 工具设计数据隔离方案，具体地、并排地对比 RLS 模型和 Schema-per-Tenant 模型的实现细节、优缺点和适用场景。

### 场景描述

我们正在构建一个名为 "ProjectHub" 的项目管理 SaaS 应用。每个注册的公司（租户）都可以在平台上创建自己的项目（`projects`）和任务（`tasks`）。

**核心需求：**
1.  **数据隔离**：ACME 公司的用户绝对不能看到 Globex 公司的项目和任务。
2.  **用户角色**：在租户内部，可能还有更复杂的角色（如管理员、普通成员），但这超出了本节范围，我们假设租户内的所有用户权限相同。
3.  **可扩展性**：系统需要能够支持未来成千上万个公司注册使用。

我们将用两种不同的架构模式来实现这个需求。

---

## 方案一：RLS - 共享数据库，共享 Schema

这是更现代、更具扩展性的方案。所有租户的数据都存储在公共的 `projects` 和 `tasks` 表中，通过 `tenant_id` 列和 RLS 策略进行隔离。

#### 1. 表结构设计

```sql
CREATE TABLE projects (
    id SERIAL PRIMARY KEY,
    tenant_id TEXT NOT NULL,
    name TEXT NOT NULL
);

CREATE TABLE tasks (
    id SERIAL PRIMARY KEY,
    tenant_id TEXT NOT NULL,
    project_id INT REFERENCES projects(id),
    title TEXT NOT NULL,
    completed BOOLEAN DEFAULT FALSE
);
```

#### 2. RLS 策略定义

```sql
-- 在两张表上启用 RLS
ALTER TABLE projects ENABLE ROW LEVEL SECURITY;
ALTER TABLE tasks ENABLE ROW LEVEL SECURITY;

-- 为 projects 表创建策略
CREATE POLICY tenant_projects_policy ON projects
    FOR ALL TO PUBLIC
    USING (tenant_id = current_setting('app.tenant_id', true))
    WITH CHECK (tenant_id = current_setting('app.tenant_id', true));

-- 为 tasks 表创建策略
CREATE POLICY tenant_tasks_policy ON tasks
    FOR ALL TO PUBLIC
    USING (tenant_id = current_setting('app.tenant_id', true))
    WITH CHECK (tenant_id = current_setting('app.tenant_id', true));
```
*`current_setting('app.tenant_id', true)` 中的第二个参数 `true` 表示如果参数未设置，不会报错，而是返回 `NULL`，这可以防止意外的数据暴露。*

#### 3. 应用层交互

当一个来自 'acme' 公司的用户登录时，应用在处理其每个请求的数据库连接时，都需要：
1.  开启一个事务。
2.  `SET LOCAL app.tenant_id = 'acme';`
3.  执行业务 SQL 查询。
4.  提交事务。

**优点：**
-   **易于维护**：当需要为 `tasks` 表增加一个 `due_date` 列时，只需执行一次 `ALTER TABLE tasks ADD COLUMN due_date DATE;`。
-   **易于分析**：`SELECT count(*) FROM projects;` 可以轻松统计出平台上的项目总数。
-   **扩展性好**：增加新租户只是在表中增加新行，对数据库的元数据没有影响，可以平滑扩展到海量租户。

---

## 方案二：Schema-per-Tenant - 独立 Schema

这个方案为每个租户提供逻辑上独立的命名空间。

#### 1. 租户管理

需要一个公共的 `public.tenants` 表来管理所有租户。
```sql
CREATE TABLE public.tenants (
    id SERIAL PRIMARY KEY,
    name TEXT NOT NULL,
    schema_name TEXT NOT NULL UNIQUE
);
```

#### 2. 新租户注册流程

当 'acme' 公司注册时，应用需要执行一系列操作：
1.  `INSERT INTO public.tenants (name, schema_name) VALUES ('ACME Corp', 'tenant_acme');`
2.  `CREATE SCHEMA tenant_acme;`
3.  在 `tenant_acme` Schema 内创建 `projects` 和 `tasks` 表：
    ```sql
    CREATE TABLE tenant_acme.projects ( ... ); -- 无需 tenant_id 列
    CREATE TABLE tenant_acme.tasks ( ... );    -- 无需 tenant_id 列
    ```

#### 3. 应用层交互

当一个来自 'acme' 公司的用户登录时，应用在处理其请求时：
1.  `SET search_path TO tenant_acme, public;`
2.  执行业务 SQL 查询，如 `SELECT * FROM projects;` (这将自动指向 `tenant_acme.projects`)。

**优点：**
-   **数据强隔离**：租户的数据在物理上更隔离，没有意外混淆的风险。
-   **易于删除租户**：当一个租户注销时，只需 `DROP SCHEMA tenant_acme CASCADE;` 即可干净地删除其所有数据。

---

## 最终对比与决策

| 考量点 | RLS 方案 | Schema-per-Tenant 方案 | 决策建议 |
| :--- | :--- | :--- | :--- |
| **开发敏捷性** | **更高**。修改表结构非常简单。 | **较低**。Schema 迁移是主要痛点。 | 如果你的应用需要快速迭代，**RLS** 更优。 |
| **运维复杂度** | **较低**。标准的数据库运维。 | **较高**。需要自动化脚本来管理大量的 Schema。 | 如果运维资源有限，**RLS** 更易于管理。 |
| **长期扩展性** | **极好**。可以支持数十万租户。 | **有限**。租户过多可能导致性能问题和管理噩梦。 | 如果目标是大规模 SaaS，**RLS** 是必然选择。 |
| **数据隔离感** | 逻辑隔离，依赖策略的正确性。 | **物理隔离感更强**，更符合传统安全模型。 | 如果客户对数据隔离有极强的心理要求，**Schema** 方案更具说服力。 |
| **租户定制化** | 困难。所有租户共享表结构。 | **容易**。可以单独修改某个租户的 Schema。 | 如果需要为大客户提供定制化功能，**Schema** 方案更灵活。 |

**结论：**
对于我们这个需要支持海量用户的通用项目管理 SaaS "ProjectHub" 来说，**RLS（共享 Schema）方案是更明智、更具前瞻性的选择**。它牺牲了一点物理隔离感，但换来了无与伦比的**可维护性**和**可扩展性**，这对于一个长期发展的 SaaS 产品是至关重要的。

Schema-per-Tenant 模型则更像一个“精品店”模式，适合租户数量不多、但每个租户客单价高、且有定制化需求的场景。
