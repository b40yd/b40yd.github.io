+++
title = "PostgreSQL实战指南：多领域数据库应用与实践 - 第24章：高可用性与故障切换"
date = 2025-07-12
lastmod = 2025-07-12T11:15:00+08:00
tags = ["PostgreSQL", "practical", "guide", "book", "dba", "administration", "high-availability", "failover", "patroni"]
categories = ["PostgreSQL", "practical", "guide", "book", "dba"]
draft = false
author = "b40yd"
+++

## PostgreSQL实战指南：多领域数据库应用与实践

-----

### 第六部分：PostgreSQL高级管理与优化

本部分聚焦于 PostgreSQL 的高级运维和管理，内容涵盖高可用架构、备份恢复、安全加固、性能诊断与调优，以及通过扩展开发来增强数据库功能。旨在帮助数据库管理员（DBA）和开发人员精通 PostgreSQL 的管理与维护，确保数据库系统的稳定、安全与高效。

-----

#### 第24章：高可用性与故障切换

对于任何生产系统而言，数据库的持续可用性都是至关重要的。高可用性（High Availability, HA）的目标是最大限度地减少因硬件故障、软件错误或计划内维护导致的停机时间。在 PostgreSQL 中，高可用性通常通过构建主从复制集群来实现，并配合自动故障切换机制。

##### 24.1 基于流复制的主从架构

PostgreSQL 内置的流复制（Streaming Replication）是构建高可用性的基础。

- **主节点 (Primary/Master)**: 接收所有写操作的数据库实例。
- **备节点 (Standby/Replica)**: 从主节点接收 WAL（预写日志）记录并应用的只读副本。备节点可以用于读取扩展（分担读请求），更重要的是，它可以在主节点发生故障时接管服务。

**复制模式:**

- **异步复制 (Asynchronous Replication)**: 主节点在将事务提交给客户端后，再将 WAL 记录发送给备节点。性能最高，但如果主节点在 WAL 发送前崩溃，可能会有少量数据丢失（RPO > 0）。这是默认和最常用的模式。
- **同步复制 (Synchronous Replication)**: 主节点必须等待至少一个备节点确认已接收并写入 WAL 记录后，才向客户端返回提交成功。这能保证数据零丢失（RPO = 0），但会增加写操作的延迟。

##### 24.2 故障切换 (Failover)

当主节点发生故障时，需要将一个备节点提升（promote）为新的主节点，这个过程称为故障切换。

- **手动切换**: DBA 确认主节点故障后，登录到选定的备节点，执行 `pg_promote()` 函数或 `pg_ctl promote` 命令，将其提升为新的主节点。然后需要更新其他备节点去跟随新的主节点，并通知应用程序连接到新的主节点。这个过程需要人工干预，恢复时间（RTO）较长。
- **自动切换**: 为了缩短恢复时间，需要一个外部工具来监控集群状态，并在检测到主节点故障时自动执行切换流程。

##### 24.3 使用 Patroni 实现自动故障切换

`Patroni` 是一个使用 Python 编写的、成熟的 PostgreSQL 高可用解决方案模板。它利用分布式共识存储（Distributed Consensus Store, DCS），如 Etcd、Consul 或 ZooKeeper，来管理集群状态，并自动处理故障切换。

**Patroni 架构:**

- **Patroni Daemon**: 在每个 PostgreSQL 节点上运行一个 Patroni 守护进程。
- **DCS (e.g., Etcd)**: 作为集群的“真理之源”。所有节点通过它来选举 Leader（主节点），并报告自己的状态。
- **HAProxy/PgBouncer**: 通常在 Patroni 集群前部署一个负载均衡器或连接池。Patroni 会在主节点变更后，自动更新负载均衡器的配置，将写请求路由到新的主节点，从而对应用程序实现透明切换。

**工作原理:**

