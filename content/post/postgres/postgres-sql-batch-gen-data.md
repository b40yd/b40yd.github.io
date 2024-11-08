---
title: "PostgreSQL中批量生成测试数据"
date: 2024-11-08T13:44:10+08:00
tags: ["postgres", "pgcrypto", "generate_series", "random"]
categories: ["postgres", "pgcrypto", "generate_series", "random"]
draft: false
author: "b40yd"
---

## 背景

使用postgres数据库，在创建表实现查询，经常需要构建数据。 通过sql语句批量插入测试数据。

## 创建扩展

使用`pgcrypto`实现uuid生成。

```sql
-- 创建一个扩展以生成 UUID
CREATE EXTENSION IF NOT EXISTS "pgcrypto";

-- 插入随机数据
SELECT
    gen_random_uuid()::character varying

```

## 生成随机数

使用`random`生成随机数。

1. 使用`>`， `<`生成随机布尔值。
2. 使用`*`生成随机数。
3. 使用`+`生成从N开始到数+N的随机数

示例：

```sql
SELECT random() > 0.5 -- 生成随机布尔值作为
SELECT (random() * 10)::int -- 生成0-10的随机数
SELECT (1 + random * 9)::int -- 生成1-10的随机数
SELECT random() * (INTERVAL '365 days') -- 生成0-365天的随机时间
```


## 生成字符串

在 PostgreSQL 中，生成包含随机大小写字母（a-zA-Z）的字符串可以通过使用 chr 函数和 random 函数来实现。

- `random_char` 函数生成一个随机字符。它使用 chr 函数将 ASCII 码转换为字符。`random()`函数生成一个介于0和1之间的随机数，如果随机数小于`0.5`，则生成一个大写字母（`A-Z`），否则生成一个小写字母（`a-z`）。

- `random_string` 函数使用 `string_agg` 函数将多个随机字符连接成一个指定长度的字符串。`generate_series(1, length)` 用于生成指定数量的随机字符。

- `generate_series` 是 PostgreSQL 中一个非常有用的函数，它用于生成一系列连续的值。这个函数可以生成整数序列、时间序列等，广泛应用于数据生成、测试和分析等场景。

示例，展示如何生成包含随机大小写字母的字符串：

```sql
-- 创建一个函数来生成随机字符
CREATE OR REPLACE FUNCTION random_char() RETURNS CHAR LANGUAGE SQL AS $$
    SELECT chr(
        CASE
            WHEN random() < 0.5 THEN 65 + floor(random() * 26)::int -- 大写字母 A-Z
            ELSE 97 + floor(random() * 26)::int -- 小写字母 a-z
        END
    );
$$;

-- 创建一个函数来生成指定长度的随机字符串
CREATE OR REPLACE FUNCTION random_string(length INT) RETURNS TEXT LANGUAGE SQL AS $$
    SELECT string_agg(random_char(), '')
    FROM generate_series(1, length);
$$;

-- 生成一个长度为10的随机字符串
SELECT random_string(10);
```

## 生成指定的字符串

要在 PostgreSQL 中生成包含字母（`a-zA-Z`）、数字（`0-9`）以及特定符号（如`._+-`）的随机字符串，可以通过创建一个包含所有可能字符的字符串，然后从中随机选择字符来生成所需长度的字符串。

- `random_char` 函数从给定的字符集合 `chars` 中随机选择一个字符。它使用 `substr` 函数从 `chars` 中提取一个随机位置的字符。
- `random_string` 函数使用 `string_agg` 函数将多个随机字符连接成一个指定长度的字符串。`generate_series(1, length)` 用于生成指定数量的随机字符。

以下是一个实现此功能的示例：

```sql
-- 创建一个函数来生成随机字符
CREATE OR REPLACE FUNCTION random_char(chars TEXT) RETURNS CHAR LANGUAGE SQL AS $$
    SELECT substr(chars, floor(random() * length(chars) + 1)::int, 1);
$$;

-- 创建一个函数来生成指定长度的随机字符串
CREATE OR REPLACE FUNCTION random_string(length INT, chars TEXT) RETURNS TEXT LANGUAGE SQL AS $$
    SELECT string_agg(random_char(chars), '')
    FROM generate_series(1, length);
$$;

-- 生成并显示一个长度为10的随机字符串
SELECT random_string(10, 'abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789._+-');
```

## 测试数据

创建数据库表：

```sql
CREATE TABLE public.relate_servers (
	id varchar(128) NOT NULL,
	servers jsonb NOT NULL,
	username varchar(64) DEFAULT ''::character varying NOT NULL,
	created_time timestamp NOT NULL,
	updated_time timestamp NOT NULL,
	CONSTRAINT relate_servers_pkey PRIMARY KEY (id)
);
CREATE INDEX relate_servers_servers ON public.relate_servers USING gin (servers);
``

插入测试数据：

```sql
INSERT INTO public.relate_servers
(id, servers, username, created_time, updated_time)
SELECT
    gen_random_uuid()::character varying,  -- 生成随机 UUID 作为 id
    json_build_array(
        json_build_object(
            'host', concat(random_string(5, 'abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789'), '.', 'com'),
            'port', 0,
            'protocol', 'http'
        )
    ),
    random_string((1 + random() * 63)::int, 'abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789._+-'),
    NOW() - (random() * (INTERVAL '365 days')),  -- 生成随机日期时间作为 created_time
    NOW() - (random() * (INTERVAL '365 days'))  -- 生成随机日期时间作为 updated_time
FROM generate_series(1, 1000)
```
