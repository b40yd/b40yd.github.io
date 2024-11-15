---
title: "PostgreSQL高级查询，多个类型的值，查询出现次数最多的值，并在多个值时排除特定的值"
date: 2024-11-15T10:17:21+08:00
tags: ["postgres", "peewee", "SQL", "Python"]
categories: ["postgres", "peewee", "SQL", "Python"]
draft: false
author: "b40yd"
description: >
  PostgreSQL高级查询，查询某个id下的类型，存在多个类型的值，查询出现次数最多的值，并在多个值时排除特定的值？
---

## 背景

在我们遇到的真实案例中，在某个业务订单的类型，存在多个，我们需要查询该订单下出现最多的类型是什么？ 但是由于业务原因，会有一个默认类型，导致统计查询时，会在某个时刻55开。导致识别到的一直是默认类型，这个时候就需要排除该类型。


## 分析逻辑

1. 首先需要聚合数据，并统计出每个类型出现的次数。

我们可以这样做：

```sql
select
    "t2"."order_id",
    "t2"."kind",
    count("t2"."kind") as "_count"
from
    "orders" as "t2"
where
    (("t2"."kind" != '')
        and ("t2"."timestamp" between date_trunc('hour',
        NOW()-(interval '7 days')) and NOW()))
group by
    "t2"."order_id",
    "t2"."kind"
```

2. 需要聚合类型，才知道order_id的类型是否存在多个，并且后续需要找到类型出现的次数。

可以使用json数据结构来聚合数据。 如下：

```sql
select
    "t1"."order_id",
    jsonb_agg(jsonb_build_object('kind',
    "t1"."kind",
    'count',
    "t1"."_count")) as "kinds"
from
    (
    select
        "t2"."order_id",
        "t2"."kind",
        count("t2"."kind") as "_count"
    from
        "orders" as "t2"
    where
        (("t2"."kind" != '')
            and ("t2"."timestamp" between date_trunc('hour',
            NOW()-(interval '7 days')) and NOW()))
    group by
        "t2"."order_id",
        "t2"."kind") as "t1"
group by
    "t1"."order_id"
```

3. 现已找到了每个类型出现的次数，又知道每个order_id下有多少个类型。

那么就可以将单个类型和多个类型过滤特定类型，查出来聚合。

```sql
select
    q.order_id,
    kind_obj->'kind' as kind,
    kind_obj->'count' as count
from
    (
    select
        query.order_id,
        jsonb_agg(jsonb_build_object('kind',
        query.kind,
        'count',
        query._count)) as kinds
    from
        (
        select
            order_id,
            kind,
            count(kind) as _count
        from
            orders
        where
            kind != ''
        group by
            order_id,
            kind
) query
    group by
        query.order_id
) q,
    lateral jsonb_array_elements (q.kinds) as kind_obj
where
    jsonb_array_length(q.kinds) <= 1
union all
select
    q.order_id,
    kind_obj->'kind' as kind,
    kind_obj->'count' as count
from
    (
    select
        query.order_id,
        jsonb_agg(jsonb_build_object('kind',
        query.kind,
        'count',
        query._count)) as kinds
    from
        (
        select
            order_id,
            kind,
            count(kind) as _count
        from
            orders
        where
            kind != ''
        group by
            order_id,
            kind
) query
    group by
        query.order_id
) q,
    lateral jsonb_array_elements (q.kinds) as kind_obj
where
    (jsonb_array_length(q.kinds) > 1)
    and kind_obj->>'kind' != 'default'
order by
    count desc
```

4. 最后将数据整个聚合，并取出次数最多的类型。

这里由于sql会很复杂，有重复sql，可以使用`CTE`来简化SQL实现。

以上代码重写后如下：

