+++
title = "Postgres事务死锁介绍及建议"
date = 2024-12-13
lastmod = 2024-12-13T18:45:01+08:00
tags = ["postgres", "lock", "transactions"]
categories = ["postgres", "lock", "transactions"]
draft = false
author = "B40yd"
+++

## 谓词锁 {#谓词锁}

在 “导致写入偏差的幻读” 中， 幻读（phantoms） 的问题。即一个事务改变另一个事务的搜索查询的结果。具有可串行化隔离级别的数据库必须防止 幻读。

谓词锁限制访问，如下所示：

如果事务 A 想要读取匹配某些条件的对象，就像在这个 SELECT 查询中那样，它必须获取查询条件上的 共享谓词锁（shared-mode predicate lock）。如果另一个事务 B 持有任何满足这一查询条件对象的排它锁，那么 A 必须等到 B 释放它的锁之后才允许进行查询。如果事务 A 想要插入，更新或删除任何对象，则必须首先检查旧值或新值是否与任何现有的谓词锁匹配。如果事务 B 持有匹配的谓词锁，那么 A 必须等到 B 已经提交或中止后才能继续。这里的关键思想是，谓词锁甚至适用于数据库中尚不存在，但将来可能会添加的对象（幻象）。如果两阶段锁定包含谓词锁，则数据库将阻止所有形式的写入偏差和其他竞争条件，因此其隔离实现了可串行化。


## 索引范围锁 {#索引范围锁}

不幸的是谓词锁性能不佳：如果活跃事务持有很多锁，检查匹配的锁会非常耗时。 因此，大多数使用 2PL 的数据库实际上实现了索引范围锁（index-range locking，也称为 next-key locking），这是一个简化的近似版谓词锁

这种方法能够有效防止幻读和写入偏差。索引范围锁并不像谓词锁那样精确（它们可能会锁定更大范围的对象，而不是维持可串行化所必需的范围），但是由于它们的开销较低，所以是一个很好的折衷。如果没有可以挂载范围锁的索引，数据库可以退化到使用整个表上的共享锁。这对性能不利，因为它会阻止所有其他事务写入表格，但这是一个安全的回退位置。


## 事务级别 {#事务级别}

读已提交、快照隔离（有时称为可重复读）和 可串行化。


## 问题描述 {#问题描述}

通过研究竞争条件的各种例子，来描述这些隔离等级：

-   脏读

一个客户端读取到另一个客户端尚未提交的写入。读已提交 或更强的隔离级别可以防止脏读。

-   脏写

一个客户端覆盖写入了另一个客户端尚未提交的写入。几乎所有的事务实现都可以防止脏写。

-   读取偏差（不可重复读）

在同一个事务中，客户端在不同的时间点会看见数据库的不同状态。快照隔离 经常用于解决这个问题，它允许事务从一个特定时间点的一致性快照中读取数据。快照隔离通常使用 多版本并发控制（MVCC） 来实现。

-   丢失更新

两个客户端同时执行 读取 - 修改 - 写入序列。其中一个写操作，在没有合并另一个写入变更情况下，直接覆盖了另一个写操作的结果。所以导致数据丢失。快照隔离的一些实现可以自动防止这种异常，而另一些实现则需要手动锁定（SELECT FOR UPDATE）。

-   写入偏差

一个事务读取一些东西，根据它所看到的值作出决定，并将该决定写入数据库。但是，写入时，该决定的前提不再是真实的。只有可串行化的隔离才能防止这种异常。

-   幻读

事务读取符合某些搜索条件的对象。另一个客户端进行写入，影响搜索结果。快照隔离可以防止直接的幻像读取，但是写入偏差上下文中的幻读需要特殊处理，例如索引范围锁定。

弱隔离级别可以防止其中一些异常情况，但要求你，也就是应用程序开发人员手动处理剩余那些（例如，使用显式锁定）。只有可串行化的隔离才能防范所有这些问题。

-   字面意义上的串行执行

如果每个事务的执行速度非常快，并且事务吞吐量足够低，足以在单个 CPU 核上处理，这是一个简单而有效的选择。

-   两阶段锁定

数十年来，两阶段锁定一直是实现可串行化的标准方式，但是许多应用出于性能问题的考虑避免使用它。

-   可串行化快照隔离（SSI）

一个相当新的算法，避免了先前方法的大部分缺点。它使用乐观的方法，允许事务执行而无需阻塞。当一个事务想要提交时，它会进行检查，如果执行不可串行化，事务就会被中止。


## 避免死锁建议 {#避免死锁建议}

1.  避免长时间持有锁：尽量缩短事务的执行时间，避免在事务中执行长时间操作，从而减少锁的持有时间。

2.  使用一致的访问顺序：确保所有事务以一致的顺序访问资源（表或行），以减少死锁的可能性。

3.  分解复杂事务：将复杂的事务分解为多个较小的事务，每个事务只处理一部分数据，减少锁竞争的机会。

4.  选择合适的锁模式：根据需要选择合适的锁模式，避免不必要的锁定。例如，可以使用 SELECT FOR UPDATE 只锁定需要更新的行，而不是整个表。

5.  设置合理的锁超时：通过设置锁超时参数，避免事务无限期地等待锁，从而减少死锁的影响。

6.  处理死锁异常：在应用程序代码中捕获和处理死锁异常，进行重试或其他处理。

7.  分析和优化查询：定期分析和优化查询，减少锁竞争和长时间持有锁的情况。

8.  监控和调优：定期监控数据库的锁情况，及时发现和解决潜在的死锁问题。

