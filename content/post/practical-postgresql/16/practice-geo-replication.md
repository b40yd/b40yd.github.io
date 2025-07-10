+++
title = "第十六章 多主复制与读写分离 - 第三节 实战：跨地域部署下的数据同步方案"
date = 2025-07-10
lastmod = 2025-07-10T01:00:00+08:00
tags = ["PostgreSQL", "practical", "book", "replication", "logical", "geo-distributed"]
categories = ["PostgreSQL", "practical", "book"]
draft = false
author = "b40yd"
+++

## 第三节 实战：跨地域部署下的数据同步方案

> **目标**：利用 PostgreSQL 内置的逻辑复制功能，设计并实现一个跨地域的数据分发和汇总系统，以解决全球化应用中的数据就近访问和中央分析的需求。

### 场景描述

我们是一家全球性的在线教育公司，在世界各地（如 `us-east-1`, `eu-west-1`, `ap-southeast-1`）都部署了应用服务器和数据库。

**业务需求：**
1.  **课程目录的全球分发**：公司的核心课程目录（`courses` 表）由总部的运营团队维护。这张表需要被快速、可靠地复制到所有区域的数据库中，以确保全球用户都能看到最新的课程列表。
2.  **区域用户数据的中央汇总**：每个区域的数据库都有一个 `regional_users` 表，记录该区域注册的用户。我们需要将这些分散的用户数据，实时地汇总到一个位于总部的中央数据仓库（`central_dw`）中，以进行统一的用户行为分析。

这个场景完美地展示了逻辑复制的两个核心优势：**一对多分发**和**多对一汇总**。

---

## 🏛️ 第一步：架构设计与准备

我们将模拟一个包含三个节点的架构：
1.  **总部主库 (HQ-Master)**：端口 `5432`。包含 `courses` 表，并作为发布者。
2.  **美国区域库 (US-Region)**：端口 `5433`。订阅总部的 `courses` 表，并发布自己的 `regional_users` 表。
3.  **欧洲区域库 (EU-Region)**：端口 `5434`。订阅总部的 `courses` 表，并发布自己的 `regional_users` 表。
4.  **中央数据仓库 (Central-DW)**：端口 `5435`。订阅所有区域库的 `regional_users` 表。

*(为简化，所有节点都在本地模拟。生产环境中它们将位于不同地理位置的服务器上)*

**准备工作：**
-   确保所有节点的 `postgresql.conf` 中 `wal_level = logical`。
-   确保所有节点的 `pg_hba.conf` 允许相互之间的复制连接。

---

## 🚀 第二步：实现课程目录的“一对多”分发

#### 1. 在总部主库 (HQ) 上创建 `courses` 表和发布

```sql
-- 连接到 HQ-Master (p5432)
CREATE TABLE courses (
    course_id INT PRIMARY KEY,
    title TEXT NOT NULL,
    last_updated TIMESTAMPTZ
);

-- 创建一个针对 courses 表的发布
CREATE PUBLICATION courses_pub FOR TABLE courses;
```

#### 2. 在区域库 (US & EU) 上创建订阅

```sql
-- 连接到 US-Region (p5433)
-- 先创建一张结构相同的表
CREATE TABLE courses (
    course_id INT PRIMARY KEY,
    title TEXT NOT NULL,
    last_updated TIMESTAMPTZ
);

-- 创建订阅，连接到总部的发布
CREATE SUBSCRIPTION courses_sub
    CONNECTION 'host=localhost port=5432 user=your_user password=your_pass dbname=your_db'
    PUBLICATION courses_pub;

-- 在 EU-Region (p5434) 上重复以上步骤
```

#### 3. 验证分发

现在，在总部的 `courses` 表中进行任何 `INSERT`, `UPDATE`, `DELETE` 操作，变更都会在几毫秒内被自动复制到美国和欧洲的区域数据库中。

```sql
-- 在 HQ-Master (p5432) 上操作
INSERT INTO courses VALUES (101, 'Advanced PostgreSQL', NOW());

-- 在 US-Region (p5433) 上查询
SELECT * FROM courses WHERE course_id = 101; -- 你将能看到数据
```

---

## 🔄 第三步：实现用户数据的“多对一”汇总

#### 1. 在每个区域库 (US & EU) 上创建 `regional_users` 表和发布

```sql
-- 连接到 US-Region (p5433)
CREATE TABLE regional_users (
    user_id BIGSERIAL PRIMARY KEY,
    email TEXT NOT NULL,
    region TEXT DEFAULT 'US' -- 标记数据来源
);
CREATE PUBLICATION us_users_pub FOR TABLE regional_users;

-- 连接到 EU-Region (p5434)
CREATE TABLE regional_users (
    user_id BIGSERIAL PRIMARY KEY,
    email TEXT NOT NULL,
    region TEXT DEFAULT 'EU'
);
CREATE PUBLICATION eu_users_pub FOR TABLE regional_users;
```

#### 2. 在中央数据仓库 (DW) 上创建目标表和订阅

```sql
-- 连接到 Central-DW (p5435)
-- 目标表结构需要兼容所有源表
CREATE TABLE all_users (
    user_id BIGINT, -- 注意：不能是 SERIAL，因为值是从源头复制过来的
    email TEXT NOT NULL,
    region TEXT,
    -- 添加一个主键，但不能是 user_id，因为它在不同区域可能重复
    dw_id BIGSERIAL PRIMARY KEY
);

-- 创建对美国区域的订阅
CREATE SUBSCRIPTION us_users_sub
    CONNECTION 'host=localhost port=5433 ...'
    PUBLICATION us_users_pub;

-- 创建对欧洲区域的订阅
CREATE SUBSCRIPTION eu_users_sub
    CONNECTION 'host=localhost port=5434 ...'
    PUBLICATION eu_users_pub;
```

#### 3. 验证汇总

现在，在美国或欧洲区域库的 `regional_users` 表中注册新用户，这些新用户记录会立即出现在中央数据仓库的 `all_users` 表中，并带有正确的 `region` 标识。

```sql
-- 在 EU-Region (p5434) 上操作
INSERT INTO regional_users (email) VALUES ('helga@example.de');

-- 在 Central-DW (p5435) 上查询
SELECT * FROM all_users WHERE email = 'helga@example.de';
-- 你将能看到来自欧洲的新用户记录
```

---

## 📌 小结

本实战通过一个具体的业务场景，展示了逻辑复制的强大灵活性：
1.  **一对多分发**：轻松地将中央的“权威”数据（如配置表、目录表）分发到全球各地的边缘节点，保证了数据的一致性。
2.  **多对一汇总**：无缝地将来自多个源头的数据流，实时地聚合到一个中央数据库中，为数据分析和商业智能提供了强大的数据基础。

与物理复制的“全有或全无”相比，逻辑复制提供了一种**精细化、可编程**的数据流动控制能力。它不仅仅是一种高可用或扩展技术，更是一种强大的**数据集成工具**，能够帮助我们构建出适应复杂业务需求的、地理分布式的、松耦合的现代数据架构。
