+++
title = "PostgreSQL 数据库实战指南"
lastmod = 2025-07-04T01:00:00+08:00
tags = ["PostgreSQL", "practical", "book"]
categories = ["PostgreSQL", "practical", "book"]
draft = false
author = "b40yd"
[menu.main]
  identifier = "PostgreSQL"
  weight = 12
  name = "PostgreSQL"
+++


## 📚书籍概述

PostgreSQL 不再只是一个传统的关系型数据库。随着插件生态的发展和原生特性的增强，它已经演变为一个支持多种数据模型（关系型、JSON、图、时序、空间）、多租户架构、分布式部署、流式处理等场景的通用数据库平台。

本书将围绕 PostgreSQL 最新版（v17）的功能体系，结合企业级实战案例，深入讲解其在不同数据库领域中的应用与优化技巧，帮助读者掌握 PostgreSQL 在现代数据架构中的核心地位。

---

## 🧩全书结构概览

| 部分 | 内容主题 | 章节数量 |
|------|----------|-----------|
| 第一部分 | 关系型数据库实战 | 6章 |
| 第二部分 | 图数据库支持与实战 | 3章 |
| 第三部分 | NoSQL 能力扩展实战 | 4章 |
| 第四部分 | 分布式数据库架构实战 | 5章 |
| 第五部分 | 高可用、安全与性能调优 | 4章 |

---

## 📘第一部分：关系型数据库实战（共6章）

### 第1章 PostgreSQL 基础与安装配置
- [安装 PostgreSQL 17（Linux/Windows/Docker）](/post/practical-postgresql/1.1-install)
- [初始化集群与基本配置（`pg_hba.conf`, `postgresql.conf`）](/post/practical-postgresql/1.2-init-cluster-config)
- [使用 `psql` 工具与图形化客户端（如 pgAdmin）](/post/practical-postgresql/1.3-admin)
- [用户权限管理基础](/post/practical-postgresql/1.4-user-control)
- [实战：搭建本地开发环境并导入样例数据集](/post/practical-postgresql/1.5-local-dev-data-sample)

### 第2章 SQL 语法与高级查询
- SELECT、JOIN、CTE、窗口函数详解
- 索引类型（B-tree, Hash, GiST, SP-GiST, BRIN, Bloom）
- 查询优化技巧与执行计划解读
- 实战：使用窗口函数分析销售数据趋势

### 第3章 事务控制与并发机制
- ACID 特性实现机制
- 多版本并发控制（MVCC）原理
- 锁机制与死锁检测
- 实战：高并发下单系统事务设计

### 第4章 函数、触发器与过程语言
- PL/pgSQL 编写存储过程
- 触发器的定义与使用场景
- 支持的其他语言：PL/Python、PL/Perl、PL/V8
- 实战：订单状态变更自动记录日志

### 第5章 表分区与继承
- 范围、列表、哈希分区策略
- 原生分区表 vs 继承表对比
- 自动分区策略设置
- 实战：按年份分区的历史数据归档系统

### 第6章 扩展与插件生态
- 常用扩展介绍（`uuid-ossp`, `hstore`, `citext`, `ltree`）
- 如何安装和使用扩展
- 实战：使用 `pg_trgm` 提升模糊搜索效率

---

## 📘第二部分：图数据库支持与实战（共3章）

### 第7章 PostgreSQL 中的图数据库能力
- 使用 `LTree` 实现树形结构
- `pgGraph` 插件简介（或使用 `Apache AGE`）
- 图遍历与路径查找
- 实战：组织架构图的构建与查询

### 第8章 使用 JSON + SQL 模拟图模型
- 使用 JSONB 存储节点与边关系
- 构建图结构并进行递归查询
- 实战：社交网络中好友推荐算法模拟

### 第9章 Apache AGE 集成与图数据库实战
- Apache AGE 简介与安装
- Cypher 查询语言支持
- 图数据库与关系数据混合查询
- 实战：知识图谱的构建与查询优化

---

## 📘第三部分：NoSQL 能力扩展实战（共4章）

### 第10章 JSON 与 JSONB 数据类型深度解析
- JSON vs JSONB 的区别
- GIN 索引与 JSONB 查询优化
- 支持的操作符与函数
- 实战：日志系统的灵活字段存储与检索

### 第11章 文档型数据库风格操作
- 使用 `jsonpath` 进行复杂文档查询
- 动态 Schema 设计与更新策略
- 实战：电商商品信息灵活字段管理

### 第12章 时间序列数据处理
- TimescaleDB 插件集成与使用
- hypertable 创建与压缩策略
- 实战：物联网设备监控数据存储与分析

### 第13章 全文搜索与向量相似度匹配
- `tsvector` 与 `GIN` 索引全文搜索
- 使用 `pgvector` 插件进行向量检索
- 实战：图像特征匹配与文本语义搜索

---

## 📘第四部分：分布式数据库架构实战（共5章）

### 第14章 PostgreSQL 的分布式方案概览
- Citus 扩展简介
- Postgres-XC / Postgres-XL 对比
- 实战：选择适合业务场景的分布式架构

### 第15章 Citus 扩展实战
- 安装 Citus 并创建分布式表
- 分片策略（hash, range, append）
- 分布式 JOIN 与聚合查询
- 实战：用户行为数据的分布式统计分析

### 第16章 多主复制与读写分离
- 使用 BDR（Bi-Directional Replication）
- 逻辑复制与物理复制对比
- 实战：跨地域部署下的数据同步方案

### 第17章 PostgreSQL + Kubernetes 实战
- 使用 Crunchy Data Operator 部署
- 自动备份、恢复、扩缩容
- 实战：云原生环境下 PostgreSQL 集群管理

### 第18章 多租户架构设计
- 行级安全策略（RLS）与 Row-Level Security
- 使用模式隔离或多租户扩展（如 TenantKit）
- 实战：SaaS 应用中的多租户数据隔离

---

## 📘第五部分：高可用、安全与性能调优（共4章）

### 第19章 高可用与故障转移
- 流复制（Streaming Replication）配置
- Patroni + etcd 高可用集群部署
- 实战：实现自动故障切换的生产级架构

### 第20章 安全机制与合规性
- SSL/TLS 加密连接
- 行级安全与列级权限控制
- 数据脱敏与审计日志
- 实战：金融系统中的访问控制与审计

### 第21章 性能调优实战
- 查询优化技巧（索引、重写、缓存）
- 配置参数调优（shared_buffers, work_mem, etc.）
- 实战：电商平台高峰期性能瓶颈排查

### 第22章 监控与运维自动化
- Prometheus + Grafana 监控 PostgreSQL
- 使用 `pgBadger` 分析慢查询日志
- 自动化巡检脚本编写
- 实战：建立完整的数据库健康检查体系
