+++
title = "第十三章 全文搜索与向量相似度匹配 - 第三节 实战：图像特征匹配与文本语义搜索"
date = 2025-07-10
lastmod = 2025-07-10T01:00:00+08:00
tags = ["PostgreSQL", "practical", "book", "vector", "pgvector", "image search", "semantic search"]
categories = ["PostgreSQL", "practical", "book"]
draft = false
author = "b40yd"
+++

## 第三节 实战：图像特征匹配与文本语义搜索

> **目标**：通过一个综合性实战，将 `pgvector` 应用于两个核心 AI 场景：1) 基于图像内容的“以图搜图”；2) 基于文本语义的“智能问答”。

本实战将模拟一个多媒体内容库的后端系统。该系统需要存储图片和相关描述文本，并支持两种高级搜索功能：
-   **图像搜索**：用户上传一张图片，系统返回内容最相似的其他图片。
-   **文本搜索**：用户输入一个问题，系统返回最能回答该问题的文本片段。

这两种功能都依赖于同一个核心技术：**向量嵌入和相似度搜索**。

---

### 核心流程

1.  **数据入库**：
    -   对于**图片**，使用一个预训练的图像嵌入模型（如 CLIP, ResNet）将其转换为一个向量。
    -   对于**文本**，使用一个预训练的文本嵌入模型（如 SBERT, all-MiniLM）将其转换为另一个向量。
    -   将原始数据和其对应的向量一起存入 PostgreSQL。
2.  **搜索**：
    -   将用户的**查询图片/文本**通过**相同的嵌入模型**转换为一个查询向量。
    -   使用 `pgvector` 在数据库中执行最近邻搜索，找到与查询向量最相似的向量。
    -   返回这些向量对应的原始数据（图片URL或文本内容）。

---

## 🏛️ 第一步：设计数据表

我们创建一个 `multimedia_assets` 表来统一存储图片和文本信息。

```sql
-- 确保 pgvector 扩展已启用
CREATE EXTENSION IF NOT EXISTS vector;

CREATE TABLE multimedia_assets (
    id SERIAL PRIMARY KEY,
    asset_type TEXT NOT NULL, -- 'image' or 'text'
    -- 对于图片，是 URL；对于文本，是内容本身
    content TEXT NOT NULL,
    -- 使用一个统一的字段存储向量，维度取决于所选模型
    embedding VECTOR(384)
);

-- 创建 HNSW 索引以加速相似度搜索
-- 我们使用余弦距离，因为它在语义空间中表现良好
CREATE INDEX idx_assets_embedding_hnsw ON multimedia_assets
USING HNSW (embedding vector_cosine_ops);
```

---

## ✍️ 第二步：模拟数据入库

在真实应用中，`embedding` 字段的值是通过调用 Python 等语言中的深度学习模型库（如 `transformers`, `timm`）生成的。这里我们手动插入一些简化的、示意性的向量。

```sql
-- 插入图片数据 (URL + 图像向量)
INSERT INTO multimedia_assets (asset_type, content, embedding) VALUES
('image', 'https://example.com/images/cat_on_sofa.jpg', '[0.9, 0.1, 0.2, ...]'), -- 猫的向量
('image', 'https://example.com/images/lion_in_savanna.jpg', '[0.8, 0.2, 0.1, ...]'), -- 狮子的向量 (与猫相似)
('image', 'https://example.com/images/dog_playing_fetch.jpg', '[0.1, 0.9, 0.3, ...]'), -- 狗的向量

-- 插入文本数据 (内容 + 文本向量)
('text', 'The financial market showed a significant downturn last quarter.', '[-0.5, 0.8, 0.7, ...]'), -- 金融新闻向量
('text', 'Stock prices are influenced by quarterly earnings reports.', '[-0.4, 0.9, 0.6, ...]'), -- 股票新闻向量
('text', 'The best recipes for homemade pasta.', '[0.2, -0.7, 0.5, ...]'); -- 烹饪食谱向量
```

---

## 🔍 第三步：执行“以图搜图”

**场景**：用户上传了一张新的猫的图片。应用将其转换为查询向量 `[0.88, 0.12, 0.18, ...]`。现在，我们需要在数据库中找到最相似的图片。

```sql
-- 准备查询向量
-- 在真实应用中，这个向量由你的 AI 模型生成
-- 这里我们直接写入 SQL
SELECT
    id,
    content AS image_url,
    1 - (embedding <=> '[0.88, 0.12, 0.18, ...]') AS similarity
FROM multimedia_assets
WHERE asset_type = 'image' -- 只在图片中搜索
ORDER BY embedding <=> '[0.88, 0.12, 0.18, ...]'::vector
LIMIT 5;
```
**预期结果**：
查询会返回 `cat_on_sofa.jpg` 和 `lion_in_savanna.jpg`，因为它们的向量与查询向量的余弦距离最近。`dog_playing_fetch.jpg` 则会因为距离较远而排在后面或不被返回。HNSW 索引确保了这个查询即使在百万级图片库中也能在毫秒内完成。

---

## 🗣️ 第四步：执行语义文本搜索（智能问答）

**场景**：用户在搜索框输入问题：“What affects the stock market?”。应用将其转换为查询向量 `[-0.45, 0.85, 0.65, ...]`。

```sql
-- 准备查询向量
SELECT
    id,
    content AS relevant_text,
    1 - (embedding <=> '[-0.45, 0.85, 0.65, ...]') AS relevance_score
FROM multimedia_assets
WHERE asset_type = 'text' -- 只在文本中搜索
ORDER BY embedding <=> '[-0.45, 0.85, 0.65, ...]'::vector
LIMIT 3;
```
**预期结果**：
查询会返回关于“金融市场”和“股票价格”的两段文本，因为它们的语义与问题最相关。关于“烹饪食谱”的文本则会被忽略。这远比基于关键词“stock”或“market”的传统搜索要智能和准确得多。

---

## 📌 小结

本实战清晰地展示了 `pgvector` 如何赋能两种前沿的 AI 搜索应用：
1.  **跨模态理解**：通过将不同类型的数据（图片、文本）映射到同一个向量空间，`pgvector` 实现了基于内容和语义的统一检索。
2.  **架构简化**：你不再需要一个独立的向量数据库（如 Pinecone, Weaviate）。可以直接在主业务数据库 PostgreSQL 中存储和查询向量，极大地降低了技术栈的复杂性、运维成本和数据同步的延迟。
3.  **高性能**：HNSW 索引提供了与专业向量数据库相媲美的查询性能，足以支撑大规模的生产应用。

随着 AI 技术的普及，将向量搜索能力集成到关系型数据库中已成为一种趋势。`pgvector` 使 PostgreSQL 在这场技术变革中走在了前列，为开发者提供了一个强大、成熟且高度集成的 AI 应用后端解决方案。
