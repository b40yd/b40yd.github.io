+++
title = "第二十章 安全机制与合规性 - 第一节：SSL/TLS 加密连接"
date = 2025-07-10
lastmod = 2025-07-10T01:00:00+08:00
tags = ["PostgreSQL", "practical", "book", "security", "ssl", "tls", "encryption"]
categories = ["PostgreSQL", "practical", "book"]
draft = false
author = "b40yd"
+++

## 第二十章 安全机制与合规性
### 第一节 SSL/TLS 加密连接

> **目标**：理解在客户端和服务器之间加密数据库连接的重要性，并掌握在 PostgreSQL 中配置 SSL/TLS 的完整流程，以防止数据在传输过程中被窃听或篡改。

默认情况下，PostgreSQL 客户端和服务器之间的通信是**未加密的**。这意味着，如果攻击者能够嗅探两者之间的网络流量，他们就可以直接读取到你发送的 SQL 查询和数据库返回的结果，包括用户名、密码和所有敏感的业务数据。

为了防止这种中间人攻击（Man-in-the-Middle, MitM），我们必须启用 **SSL/TLS (Secure Sockets Layer / Transport Layer Security)** 来对所有网络通信进行加密。

---

### SSL/TLS 的工作原理

启用 SSL/TLS 后，客户端和服务器在建立连接时会执行一次“握手”：
1.  服务器向客户端出示自己的**数字证书（Server Certificate）**。
2.  客户端验证该证书是否由一个可信的**证书颁发机构（Certificate Authority, CA）**签发。
3.  验证通过后，双方协商出一个对称加密密钥。
4.  此后所有的通信都使用这个密钥进行加密。

---

### 配置流程

我们将配置一个要求客户端必须使用 SSL/TLS 加密连接的 PostgreSQL 服务器。

**先决条件：**
-   PostgreSQL 服务器已经安装。
-   系统中已安装 OpenSSL 工具包。

#### 第一步：生成 SSL 证书文件

在生产环境中，你应该从一个公共的 CA（如 Let's Encrypt）获取证书。为了演示，我们将创建一个自签名的 CA 和服务器证书。

**1. 创建私有的证书颁发机构 (CA)**
```bash
# 创建 CA 的私钥
openssl genrsa -out ca.key 4096

# 创建 CA 的根证书 (ca.crt)
openssl req -new -x509 -days 3650 -key ca.key -out ca.crt -subj "/CN=MyTestCA"
```

**2. 创建服务器的私钥和证书签名请求 (CSR)**
```bash
# 创建服务器私钥
openssl genrsa -out server.key 4096

# 创建证书签名请求 (CSR)
# Common Name (CN) 必须是服务器的域名或 IP 地址
openssl req -new -key server.key -out server.csr -subj "/CN=db.example.com"
```

**3. 使用我们的 CA 签署服务器证书**
```bash
openssl x509 -req -days 365 -in server.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out server.crt
```
现在，我们得到了三个核心文件：`ca.crt`, `server.key`, `server.crt`。

**4. 设置文件权限**
私钥文件必须是严格保密的。
```bash
chmod 600 ca.key server.key
```

#### 第二步：在 PostgreSQL 服务器上配置 SSL

**1. 复制证书文件**
将 `ca.crt`, `server.key`, `server.crt` 复制到 PostgreSQL 的数据目录中（例如 `/var/lib/postgresql/15/main/`）。确保 `postgres` 系统用户有权限读取它们。

**2. 修改 `postgresql.conf`**
```ini
# postgresql.conf

# 1. 启用 SSL
ssl = on

# 2. 指定证书文件路径
ssl_cert_file = 'server.crt'
ssl_key_file = 'server.key'
ssl_ca_file = 'ca.crt'
```

**3. 修改 `pg_hba.conf` 以强制使用 SSL**
`pg_hba.conf` 提供了两种方式来控制 SSL 连接：
-   `hostssl`: **只允许** SSL 加密的连接。
-   `hostnossl`: **只允许**未加密的连接。

我们将所有非本地的连接都强制为 `hostssl`。
```conf
# pg_hba.conf

# TYPE      DATABASE        USER        ADDRESS         METHOD
# 只接受通过 SSL 加密的连接
hostssl     all             all         0.0.0.0/0       scram-sha-256

# 本地连接可以不加密 (可选)
host        all             all         127.0.0.1/32    scram-sha-256
```

**4. 重启 PostgreSQL 服务器**
```bash
sudo systemctl restart postgresql-15
```

---

### 第三步：客户端连接测试

现在，客户端（如 `psql` 或其他应用）连接时，必须提供正确的 SSL 参数。

**`psql` 连接字符串：**
```bash
psql "host=db.example.com dbname=mydb user=myuser \
      sslmode=verify-full \
      sslrootcert=path/to/ca.crt"
```

**`sslmode` 参数详解：**
这是客户端最重要的 SSL 配置，它决定了安全级别。
-   `disable`: 不使用 SSL。
-   `allow`: 优先不使用 SSL，如果服务器强制要求，则使用。
-   `prefer`: **(默认值)** 优先使用 SSL，如果服务器不支持，则退回非加密。
-   `require`: 必须使用 SSL，但不对服务器证书进行验证（容易受到中间人攻击）。
-   `verify-ca`: 必须使用 SSL，并验证服务器证书是由一个可信的 CA 签发的。
-   `verify-full`: **(最安全)** 除了 `verify-ca` 的所有检查外，还会验证服务器证书中的 `CN` (Common Name) 是否与你连接的主机名匹配。

**为了达到真正的安全，客户端应始终使用 `sslmode=verify-full`。**

如果尝试使用不加密的方式连接，服务器会根据 `pg_hba.conf` 的规则拒绝连接。
```bash
psql "host=db.example.com ... sslmode=disable"
# psql: error: server does not support SSL, but SSL was required
```

---

## 📌 小结

-   在任何生产环境中，**必须启用 SSL/TLS** 来加密 PostgreSQL 的网络通信，这是安全基线。
-   配置过程分为三步：**生成证书** -> **配置 `postgresql.conf`** -> **配置 `pg_hba.conf`**。
-   `pg_hba.conf` 中的 `hostssl` 记录是**强制客户端使用加密连接**的关键。
-   客户端必须使用 `sslmode=verify-full` 并提供 CA 根证书，才能实现对服务器身份的有效验证，防止中间人攻击。

通过正确配置 SSL/TLS，你可以确保你的数据在从应用到数据库的整个传输链路中都是机密和完整的，满足了大多数安全合规性（如 PCI-DSS, HIPAA）的基本要求。