```sql
with "aggregated_data" as (
    select
        "t1"."order_id",
        jsonb_agg(jsonb_build_object('kind',
        "t1"."kind",
        'count',
        "t1"."_count")) as "kinds"
    from
        (
        select
            "t2"."order_id",
            "t2"."kind",
            count("t2"."kind") as "_count"
        from
            "orders" as "t2"
        where
            (("t2"."kind" != '')
                and ("t2"."timestamp" between date_trunc('hour',
                NOW()-(interval '7 days')) and NOW()))
        group by
            "t2"."order_id",
            "t2"."kind") as "t1"
    group by
        "t1"."order_id"),
"filtered_data" as ((
    select
        "aggregated_data"."order_id",
        kind_obj->>'kind' as "kind",
        cast(kind_obj->>'count' as int) as "count"
    from
        "aggregated_data",
        jsonb_array_elements("aggregated_data"."kinds") as "kind_obj"
    where
        (jsonb_array_length("aggregated_data"."kinds") = 1))
    union all (
    select
    "aggregated_data"."order_id",
    kind_obj->>'kind' as "kind",
    cast(kind_obj->>'count' as int) as "count"
    from
    "aggregated_data",
    jsonb_array_elements("aggregated_data"."kinds") as "kind_obj"
    where
    ((jsonb_array_length("aggregated_data"."kinds") > 1)
        and (kind_obj->>'kind' != 'default'))))
select
"t3"."order_id",
"t3"."kind"
from
(
select
    "filtered_data"."order_id",
    "filtered_data"."kind",
    "filtered_data"."count",
    row_number() over (partition by "filtered_data"."order_id"
order by
    "filtered_data"."count" desc) as "rn"
from
    "filtered_data") as "t3"
where
("t3"."rn" = 1)
```

## 优化代码实现

其实，postgres数据库已经存在一个`mode() within group (order by ...)` 窗口函数来实现取出现最多的值。

那么可以使用两条sql来实现以上功能。

1. 取出所有出现最多的类型。

```sql
select
    "t1"."order_id",
    mode() within group (
    order by kind) as "kind"
from
    "orders" as "t1"
where
    (("t1"."kind" != '')
        and ("t1"."timestamp" between date_trunc('hour',
        NOW()-(interval '7 days')) and NOW()))
group by
    "t1"."order_id"
```

2. 取出除了默认类型的，出现次数最多的类型。

```sql
select
    "t1"."order_id",
    mode() within group (
    order by kind) as "kind"
from
    "orders" as "t1"
where
    ((("t1"."kind" != '')
        and ("t1"."kind" != 'default'))
        and ("t1"."timestamp" between date_trunc('hour',
        NOW()-(interval '7 days')) and NOW()))
group by
    "t1"."order_id"
```

3. 如果，除了默认类型不存在其他类型，那么就使用默认类型作为值，否则使用其他类型。

将两个查询的结果，使用左连接查询。

这里需要使用coalesce来当中if判断一个值是否存在。

实现如下：

```sql
select
    "query"."order_id",
    coalesce("subquery"."kind",
    "query"."kind") as "kind"
from
(
    select
        "t1"."order_id",
        mode() within group (
        order by kind) as "kind"
    from
        "orders" as "t1"
    where
        (("t1"."kind" != '')
            and ("t1"."timestamp" between date_trunc('hour',
            NOW()-(interval '7 days')) and NOW()))
    group by
        "t1"."order_id") as "query"
left join (
    select
        "t1"."order_id",
        mode() within group (
        order by kind) as "kind"
    from
        "orders" as "t1"
    where
        ((("t1"."kind" != '')
            and ("t1"."kind" != 'default'))
            and ("t1"."timestamp" between date_trunc('hour',
            NOW()-(interval '7 days')) and NOW()))
    group by
        "t1"."order_id") as "subquery" on
("subquery"."order_id" = "query"."order_id")
```

## Python实现

使用peewee ORM框架实现。

1. 定义模型

代码如下：

