+++
title = "第十九章 高可用与故障转移 - 第二节：Patroni + etcd 高可用集群部署"
date = 2025-07-10
lastmod = 2025-07-10T01:00:00+08:00
tags = ["PostgreSQL", "practical", "book", "high availability", "patroni", "etcd"]
categories = ["PostgreSQL", "practical", "book"]
draft = false
author = "b40yd"
+++

## 第二节 Patroni + etcd 高可用集群部署

> **目标**：理解手动管理故障转移的痛点，并学习如何使用业界标准的自动化高可用解决方案——Patroni，配合 etcd 分布式存储，来构建一个能够自动进行故障检测和主备切换的、生产级的 PostgreSQL 集群。

上一节我们手动搭建的流复制集群，虽然实现了数据备份，但并非真正“高可用”。因为当主节点宕机时，从“发现故障”到“手动提升新主”再到“通知应用切换连接”，整个过程需要人工干预，耗时漫长且容易出错。

为了实现**自动故障转移（Automatic Failover）**，我们需要一个“集群管理器”来扮演机器人 DBA 的角色。**Patroni** 就是目前最流行、最强大的 PostgreSQL 集群管理器。

---

### Patroni 架构

Patroni 并非一个大一统的程序，而是一个由多个成熟组件协同工作的模板化解决方案。

**核心组件：**
1.  **Patroni**: 这是一个用 Python 编写的守护进程，它运行在**每一个** PostgreSQL 节点上。它负责：
    -   监控本地 PostgreSQL 实例的健康状况。
    -   管理本地 PostgreSQL 的配置文件（`postgresql.conf`）。
    -   与其他节点的 Patroni 进程通信。
    -   执行主节点选举（Leader Election）和故障转移（Failover）操作。

2.  **分布式配置存储 (DCS)**: 这是集群的“真理之源”和“公告板”。所有节点都通过它来共享集群状态。Patroni 支持多种 DCS，最常用的是：
    -   **etcd**: 一个高可用的键值存储，由 CoreOS 开发，非常可靠。
    -   **Consul**: 由 HashiCorp 开发的服务发现和配置工具。
    -   **ZooKeeper**: Apache 的经典分布式协调服务。

3.  **HAProxy (可选，但推荐)**: 这是一个高性能的负载均衡器。我们可以配置 HAProxy 来持续检查哪个节点是主节点，并自动地将应用的写请求路由到当前的主节点，将读请求分发到所有副本节点。这解决了应用连接切换的问题。

![Patroni Architecture](https://patroni.readthedocs.io/en/latest/_images/patroni_conns.png)
*图片来源: Patroni 官方文档*

---

### 工作流程

1.  **启动**：所有节点的 Patroni 进程启动后，会去 DCS 中尝试获取一个“主节点锁”。第一个获取到锁的节点成为主节点（Leader）。
2.  **成为主节点**：获取到锁的 Patroni 进程会立即启动并配置本地的 PostgreSQL 实例作为主节点。它会定期地向 DCS 更新自己的状态，以维持这个“锁”（心跳）。
3.  **成为副本节点**：其他没有获取到锁的节点，会从 DCS 中读取当前主节点的信息，然后配置本地的 PostgreSQL 实例作为副本，并从主节点开始流复制。
4.  **健康检查**：每个 Patroni 进程都在持续监控自己的 PostgreSQL 实例。主节点的 Patroni 也在监控所有副本的复制延迟。
5.  **故障转移**：
    -   如果主节点的 Patroni 进程或其 PostgreSQL 实例宕机，它将无法再更新 DCS 中的“锁”。
    -   “锁”会过期并被释放。
    -   所有副本节点的 Patroni 会立即感知到锁被释放，并开始一场新的**主节点选举**。
    -   复制延迟最低、最健康的那个副本节点会赢得选举，获取新的“锁”，并立即将自己提升（Promote）为新的主节点。
    -   其他副本节点会从 DCS 中发现主节点已变更，并自动地切换到从新的主节点进行复制。
    -   HAProxy 检测到主节点变更后，会自动更新其路由表，将新的写请求导向新的主节点。

整个过程完全自动化，通常在 10-30 秒内完成。

---

### 部署示例 (简要)

部署一个完整的 Patroni 集群涉及较多配置，这里给出一个简化的 `patroni.yml` 配置文件示例。每个节点上的这份文件内容基本相同。

```yaml
# patroni.yml
bootstrap:
  dcs:
    ttl: 30
    loop_wait: 10
    retry_timeout: 10
    # etcd 集群的配置
    etcd:
      hosts:
      - etcd1:2379
      - etcd2:2379
      - etcd3:2379
  # 初始化新集群时执行的脚本
  initdb:
  - auth-method: md5
    encoding: UTF8
    locale: en_US.UTF-8
  # 创建超级用户和复制用户
  pg_hba:
  - host all all 0.0.0.0/0 md5
  - host replication replicator 0.0.0.0/0 md5
  users:
    admin:
      password: 'strong_password'
      options: [createdb, createrole]
    replicator:
      password: 'repl_password'
      options: [replication]

# PostgreSQL 实例的配置
postgresql:
  listen: '0.0.0.0:5432'
  connect_address: '192.168.1.10:5432' # 当前节点的IP和端口
  data_dir: /data/postgresql
  # Patroni 会自动管理的参数
  parameters:
    max_connections: 100
    shared_buffers: 256MB
    wal_level: replica
    # ...

# REST API 配置，用于 Patroni 节点间通信和 patronictl 管理
restapi:
  listen: '0.0.0.0:8008'
  connect_address: '192.168.1.10:8008'
```

**启动集群：**
在每个节点上运行：
```bash
patroni /path/to/patroni.yml
```

**管理集群：**
使用 `patronictl` 命令行工具来查看和管理集群。
```bash
# 列出集群成员和状态
patronictl -c /path/to/patroni.yml list

# 手动触发一次故障转移
patronictl -c /path/to/patroni.yml switchover
```

---

## 📌 小结

-   Patroni 是一个**模板化**的、**与具体DCS解耦**的 PostgreSQL 高可用解决方案，它将运维专家的故障处理流程代码化。
-   它通过**分布式配置存储（如 etcd）**来维护集群的全局状态，并进行**主节点选举**。
-   它实现了**完全自动化的故障检测和转移**，极大地提升了数据库的可用性（Availability）和可靠性（Reliability）。
-   结合 **HAProxy** 等负载均衡器，可以为应用程序提供一个统一的、自动切换的数据库入口。

对于任何需要高可用的生产 PostgreSQL 环境，**Patroni + etcd/Consul + HAProxy** 的组合是目前业界公认的、最健壮、最可靠的黄金标准架构。
