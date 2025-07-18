+++
title = "PostgreSQL 数据库实战指南 - 安装 PostgreSQL 17（Linux/Windows/Docker）"
date = 2025-07-04
lastmod = 2025-07-04T01:01:00+08:00
tags = ["PostgreSQL", "practical", "book", "install", "Linux", "Windows", "Docker"]
categories = ["PostgreSQL", "practical", "book", "install", "Linux", "Windows", "Docker"]
draft = false
author = "b40yd"
+++

# 第一章 PostgreSQL 基础与安装配置
## 第一节 安装 PostgreSQL 17（Linux / Windows / Docker）

> **目标**：掌握在不同操作系统平台下安装 PostgreSQL 17 的方法，并完成基础环境配置，为后续学习和实战打下坚实基础。

PostgreSQL 是一个开源的、功能强大的关系型数据库管理系统。随着其生态系统的不断扩展，PostgreSQL 已成为企业级数据架构中的重要组成部分。本节将详细介绍如何在 Linux、Windows 和 Docker 环境中安装 PostgreSQL 17，并简要介绍安装后的基本配置流程。

---

## 🐧 一、在 Linux 上安装 PostgreSQL 17

目前主流 Linux 发行版（如 Ubuntu、CentOS、Debian）都提供了官方或第三方源支持 PostgreSQL 安装。

### 1. Ubuntu 22.04 LTS 安装步骤

```bash
# 添加 PostgreSQL 官方仓库
sudo sh -c 'echo "deb https://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list'

# 导入仓库签名密钥
wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add -

# 更新软件包列表并安装 PostgreSQL 17
sudo apt update
sudo apt install postgresql-17

# 启动服务并设置开机自启
sudo systemctl start postgresql
sudo systemctl enable postgresql
```

### 2. CentOS Stream 9 安装步骤

```bash
# 下载并安装 PostgreSQL 官方仓库 RPM 包
dnf install -y https://download.postgresql.org/pub/repos/yum/reporpms/EL-9-x86_64/pgdg-redhat-repo-latest.noarch.rpm

# 禁用内置模块并启用 PostgreSQL 17 模块
dnf -qy module disable postgresql
dnf install -y postgresql17-server

# 初始化数据库并启动服务
/usr/pgsql-17/bin/postgresql-17-setup initdb
systemctl enable postgresql-17
systemctl start postgresql-17
```

### ✅ 安装验证

```bash
# 查看服务状态
systemctl status postgresql

# 切换到 postgres 用户并连接数据库
sudo -i -u postgres
psql -c "SELECT version();"
```

---

## 💻 二、在 Windows 上安装 PostgreSQL 17

PostgreSQL 提供了适用于 Windows 的图形化安装程序，推荐使用 [EnterpriseDB](https://www.enterprisedb.com/downloads/postgres-postgresql-downloads) 提供的安装包。

### 安装步骤：

1. 访问 [PostgreSQL 官方下载页面](https://www.postgresql.org/download/windows/)，选择 “Interactive installer by EnterpriseDB”。
2. 下载安装包并运行：
   - 选择安装路径（建议默认）
   - 设置 `postgres` 超级用户密码
   - 选择组件（建议全选）
   - 设置端口号（默认 5432）
   - 启动服务并注册为 Windows 服务

### ✅ 安装验证

1. 打开 `SQL Shell (psql)`，输入以下命令查看版本信息：

```sql
SELECT version();
```

2. 使用 `pgAdmin 4` 图形界面工具管理数据库。

---

## 🐳 三、使用 Docker 安装 PostgreSQL 17

Docker 是快速部署 PostgreSQL 的理想方式，特别适合开发测试环境。

### 启动 PostgreSQL 17 容器：

```bash
docker run -d \
  --name postgres17 \
  -e POSTGRES_USER=admin \
  -e POSTGRES_PASSWORD=secret \
  -e POSTGRES_DB=mydb \
  -p 5432:5432 \
  postgres:17
```

### 参数说明：

| 参数 | 说明 |
|------|------|
| `-d` | 后台运行容器 |
| `--name` | 容器名称 |
| `-e POSTGRES_USER` | 设置管理员用户名 |
| `-e POSTGRES_PASSWORD` | 设置管理员密码 |
| `-e POSTGRES_DB` | 创建默认数据库 |
| `-p` | 映射宿主机端口 |

### ✅ 安装验证

进入容器内部执行查询：

```bash
docker exec -it postgres17 psql -U admin -d mydb
```

在 psql 中输入：

```sql
SELECT version();
```

---

## ⚙️ 四、安装后基础配置建议

无论你使用哪种方式安装 PostgreSQL，都建议进行以下基础配置以确保安全性和可用性：

### 1. 修改监听地址（允许远程访问）

编辑配置文件（通常位于 `/etc/postgresql/17/main/postgresql.conf` 或类似路径）：

```conf
listen_addresses = '*'
```

### 2. 配置远程连接权限

编辑 `pg_hba.conf` 文件，添加如下规则以允许特定 IP 连接：

```conf
host    all             all             192.168.1.0/24        md5
```

重启 PostgreSQL 服务使配置生效：

```bash
sudo systemctl restart postgresql
```

### 3. 创建新用户与数据库（可选）

```bash
sudo -i -u postgres
createuser --interactive
createdb myproject
```

---

## 📌 小结

| 平台 | 优点 | 推荐场景 |
|------|------|----------|
| Linux | 灵活、可控性强 | 生产环境、服务器部署 |
| Windows | 图形界面友好 | 开发者本地调试 |
| Docker | 快速部署、隔离性好 | 测试环境、CI/CD 流程 |

通过本节的学习，你应该已经能够在多种平台上成功安装 PostgreSQL 17，并完成了基础配置。接下来我们将进一步了解 PostgreSQL 的核心工具 `psql` 以及图形化客户端的使用。
