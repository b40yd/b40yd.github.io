+++
title = "第十章 JSON 与 JSONB 数据类型深度解析 - 第三节：支持的操作符与函数"
date = 2025-07-10
lastmod = 2025-07-10T01:00:00+08:00
tags = ["PostgreSQL", "practical", "book", "nosql", "jsonb", "operators", "functions"]
categories = ["PostgreSQL", "practical", "book"]
draft = false
author = "b40yd"
+++

## 第三节 支持的操作符与函数

> **目标**：全面了解 PostgreSQL 为 `JSONB` 提供的丰富操作符和函数，掌握数据提取、修改和处理的核心工具。

除了上一节介绍的用于索引优化的 `@>`、`?` 等操作符外，PostgreSQL 还提供了一整套强大的工具集来处理 `JSONB` 数据。本节将分类介绍最常用的操作符和函数。

---

### 一、数据提取操作符

这些操作符用于从 `JSONB` 文档中获取数据。

| 操作符 | 返回类型 | 描述 | 示例 |
| :--- | :--- | :--- | :--- |
| `->` | `jsonb` | 按**键**（对象）或**索引**（数组）提取子 `JSONB` | ` '{"a": {"b": 1}}'::jsonb -> 'a' ` -> `{"b": 1}` |
| `->>` | `text` | 按**键**或**索引**提取值为**文本** | ` '{"a": 1}'::jsonb ->> 'a' ` -> `'1'` |
| `#>` | `jsonb` | 按**路径**提取子 `JSONB` | ` '{"a": {"b": [1,2]}}'::jsonb #> '{a,b,1}' ` -> `2` |
| `#>>` | `text` | 按**路径**提取值为**文本** | ` '{"a": {"b": [1,2]}}'::jsonb #>> '{a,b,1}' ` -> `'2'` |

**关键区别：**
-   单箭头 `->` 返回 `jsonb`，双箭头 `->>` 返回 `text`。
-   `#>` 和 `#>>` 使用一个文本数组 `'{key,index,...}'` 来表示路径，非常适合提取深层嵌套的数据。

**示例：**
```sql
WITH data AS (
    SELECT '{
        "name": "Alice",
        "contact": {
            "emails": ["alice@work.com", "alice@home.com"]
        },
        "orders": [
            {"id": 101, "amount": 150},
            {"id": 102, "amount": 200}
        ]
    }'::jsonb AS doc
)
SELECT
    doc -> 'name' AS name_json, -- "Alice" (jsonb)
    doc ->> 'name' AS name_text, -- Alice (text)
    doc #> '{contact,emails,0}' AS first_email_json, -- "alice@work.com" (jsonb)
    doc #>> '{orders,1,amount}' AS second_order_amount -- 200 (text)
FROM data;
```

---

### 二、`JSONB` 修改与合并

#### 1. 合并操作符 `||`

`||` 操作符用于将两个 `JSONB` 值合并成一个新的 `JSONB` 值。
-   **合并对象**：如果键冲突，右侧的值会覆盖左侧的值。
-   **合并数组**：直接将两个数组连接起来。

```sql
-- 合并对象
SELECT '{"a": 1, "b": 2}'::jsonb || '{"b": 3, "c": 4}'::jsonb;
-- 结果: {"a": 1, "b": 3, "c": 4}

-- 合并数组
SELECT '[1, 2]'::jsonb || '[3, 4]'::jsonb;
-- 结果: [1, 2, 3, 4]
```

#### 2. 删除操作符 `-` 和 `#-`

-   `-`: 从对象中按**键**删除，或从数组中按**索引**删除。
-   `#-`: 按**路径**删除。

```sql
-- 从对象中删除键 'b'
SELECT '{"a": 1, "b": 2}'::jsonb - 'b';
-- 结果: {"a": 1}

-- 从数组中删除索引为 1 的元素
SELECT '["a", "b", "c"]'::jsonb - 1;
-- 结果: ["a", "c"]

-- 按路径删除
SELECT '{"a": {"b": 1, "c": 2}}'::jsonb #- '{a,b}';
-- 结果: {"a": {"c": 2}}
```

#### 3. `jsonb_set()` 函数

`jsonb_set()` 是一个更强大的更新函数，它可以在指定路径上替换或插入值。
`jsonb_set(target, path, new_value, [create_if_missing])`

```sql
-- 更新 Alice 的第一个 email
SELECT jsonb_set(
    '{"name": "Alice", "emails": ["a@a.com"]}'::jsonb,
    '{emails,0}',
    '"new@a.com"'::jsonb
);
-- 结果: {"name": "Alice", "emails": ["new@a.com"]}

-- 在不存在的路径上创建新值 (第四个参数为 true)
SELECT jsonb_set(
    '{"name": "Alice"}'::jsonb,
    '{contact,phone}',
    '"12345"'::jsonb,
    true
);
-- 结果: {"name": "Alice", "contact": {"phone": "12345"}}
```

---

### 三、常用的 `JSONB` 创建和处理函数

| 函数 | 描述 |
| :--- | :--- |
| `jsonb_build_object(...)` | 从键值对列表创建 `JSONB` 对象。 |
| `jsonb_build_array(...)` | 从元素列表创建 `JSONB` 数组。 |
| `jsonb_pretty(...)` | 将 `JSONB` 格式化为带缩进的可读文本。 |
| `jsonb_typeof(...)` | 返回 `JSONB` 值的顶层类型 (`object`, `array`, `string`, `number`, `boolean`, `null`)。 |
| `jsonb_array_length(...)` | 返回 `JSONB` 数组的长度。 |
| `jsonb_object_keys(...)` | 以集合形式返回 `JSONB` 对象的所有顶层键。 |
| `jsonb_each(...)` | 将 `JSONB` 对象展开为键值对集合。 |
| `jsonb_array_elements(...)` | 将 `JSONB` 数组展开为元素集合。 |

**示例：**
```sql
-- 使用 jsonb_build_object 创建对象
SELECT jsonb_build_object('name', 'Bob', 'age', 30);

-- 使用 jsonb_array_elements 展开数组
SELECT value FROM jsonb_array_elements('[1, "a", true]'::jsonb);
```

---

## 📌 小结

PostgreSQL 为 `JSONB` 提供了一个功能极其丰富的“工具箱”。
-   **提取数据**：使用 `->`, `->>`, `#>`, `#>>` 来精确获取你需要的信息。
-   **修改数据**：使用 `||`, `-`, `#-` 和 `jsonb_set` 来动态地构建和修改 JSON 文档。
-   **处理数据**：利用各种 `jsonb_*` 函数，可以在 SQL层面完成复杂的 JSON 创建和解构任务。

熟练掌握这些操作符和函数，你就可以在 PostgreSQL 中像操作原生 NoSQL 数据库一样，灵活自如地处理半结构化数据。在下一节，我们将通过一个实战案例，将这些工具融会贯通。
