+++
title = "第七章 PostgreSQL 中的图数据库能力 - 第四节 实战：组织架构图的构建与查询"
date = 2025-07-10
lastmod = 2025-07-10T01:00:00+08:00
tags = ["PostgreSQL", "practical", "book", "graph", "ltree", "apache age", "org chart"]
categories = ["PostgreSQL", "practical", "book"]
draft = false
author = "b40yd"
+++

## 第四节 实战：组织架构图的构建与查询

> **目标**：通过一个完整的组织架构图实战案例，综合运用并对比本章介绍的两种图数据处理方法——`ltree` 和 Apache AGE，理解它们各自的优缺点和最佳应用场景。

组织架构是企业管理中的核心数据模型，它天生就是一种层级分明的树形图。在这个实战中，我们将构建一个包含部门、职位和员工的组织架构，并用两种不同的技术方案来实现和查询它。

**业务需求：**
1.  存储公司、部门、员工的层级关系。
2.  查询某部门下的所有员工。
3.  查询某员工的管理链条（汇报线）。
4.  （扩展需求）处理非树形关系，如“虚线汇报”或“项目协作”。

---

## 方案一：使用 `ltree` 实现

`ltree` 非常适合表示纯粹的树形结构，如标准的组织汇报线。

### 1. 数据建模

我们复用本章第一节的 `organization` 表结构。

```sql
CREATE TABLE organization_ltree (
    id SERIAL PRIMARY KEY,
    name TEXT NOT NULL,
    type VARCHAR(20) NOT NULL, -- 'Department', 'Employee'
    path LTREE NOT NULL UNIQUE
);
CREATE INDEX idx_org_ltree_path ON organization_ltree USING GIN (path);
```

### 2. 数据插入

```sql
INSERT INTO organization_ltree (name, type, path) VALUES
('CEO Office', 'Department', '1'),
('CTO', 'Employee', '1.2'),
('Engineering', 'Department', '1.2.3'),
('Frontend Team', 'Team', '1.2.3.4'),
('Backend Team', 'Team', '1.2.3.5'),
('Alice (Lead)', 'Employee', '1.2.3.4.6'),
('Bob', 'Employee', '1.2.3.4.7'),
('Charlie (Lead)', 'Employee', '1.2.3.5.8');
```

### 3. 查询实现

**查询 1：查找“Engineering”部门下的所有员工**
```sql
SELECT name FROM organization_ltree
WHERE path <@ '1.2.3' AND type = 'Employee';
```

**查询 2：查找员工“Bob”的管理链条**
```sql
SELECT name, type FROM organization_ltree
WHERE path @> '1.2.3.4.7' ORDER BY path;
```

### `ltree` 方案小结

-   **优点**：对于严格的树形结构，`ltree` 查询极其高效、语法简洁。
-   **缺点**：无法自然地表示非树形关系。例如，如果 Bob 需要向“CEO Office”虚线汇报，`ltree` 模型就很难表达这种多重领导关系。

---

## 方案二：使用 Apache AGE 实现

当组织架构中出现复杂的非树形关系时，AGE 的图模型就显示出其强大的灵活性。

### 1. 数据建模

在 AGE 中，部门、员工等都是**节点（Node）**，而汇报关系、协作关系则是**边（Edge）**。

### 2. 数据插入 (Cypher)

