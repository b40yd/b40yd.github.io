+++
title = "第十五章 Citus 扩展实战 - 第一节：安装 Citus 并创建分布式表"
date = 2025-07-10
lastmod = 2025-07-10T01:00:00+08:00
tags = ["PostgreSQL", "practical", "book", "distributed", "citus", "install"]
categories = ["PostgreSQL", "practical", "book"]
draft = false
author = "b40yd"
+++

## 第十五章 Citus 扩展实战
### 第一节 安装 Citus 并创建分布式表

> **目标**：学习在本地环境中搭建一个简单的 Citus 分布式集群（一个协调器和两个工作节点），并掌握创建分布式表的核心命令。

理论之后，实践开始。本节我们将亲手搭建一个 Citus 集群。为了简化，我们将在同一台机器上运行三个不同的 PostgreSQL 实例来模拟协调器和工作节点。在生产环境中，这些节点会部署在不同的物理服务器上。

---

### 第一步：安装 Citus 扩展

首先，我们需要在系统中安装 Citus 扩展包。

**Debian/Ubuntu:**
```bash
# 添加 Citus 官方软件源
curl https://install.citusdata.com/community/deb.sh | sudo bash

# 安装与你的 PostgreSQL 版本匹配的包
sudo apt-get install -y postgresql-15-citus-12.1
```
*(请将版本号 `15` 和 `12.1` 替换为最新的稳定版本)*

**Red Hat/CentOS:**
```bash
# 添加 Citus 官方软件源
curl https://install.citusdata.com/community/rpm.sh | sudo bash

# 安装
sudo yum install -y citus-12.1_15
```

---

### 第二步：初始化集群节点

我们将使用不同的端口（5432, 5433, 5434）来模拟三个独立的 PostgreSQL 实例。

**1. 创建数据目录**
```bash
mkdir -p ~/citus_cluster/coordinator
mkdir -p ~/citus_cluster/worker1
mkdir -p ~/citus_cluster/worker2
```

**2. 初始化每个节点**
```bash
# 初始化协调器
initdb -D ~/citus_cluster/coordinator

# 初始化工作节点1
initdb -D ~/citus_cluster/worker1

# 初始化工作节点2
initdb -D ~/citus_cluster/worker2
```

**3. 修改配置文件**
为每个节点设置不同的端口，并加载 Citus 扩展。

**协调器 `~/citus_cluster/coordinator/postgresql.conf`:**
```ini
port = 5432
shared_preload_libraries = 'citus'
```

**工作节点1 `~/citus_cluster/worker1/postgresql.conf`:**
```ini
port = 5433
shared_preload_libraries = 'citus'
```

**工作节点2 `~/citus_cluster/worker2/postgresql.conf`:**
```ini
port = 5434
shared_preload_libraries = 'citus'
```

**4. 启动所有节点**
```bash
pg_ctl -D ~/citus_cluster/coordinator -l coordinator.log start
pg_ctl -D ~/citus_cluster/worker1 -l worker1.log start
pg_ctl -D ~/citus_cluster/worker2 -l worker2.log start
```

---

### 第三步：配置集群

现在，所有节点都在独立运行。我们需要告诉协调器，哪些是它的工作节点。

**1. 在每个节点上创建 Citus 扩展**
```bash
psql -p 5432 -c "CREATE EXTENSION citus;"
psql -p 5433 -c "CREATE EXTENSION citus;"
psql -p 5434 -c "CREATE EXTENSION citus;"
```

**2. 将工作节点信息添加到协调器**
连接到协调器节点，并执行 `citus_add_node` 函数。
```bash
psql -p 5432
```
在 `psql` 提示符中执行：
```sql
-- 添加第一个工作节点
SELECT * from citus_add_node('localhost', 5433);

-- 添加第二个工作节点
SELECT * from citus_add_node('localhost', 5434);
```

**3. 验证集群状态**
在协调器上，你可以查看 `citus_nodes` 视图来确认所有节点都已成功加入集群。
```sql
SELECT * FROM citus_nodes;
```
你应该能看到一个协调器节点和两个状态为 `active` 的工作节点。

---

### 第四步：创建第一个分布式表

集群已经就绪，现在我们可以创建并分发一个表了。我们将使用一个简化的电商 `products` 表作为例子。

**1. 在协调器上创建表**
```sql
-- 仍然连接在协调器的 psql (端口 5432)
CREATE TABLE products (
    product_id BIGINT NOT NULL,
    name TEXT,
    category TEXT,
    PRIMARY KEY (product_id)
);
```
此时，`products` 表只存在于协调器节点上，它是一个普通的本地表。

**2. 将表分布式**
使用 `create_distributed_table()` 函数，指定表名和分布列。我们将使用 `product_id` 作为分布列。
```sql
SELECT create_distributed_table('products', 'product_id');
```
执行此命令后，Citus 会：
-   在协调器上将 `products` 表转换成一个分布式表的元数据入口。
-   自动连接到所有工作节点，并在每个节点上创建表的**分片（Shards）**。
-   将数据插入的逻辑路由到工作节点。

**3. 插入数据并验证**
```sql
-- 插入数据
INSERT INTO products (product_id, name, category) VALUES
(1, 'Laptop', 'Electronics'),
(2, 'T-Shirt', 'Apparel'),
(3, 'Book', 'Books'),
(4, 'Keyboard', 'Electronics');
```
数据现在已经被自动地、透明地存储到了工作节点的分片上。你无法直接从协调器上看到物理数据文件，但可以通过查询 `citus_shards` 视图来查看分片的分布情况。

---

## 📌 小结

恭喜你，你已经成功搭建并运行了一个本地的 Citus 分布式数据库集群！
-   我们学会了如何**初始化和配置**多个 PostgreSQL 实例来模拟集群。
-   掌握了使用 `citus_add_node` 函数将工作节点**注册**到协调器。
-   掌握了使用 `create_distributed_table` 函数将一个普通表**转换**为分布式表的核心操作。

虽然本地模拟集群简化了网络配置，但其核心步骤与在生产环境中部署多台物理服务器是完全一致的。在下一节，我们将深入探讨 Citus 的分片策略。
