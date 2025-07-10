+++
title = "第十九章 高可用与故障转移 - 第三节 实战：实现自动故障切换的生产级架构"
date = 2025-07-10
lastmod = 2025-07-10T01:00:00+08:00
tags = ["PostgreSQL", "practical", "book", "high availability", "patroni", "failover"]
categories = ["PostgreSQL", "practical", "book"]
draft = false
author = "b40yd"
+++

## 第三节 实战：实现自动故障切换的生产级架构

> **目标**：通过一个模拟实战，观察并理解 Patroni 集群在主节点发生故障时，是如何自动完成故障检测、领导者选举和主备切换的，并体会 HAProxy 如何配合实现对应用层的透明切换。

本实战将模拟一个由三个 PostgreSQL 节点组成的 Patroni 集群，并配合一个 HAProxy 实例作为流量入口。我们将手动“杀死”主节点，并观察整个系统的自愈过程。

**架构：**
-   `node1`, `node2`, `node3`: 运行 Patroni 和 PostgreSQL。
-   `etcd`: 分布式配置存储 (假设已部署好)。
-   `haproxy`: 负载均衡器，将流量导向活动的 PostgreSQL 主节点。

---

### 第一步：配置 HAProxy

HAProxy 的配置是实现对应用透明的关键。它会定期地向 Patroni 提供的 REST API 端点发送 HTTP 健康检查请求。

Patroni 的 REST API 端点（如 `http://node1:8008/primary`）在节点是主节点时会返回 `HTTP 200 OK`，否则会返回 `HTTP 503 Service Unavailable`。HAProxy 就利用这个机制来动态地识别当前的主节点。

**`haproxy.cfg` 配置文件示例：**
```cfg
global
    maxconn 2000

defaults
    mode tcp
    timeout connect 5s
    timeout client 1m
    timeout server 1m

listen stats
    bind *:7000
    mode http
    stats enable
    stats uri /

# 读写端口，永远指向主节点
listen postgres_readwrite
    bind *:5000
    mode tcp
    option httpchk
    # 向每个节点的 Patroni API 发送健康检查
    server pg_node1 node1:5432 check port 8008 inter 2s
    server pg_node2 node2:5432 check port 8008 inter 2s
    server pg_node3 node3:5432 check port 8008 inter 2s

# 只读端口，指向所有健康的节点（包括主节点和副本）
listen postgres_readonly
    bind *:5001
    mode tcp
    balance roundrobin # 轮询分发
    option httpchk
    # 健康检查端点是 /replica，只要节点健康就会返回 200
    http-check expect status 200
    server pg_node1 node1:5432 check port 8008 inter 2s
    server pg_node2 node2:5432 check port 8008 inter 2s
    server pg_node3 node3:5432 check port 8008 inter 2s
```
**应用连接方式：**
-   需要执行**写操作**的应用，连接到 `haproxy:5000`。
-   需要执行**读操作**的应用，连接到 `haproxy:5001`。

---

### 第二步：启动集群并检查初始状态

1.  启动 etcd 集群。
2.  在 `node1`, `node2`, `node3` 上分别启动 Patroni 进程。
3.  启动 HAProxy。

使用 `patronictl` 查看集群状态：
```bash
patronictl -c patroni.yml list
+ Cluster: my_cluster (7049948234234234) ----+----+-----------+
| Member | Host  | Role   | State   | TL | Lag in MB |
+--------+-------+--------+---------+----+-----------+
| node1  | node1 | Leader | running |  1 |           |
| node2  | node2 | Replica| running |  1 |         0 |
| node3  | node3 | Replica| running |  1 |         0 |
+--------+-------+--------+---------+----+-----------+
```
可以看到，`node1` 当前是主节点（Leader）。此时，所有到 `haproxy:5000` 的连接都会被转发到 `node1`。

---

### 第三步：模拟主节点故障

我们直接在 `node1` 上停止 Patroni 和 PostgreSQL 服务，以模拟一次突然的宕机。

```bash
# 在 node1 服务器上
sudo systemctl stop patroni
sudo systemctl stop postgresql
```

---

### 第四步：观察自动故障转移

现在，我们来观察系统的反应。

**1. Patroni 的反应 (在另一个节点上观察)**
再次运行 `patronictl list`，你会在几秒钟内看到变化：
```bash
# 第一次观察 (node1 刚宕机)
patronictl -c patroni.yml list
+ Cluster: my_cluster (7049948234234234) ----+----+-----------+
| Member | Host  | Role   | State   | TL | Lag in MB |
+--------+-------+--------+---------+----+-----------+
| node1  | node1 | Leader | stopping|  1 |   unknown | # 状态变为 stopping
| node2  | node2 | Replica| running |  1 |         0 |
| node3  | node3 | Replica| running |  1 |         0 |
+--------+-------+--------+---------+----+-----------+

# 几秒后再次观察 (选举完成)
patronictl -c patroni.yml list
+ Cluster: my_cluster (7049948234234234) ----+----+-----------+
| Member | Host  | Role   | State   | TL | Lag in MB |
+--------+-------+--------+---------+----+-----------+
| node1  | node1 |        | failed  |    |   unknown | # node1 被标记为 failed
| node2  | node2 | Leader | running |  2 |           | # node2 被提升为新的 Leader
| node3  | node3 | Replica| running |  2 |         0 | # node3 开始从 node2 复制
+--------+-------+--------+---------+----+-----------+
```
-   `node1` 在 DCS 中的锁过期，被标记为 `failed`。
-   `node2` 和 `node3` 进行了新的领导者选举，`node2` 获胜。
-   `node2` 将自己提升为新的主节点，时间线（Timeline ID, `TL`）从 1 增加到了 2。
-   `node3` 自动检测到主节点变更，并开始从 `node2` 进行流复制。

**2. HAProxy 的反应**
在故障期间，HAProxy 对 `node1` 的健康检查会失败。当 `node2` 成为新的主节点后，`http://node2:8008/primary` 会开始返回 `HTTP 200`。HAProxy 会立即检测到这个变化，并将所有到 `*:5000` 的新连接自动地、无缝地转发到 `node2`。

**3. 应用的感受**
-   在 `node1` 宕机到 `node2` 被提升并被 HAProxy 识别的这段时间（通常是 10-30 秒），应用到写端口 `5000` 的连接会失败或被挂起。
-   一旦故障转移完成，应用无需任何代码修改或配置变更，新的连接就会自动成功，写服务恢复。
-   读端口 `5001` 的服务中断时间更短，因为它只需要将 `node1` 从健康的后端池中移除即可。

---

## 📌 小结

本实战清晰地展示了一个生产级的 PostgreSQL 高可用架构是如何工作的。
-   **Patroni** 负责集群内部的**状态管理**和**故障转移决策**。
-   **DCS (etcd)** 负责提供一个**一致性的、可靠的平台**来存储集群状态和实现领导者选举。
-   **HAProxy** 负责**流量路由**，为应用程序提供一个**稳定的、自动切换的**数据库访问入口。

这三个组件的协同工作，将 PostgreSQL 从一个单点的、需要人工干预的数据库，转变为一个具备**自愈能力（Self-Healing）**、能够抵御节点故障的、真正意义上的高可用系统。这是现代关键业务系统数据库架构的基石。
