+++
title = "第六章 扩展与插件生态 - 第二节：如何安装和使用扩展"
date = 2025-07-10
lastmod = 2025-07-10T01:00:00+08:00
tags = ["PostgreSQL", "practical", "book", "extensions", "create extension"]
categories = ["PostgreSQL", "practical", "book"]
draft = false
author = "b40yd"
+++

## 第二节 如何安装和使用扩展

> **目标**：掌握在 PostgreSQL 中查找、安装、管理和卸载扩展的标准流程，为在项目中安全、高效地使用扩展打下坚实基础。

了解了扩展的强大功能后，下一步就是学习如何将它们集成到你的数据库中。PostgreSQL 提供了一套简单而标准的命令来管理扩展的整个生命周期。

---

### 扩展的组成部分

一个 PostgreSQL 扩展通常由以下部分组成：
- **SQL 文件**：一个或多个 `.sql` 文件，定义了扩展的函数、数据类型、操作符等。
- **控制文件**：一个 `.control` 文件，包含了扩展的元数据，如版本号、依赖关系等。
- **共享库**：一个或多个 `.so` (Linux/macOS) 或 `.dll` (Windows) 文件，包含了用 C 语言等编译型语言编写的底层代码。

---

## ⚙️ 第一步：系统层面的安装

在使用 `CREATE EXTENSION` 命令之前，必须确保扩展的文件（SQL、control、共享库）已经存在于 PostgreSQL 的文件系统中。这个过程通常通过系统的包管理器来完成。

### 对于 `contrib` 模块中的官方扩展

`contrib` 模块包含了像 `hstore`, `pg_trgm`, `ltree` 等许多常用扩展。安装它通常非常简单。

**Debian/Ubuntu:**
```bash
# 安装与你的 PostgreSQL 版本对应的 contrib 包
sudo apt-get install postgresql-contrib-17
```

**Red Hat/CentOS/Fedora:**
```bash
sudo dnf install postgresql17-contrib
```

### 对于第三方扩展

对于像 `pg_partman`, `pgvector`, `TimescaleDB` 这样的第三方扩展，你需要遵循它们各自的安装文档。这可能涉及添加新的软件源（repository）、从源码编译或使用特定的安装脚本。

---

## 🔎 第二步：查找可用的扩展

一旦系统层面安装完成，你就可以在 `psql` 中查看当前数据库实例所有可用的扩展。

```sql
SELECT * FROM pg_available_extensions;
```
或者使用 `psql` 的元命令 `\dx+`，它会显示更详细的信息，包括已安装的版本和文件路径。

---

## 🚀 第三步：在数据库中安装（启用）扩展

`CREATE EXTENSION` 命令负责在**当前数据库**中加载扩展的 SQL 对象，使其真正可用。这个操作需要超级用户权限，或者被授权的用户。

**基本语法：**
```sql
CREATE EXTENSION extension_name;
```

**推荐使用的安全写法：**
`IF NOT EXISTS` 可以防止在扩展已存在时报错，这在自动化部署脚本中非常有用。
```sql
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
```
**注意**：扩展是安装在特定数据库中的，而不是整个 PostgreSQL 实例。如果你有多个数据库需要使用同一个扩展，你必须在每一个数据库中都执行 `CREATE EXTENSION`。

---

## 📋 第四步：查看已安装的扩展

要列出当前数据库中所有已安装的扩展及其版本，可以使用 `psql` 的元命令 `\dx`。

```sql
\dx
```
**输出示例：**
```
                                     List of installed extensions
  Name   | Version |   Schema   |                            Description
---------+---------+------------+------------------------------------------------------------------
 citext  | 1.6     | public     | data type for case-insensitive character strings
 hstore  | 1.8     | public     | data type for storing sets of (key, value) pairs
 plpgsql | 1.0     | pg_catalog | PL/pgSQL procedural language
 uuid-ossp | 1.1   | public     | generate universally unique identifiers (UUIDs)
```

---

## ⬆️ 第五步：更新扩展

当 PostgreSQL 主版本升级或扩展自身发布新版本时，你可能需要更新已安装的扩展。

```sql
ALTER EXTENSION extension_name UPDATE;
```
或者指定一个特定版本：
```sql
ALTER EXTENSION extension_name UPDATE TO '2.0';
```
在进行 PostgreSQL 大版本升级（例如，从 16 升级到 17）之前，检查并更新所有已安装的扩展是一个非常重要的步骤。

---

## 🗑️ 第六步：卸载扩展

如果你不再需要某个扩展，可以使用 `DROP EXTENSION` 命令将其从数据库中移除。这将删除所有由该扩展创建的函数、类型等对象。

**警告：这是一个危险的操作！** 如果有任何数据库对象（如表中的列）正在使用该扩展提供的类型，命令将会失败，除非你使用 `CASCADE`。

**基本语法：**
```sql
DROP EXTENSION extension_name;
```

**级联删除（请极度谨慎使用）：**
`CASCADE` 会一并删除所有依赖于该扩展的对象。这可能导致数据丢失，使用前请务必确认后果。
```sql
-- 示例：如果一个表使用了 citext 类型，此命令会连带删除该表或列
DROP EXTENSION citext CASCADE;
```

---

## 📌 小结

| 命令 | 功能 |
| :--- | :--- |
| `apt-get install ...` | 在操作系统层面安装扩展文件 |
| `SELECT * FROM pg_available_extensions;` | 查看所有可用的扩展 |
| `CREATE EXTENSION ...;` | 在当前数据库中启用一个扩展 |
| `\dx` | 列出当前数据库中已安装的扩展 |
| `ALTER EXTENSION ... UPDATE;` | 更新一个已安装的扩展到新版本 |
| `DROP EXTENSION ...;` | 从当前数据库中卸载一个扩展 |

掌握这套标准流程，你就可以自信地利用 PostgreSQL 庞大的扩展生态，为你的应用添加各种强大的新功能了。
