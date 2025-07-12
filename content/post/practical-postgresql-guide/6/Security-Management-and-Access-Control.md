+++
title = "PostgreSQL实战指南：多领域数据库应用与实践 - 第26章：安全性管理与权限控制"
date = 2025-07-12
lastmod = 2025-07-12T11:25:00+08:00
tags = ["PostgreSQL", "practical", "guide", "book", "dba", "administration", "security", "access-control", "rls"]
categories = ["PostgreSQL", "practical", "guide", "book", "dba"]
draft = false
author = "b40yd"
+++

## PostgreSQL实战指南：多领域数据库应用与实践

-----

### 第六部分：PostgreSQL高级管理与优化

本部分聚焦于 PostgreSQL 的高级运维和管理，内容涵盖高可用架构、备份恢复、安全加固、性能诊断与调优，以及通过扩展开发来增强数据库功能。旨在帮助数据库管理员（DBA）和开发人员精通 PostgreSQL 的管理与维护，确保数据库系统的稳定、安全与高效。

-----

#### 第26章：安全性管理与权限控制

数据库安全是系统安全的基石。PostgreSQL 提供了分层、精细的安全模型，涵盖了从客户端连接认证、对象级权限控制到行级数据访问的方方面面。本章将深入探讨 PostgreSQL 的安全机制，学习如何配置一个安全、可靠的数据库环境。

##### 26.1 认证机制：`pg_hba.conf`

PostgreSQL 的客户端认证由一个名为 `pg_hba.conf` (Host-Based Authentication) 的配置文件集中管理。这个文件中的每一行都是一条规则，它定义了允许哪些用户、从哪些 IP 地址、使用哪种认证方法来访问哪个数据库。PostgreSQL 会自上而下地匹配这些规则。

**`pg_hba.conf` 规则格式:**
`TYPE DATABASE USER ADDRESS METHOD [OPTIONS]`

- **`TYPE`**: 连接类型，如 `local` (Unix-domain socket), `host` (TCP/IP), `hostssl` (SSL加密的TCP/IP)。
- **`DATABASE`**: 目标数据库名，`all` 表示所有。
- **`USER`**: 数据库用户名，`all` 表示所有。
- **`ADDRESS`**: 客户端 IP 地址范围，使用 CIDR 表示法（如 `192.168.1.0/24`）。
- **`METHOD`**: 认证方法，常用的有：
    - `trust`: 无条件信任，不推荐在生产环境使用。
    - `reject`: 无条件拒绝。
    - `md5`: 要求提供 MD5 加密的密码。
    - `scram-sha-256`: (PostgreSQL 10+) 更安全的基于 SCRAM-SHA-256 的密码质询/响应机制，是目前推荐的密码认证方法。
    - `peer`: (用于 `local` 连接) 从内核获取客户端的操作系统用户名，并检查其是否与请求的数据库用户名匹配。
    - `ident`: (用于 `host` 连接) 连接到客户端的 ident 服务器获取操作系统用户名。
    - `ldap`: 使用 LDAP 服务器进行认证。
    - `cert`: 使用 SSL 客户端证书进行认证。

**安全配置示例:**
```conf
# /etc/postgresql/14/main/pg_hba.conf

# TYPE  DATABASE        USER            ADDRESS                 METHOD

# 本地 Unix socket 连接使用 peer 认证
local   all             all                                     peer

# 来自本地网络的 IPv4 连接，要求提供安全的密码
host    all             all             192.168.1.0/24          scram-sha-256

# 来自特定应用服务器的连接
host    my_app_db       app_user        192.168.1.100/32        scram-sha-256

# 拒绝所有其他连接
host    all             all             0.0.0.0/0               reject
```

##### 26.2 权限模型：用户、角色与 `GRANT`/`REVOKE`

PostgreSQL 的权限管理基于“角色”（Role）的概念。实际上，用户（User）只是一个带有 `LOGIN` 权限的角色的别名。你可以创建角色，将权限授予角色，然后再将角色授予用户，从而实现权限的灵活管理。

- **`CREATE ROLE` / `CREATE USER`**: 创建角色/用户。
- **`GRANT`**: 授予权限。
- **`REVOKE`**: 撤销权限。

**权限类型:**
- **对象权限**: `SELECT`, `INSERT`, `UPDATE`, `DELETE` (对表), `USAGE` (对 schema), `EXECUTE` (对函数) 等。
- **角色权限**: 一个角色可以被授予给另一个角色，从而继承其所有权限。

**权限管理最佳实践:**
遵循“最小权限原则”（Principle of Least Privilege），即只授予用户完成其工作所必需的最少权限。

