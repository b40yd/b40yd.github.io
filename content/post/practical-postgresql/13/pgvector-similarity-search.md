+++
title = "第十三章 全文搜索与向量相似度匹配 - 第二节：使用 pgvector 插件进行向量检索"
date = 2025-07-10
lastmod = 2025-07-10T01:00:00+08:00
tags = ["PostgreSQL", "practical", "book", "vector", "pgvector", "ai", "similarity search"]
categories = ["PostgreSQL", "practical", "book"]
draft = false
author = "b40yd"
+++

## 第二节 使用 `pgvector` 插件进行向量检索

> **目标**：了解向量嵌入（Vector Embeddings）的基本概念，并学习如何使用 `pgvector` 扩展在 PostgreSQL 中存储向量，并执行高效的相似度搜索。

随着人工智能（AI）和机器学习（ML）的兴起，一种全新的搜索范式——**向量相似度搜索**——变得越来越重要。它不再依赖于关键词匹配，而是通过理解数据的**语义（Semantic）**来进行搜索。

**核心思想**：
1.  使用一个深度学习模型（称为“嵌入模型”，Embedding Model），将复杂的数据（如文本、图片、音频）转换成一个由数字组成的**向量（Vector）**。
2.  这个向量可以被看作是数据在多维空间中的一个“坐标”，它捕捉了数据的核心语义特征。
3.  语义上相似的数据，其向量在空间中的距离也更近。
4.  通过计算向量之间的距离（如欧氏距离、余弦相似度），我们就可以找到与查询内容最相似的数据。

**`pgvector`** 是一个流行的 PostgreSQL 扩展，它为数据库增加了 `vector` 数据类型和一系列用于计算向量距离的操作符和索引类型，使 PostgreSQL 变身为一个功能强大的向量数据库。

---

### 安装和启用 `pgvector`

`pgvector` 是一个第三方扩展，需要从源码编译或通过包管理器安装。

**1. 从源码安装 (通用方法)**
```bash
git clone --branch v0.7.0 https://github.com/pgvector/pgvector.git
cd pgvector
make
sudo make install
```
*(请参考 `pgvector` 官方文档获取最新的稳定版本和安装指南)*

**2. 在数据库中启用**
```sql
CREATE EXTENSION vector;
```

---

### `pgvector` 的核心功能

#### 1. `vector` 数据类型

`pgvector` 引入了 `vector` 类型，用于存储向量。
`vector(n)`: 表示一个 n 维的浮点数向量。

```sql
CREATE TABLE items (
    id SERIAL PRIMARY KEY,
    embedding VECTOR(3) -- 存储一个3维向量
);

INSERT INTO items (embedding) VALUES
('[1,2,3]'),
('[4,5,6]');
```

#### 2. 距离计算操作符

`pgvector` 提供了多种操作符来计算向量间的距离。

| 操作符 | 距离类型 | 描述 |
| :--- | :--- | :--- |
| `<->` | **欧氏距离 (L2 Distance)** | 空间中两点间的直线距离。 |
| `<#>` | **内积 (Inner Product)** | 两个向量的点积。对于归一化向量，它与余弦距离相关。 |
| `<=>` | **余弦距离 (Cosine Distance)** | 衡量两个向量方向的差异，与内容本身的幅度无关。非常适合文本语义相似度。 |

**示例：**
```sql
SELECT '[1,1,1]'::vector <-> '[2,2,2]'::vector; -- 计算欧氏距离
SELECT '[1,1,1]'::vector <=> '[2,2,2]'::vector; -- 计算余弦距离
```

---

### 实战：基于文本语义的问答系统

我们将构建一个简单的系统，它可以根据问题的语义相似度，从一系列文档中找到最相关的答案。

#### 第一步：准备数据表

```sql
CREATE TABLE documents (
    id SERIAL PRIMARY KEY,
    content TEXT,
    embedding VECTOR(384) -- 假设我们使用一个输出384维向量的嵌入模型
);
```

#### 第二步：生成并存储向量

在真实应用中，我们会使用一个像 `all-MiniLM-L6-v2` 这样的句子嵌入模型，将 `content` 文本转换为向量。这里我们为了演示，手动插入一些简化的向量。

```sql
-- 假设我们已经将以下句子转换为了向量
-- 'The cat sat on the mat.' -> [1, 0, 0, ...]
-- 'The dog played in the park.' -> [0, 1, 0, ...]
-- 'Feline animals are cute.' -> [0.9, 0.1, 0, ...] (与猫的语义非常接近)

INSERT INTO documents (content, embedding) VALUES
('The cat sat on the mat.', '[1,0,0]'),
('The dog played in the park.', '[0,1,0]'),
('Feline animals are cute.', '[0.9,0.1,0]');
```

#### 第三步：执行相似度搜索

**问题**：“What are cats like?” (猫是什么样的？)
我们先将这个问题同样转换为向量，假设其向量为 `[0.8, 0, 0.1]`。

现在，我们可以在数据库中查找与这个查询向量最相似的文档。

```sql
-- 使用余弦距离 (<=>) 进行搜索
SELECT id, content, 1 - (embedding <=> '[0.8,0,0.1]') AS similarity
FROM documents
ORDER BY embedding <=> '[0.8,0,0.1]'
LIMIT 5;
```
-   `ORDER BY embedding <=> '[0.8,0,0.1]'`: 这是核心。我们按查询向量与表中每个向量的余弦距离，从小到大进行排序。距离越小，相似度越高。
-   `1 - (distance)`: 余弦距离的范围是 0 到 2。`1 - distance` 可以将其转换为一个更直观的相似度分数（-1 到 1）。

**预期结果：**
查询结果会按相似度排序，`'Feline animals are cute.'` 和 `'The cat sat on the mat.'` 会排在最前面，因为它们的向量与查询向量的夹角最小。

---

### 第四步：使用 HNSW 索引进行加速

当数据量巨大时，逐个计算向量距离（全表扫描）会非常慢。`pgvector` 支持使用 **HNSW (Hierarchical Navigable Small World)** 索引来进行**近似最近邻（Approximate Nearest Neighbor, ANN）**搜索。

HNSW 是一种图结构的索引，它能以极高的速度找到与查询向量“足够近”的邻居，虽然不保证 100% 精确，但在绝大多数场景下，其召回率和性能都非常出色。

```sql
CREATE INDEX ON documents USING HNSW (embedding vector_cosine_ops);
```
-   `vector_cosine_ops`: 指定索引使用余弦距离进行构建。

创建索引后，上面的 `ORDER BY ... LIMIT ...` 查询将会被自动加速，性能可提升数百倍。

---

## 📌 小结

`pgvector` 将 PostgreSQL 带入了 AI 时代。
1.  **语义搜索**：它让数据库不再局限于关键词匹配，而是能够理解数据背后的**语义和上下文**。
2.  **统一存储**：你可以在同一个数据库中同时存储业务数据和 AI 生成的向量嵌入，极大地简化了 AI 应用的架构。
3.  **高性能**：通过 HNSW 等现代向量索引算法，`pgvector` 提供了足以媲美专业向量数据库的查询性能。

结合 PostgreSQL 强大的 SQL 功能和 `pgvector` 的向量检索能力，你可以轻松构建各种复杂的 AI 应用，如问答系统、推荐系统、图像搜索、异常检测等。
