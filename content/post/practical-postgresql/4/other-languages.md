+++
title = "第四章 函数、触发器与过程语言 - 第三节：其他过程语言支持"
date = 2025-07-10
lastmod = 2025-07-10T01:00:00+08:00
tags = ["PostgreSQL", "practical", "book", "pl/python", "pl/v8", "pl/perl"]
categories = ["PostgreSQL", "practical", "book"]
draft = false
author = "b40yd"
+++

## 第三节 其他过程语言支持：PL/Python、PL/Perl、PL/V8

> **目标**：了解 PostgreSQL 对多种过程语言的支持，重点掌握 PL/Python 和 PL/V8 的使用方法，并学会在合适的场景中选择它们以发挥各自的优势。

虽然 PL/pgSQL 功能强大且性能优异，但 PostgreSQL 的可扩展性远不止于此。它允许用户使用自己熟悉的语言来编写数据库函数，极大地丰富了其在数据处理和分析领域的能力。本节将重点介绍几种最流行的替代过程语言。

---

### 为什么需要其他语言？

- **利用现有生态**：Python 拥有庞大的科学计算、数据分析（Pandas, NumPy）和网络请求（Requests）库，可以直接在数据库层调用。
- **特定领域优势**：JavaScript (PL/V8) 在处理 JSON 数据方面具有原生优势和灵活的语法。
- **开发人员熟悉度**：团队可能对 Python 或 JavaScript 更熟悉，从而降低学习成本，提高开发效率。

---

## 🐍 一、PL/Python

PL/Python 允许你在 PostgreSQL 中使用 Python 语言编写函数。它有两个版本：`plpythonu` (Python 2, 已废弃) 和 `plpython3u` (Python 3)。我们只关注 `plpython3u`。

### 1. 启用 PL/Python

首先，你需要在操作系统层面安装对应的包，然后在数据库中创建扩展。

**Debian/Ubuntu:**
```bash
sudo apt-get install postgresql-plpython3-17
```

**在数据库中启用:**
```sql
CREATE EXTENSION plpython3u;
```

### 2. 基本语法与示例

Python 函数通过 `plpy` 模块与 PostgreSQL 交互。

**示例：一个简单的 Python 函数**
```sql
CREATE OR REPLACE FUNCTION py_max(a INT, b INT)
RETURNS INT
LANGUAGE plpython3u
AS $$
    if a > b:
        return a
    return b
$$;

-- 调用
SELECT py_max(10, 20); -- 返回 20
```

**示例：利用 Python 标准库解析 JSON**
假设我们想从一个 JSON 字符串中提取 email。

```sql
CREATE OR REPLACE FUNCTION get_email_from_json(json_data TEXT)
RETURNS TEXT
LANGUAGE plpython3u
AS $$
    import json
    try:
        data = json.loads(json_data)
        return data.get('email')
    except json.JSONDecodeError:
        return None
$$;

-- 调用
SELECT get_email_from_json('{"name": "Alice", "email": "alice@example.com"}');
```

### 3. 数据库访问

`plpy` 模块提供了执行查询的函数，如 `plpy.execute()`。

```sql
CREATE OR REPLACE FUNCTION count_users()
RETURNS INT
LANGUAGE plpython3u
AS $$
    result = plpy.execute("SELECT COUNT(*) FROM users;")
    return result[0]['count']
$$;
```

---

## 🚀 二、PL/V8 (JavaScript)

PL/V8 是基于 Google V8 JavaScript 引擎的 PostgreSQL 过程语言。它在处理 JSON 和执行动态逻辑方面非常高效。

### 1. 启用 PL/V8

同样，需要先安装软件包，再创建扩展。

**Debian/Ubuntu (通常需要从源码编译或使用第三方源):**
```bash
# 安装依赖
sudo apt-get install -y build-essential libtinfo5 git cmake
# 克隆和编译
git clone https://github.com/plv8/plv8.git
cd plv8
make
sudo make install
```

**在数据库中启用:**
```sql
CREATE EXTENSION plv8;
```

### 2. 基本语法与示例

**示例：验证 JSON Schema**
这个函数可以检查一个 JSON 对象是否包含必要的字段。

```sql
CREATE OR REPLACE FUNCTION validate_user_profile(profile JSONB)
RETURNS BOOLEAN
LANGUAGE plv8
AS $$
    if (!profile || typeof profile !== 'object') {
        return false;
    }
    const required_keys = ['name', 'email', 'age'];
    for (const key of required_keys) {
        if (!(key in profile)) {
            return false;
        }
    }
    return true;
$$;

-- 调用
SELECT validate_user_profile('{"name": "Bob", "email": "bob@a.com", "age": 30}'); -- true
SELECT validate_user_profile('{"name": "Charlie"}'); -- false
```

---

## 🆚 三、语言选择对比

| 特性 | PL/pgSQL | PL/Python | PL/V8 (JavaScript) |
| :--- | :--- | :--- | :--- |
| **性能** | 极高，与数据库内核集成紧密 | 较好，但有 Python 解释器开销 | 非常高，得益于 V8 JIT 编译 |
| **生态系统** | 有限，依赖 PostgreSQL 扩展 | 极其丰富 (数据科学、Web 等) | 丰富 (NPM 生态，但需工具打包) |
| **JSON 处理** | 功能强大，但语法较繁琐 | 良好，有原生 `json` 库 | 极佳，原生支持，语法简洁 |
| **安全性** | 安全，运行在数据库沙箱内 | "U"代表不受信任，可访问文件系统 | 默认在沙箱内运行，相对安全 |
| **适用场景** | 传统数据库逻辑、高性能计算 | 数据分析、ETL、调用外部 API | 复杂 JSON 操作、Web 应用后端逻辑 |

---

## 📌 小结

- **PL/pgSQL** 仍然是编写高性能、纯数据库逻辑的首选。
- **PL/Python** 是当你需要在数据库中进行复杂数据分析、科学计算或与外部系统交互时的强大武器。
- **PL/V8** 在处理 JSON 数据和实现动态业务规则方面表现出色，是现代 Web 应用的理想搭档。

选择哪种语言取决于你的具体需求、团队的技术栈以及对性能和安全性的考量。PostgreSQL 的多语言支持为你提供了极大的灵活性，善用它们能让你的数据库如虎添翼。
