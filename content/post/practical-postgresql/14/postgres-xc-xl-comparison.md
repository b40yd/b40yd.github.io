+++
title = "第十四章 PostgreSQL 的分布式方案概览 - 第二节：Postgres-XC / Postgres-XL 对比"
date = 2025-07-10
lastmod = 2025-07-10T01:00:00+08:00
tags = ["PostgreSQL", "practical", "book", "distributed", "citus", "postgres-xl"]
categories = ["PostgreSQL", "practical", "book"]
draft = false
author = "b40yd"
+++

## 第二节 Postgres-XC / Postgres-XL 对比

> **目标**：了解 Citus 之外的其他 PostgreSQL 分布式探索，重点理解 Postgres-XC/XL 的架构思想，并将其与 Citus 进行对比，从而明白为什么 Citus 的“扩展”模式在当今更胜一筹。

在 Citus 成为主流之前，PostgreSQL 社区对分布式进行了多种探索。其中，**Postgres-XC** 和其后续演进版本 **Postgres-XL** 是非常重要的先行者。它们与 Citus 采用了截然不同的架构理念，了解其差异有助于我们更深刻地理解分布式数据库的设计哲学。

---

### Postgres-XC/XL 的核心架构

与 Citus 不同，Postgres-XC/XL **不是 PostgreSQL 的扩展，而是 PostgreSQL 的一个分支（Fork）**。这意味着它们拥有自己独立的、经过深度修改的源码。

其架构通常包含三个核心组件：
1.  **GTM (Global Transaction Manager)**：全局事务管理器。这是一个独立的、中心化的组件，负责管理整个集群的事务一致性和快照隔离（MVCC）。它是保证集群ACID特性的关键，但同时也可能成为性能瓶颈和单点故障。
2.  **Coordinators (协调器)**：与 Citus 类似，负责接收、解析和分发查询。
3.  **Datanodes (数据节点)**：与 Citus 类似，负责存储数据的分片。

Postgres-XC/XL 的设计目标是提供一个**完全透明**的、**强一致性**的分布式 PostgreSQL 体验，力求让用户感觉就像在使用一个无限大的单机 PostgreSQL。为了实现这个目标，它采用了复杂的两阶段提交（2PC）协议，并由 GTM 进行全局协调。

---

### 对比：Citus vs. Postgres-XL

| 特性 | Citus | Postgres-XL | 优势方 |
| :--- | :--- | :--- | :--- |
| **架构模型** | **PostgreSQL 扩展** | **PostgreSQL 分支 (Fork)** | **Citus** |
| **版本兼容性** | **紧跟 PostgreSQL 主版本**，可以快速支持最新特性。 | **严重滞后**。由于代码库是分支，每次合并主线更新都非常困难。 | **Citus** |
| **易用性** | 可以从单机 PostgreSQL + Citus 扩展开始，**平滑地扩展**为集群。 | 必须从一开始就部署一个包含 GTM、协调器和数据节点的**完整集群**。 | **Citus** |
| **事务模型** | 针对特定场景优化，对分布式事务有一定限制，但更具扩展性。 | 提供**全局强一致性**的事务，但 GTM 易成为瓶颈。 | 各有侧重 |
| **生态系统** | **完全兼容**所有 PostgreSQL 工具和扩展。 | 可能与某些工具或扩展不兼容。 | **Citus** |
| **社区与维护** | **非常活跃**，由微软团队和开源社区共同维护。 | 发展已基本停滞。 | **Citus** |

---

### 为什么“扩展”模式最终胜出？

Postgres-XL 的理念非常宏大，但其“分支”模式带来了致命的弱点。

1.  **“追赶”的困境**：PostgreSQL 自身在以极快的速度发展，每年都会发布一个包含大量新特性和性能优化的主版本。作为一个分支，Postgres-XL 需要投入巨大的人力才能将这些变更合并到自己的代码中，这导致它永远落后于主流的 PostgreSQL 版本。用户无法及时享受到最新的功能，例如 `JSONB` 的改进、`jsonpath`、逻辑复制的增强等。

2.  **生态的隔阂**：Citus 作为一个扩展，可以与 `pgvector`, `TimescaleDB` 等成千上万个其他 PostgreSQL 扩展无缝协作。而 Postgres-XL 则可能因为其内部代码的修改而与其他扩展不兼容。

3.  **运维的复杂性**：GTM 组件的引入，增加了系统的复杂度和潜在的故障点。而 Citus 的架构更简洁，协调器本身就是一个标准的 PostgreSQL 节点，运维起来更符合 DBA 的习惯。

Citus 的成功在于它的**务实和专注**。它没有试图完美地模拟一个单机 PostgreSQL 的所有行为，而是聚焦于解决多租户和实时分析这两个核心场景，并为此提供了极致的性能和可扩展性。更重要的是，它选择了一条与 PostgreSQL 主线共同成长的“扩展”之路，而不是与之分道扬镳的“分支”之路。

---

## 📌 小结

-   Postgres-XC/XL 是值得尊敬的分布式先行者，它们验证了在 PostgreSQL 上实现分布式集群的可行性。
-   然而，其**分支（Fork）架构**和**中心化的 GTM** 设计，在版本迭代速度和生态兼容性上存在天然的缺陷。
-   Citus 的**扩展（Extension）架构**被证明是更成功、更可持续的模式。它让用户无需离开标准 PostgreSQL 的世界，就能享受到分布式带来的好处。

因此，对于所有新项目而言，**Citus 是当前在 PostgreSQL 生态中构建分布式系统的、无可争议的首选方案**。
