+++
title = "PostgreSQL 数据库实战指南 - 初始化集群与基本配置"
lastmod = 2025-07-04T02:00:00+08:00
tags = ["PostgreSQL", "practical", "book", "database", "cluster", "config"]
categories = ["PostgreSQL", "practical", "book", "database", "cluster", "config"]
draft = false
author = "b40yd"
+++

# 第一章 PostgreSQL 基础与安装配置
## 第二节 初始化集群与基本配置（pg_hba.conf, postgresql.conf）

> **目标**：掌握 PostgreSQL 集群的初始化流程，理解关键配置文件的作用，并能够根据实际需求修改 `postgresql.conf` 和 `pg_hba.conf` 文件以完成数据库的基本配置。

在成功安装 PostgreSQL 后，下一步通常是初始化一个“数据库集群”（Database Cluster），它是 PostgreSQL 用来组织一组数据库实例的逻辑单位。每个集群包含多个数据库和共享的全局对象（如角色、表空间等）。本节将介绍如何手动初始化集群，并深入讲解两个核心配置文件：

- `postgresql.conf`：主配置文件，用于设置数据库引擎的核心参数。
- `pg_hba.conf`：客户端认证配置文件，用于控制谁可以从哪里连接到数据库。

---

## 🧱 一、初始化数据库集群

在大多数 Linux 发行版中，安装 PostgreSQL 时会自动初始化默认集群。但在某些场景下（例如自定义路径部署或从源码编译），我们需要手动初始化集群。

### 手动初始化命令如下：

```bash
sudo -i -u postgres
initdb -D /usr/local/pgsql/data
```

- `-D` 指定数据目录（Data Directory），你也可以使用其他路径，如 `/var/lib/postgresql/17/main`。
- `initdb` 会创建必要的系统表、模板数据库（`template0` 和 `template1`）以及初始用户 `postgres`。

### 启动数据库服务：

```bash
pg_ctl -D /usr/local/pgsql/data -l logfile start
```

你可以通过以下命令验证是否启动成功：

```bash
psql -c "SELECT version();"
```

---

## ⚙️ 二、postgresql.conf：主配置文件详解

`postgresql.conf` 是 PostgreSQL 的核心配置文件，位于数据目录中（如 `/etc/postgresql/17/main/postgresql.conf` 或 `/usr/local/pgsql/data/postgresql.conf`）。

### 常见配置项说明：

| 配置项 | 默认值 | 描述 |
|--------|--------|------|
| `listen_addresses` | `localhost` | 监听的 IP 地址，设为 `*` 表示监听所有接口 |
| `port` | `5432` | 数据库监听端口 |
| `max_connections` | `100` | 最大并发连接数 |
| `shared_buffers` | `128MB` | 共享内存缓冲区大小，建议为物理内存的 25% |
| `work_mem` | `4MB` | 排序和哈希操作使用的内存量 |
| `maintenance_work_mem` | `64MB` | 维护操作（如 VACUUM、CREATE INDEX）使用的内存 |
| `checkpoint_segments` | `3` | 检查点之间写入的 WAL 段数量（v17 中已被废弃） |
| `checkpoint_timeout` | `5min` | 检查点之间的最大时间间隔 |
| `logging_collector` | `off` | 是否启用日志收集器 |
| `log_directory` | `pg_log` | 日志文件存储目录 |
| `log_filename` | `postgresql-%Y-%m-%d_%H%M%S.log` | 日志文件命名格式 |

### 示例：优化生产环境配置（假设服务器有 16GB 内存）

```conf
listen_addresses = '*'
port = 5432
max_connections = 200
shared_buffers = 4GB
work_mem = 64MB
maintenance_work_mem = 1GB
checkpoint_segments = 16
checkpoint_timeout = 15min
logging_collector = on
log_directory = 'pg_log'
log_filename = 'postgresql-%a.log'
log_truncate_on_rotation = on
log_rotation_age = 1d
```

### 修改后重启服务生效：

```bash
sudo systemctl restart postgresql
```

---

## 🔐 三、pg_hba.conf：客户端认证配置文件详解