```sql
-- 确保已创建图并设置 search_path
SELECT create_graph('organization_graph');
SET search_path = organization_graph, public;

-- 使用 Cypher 创建图结构
SELECT * FROM cypher('organization_graph', $$
    CREATE (ceo_office:Department {name: 'CEO Office'}),
           (cto:Employee {name: 'CTO'}),
           (eng:Department {name: 'Engineering'}),
           (frontend:Team {name: 'Frontend Team'}),
           (backend:Team {name: 'Backend Team'}),
           (alice:Employee {name: 'Alice (Lead)'}),
           (bob:Employee {name: 'Bob'}),
           (charlie:Employee {name: 'Charlie (Lead)'}),
           
           -- 建立汇报关系 (MANAGES)
           (ceo_office)-[:CONTAINS]->(cto),
           (cto)-[:MANAGES]->(eng),
           (eng)-[:CONTAINS]->(frontend),
           (eng)-[:CONTAINS]->(backend),
           (frontend)-[:CONTAINS]->(alice),
           (frontend)-[:CONTAINS]->(bob),
           (backend)-[:CONTAINS]->(charlie),
           (alice)-[:REPORTS_TO]->(frontend),
           (bob)-[:REPORTS_TO]->(alice),
           (charlie)-[:REPORTS_TO]->(backend),

           -- 增加一条虚线汇报关系
           (bob)-[:DOTTED_LINE_REPORTS_TO]->(ceo_office)
$$) AS (result agtype);
```

### 3. 查询实现 (Cypher)

**查询 1：查找“Engineering”部门下的所有员工**
这是一个可变长度的路径查询。
```sql
SELECT * FROM cypher('organization_graph', $$
    MATCH (d:Department {name: 'Engineering'})-[:CONTAINS*1..]->(e:Employee)
    RETURN e.name
$$) AS (employee_name agtype);
```

**查询 2：查找员工“Bob”的实线管理链条**
```sql
SELECT * FROM cypher('organization_graph', $$
    MATCH (e:Employee {name: 'Bob'})
    OPTIONAL MATCH (e)-[:REPORTS_TO]->(m1)
    OPTIONAL MATCH (m1)-[:REPORTS_TO]->(m2)
    OPTIONAL MATCH (m2)-[:REPORTS_TO]->(m3)
    RETURN [e.name, m1.name, m2.name, m3.name] AS reporting_line
$$) AS (reporting_line agtype);
```
`nodes(path)` 函数可以提取路径上的所有节点。

**查询 3：查找向“CEO Office”汇报的所有人（包括实线和虚线）**
```sql
SELECT * FROM cypher('organization_graph', $$
    MATCH (p)-[r:REPORTS_TO]->(d:Department {name: 'CEO Office'})
    RETURN p.name, type(r) AS report_type
$$) AS (person_name agtype, report_type agtype)
UNION
SELECT * FROM cypher('organization_graph', $$
    MATCH (p)-[r:DOTTED_LINE_REPORTS_TO]->(d:Department {name: 'CEO Office'})
    RETURN p.name, type(r) AS report_type
$$) AS (person_name agtype, report_type agtype);
```
-   `[:REPORTS_TO|:DOTTED_LINE_REPORTS_TO]` 语法可以同时匹配多种类型的边。
-   这个查询完美地解决了 `ltree` 方案的痛点。

---

## 结论：如何选择？

| 特性 | `ltree` | Apache AGE (图模型) |
| :--- | :--- | :--- |
| **模型复杂度** | **简单**，一张表即可 | **中等**，需要理解节点、边、标签 |
| **适用场景** | **严格的树形结构**，如商品分类、地理区域 | **复杂的网络结构**，如社交网络、多重汇报关系 |
| **查询性能** | 极高（对于树查询） | 高（对于图遍历） |
| **灵活性** | 较低，难以表示非树形关系 | **极高**，可以随意添加新的节点和关系类型 |
| **生态与标准** | PostgreSQL 特有 | 使用开放的 openCypher 标准，易于迁移 |

**选择建议：**

-   如果你的数据模型是**稳定且严格的树形结构**，并且追求极致的查询性能和最简单的实现，**`ltree` 是最佳选择**。
-   如果你的数据模型中存在**多对多、网状、或多种类型的复杂关系**，或者你预期未来可能出现这类需求，那么**Apache AGE 提供的真图数据库能力是更强大、更灵活的解决方案**。

通过这个实战，我们看到 PostgreSQL 借助其强大的扩展性，为我们提供了从简单到复杂的多种图数据处理方案，开发者可以根据业务的实际需求，选择最合适的工具。
