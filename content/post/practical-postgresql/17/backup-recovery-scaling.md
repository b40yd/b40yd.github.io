+++
title = "第十七章 PostgreSQL + Kubernetes 实战 - 第二节：自动备份、恢复、扩缩容"
date = 2025-07-10
lastmod = 2025-07-10T01:00:00+08:00
tags = ["PostgreSQL", "practical", "book", "kubernetes", "operator", "backup", "scaling"]
categories = ["PostgreSQL", "practical", "book"]
draft = false
author = "b40yd"
+++

## 第二节 自动备份、恢复、扩缩容

> **目标**：学习如何利用 PostgreSQL Operator (PGO) 的声明式 API，轻松地执行备份、恢复和集群扩缩容等高级运维操作，体验云原生数据库管理的便捷性。

PGO 的强大之处不仅在于简化了初始部署，更在于它将复杂的 Day-2 运维操作（即部署后期的日常管理）也变成了简单的 YAML 修改。

---

### 一、备份 (Backups)

PGO 内部集成了业界领先的 PostgreSQL 备份和恢复工具——**pgBackRest**。它支持全量备份（Full）、增量备份（Incremental）、差异备份（Differential）以及时间点恢复（Point-in-Time Recovery, PITR）。

#### 1. 配置备份策略

在上一节的 `PostgresCluster` 定义中，我们已经包含了 `backups.pgbackrest` 部分。我们可以对其进行扩展，来定义备份计划。

```yaml
# ... (spec from previous section)
  backups:
    pgbackrest:
      # ... (repo configuration)
      global:
        repo1-retention-full: "3" # 保留最近3次全量备份
        repo1-retention-full-type: count
      manual:
        repoName: repo1
        options:
        - --type=full # 手动备份类型为全量
      jobs: # 定义定时备份任务
      - name: everyday-full-backup
        schedule: "0 1 * * *" # 每天凌晨1点执行
        repoName: repo1
        options:
        - --type=full
      - name: every-30min-incr-backup
        schedule: "*/30 * * * *" # 每30分钟执行一次增量备份
        repoName: repo1
        options:
        - --type=incr
```
只需 `kubectl apply` 这个更新后的 YAML，PGO 就会自动创建相应的 `CronJob` 来执行定时备份。

#### 2. 执行手动备份

有时我们需要在重大变更前执行一次手动备份。只需修改 `manual.repoName` 并 `apply` 即可触发。但更直接的方式是创建一个一次性的 `Job`。

或者，PGO v5.4+ 引入了更简单的注解方式：
```bash
kubectl annotate postgrescluster hippo -n pgo --overwrite \
  postgres-operator.crunchydata.com/pgbackrest-backup='{"repoName": "repo1", "options": ["--type=full"]}'
```
这个命令会立即触发一次全量备份。

---

### 二、恢复 (Restore)

恢复数据是 DBA 的核心职责，PGO 让这个过程也变得非常简单。

#### 1. 克隆一个新集群 (Clone)

最安全的恢复方式是**克隆**。即从一个现有集群的备份中，创建一个全新的、独立的集群。这对于搭建测试环境或验证备份有效性非常有用。

只需创建一个新的 `PostgresCluster` YAML，并在 `dataSource` 部分指定源集群和备份。

```yaml
# restore-cluster.yaml
apiVersion: postgres-operator.crunchydata.com/v1beta1
kind: PostgresCluster
metadata:
  name: hippo-restored
spec:
  # ... (postgresVersion, instances, etc.)
  dataSource:
    postgresCluster:
      clusterName: hippo # 源集群名称
      repoName: repo1    # 从哪个备份仓库恢复
```
`kubectl apply` 这个文件后，PGO 就会自动创建一个名为 `hippo-restored` 的新集群，其数据与 `hippo` 集群的最近一次备份完全一致。

#### 2. 时间点恢复 (Point-in-Time Recovery, PITR)

如果你需要将数据恢复到某个特定的时间点（例如，在误删数据之前），可以在 `dataSource` 中指定 `pitr` 选项。

```yaml
# ...
  dataSource:
    postgresCluster:
      clusterName: hippo
      repoName: repo1
      pitr:
        time: "2025-07-10 14:30:00+00" # 恢复到这个时间点
```

---

### 三、扩缩容 (Scaling)

随着业务量的变化，我们可能需要增加或减少数据库实例。

#### 1. 扩容副本 (Scaling Out Replicas)

这是最常见的操作，用于增强集群的读性能和高可用性。只需修改 `PostgresCluster` YAML 中的 `replicas` 字段即可。

**将副本从 2 个增加到 4 个：**
```yaml
# ...
  instances:
    - name: instance1
      replicas: 4 # 原来是 2
# ...
```
`kubectl apply` 后，PGO 会自动创建两个新的 Pod，配置流复制，并将其加入到只读副本服务中。整个过程对应用透明。

#### 2. 扩容存储 (Scaling Up Storage)

如果数据量增长导致磁盘空间不足，可以直接修改 `dataVolumeClaimSpec` 中的 `storage` 请求大小。

**将存储从 5Gi 增加到 10Gi：**
```yaml
# ...
      dataVolumeClaimSpec:
        resources:
          requests:
            storage: 10Gi # 原来是 5Gi
# ...
```
`kubectl apply` 后，PGO 会安全地扩展每个实例的 `PersistentVolume`。**注意**：这需要你的 K8s 存储类（`StorageClass`）支持在线扩容。

---

## 📌 小结

PostgreSQL Operator 将传统数据库运维中那些复杂、高风险、需要DBA手动执行的脚本化流程，转变成了**声明式的、自动化的、可版本控制的**云原生工作流。
-   **备份**：通过 `pgbackrest` 的集成和 `CronJob` 的自动化，实现了可靠、定时的备份。
-   **恢复**：通过 `dataSource` 声明，可以安全地将数据克隆到新集群或执行精准的时间点恢复。
-   **扩缩容**：无论是增加副本实例还是扩展存储空间，都只需简单地修改 YAML 文件并应用，PGO 会处理好所有底层的复杂逻辑。

这种将运维操作“代码化”的能力，是 DevOps 和 GitOps 理念在数据库管理领域的完美实践，它极大地提升了团队的敏捷性、可靠性和运维效率。
