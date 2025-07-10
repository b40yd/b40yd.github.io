+++
title = "第五章 表分区与继承 - 第三节：自动分区管理"
date = 2025-07-10
lastmod = 2025-07-10T01:00:00+08:00
tags = ["PostgreSQL", "practical", "book", "partitioning", "automation", "pg_partman"]
categories = ["PostgreSQL", "practical", "book"]
draft = false
author = "b40yd"
+++

## 第三节 自动分区管理

> **目标**：解决手动创建和维护分区的痛点，学习如何使用自动化脚本或 `pg_partman` 扩展来实现分区的自动创建和删除，确保分区表能够“无人值守”地运行。

虽然原生分区极大地简化了分区的定义和使用，但一个现实问题依然存在：对于时间序列数据（如按月或按日分区），我们必须提前创建未来的分区。如果忘记创建，新的数据将无法插入，导致业务中断。同样，过期的历史分区也需要被定期清理（分离或删除）以释放存储空间。

手动执行这些任务是乏味且不可靠的。本节将介绍两种实现分区自动化管理的主流方法。

---

## 🛠️ 方法一：使用自定义脚本 + 定时任务 (Cron)

对于简单的、有规律的分区需求，我们可以编写一个简单的 SQL 或 Shell 脚本，并使用系统的定时任务工具（如 Linux 的 `cron`）来定期执行它。

### 场景：为 `access_logs` 表自动创建下个月的分区

假设我们有一个按月分区的 `access_logs` 表，我们希望系统在每个月的中旬（例如15号）自动创建下下个月的分区，并自动删除13个月前的旧分区。

#### 1. 编写 SQL 函数

在数据库中创建一个函数来封装分区管理逻辑，这样更易于维护和测试。

```sql
CREATE OR REPLACE FUNCTION manage_access_logs_partitions()
RETURNS VOID AS $$
DECLARE
    next_month_start DATE;
    next_month_end DATE;
    next_month_partition_name TEXT;
    
    old_month_start DATE;
    old_month_partition_name TEXT;
BEGIN
    -- 1. 创建未来分区
    -- 计算下下个月的起始和结束日期
    next_month_start := date_trunc('month', NOW() + INTERVAL '2 months');
    next_month_end := next_month_start + INTERVAL '1 month';
    next_month_partition_name := 'access_logs_y' || to_char(next_month_start, 'YYYY') || 'm' || to_char(next_month_start, 'MM');

    -- 检查分区是否已存在，不存在则创建
    IF NOT EXISTS (SELECT 1 FROM pg_class WHERE relname = next_month_partition_name) THEN
        EXECUTE format(
            'CREATE TABLE %I PARTITION OF access_logs FOR VALUES FROM (%L) TO (%L);',
            next_month_partition_name,
            next_month_start,
            next_month_end
        );
        RAISE NOTICE 'Partition % created.', next_month_partition_name;
    END IF;

    -- 2. 删除旧分区
    -- 计算13个月前的分区名
    old_month_start := date_trunc('month', NOW() - INTERVAL '13 months');
    old_month_partition_name := 'access_logs_y' || to_char(old_month_start, 'YYYY') || 'm' || to_char(old_month_start, 'MM');

    -- 检查旧分区是否存在，存在则删除
    IF EXISTS (SELECT 1 FROM pg_class WHERE relname = old_month_partition_name) THEN
        EXECUTE format('DROP TABLE %I;', old_month_partition_name);
        RAISE NOTICE 'Partition % dropped.', old_month_partition_name;
    END IF;
END;
$$ LANGUAGE plpgsql;
```

#### 2. 设置 Cron 定时任务

现在，我们可以设置一个 `cron` 作业来每月执行这个函数。

打开 crontab 编辑器：
```bash
crontab -e
```

添加以下行，表示在每个月的15号凌晨2点执行任务：
```crontab
# m h  dom mon dow   command
0 2 15 * * psql -h localhost -U your_user -d your_db -c "SELECT manage_access_logs_partitions();"
```

**优点**：
- 灵活，完全可定制。
- 不需要安装任何外部依赖。

**缺点**：
- 需要自己编写和维护 SQL 逻辑，对于复杂场景容易出错。
- 跨平台兼容性差（Cron 是类 Unix 系统工具）。

---

## 🚀 方法二：使用 `pg_partman` 扩展

`pg_partman` 是一个功能极其强大的 PostgreSQL 扩展，专门用于自动化管理基于时间和序列号的分区表。它几乎是处理分区自动化问题的“标准答案”。

### 1. 安装 `pg_partman`

**Debian/Ubuntu:**
```bash
sudo apt-get install postgresql-17-partman
```

**在数据库中启用:**
```sql
CREATE SCHEMA partman;
CREATE EXTENSION pg_partman SCHEMA partman;
```

### 2. 配置 `pg_partman`

`pg_partman` 的核心是 `partman.create_parent` 函数，它会注册一个分区表并为其设置自动化策略。

```sql
-- 假设 access_logs 表已按上一节的方式创建
-- 调用 create_parent 来让 pg_partman接管它
SELECT partman.create_parent(
    p_parent_table => 'public.access_logs',
    p_control => 'log_time',
    p_type => 'native',
    p_interval => 'monthly',
    p_premake => 4,  -- 提前创建未来4个分区
    p_start_partition => to_char(now(), 'YYYY-MM-DD')
);
```
- `p_parent_table`: 要管理的分区主表。
- `p_control`: 分区键列。
- `p_type`: 分区类型，`native` 表示原生分区。
- `p_interval`: 分区间隔，可以是 `daily`, `weekly`, `monthly`, `yearly` 等。
- `p_premake`: 提前创建多少个未来的分区。这是防止数据插入失败的关键。

### 3. 运行维护任务

`pg_partman` 不会自动运行。你需要调用它的维护函数 `partman.run_maintenance()` 来执行实际的分区创建和删除操作。通常也是通过 `cron` 来定期调用。

```sql
-- 更新 retention 策略，让 pg_partman 自动删除超过12个月的旧分区
UPDATE partman.part_config 
SET retention = '12 months', retention_keep_table = false
WHERE parent_table = 'public.access_logs';
```
- `retention`: 保留多长时间的数据。
- `retention_keep_table`: `false` 表示直接 `DROP` 旧分区，`true` 表示 `DETACH`。

**设置 Cron 定时任务:**
`pg_partman` 建议每小时运行一次维护任务，它足够智能，只在需要时才执行操作。
```crontab
# m h  dom mon dow   command
0 * * * * psql -h localhost -U your_user -d your_db -c "SELECT partman.run_maintenance();"
```

**优点**：
- 功能强大，身经百战，非常可靠。
- 配置驱动，无需编写复杂的 SQL。
- 支持自动清理、保留策略、多级分区等高级功能。

**缺点**：
- 需要安装和学习一个新扩展。

---

## 📌 结论

| 方法 | 适用场景 | 优点 | 缺点 |
| :--- | :--- | :--- | :--- |
| **自定义脚本** | 简单、固定的分区需求；不希望引入外部依赖。 | 灵活、无依赖 | 需自行维护、易出错、功能有限 |
| **`pg_partman`** | 几乎所有生产环境中的时间序列分区表。 | **强大、可靠、功能丰富、配置驱动** | 需安装和学习扩展 |

对于任何严肃的生产环境应用，**强烈推荐使用 `pg_partman`**。它为你处理了所有复杂的边界情况，让你能专注于业务逻辑，而不是底层的分区维护。自定义脚本只适用于非常简单的、一次性的或非关键的场景。
