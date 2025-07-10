+++
title = "第十二章 时间序列数据处理 - 第二节：hypertable 创建与压缩策略"
date = 2025-07-10
lastmod = 2025-07-10T01:00:00+08:00
tags = ["PostgreSQL", "practical", "book", "time series", "timescaledb", "compression", "retention"]
categories = ["PostgreSQL", "practical", "book"]
draft = false
author = "b40yd"
+++

## 第二节 Hypertable 创建与压缩策略

> **目标**：深入学习 `create_hypertable` 函数的高级用法，并掌握 TimescaleDB 的核心数据管理功能：列式压缩（Columnar Compression）和数据保留策略（Data Retention）。

创建基本的 Hypertable 很简单，但 TimescaleDB 的强大之处在于它提供了丰富的配置选项和策略，来帮助我们高效地管理海量的时序数据，尤其是在存储成本和查询性能之间取得平衡。

---

### 一、高级 Hypertable 创建

`create_hypertable` 函数支持更多参数，让我们能更精细地控制分区行为。

#### 1. 空间分区 (Space Partitioning)

除了按时间分区，TimescaleDB 还支持在另一个维度（如 `device_id`, `location`）上进行**空间分区**。这对于多租户或多设备场景下的查询性能有巨大提升。

```sql
-- 准备一个包含位置信息的传感器数据表
CREATE TABLE sensor_data (
    time TIMESTAMPTZ NOT NULL,
    location TEXT NOT NULL,
    device_id TEXT NOT NULL,
    value DOUBLE PRECISION
);

-- 创建一个同时按时间和空间分区的 Hypertable
SELECT create_hypertable(
    'sensor_data',
    'time',
    'location',           -- 空间分区键
    number_partitions => 4  -- 空间分区的数量
);
```
这个设置会首先按时间创建 Chunks，然后在每个时间 Chunk 内部，再根据 `location` 的哈希值将数据分布到 4 个空间分区中。当你的查询同时包含 `time` 和 `location` 条件时（例如 `WHERE time > ... AND location = 'room-a'`），性能会得到极大提升。

#### 2. 调整 Chunk 时间间隔

默认的 7 天时间间隔不一定适合所有场景。如果数据量极大，你可能希望每天甚至每小时创建一个 Chunk。

```sql
SELECT create_hypertable('sensor_data', 'time', chunk_time_interval => INTERVAL '1 day');
```
或者，如果数据量不大，可以设置为一个月：
```sql
SELECT create_hypertable('sensor_data', 'time', chunk_time_interval => INTERVAL '1 month');
```

---

### 二、列式压缩 (Columnar Compression)

这是 TimescaleDB 最具吸引力的功能之一。时序数据通常具有高度的重复性（相同的时间戳、相同的设备ID等），非常适合压缩。

TimescaleDB 的压缩是**列式的**，它将每个 Chunk 从“行存”格式转换为“列存”格式，并对每一列应用最适合其数据类型的压缩算法（如 Gorilla、Delta-delta、LZ4）。

**压缩带来的好处：**
1.  **极高的存储节省**：通常可以达到 90% - 98% 的压缩率，极大地降低了存储成本。
2.  **更快的分析查询**：对于只涉及少数几列的分析查询（如 `SELECT avg(temperature) FROM ...`），列式存储只需读取所需的列，大大减少了 I/O，查询速度可以提升数十倍。

**压缩的代价：**
-   被压缩的 Chunk **默认是只读的**。你不能对其中的数据进行 `UPDATE` 或 `DELETE`。（可以先解压，修改后再重新压缩，但这很低效）。

因此，压缩非常适合那些**不再会发生变化的旧数据**。

#### 启用压缩策略

我们可以设置一个策略，让 TimescaleDB 自动压缩超过一定时间的旧数据。

```sql
-- 为 sensor_data 表启用压缩
ALTER TABLE sensor_data SET (
    timescaledb.compress,
    timescaledb.compress_segmentby = 'location, device_id'
);

-- 添加一个策略，自动压缩超过 7 天的旧数据
SELECT add_compression_policy('sensor_data', INTERVAL '7 days');
```
-   `compress_segmentby`: 指定一个或多个列作为“压缩段”。在同一个段内的数据会被组织在一起，这对于按段进行分组的查询（如 `GROUP BY device_id`）性能极好。

---

### 三、数据保留策略 (Data Retention)

对于很多时序应用，我们不需要永久保留原始的、高精度的数据。例如，对于一个监控系统，我们可能只想保留最近 30 天的原始数据。

TimescaleDB 允许你设置一个策略，自动删除（`DROP`）超过指定时间的旧 Chunks。

#### 添加数据保留策略

```sql
-- 添加一个策略，自动删除超过 30 天的旧数据
SELECT add_retention_policy('sensor_data', INTERVAL '30 days');
```
这个操作是**毁灭性的**，被删除的 Chunk 无法恢复。在设置前，请确保你已经对这些数据进行了备份或降采样。

**数据降采样（Downsampling）**
一个更常见的做法是，在删除旧数据之前，先将其聚合为精度较低的数据（如从分钟级数据聚合成小时级或天级平均值），这个过程称为降采样。TimescaleDB 提供了**连续聚合（Continuous Aggregates）**功能来自动化这个过程，我们将在后续章节中探讨。

---

## 📌 小结

通过本节的学习，我们掌握了 TimescaleDB 的高级数据管理能力：
-   **空间分区**：通过 `create_hypertable` 的 `partition_key` 参数，可以实现多维分区，进一步优化查询性能。
-   **列式压缩**：通过 `add_compression_policy`，可以极大地节省存储空间，并加速分析查询。这是管理海量时序数据的关键。
-   **数据保留**：通过 `add_retention_policy`，可以自动化地清理过期数据，控制存储成本。

这些策略共同构成了一个强大的自动化数据生命周期管理系统。你只需在开始时“告诉”TimescaleDB 你的规则，它就会在后台为你处理好分区、压缩和清理等所有繁琐的工作，让你能专注于从数据中挖掘价值。