`pg_hba.conf` 控制客户端访问权限，决定哪些用户可以从哪些主机连接到哪些数据库，以及使用什么方式进行认证。

该文件也位于数据目录中（如 `/etc/postgresql/17/main/pg_hba.conf`）。

### 文件结构说明：

每一行为一条规则，格式如下：

```
<type> <database> <user> <address> <auth-method>
```

| 字段 | 说明 |
|------|------|
| type | 连接类型：`local`（本地）、`host`（TCP/IP） |
| database | 要连接的数据库名，可为 `all` |
| user | 用户名，可为 `all` |
| address | 客户端 IP 地址或 CIDR 范围（仅 TCP/IP 类型） |
| auth-method | 认证方式：`trust`, `peer`, `ident`, `password`, `md5`, `scram-sha-256` 等 |

### 示例配置：

```conf
# 允许本地用户无密码登录所有数据库
local   all             all                                     peer

# 允许远程 IP 192.168.1.0/24 使用 md5 密码登录所有数据库
host    all             all             192.168.1.0/24        md5

# 允许特定用户 admin 登录 mydb 数据库，使用 SCRAM 认证
host    mydb            admin           192.168.10.0/24       scram-sha-256

# 限制只读用户只读访问
host    readonly_db     reader          10.0.0.0/16           md5
```

### 常用认证方式对比：

| 方法 | 安全性 | 说明 |
|------|--------|------|
| trust | ⚠️低 | 不进行认证，任何尝试都允许连接 |
| peer/ident | ✅中 | 依赖操作系统用户名匹配 |
| password/md5 | ✅中 | 明文或加密密码认证 |
| scram-sha-256 | ✅高 | 推荐使用，支持更强的密码哈希机制 |

### 应用配置变更：

修改完 `pg_hba.conf` 后，需重新加载配置使更改生效：

```bash
sudo pg_ctl reload -D /usr/local/pgsql/data
```

或使用服务管理命令：

```bash
sudo systemctl reload postgresql
```

---

## 📝 四、实用工具推荐

### 1. 查看当前配置信息

```sql
-- 查看 listen_addresses
SHOW listen_addresses;

-- 查看 max_connections
SHOW max_connections;

-- 查看所有配置项
SELECT name, setting FROM pg_settings;
```

### 2. 使用 `pg_config` 工具查看安装路径

```bash
pg_config --bindir      # 可执行文件路径
pg_config --datadir     # 默认数据目录
pg_config --docdir      # 文档路径
```

---

## 🧪 五、实战演练：配置远程访问并创建测试用户

### 步骤 1：修改 `pg_hba.conf`

添加一行允许远程访问：

```conf
host    all             all             0.0.0.0/0               md5
```

> ⚠️ 生产环境请不要使用 `0.0.0.0/0`，应限制具体 IP 段。

### 步骤 2：修改 `postgresql.conf`

确保监听地址已开启：

```conf
listen_addresses = '*'
```

### 步骤 3：创建测试用户和数据库

```bash
sudo -i -u postgres
createuser testuser
createdb testdb
```

然后编辑 `.pgpass` 文件设置密码：

```bash
echo "localhost:5432:testdb:testuser:yourpassword" >> ~/.pgpass
chmod 600 ~/.pgpass
```

### 步骤 4：远程连接测试

使用 psql 或图形工具（如 DBeaver、pgAdmin）连接 PostgreSQL：

```bash
psql -h your_postgres_server_ip -U testuser -d testdb
```

---

## 📌 小结

| 文件 | 作用 | 关键配置项 |
|------|------|------------|
| `postgresql.conf` | 控制数据库引擎运行参数 | `listen_addresses`, `max_connections`, `shared_buffers`, `work_mem` |
| `pg_hba.conf` | 控制客户端访问权限 | `host`, `local`, `auth-method` |

通过本节的学习，你应该已经掌握了 PostgreSQL 集群的初始化流程，以及如何通过 `postgresql.conf` 和 `pg_hba.conf` 文件进行基础配置。下一节我们将介绍 PostgreSQL 的核心工具 `psql` 与图形化客户端（如 pgAdmin）的使用。
