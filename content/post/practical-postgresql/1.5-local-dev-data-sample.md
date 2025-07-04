+++
title = "PostgreSQL 数据库实战指南 - 实战：搭建本地开发环境并导入样例数据集"
lastmod = 2025-07-04T05:00:00+08:00
tags = ["PostgreSQL", "practical", "book", "data", "samples"]
categories = ["PostgreSQL", "practical", "book", "data", "samples"]
draft = false
author = "b40yd"
+++


# 第一章 PostgreSQL 基础与安装配置
## 第五节 实战：搭建本地开发环境并导入样例数据集

> **目标**：通过本节实战操作，掌握如何在本地环境中部署 PostgreSQL 开发环境，并成功导入一个常用的数据集（如 Northwind 或 IMDb），为后续 SQL 查询、性能优化和业务分析打下基础。

在前面的章节中，我们已经学习了 PostgreSQL 的安装、基本配置、用户权限管理等内容。现在我们将把这些知识综合运用起来，搭建一个完整的本地开发环境，并导入一个真实的数据集进行后续练习。

本节将涵盖以下内容：

- 安装 PostgreSQL 17 开发环境
- 创建专用数据库和用户
- 下载并准备样例数据集（以 Northwind 为例）
- 使用 `psql` 和图形工具导入数据
- 验证导入结果并执行简单查询

---

## 🧰 一、环境准备

### 1. 确保 PostgreSQL 已安装并运行

根据前几节的内容，确保你已经在本地环境中安装并启动了 PostgreSQL 17。

```bash
# 查看服务状态（Linux）
sudo systemctl status postgresql

# 连接数据库验证版本
psql -c "SELECT version();"
```

### 2. 安装辅助工具（可选）

建议安装以下工具以便更高效地处理数据：

- `curl` / `wget`：下载文件
- `unzip`：解压 ZIP 文件
- `pgAdmin 4`（可选）：用于图形化操作

---

## 🗃️ 二、创建开发用数据库与用户

为了保持环境整洁，我们创建一个专用数据库和用户来管理我们的样例数据。

### 1. 创建角色和数据库

```bash
sudo -i -u postgres
```

```sql
-- 创建用户
CREATE USER dev_user WITH LOGIN PASSWORD 'devpass';

-- 创建数据库
CREATE DATABASE northwind OWNER dev_user;

-- 授予连接权限
GRANT CONNECT ON DATABASE northwind TO dev_user;

-- 授予模式使用权限
\c northwind
GRANT USAGE ON SCHEMA public TO dev_user;
GRANT CREATE ON SCHEMA public TO dev_user;
```

### 2. 切换到新用户登录数据库

```bash
psql -U dev_user -d northwind
```

---

## 📥 三、下载并准备 Northwind 样例数据集

Northwind 是一个经典的示例数据库，最初由 Microsoft 提供，包含客户、订单、产品、供应商等常见业务实体。

### 1. 下载数据集

我们可以使用社区维护的 PostgreSQL 版本 Northwind 数据集。

```bash
cd ~/Downloads
curl -O https://raw.githubusercontent.com/pthom/northwind_psql/master/northwind.sql
```

该文件是一个完整的 `.sql` 脚本，包含了所有表结构和数据插入语句。

---

## 📤 四、导入数据集到 PostgreSQL

你可以使用命令行或图形工具导入数据。

### 方法 1：使用 `psql` 导入

```bash
psql -U dev_user -d northwind -f ~/Downloads/northwind.sql
```

如果提示权限问题，请切换回 `postgres` 用户执行：

```bash
sudo -i -u postgres
psql -U dev_user -d northwind -f ~/Downloads/northwind.sql
```

### 方法 2：使用 pgAdmin 4 导入

1. 打开 pgAdmin 4。
2. 展开连接 → 选择 `northwind` 数据库。
3. 右键点击数据库 → “Query Tool”。
4. 点击左上角 “Open File” 图标，加载 `northwind.sql` 文件。
5. 点击 ▶️ 执行按钮开始导入。

---

## 🔍 五、验证数据是否导入成功

### 1. 查看表结构

```sql
\d
```

你应该能看到类似如下表格：

```
public | categories       | table | dev_user
public | customers        | table | dev_user
public | employees        | table | dev_user
public | order_details    | table | dev_user
public | orders           | table | dev_user
public | products         | table | dev_user
...
```

### 2. 查询部分数据

```sql
-- 查看前 5 条订单记录
SELECT * FROM orders LIMIT 5;

-- 统计产品总数
SELECT COUNT(*) FROM products;

-- 查询某个客户的订单数量
SELECT c.company_name, COUNT(o.order_id) AS total_orders
FROM customers c
JOIN orders o ON c.customer_id = o.customer_id
GROUP BY c.company_name
ORDER BY total_orders DESC
LIMIT 10;
```

---

## 📊 六、扩展建议：尝试其他数据集（如 IMDb）

除了 Northwind，IMDb 数据集也是一个很好的练习对象，适合练习大数据量下的查询优化技巧。

### 1. 下载 IMDb 数据集

访问 [IMDb Dataset](https://www.imdb.com/interfaces/) 页面，下载以下文件之一：

- `title.basics.tsv.gz`
- `title.ratings.tsv.gz`
- `name.basics.tsv.gz`

### 2. 解压并转换为 CSV

```bash
gunzip title.basics.tsv.gz
mv title.basics.tsv title_basics.csv
```

### 3. 创建表并导入数据

```sql
CREATE TABLE title_basics (
    tconst TEXT PRIMARY KEY,
    title_type TEXT,
    primary_title TEXT,
    start_year INTEGER,
    end_year INTEGER,
    runtime_minutes INTEGER,
    genres TEXT
);

COPY title_basics FROM '/path/to/title_basics.csv' DELIMITER '\t' CSV HEADER;
```

> ⚠️ 注意：CSV 文件路径需为 PostgreSQL 可读路径，且运行 COPY 命令的用户需要有权限。

---

## 📌 小结

| 步骤 | 内容 |
|------|------|
| 安装 PostgreSQL | Linux/Windows/Docker 安装方式均可 |
| 创建开发用户和数据库 | 使用最小权限原则分配资源 |
| 获取数据集 | Northwind / IMDb 等开源数据集 |
| 导入数据 | 使用 `psql` 或 pgAdmin 4 执行脚本或导入文件 |
| 验证数据 | 使用 `\d` 查看结构，SQL 查询验证内容 |

通过本次实战，你已经成功搭建了一个本地 PostgreSQL 开发环境，并导入了经典数据集，具备了后续 SQL 查询、索引优化、视图创建等操作的基础条件。