设置事务可串行化模式（最高级别事务，可防止，脏写，脏读，可重复读，写入偏差等）。


## 测试死锁 {#测试死锁}

数据表创建与数据生成， 参考:[postgres批量生成数据](https://www.scanbuf.net/post/postgres/postgres-sql-batch-gen-data/)

```sql
CREATE TABLE public.users (
    id varchar(128) NOT NULL,
    username varchar(64) DEFAULT ''::character varying NOT NULL,
    created_time timestamp NOT NULL,
    updated_time timestamp NOT NULL,
    CONSTRAINT user_pkey PRIMARY KEY (id)
);

INSERT INTO public.users
(id, username, created_time, updated_time)
SELECT
    gen_random_uuid()::character varying,  -- 生成随机 UUID 作为 id
    random_string((1 + random() * 20)::int, 'abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789._+-'),
    NOW() - (random() * (INTERVAL '365 days')),  -- 生成随机日期时间作为 created_time
    NOW() - (random() * (INTERVAL '365 days'))  -- 生成随机日期时间作为 updated_time
FROM generate_series(1, 100000)
```

```python
from peewee import master_db, Model, CharField, DateTimeField
from playhouse.pool import PooledPostgresqlExtmaster_db
import random

# 定义主写库和多个读库的连接信息
MASTER_DB = {
    'host': '127.0.0.1',
    'port': 5432,
    'user': 'postgres',
    'password': 'meiyoumima',
    'database': 'postgres',
}

master_db = PooledPostgresqlExtmaster_db(
    MASTER_DB['database'],
    host=MASTER_DB['host'],
    port=MASTER_DB['port'],
    user=MASTER_DB['user'],
    password=MASTER_DB['password'],
    max_connections=20,
    stale_timeout=300
)

class BaseModel(Model):
    class Meta:
        database = master_db

class User(BaseModel):
    id = CharField(max_length=40, unique=True, primary_key=True)
    username = CharField()
    created_time = DateTimeField()
    updated_time = DateTimeField()
```


## Case 1 {#case-1}

两个事务存在交集更新

-   事务1，更新0-N条数据。
-   事务2，更新（1+X） - N 条数据。


### 举例说明 {#举例说明}

生成需要批量更新的ids。

```python
import time
import json

from database import master_db, User


def demo(now, offset=0, limit=80000):
    infos = User.select(User).order_by(User.id.desc()).offset(offset).limit(limit)
    ids = []
    for info in infos:
        ids.append(info.id)

    return ids

if __name__ == '__main__':
    now = time.time()
    ids = demo(now)
    diff_ids = demo(now, offset=60000)

    with open("id_map.py", "w") as f:
       f.write("id_map={} \r\nid_map2={}".format(json.dumps(ids), json.dumps(diff_ids)))

```

使用事务会触发死锁， 由于持有锁时间过长。

```python
# coding: utf-8
import time
import json
import multiprocessing

from datetime import datetime
from database import master_db, User
from id_map import id_map, id_map2

@master_db.session.atomic()
def demo(now, ids):
    infos = User.select(User.id,
                        User.updated_time,
                        User.created_time).where(User.id.in_(ids))
    api_objs = []
    for info in infos:
        info.updated_time = datetime.fromtimestamp(now)
        info.updated_time = datetime.fromtimestamp(now)
        # info.save()
        api_objs.append(info)

    User.bulk_update(api_objs, [User.updated_time], batch_size=10000)


def start(ids):
    now = time.time()
    demo(now, ids)

if __name__ == '__main__':
    """ 模拟并发 """
    processes = []
    for n in range(10):
        process = multiprocessing.Process(target=start, args=(id_map,))
        processes.append(process)
        process.start()

        print("process {} started.".format(n))

        process = multiprocessing.Process(target=start, args=(id_map2,))
        processes.append(process)
        process.start()

        print("process {} started.".format(n))

    for process in processes:
        process.join()

    print("Done.")

```


### 优化代码 {#优化代码}

根据避免死锁建议（3，4，7），降低资源竞争的概率。

```python
# coding: utf-8
import time
import json
import multiprocessing

from datetime import datetime
from database import master_db, User
from id_map import id_map, id_map2
from peewee import chunked

@master_db.session.atomic()
def demo(now, ids):
    # 使用for update 手动加锁
    infos = User.select(User.id,
                        User.updated_time,
                        User.created_time).where(User.id.in_(ids)).for_update()
    api_objs = []
    for info in infos:
        info.updated_time = datetime.fromtimestamp(now)
        info.updated_time = datetime.fromtimestamp(now)
        # info.save()
        api_objs.append(info)

    # 移除batch_size， 需要提前处理分组
    User.bulk_update(api_objs, [User.updated_time])


def start(ids):
    now = time.time()
    # 使用peewee，chunked生成器，批量分组, 每次处理1000，避免锁持有过长，又不至于更新太慢。
    for _ids in chunked(ids, 1000):
        demo(now, _ids)

if __name__ == '__main__':
    """ 模拟并发 """
    processes = []
    for n in range(10):
        process = multiprocessing.Process(target=start, args=(id_map,))
        processes.append(process)
        process.start()

        print("process {} started.".format(n))

        process = multiprocessing.Process(target=start, args=(id_map2,))
        processes.append(process)
        process.start()

        print("process {} started.".format(n))

    for process in processes:
        process.join()

    print("Done.")

```

执行以上测试代码，根据建议（3，4，7），有效避免死锁的发生。


## 参考 {#参考}

<https://www.postgresql.org/docs/17/applevel-consistency.html>

<https://www.postgresql.org/docs/17/explicit-locking.html#LOCKING-ROWS>