```sql
-- 1. 创建一个只读角色
CREATE ROLE readonly_role;
GRANT CONNECT ON DATABASE my_db TO readonly_role;
GRANT USAGE ON SCHEMA public TO readonly_role;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO readonly_role;
-- 对于未来新建的表，也自动授予 SELECT 权限
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT SELECT ON TABLES TO readonly_role;

-- 2. 创建一个只写角色
CREATE ROLE writeonly_role;
GRANT CONNECT ON DATABASE my_db TO writeonly_role;
GRANT USAGE ON SCHEMA public TO writeonly_role;
GRANT INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO writeonly_role;
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT INSERT, UPDATE, DELETE ON TABLES TO writeonly_role;

-- 3. 创建具体用户，并授予角色
CREATE USER analyst_user WITH PASSWORD 'secure_password';
CREATE USER app_user WITH PASSWORD 'another_secure_password';

GRANT readonly_role TO analyst_user;
GRANT writeonly_role TO app_user;
```

##### 26.3 行级安全 (Row-Level Security, RLS)

在某些场景下，我们需要更精细的访问控制，例如，在多租户应用中，确保一个租户的用户只能看到属于自己租户的数据。行级安全（RLS）就是为此而设计的。

RLS 通过为表定义“安全策略”（Policy）来实现。当一个表启用了 RLS 后，用户对该表的任何查询（包括 `SELECT`, `INSERT`, `UPDATE`, `DELETE`）都必须先通过策略的检查。

**RLS 步骤:**
1.  在表上启用 RLS: `ALTER TABLE ... ENABLE ROW LEVEL SECURITY`。
2.  创建策略: `CREATE POLICY ... ON ... USING (expression)`。`USING` 子句中的表达式返回布尔值，只有当表达式为 `true` 时，行才是可见的。

##### 26.4 场景实战：实现多租户数据隔离

**业务场景描述:**

在一个多租户 SaaS 应用中，所有租户的数据都存储在同一个 `documents` 表中，该表有一个 `tenant_id` 列来区分数据归属。我们需要确保用户查询时，只能访问到与自己 `tenant_id` 匹配的文档。

**实现步骤:**

```sql
-- 1. 准备表和数据
CREATE TABLE documents (
    id SERIAL PRIMARY KEY,
    tenant_id TEXT NOT NULL,
    content TEXT
);

INSERT INTO documents (tenant_id, content) VALUES
('tenant_a', 'Document 1 for A'),
('tenant_a', 'Document 2 for A'),
('tenant_b', 'Document 1 for B');

-- 2. 创建用户，并使用 set_config 设置其租户ID
CREATE USER user_a WITH PASSWORD 'pass_a';
CREATE USER user_b WITH PASSWORD 'pass_b';

-- 3. 在表上启用 RLS
ALTER TABLE documents ENABLE ROW LEVEL SECURITY;
-- 注意：默认情况下，表的所有者(通常是超级用户)不受 RLS 限制。
-- 如果想让所有者也受限，使用 ALTER TABLE ... FORCE ROW LEVEL SECURITY;

-- 4. 创建安全策略
-- 这个策略会检查当前会话的 'app.tenant_id' 变量是否与行中的 tenant_id 列匹配
CREATE POLICY tenant_isolation_policy ON documents
    FOR ALL -- 对 SELECT, INSERT, UPDATE, DELETE 都生效
    USING (tenant_id = current_setting('app.tenant_id', true))
    WITH CHECK (tenant_id = current_setting('app.tenant_id', true));

-- 'WITH CHECK' 子句对 INSERT 和 UPDATE 生效，防止用户插入或更新不属于自己租户的数据
```

**验证 RLS:**

现在，当用户连接到数据库后，应用程序需要在其会话中设置 `app.tenant_id` 变量。

```sql
-- 以 user_a 的身份连接并设置会话变量
SET app.tenant_id = 'tenant_a';

-- user_a 执行查询
SELECT * FROM documents;
-- 结果:
-- id | tenant_id | content
-- ---|-----------|------------------
--  1 | tenant_a  | Document 1 for A
--  2 | tenant_a  | Document 2 for A
-- (user_a 看不到 tenant_b 的数据)

-- user_a 尝试插入错误的数据 (将会失败)
-- INSERT INTO documents (tenant_id, content) VALUES ('tenant_b', 'Malicious data');
-- ERROR:  new row violates WITH CHECK OPTION for policy "tenant_isolation_policy"
```

##### 26.5 总结

本章我们全面地探讨了 PostgreSQL 的安全体系。我们学习了如何通过 `pg_hba.conf` 文件来配置强大的客户端认证，掌握了基于角色的权限管理模型和最小权限原则，并深入实践了行级安全（RLS）这一实现多租户数据隔离的利器。将这些机制结合起来，可以为你的 PostgreSQL 数据库构建一个纵深防御的安全环境。

在下一章，我们将进入性能调优领域，学习如何监控数据库、分析查询计划，并找出性能瓶颈。
-----