1.  所有 Patroni 进程都尝试在 DCS 中获取一个 Leader 锁。
2.  成功获取锁的节点成为主节点，并定期更新锁的有效期。
3.  其他节点成为备节点，它们会持续监视 DCS 中的 Leader 锁。
4.  如果主节点因故未能更新锁（例如宕机或网络中断），锁会过期。
5.  其他备节点会观察到锁已过期，并开始新一轮的 Leader 选举。
6.  选举出的新 Leader 会将自己提升为新的主节点。
7.  Patroni 会自动触发 `pg_rewind` 来确保旧的主节点恢复后能作为新主节点的备库重新加入集群，防止“脑裂”（Split-Brain）。

##### 24.4 场景实战：部署一个简单的 Patroni 集群

**业务场景描述:**

为一个核心业务系统部署一个三节点的 PostgreSQL 高可用集群（一主两备），要求在主节点故障时能够自动切换，RTO 控制在30秒以内。

**环境准备 (概念性):**

- 三台服务器: `pg-node1`, `pg-node2`, `pg-node3`
- 一个 Etcd 集群
- 在每台服务器上安装 PostgreSQL 和 Patroni

**Patroni 配置文件示例 (`patroni.yml`):**

```yaml
# 每个节点的配置基本相同，只需修改 name 和 restapi.listen
scope: my_postgres_cluster  # 集群名称

bootstrap:
  dcs:
    postgresql:
      use_pg_rewind: true
  initdb:
  - auth-host: md5
  - auth-local: trust
  - encoding: UTF8
  - locale: en_US.UTF-8
  - data-checksums
  pg_hba:
  - host all all 0.0.0.0/0 md5
  - host replication replicator 127.0.0.1/32 md5

restapi:
  listen: 192.168.1.101:8008 # 当前节点的 IP 和端口
  connect_address: 192.168.1.101:8008

etcd:
  hosts: etcd1:2379,etcd2:2379,etcd3:2379 # Etcd 集群地址
  protocol: http

postgresql:
  listen: 0.0.0.0:5432
  connect_address: 192.168.1.101:5432
  data_dir: /var/lib/postgresql/14/main
  parameters:
    max_connections: 100
    shared_buffers: 256MB
  authentication:
    replication:
      username: replicator
      password: 'strong_password'
    superuser:
      username: postgres
      password: 'strong_password'
```

**启动与管理:**

1.  在所有三个节点上使用此配置启动 Patroni 服务。
2.  Patroni 会自动初始化集群，选举一个主节点，并让其他节点作为备库从主节点开始复制。
3.  使用 `patronictl` 命令行工具来管理和查看集群状态。

```bash
# 查看集群状态
patronictl -c patroni.yml list

# + Cluster: my_postgres_cluster (7041932013590211389) ----+----+-----------+
# | Member      | Host             | Role    | State   | TL | Lag in MB |
# +-------------+------------------+---------+---------+----+-----------+
# | pg-node1    | 192.168.1.101    | Leader  | running |  1 |           |
# | pg-node2    | 192.168.1.102    | Replica | running |  1 |         0 |
# | pg-node3    | 192.168.1.103    | Replica | running |  1 |         0 |
# +-------------+------------------+---------+---------+----+-----------+

# 模拟主节点故障 (在 pg-node1 上停止 patroni 和 postgresql)
# 再次查看状态，会发现 pg-node2 或 pg-node3 已经成为了新的 Leader
```

##### 24.5 总结

本章我们深入探讨了 PostgreSQL 的高可用性策略。我们从基础的流复制架构讲起，理解了同步与异步复制的区别，并认识到手动故障切换的局限性。我们重点学习了业界主流的自动故障切换解决方案 `Patroni`，了解了其基于 DCS 的工作原理和核心优势。通过一个简化的部署实例，我们看到了 `Patroni` 在自动化集群管理和实现快速故障恢复方面的强大能力。

在下一章，我们将讨论另一个与高可用性同样重要的话题：备份与恢复，学习如何制定可靠的策略来保护你的数据免于灾难。
-----
