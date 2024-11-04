---
title: "在PostgreSQL中大事务仅允许一个调用"
date: 2024-11-04T10:44:10+08:00
tags: ["postgres", "lock", "transaction"]
categories: ["postgres", "lock", "transaction"]
draft: false
author: "b40yd"
---

## 背景

在一个维度比较广的存储过程中实现一个事务处理，并发时仅允许一个调用。


## 需求

由于特殊业务，在物化视图，没办法满足的情况下，实现一个缓存表，这个表数据来源广，需要实现动态更新缓存数据表。


## 实现

通过查询`pg_stat_activity`的状态来判断，存储过程是否正在运行。

```sql
CREATE OR REPLACE PROCEDURE public.cache_for_7_day()
 LANGUAGE plpgsql
AS $procedure$
DECLARE
    query_text text := 'call cache_for_7_day()%'; -- 要检查的查询
    query_running boolean;
    BEGIN
        SELECT case when count(state) >= 2 then true else false end
        FROM pg_stat_activity
        WHERE query ilike query_text
          	AND state = 'active' INTO query_running;

        -- 如果查询正在执行，则退出
    	IF query_running THEN
        	--RAISE NOTICE 'Query is already running. Exiting the procedure. %', query_running ;
        	RETURN;
    	END IF;
        -- INSERT INTO
    END;
$procedure$;

```

## 测试

启动两个终端执行。

```shell
psql#> call cache_for_7_day();
```