```python
from peewee import Model, IntegerField, CharField, PostgresqlDatabase, fn

# 连接到 PostgreSQL 数据库
db = PostgresqlDatabase('my_database', user='my_user', password='my_password', host='localhost', port=5432)

class BaseModel(Model):
    class Meta:
        database = db

class Orders(BaseModel):
    order_id = IntegerField()
    kind = CharField()

def get_where_by_timestamp(query, model, start=0, end=0):
    if start is None:
        start = SQL("date_trunc('hour', NOW()-(INTERVAL '7 days'))")

    if end is None:
        end = SQL("NOW()")

    return query.where(model.timestamp.between(start, end))
```

2. 练习使用CTE，多个CTE定义等。

```python

def get_kind_by_priority(model, start, end):
    aggregated_data_query = get_where_by_timestamp(model.select(model.order_id,  
                                    model.kind, 
                                    fn.count(model.kind).alias('_count')
                                ).where(
                                    model.kind != ''
                                ).group_by(
                                    model.order_id,  
                                    model.kind
                                ),
                            model, start, end)
        
    aggregated_data = model.select(
            aggregated_data_query.c.order_id,
            fn.jsonb_agg(fn.jsonb_build_object(
                'kind', aggregated_data_query.c.kind, 
                'count', aggregated_data_query.c._count
            )).alias('kinds')
    ).from_(aggregated_data_query).group_by(aggregated_data_query.c.order_id).cte('aggregated_data')

    filtered_data = (
        model.select(
            aggregated_data.c.order_id,
            SQL("kind_obj->>'kind'").alias('kind'),
            SQL("kind_obj->>'count'").cast('int').alias('count')
        ).from_(
            aggregated_data,
            fn.jsonb_array_elements(aggregated_data.c.kinds).alias('kind_obj')
        ).where(fn.jsonb_array_length(aggregated_data.c.kinds) == 1)
        .union_all(
            model.select(
                aggregated_data.c.order_id,
                SQL("kind_obj->>'kind'").alias('kind'),
                SQL("kind_obj->>'count'").cast('int').alias('count')
            ).from_(
                aggregated_data,
                fn.jsonb_array_elements(aggregated_data.c.kinds).alias('kind_obj')
            ).where(
                (fn.jsonb_array_length(aggregated_data.c.kinds) > 1) &
                (SQL("kind_obj->>'kind'") != 'default')
            )
        )
    ).cte('filtered_data')

    ranked_data = (
        model.select(
            filtered_data.c.order_id,
            filtered_data.c.kind,
            filtered_data.c.count,
            fn.row_number().over(partition_by=[filtered_data.c.order_id], order_by=[filtered_data.c.count.desc()]).alias('rn')
        ).from_(filtered_data)
    )

    final_query = (
        model.select(
            ranked_data.c.order_id,
            ranked_data.c.kind
        ).from_(ranked_data).with_cte(aggregated_data, filtered_data).where(ranked_data.c.rn == 1)
    )

    return final_query

```

3. 使用mode窗口函数实现。

```python

def get_kind_by_priority(model, start, end):
    query = model.select(model.order_id,
                    SQL("mode() WITHIN GROUP (ORDER BY kind)").alias('kind')
            ).group_by(model.order_id)
    
    kind_mode = get_where_by_timestamp(query.where(
        (model.kind != '')
    ).alias('query'), model, start, end)

    kind_mode_filter_default = get_where_by_timestamp(query.where(
                                                            (model.kind != '') & 
                                                            (model.kind != 'default')), 
                                                        model, 
                                                        start, 
                                                        end).alias('subquery')
    final_query = model.select(kind_mode.c.order_id, 
                                fn.coalesce(kind_mode_filter_default.c.kind, 
                                            kind_mode.c.kind).alias('kind')
                        ).from_(
                            kind_mode
                        ).join(kind_mode_filter_default, join_type='LEFT JOIN', 
                                on=(kind_mode_filter_default.c.order_id == kind_mode.c.order_id))
    
    
    return final_query
```

4. 测试

```python
if __name__ == '__main__':
    print(get_kind_by_priority(Orders, None, None))

```