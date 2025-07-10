+++
title = "第十七章 PostgreSQL + Kubernetes 实战 - 第一节：使用 Crunchy Data Operator 部署"
date = 2025-07-10
lastmod = 2025-07-10T01:00:00+08:00
tags = ["PostgreSQL", "practical", "book", "kubernetes", "operator", "crunchy data"]
categories = ["PostgreSQL", "practical", "book"]
draft = false
author = "b40yd"
+++

## 第十七章 PostgreSQL + Kubernetes 实战
### 第一节 使用 Crunchy Data Operator 部署

> **目标**：理解在 Kubernetes (K8s) 中运行有状态应用（如 PostgreSQL）的挑战，并学习如何使用业界领先的 Crunchy Data PostgreSQL Operator 来声明式地、自动化地部署和管理 PostgreSQL 集群。

Kubernetes 已经成为容器编排的事实标准，它在管理无状态应用方面表现出色。然而，像 PostgreSQL 这样的**有状态应用（Stateful Applications）**，对数据的持久性、稳定性和一致性有极高要求，这给在 K8s 中运行它们带来了巨大挑战：
-   **持久化存储**：Pod 是“易逝”的，如何保证数据在 Pod 重启或漂移后不丢失？
-   **稳定的网络标识**：应用如何找到数据库的主节点？主节点地址变化后怎么办？
-   **复杂的状态管理**：如何处理主备切换、备份恢复、版本升级等复杂操作？

为了解决这些问题，社区引入了 **Operator 模式**。

---

### Operator 模式简介

Operator 是一种特殊的 K8s 控制器，它将**人类运维专家的知识**编码到软件中。它通过扩展 K8s API（使用 Custom Resource Definitions, CRDs）来创建、配置和管理特定应用的实例。

对于 PostgreSQL 来说，一个 Operator 就像一个 7x24 小时待命的机器人 DBA。你只需通过一个简单的 YAML 文件“告诉”Operator 你想要的 PostgreSQL 集群是什么样的（例如，“我想要一个包含1个主节点和2个副本的集群，版本为15.3”），Operator 就会自动完成所有繁琐的部署和管理工作。

**Crunchy Data PostgreSQL Operator (PGO)** 是目前社区最流行、功能最完备、最受认可的 PostgreSQL Operator 之一。

---

### 使用 PGO 部署 PostgreSQL 集群

我们将通过一个简单的步骤，在 K8s 集群中部署一个高可用的 PostgreSQL 集群。

**先决条件：**
-   一个正在运行的 Kubernetes 集群（如 Minikube, Kind, GKE, EKS 等）。
-   `kubectl` 命令行工具已配置好。

#### 第一步：安装 PGO

PGO 的安装非常简单，只需应用一个 YAML 文件即可。

```bash
# 创建一个命名空间来安装 Operator
kubectl create namespace pgo

# 从 Crunchy Data 的 GitHub 仓库安装最新版本的 PGO
kubectl apply -f https://raw.githubusercontent.com/CrunchyData/postgres-operator/v5.5.1/installers/kubectl/postgres-operator.yml
```
*(请从 PGO 官方文档获取最新的安装命令和版本)*

这个命令会在 `pgo` 命名空间中创建 PGO 的 Deployment、CRDs、RBAC 规则等所有必要的组件。

#### 第二步：定义 PostgreSQL 集群 (CRD)

安装完 Operator 后，我们就拥有了一个新的 K8s 资源类型：`PostgresCluster`。现在，我们创建一个名为 `postgres-cluster.yaml` 的文件来定义我们想要的集群。

```yaml
# postgres-cluster.yaml
apiVersion: postgres-operator.crunchydata.com/v1beta1
kind: PostgresCluster
metadata:
  name: hippo   # 集群名称
  namespace: pgo # 部署到的命名空间
spec:
  image: registry.developers.crunchydata.com/crunchydata/crunchy-postgres:ubi8-15.3-1 # 使用的镜像
  postgresVersion: 15 # PostgreSQL 主版本
  instances:
    - name: instance1 # 实例规格
      replicas: 2     # 创建2个副本 (总共1个主节点，2个副本)
      dataVolumeClaimSpec:
        accessModes:
        - "ReadWriteOnce"
        resources:
          requests:
            storage: 5Gi # 为每个实例申请 5Gi 的持久化存储
  backups:
    pgbackrest: # 使用 pgBackRest 进行备份
      image: registry.developers.crunchydata.com/crunchydata/crunchy-pgbackrest:ubi8-2.45-1
      repos:
      - name: repo1
        volume:
          volumeClaimSpec:
            accessModes:
            - "ReadWriteOnce"
            resources:
              requests:
                storage: 10Gi # 为备份仓库申请 10Gi 的存储
  users:
    - name: myuser
      databases:
      - mydatabase
      options: "SUPERUSER"
```

#### 第三步：创建集群

使用 `kubectl apply` 命令创建我们定义的 `PostgresCluster` 资源。
```bash
kubectl apply -f postgres-cluster.yaml -n pgo
```

**发生了什么？**
1.  PGO 检测到了这个新的 `PostgresCluster` 资源。
2.  它开始自动化地执行一系列操作：
    -   创建 `StatefulSet` 来管理带有持久化存储的 Pods。
    -   创建 `PersistentVolumeClaim` (PVC) 来申请存储。
    -   创建 `Services` 来提供稳定的网络入口（一个指向主节点，一个指向所有副本）。
    -   创建 `Secrets` 来安全地存储数据库用户的密码。
    -   配置 `pgBackRest` 用于备份。
    -   初始化主节点，并让副本节点通过流复制从主节点同步数据。

#### 第四步：连接到数据库

PGO 会自动创建一个 K8s Secret，其中包含了连接信息和密码。

```bash
# 获取 myuser 用户的密码
kubectl get secret hippo-pguser-myuser -n pgo --template='{{.data.password | base64decode}}'

# 通过端口转发，将本地端口 5432 映射到集群的主节点服务
kubectl port-forward svc/hippo-primary -n pgo 5432:5432
```
现在，你就可以在本地使用任何 PostgreSQL 客户端，通过 `localhost:5432` 连接到在 K8s 中运行的 PostgreSQL 集群了。

---

## 📌 小结

PostgreSQL Operator 彻底改变了在 Kubernetes 中运行数据库的方式。
-   **声明式管理**：你只需定义“最终状态”（我想要什么样的集群），Operator 会负责实现它。
-   **自动化运维**：Operator 将复杂的运维任务（如部署、高可用配置、备份）自动化，极大地降低了管理成本和人为错误的风险。
-   **云原生体验**：它让 PostgreSQL 像一个原生的 K8s 应用一样，可以被 `kubectl` 管理，并与 K8s 的生态（如监控、日志）无缝集成。

Crunchy Data PGO 是这一领域的黄金标准。通过使用它，你可以自信地、可靠地在 Kubernetes 生产环境中运行和管理你的 PostgreSQL 集群。在下一节，我们将探讨 PGO 提供的自动备份、恢复和扩缩容等高级功能。
