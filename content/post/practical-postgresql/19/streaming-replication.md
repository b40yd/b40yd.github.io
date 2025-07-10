+++
title = "第十九章 高可用与故障转移 - 第一节：流复制（Streaming Replication）配置"
date = 2025-07-10
lastmod = 2025-07-10T01:00:00+08:00
tags = ["PostgreSQL", "practical", "book", "high availability", "replication", "streaming"]
categories = ["PostgreSQL", "practical", "book"]
draft = false
author = "b40yd"
+++

# 第五部分：高可用、安全与性能调优
## 第十九章 高可用与故障转移
### 第一节 流复制（Streaming Replication）配置

> **目标**：掌握 PostgreSQL 内置的、最核心的高可用技术——流复制（Streaming Replication）的配置方法，能够手动搭建一个一主一备（Primary-Replica）的物理复制集群。

**高可用（High Availability, HA）**是生产数据库的生命线。它的核心目标是：当主数据库节点因硬件故障、软件崩溃或网络问题而宕机时，系统能够尽快地、甚至自动地切换到一个备份节点上，以最大限度地减少服务中断时间。

PostgreSQL 实现高可用的基石就是**物理流复制**。通过流复制，我们可以创建一个或多个与主节点数据完全一致的、实时的**热备份（Hot Standby）**副本。这些副本平时可以用来处理只读请求（读写分离），而在主节点故障时，可以被“提升”（Promote）为新的主节点。

本节将详细介绍手动配置一个流复制集群的步骤。

---

### 架构

我们将搭建一个最简单的 HA 集群：
-   **`node-primary`** (主节点): IP `192.168.1.10`, Port `5432`
-   **`node-replica`** (副本节点): IP `192.168.1.11`, Port `5432`

---

### 第一步：在主节点 (`node-primary`) 上进行配置

#### 1. 修改 `postgresql.conf`

需要调整几个参数来启用复制功能。
```ini
# postgresql.conf on node-primary

# 1. 监听所有网络接口，而不仅仅是 localhost
listen_addresses = '*'

# 2. 设置 WAL 级别为 replica。这是流复制的最低要求。
#    'logical' 级别也包含 'replica'。
wal_level = replica

# 3. 配置允许连接的复制客户端数量
max_wal_senders = 5 # 允许最多5个副本连接

# 4. (可选) 保留 WAL 日志的数量，以防副本断连时间过长
wal_keep_size = 512MB
```

#### 2. 修改 `pg_hba.conf` (主机认证)

需要允许副本节点以一个特殊的 `replication` 连接类型进行连接。
```conf
# pg_hba.conf on node-primary

# TYPE  DATABASE        USER            ADDRESS                 METHOD
# 允许来自副本节点IP的 'replicator' 用户进行复制连接
host    replication     replicator      192.168.1.11/32         scram-sha-256
```

#### 3. 创建一个专门用于复制的用户

出于安全考虑，不应使用超级用户进行复制。
```sql
-- 在主节点上执行
CREATE ROLE replicator WITH REPLICATION LOGIN PASSWORD 'strong_password';
```

#### 4. 重启主节点

应用以上所有配置。
```bash
sudo systemctl restart postgresql-15
```

---

### 第二步：在副本节点 (`node-replica`) 上进行配置

副本节点的数据必须是主节点的一个精确的物理拷贝。最可靠的方法是使用 `pg_basebackup` 工具来克隆主节点。

#### 1. 停止副本节点的 PostgreSQL 服务（如果正在运行）
```bash
sudo systemctl stop postgresql-15
```

#### 2. 清空数据目录并执行基础备份

```bash
# 备份旧的数据目录 (如果有)
sudo mv /var/lib/postgresql/15/main /var/lib/postgresql/15/main_old

# 以 postgres 系统用户的身份执行 pg_basebackup
sudo -u postgres pg_basebackup -h 192.168.1.10 -p 5432 \
    -U replicator -D /var/lib/postgresql/15/main \
    -Fp -Xs -P -R
```
**`pg_basebackup` 参数解析：**
-   `-h`, `-p`, `-U`: 指定主节点的主机、端口和复制用户名。
-   `-D`: 指定副本节点的新数据目录。
-   `-Fp`: 格式为 `plain`（普通文件），而不是 `tar`。
-   `-Xs`: 在备份期间，通过流式传输获取 WAL 日志。
-   `-P`: 显示进度。
-   `-R`: **非常重要**。这个参数会在副本节点的数据目录中自动创建一个 `standby.signal` 文件，并向 `postgresql.auto.conf` 文件中写入连接到主节点的 `primary_conninfo` 设置。这极大地简化了副本的配置。

#### 3. 检查副本节点的配置

`pg_basebackup -R` 会自动生成 `standby.signal` 文件，这个空文件的存在告诉 PostgreSQL 在启动时应以“备用模式”（Standby Mode）运行。

同时，`postgresql.auto.conf` 文件会被写入如下内容：
```ini
# postgresql.auto.conf on node-replica
primary_conninfo = 'user=replicator password=... host=192.168.1.10 port=5432 ...'
```

#### 4. 启动副本节点

```bash
sudo systemctl start postgresql-15
```

---

### 第三步：验证复制状态

#### 1. 在主节点上查看

```sql
-- 在主节点上执行
SELECT * FROM pg_stat_replication;
```
如果成功，你应该能看到一行记录，其中 `client_addr` 是副本节点的 IP，`state` 是 `streaming`。

#### 2. 在副本节点上查看

副本节点的日志中会显示已成功连接到主节点并开始流式复制。

#### 3. 测试数据同步

在主节点上创建一个表并插入数据：
```sql
-- 在主节点上
CREATE TABLE replication_test (id INT);
INSERT INTO replication_test VALUES (1);
```
在副本节点上查询：
```sql
-- 在副本节点上
SELECT * FROM replication_test;
-- 你应该能立即看到 id 为 1 的记录。
```
尝试在副本节点上写入数据：
```sql
-- 在副本节点上
INSERT INTO replication_test VALUES (2);
-- ERROR:  cannot execute INSERT in a read-only transaction
```
这证明了副本是只读的，符合预期。

---

## 📌 小结

我们成功地手动搭建了一个基于流复制的一主一备高可用集群。
-   **主节点**需要配置 `wal_level`, `max_wal_senders` 和 `pg_hba.conf`，以允许复制连接。
-   **副本节点**通过 `pg_basebackup -R` 工具从主节点克隆，这是最简单、最可靠的初始化方法。
-   `standby.signal` 文件和 `primary_conninfo` 设置是副本节点以备用模式启动的关键。

然而，这个手动搭建的集群还缺少一个至关重要的部分：**自动故障转移（Automatic Failover）**。如果主节点宕机，我们需要手动地在副本节点上执行 `pg_ctl promote` 将其提升为新主，并修改应用连接。在下一节，我们将介绍如何使用 Patroni 这样的工具来实现这个过程的自动化。

```