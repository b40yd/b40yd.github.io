+++
title = "第十七章 PostgreSQL + Kubernetes 实战 - 第三节 实战：云原生环境下 PostgreSQL 集群管理"
date = 2025-07-10
lastmod = 2025-07-10T01:00:00+08:00
tags = ["PostgreSQL", "practical", "book", "kubernetes", "operator", "cloud-native"]
categories = ["PostgreSQL", "practical", "book"]
draft = false
author = "b40yd"
+++

## 第三节 实战：云原生环境下 PostgreSQL 集群管理

> **目标**：通过一个端到端的实战演练，模拟在 Kubernetes 中管理一个高可用 PostgreSQL 集群的全过程，包括初始部署、水平扩容和自动故障转移演练，以体验云原生数据库的弹性和自愈能力。

本实战将带你走过一个典型的云原生 PostgreSQL 集群的生命周期，让你亲身感受 PostgreSQL Operator (PGO) 带来的自动化运维的便捷。

**先决条件：**
-   一个已安装 PGO 的 Kubernetes 集群。
-   `kubectl` 已配置。

---

### 阶段一：初始部署

我们首先部署一个基础的一主一备（`replicas: 1`）高可用集群。

**1. 定义集群 `hippo-cluster.yaml`**
```yaml
apiVersion: postgres-operator.crunchydata.com/v1beta1
kind: PostgresCluster
metadata:
  name: hippo-prod
  namespace: pgo
spec:
  postgresVersion: 15
  instances:
    - name: instance1
      replicas: 1 # 1个主节点, 1个副本节点
      dataVolumeClaimSpec:
        accessModes: ["ReadWriteOnce"]
        resources: { requests: { storage: "5Gi" } }
  backups:
    pgbackrest:
      repos:
      - name: repo1
        volume:
          volumeClaimSpec:
            accessModes: ["ReadWriteOnce"]
            resources: { requests: { storage: "5Gi" } }
```

**2. 部署集群**
```bash
kubectl apply -f hippo-cluster.yaml -n pgo
```

**3. 观察状态**
```bash
# 持续观察 Pod 的创建过程，直到都变为 Running
kubectl get pods -n pgo -w

# 查看集群的详细状态
kubectl get postgrescluster hippo-prod -n pgo -o yaml
```
在状态中，你会看到 `instances` 下有一个 `readyReplicas` 为 1，表示副本已就绪。同时，`services` 部分会显示主节点（primary）和副本（replica）的 Service 名称。

---

### 阶段二：水平扩容（Scale-Out）

随着业务发展，读取请求量大增，我们需要增加更多的只读副本来分担压力。

**1. 修改 YAML 文件**
将 `hippo-cluster.yaml` 中的 `replicas` 字段从 `1` 修改为 `3`。
```yaml
# ...
  instances:
    - name: instance1
      replicas: 3 # 从 1 改为 3
# ...
```

**2. 应用变更**
```bash
kubectl apply -f hippo-cluster.yaml -n pgo
```

**3. 验证扩容**
再次观察 Pod 状态：
```bash
kubectl get pods -n pgo -l postgres-operator.crunchydata.com/cluster=hippo-prod
```
你会看到两个新的 Pod (`hippo-prod-instance1-...`) 被创建出来。等待它们进入 `Running` 状态后，PGO 会自动将它们配置为流复制副本，并加入到 `hippo-prod-replica` 这个 Service 的后端端点中。整个扩容过程对应用层完全透明，读请求会被自动负载均衡到这些新的副本上。

---

### 阶段三：自动故障转移演练 (Auto Failover)

这是检验高可用性的关键。我们将手动删除当前的主节点 Pod，模拟一次节点宕机，并观察 PGO 如何自动进行故障转移。

**1. 找到当前的主节点**
PGO 会为主节点 Pod 添加一个特殊的标签 `postgres-operator.crunchydata.com/role=master`。
```bash
kubectl get pods -n pgo -l postgres-operator.crunchydata.com/cluster=hippo-prod,postgres-operator.crunchydata.com/role=master
```
记下这个 Pod 的名字，例如 `hippo-prod-instance1-abcde-0`。

**2. 模拟宕机：删除主节点 Pod**
```bash
kubectl delete pod hippo-prod-instance1-abcde-0 -n pgo
```

**3. 观察自愈过程**
立即用 `-w` 参数持续观察 Pod 和 Service 的变化。
```bash
# 观察 Pod 变化
kubectl get pods -n pgo -w

# 在另一个终端观察 Service 的端点变化
kubectl describe service hippo-prod-primary -n pgo
```

你会看到以下一系列由 PGO 自动完成的操作：
1.  K8s 的 `StatefulSet` 控制器检测到主节点 Pod 消失，会尝试重新创建一个同名的 Pod。
2.  **但在此之前**，PGO 的故障转移逻辑被触发。它会评估所有可用的副本节点，选择一个复制延迟最低、最健康的副本。
3.  PGO 将这个被选中的副本**提升（Promote）**为新的主节点。
4.  PGO 更新 `hippo-prod-primary` 这个 Service，将其 `Endpoints` 指向**新的主节点 Pod 的 IP 地址**。
5.  PGO 重新配置其他剩余的副本节点，让它们从**新的主节点**开始同步数据。
6.  当最初那个被删除的 Pod 被 K8s 重新创建后，PGO 会将其初始化为一个普通的副本节点，并加入集群。

整个过程通常在 **30-60秒** 内完成。在这期间，写服务会短暂中断，但一旦新的主节点被选出且 Service 更新完毕，写服务就会自动恢复。应用代码无需任何修改，因为它始终连接的是 `hippo-prod-primary` 这个稳定的 Service 地址。

---

## 📌 小结

本次实战演练清晰地展示了 PostgreSQL Operator 的核心价值：
-   **部署即高可用**：通过简单的 YAML 定义，我们就获得了一个具备高可用基础的集群。
-   **声明式扩容**：水平扩展从一个复杂的运维任务，简化为一次 `kubectl apply` 命令。
-   **自动化自愈**：面对主节点故障，Operator 能够像一位经验丰富的 DBA 一样，自动、快速、可靠地完成故障转移，恢复服务。

将 PostgreSQL 运行在 Kubernetes 上，并通过 Operator 进行管理，是现代云原生数据架构的最佳实践。它将关系型数据库的稳定可靠与云原生平台的弹性、自动化和可扩展性完美地结合在了一起。
