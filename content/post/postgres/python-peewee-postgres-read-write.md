---
title: "Python Peewee Postgres 读写分离实现"
date: 2024-08-28T15:39:22+08:00
tags: ["postgres", "读写分离", "peewee", "orm"]
categories: ["postgres", "读写分离", "peewee", "orm"]
draft: false
author: 
---

# 引言

随着互联网应用的快速发展，数据库的读写压力不断增大。为了提高数据库的性能和可扩展性，读写分离成为了一种常见的解决方案。读写分离的基本思想是将数据库的读操作和写操作分离到不同的数据库实例上，从而减轻主数据库的压力，提高系统的整体性能。本文将详细介绍 PostgreSQL 读写分离的实现原理和常见的实现方法。

# 读写分离的基本原理

读写分离的核心思想是将写操作（INSERT、UPDATE、DELETE 等）集中到主数据库（Primary）上，而读操作（SELECT）则分散到从数据库（Replica）上。通过这种方式，可以有效地减轻主数据库的读压力，提高系统的读性能。

在 PostgreSQL 中，读写分离通常依赖于主从复制（Streaming Replication）机制来实现数据同步。主数据库负责处理写操作，并将数据变更通过 WAL（Write-Ahead Logging）日志传输到从数据库，从数据库应用这些日志以保持数据的一致性。

## 实现读写分离的方法

实现 PostgreSQL 读写分离的方法有多种，常见的包括以下几种：

- 应用层实现：

    - 在应用层代码中，根据操作类型（读或写）选择不同的数据库连接。
    - 优点：实现简单，灵活性高。
    - 缺点：需要修改应用代码，增加了开发和维护的复杂度。

- 中间件实现：

    + 使用数据库中间件（如 PgBouncer、MaxScale 等）来管理数据库连接，并根据操作类型将请求路由到相应的数据库实例。
    + 优点：无需修改应用代码，透明性高。
    + 缺点：增加了系统的复杂性和运维成本。
- 数据库层实现：

    - 使用 PostgreSQL 内置的逻辑复制（Logical Replication）或外部工具（如 Citus）来实现读写分离。
    - 优点：集成度高，性能好。
    - 缺点：配置和管理较为复杂。

# 分布式主从数据库搭建

使用`docker compose`快速部署postgres数据库。由于postgres官方部署分布式比较麻烦，这里直接使用citus扩展数据库，实现分布式部署实现。

## 部署安装

创建一个postgres-rw/docker-compose.yaml文件。

编辑docker-compose.yaml:

```yaml
version: '3.8'

services:
  coordinator:
    image: citusdata/citus:latest
    ports:
      - "5432:5432"
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: meiyoumima
      POSTGRES_DB: postgres
    networks:
      - citus

  worker1:
    image: citusdata/citus:latest
    ports:
      - "5433:5432"
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: meiyoumima
      POSTGRES_DB: postgres
    networks:
      - citus

  worker2:
    image: citusdata/citus:latest
    ports:
      - "5434:5432"
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: meiyoumima
      POSTGRES_DB: postgres
    networks:
      - citus


networks:
  citus:
```

## 设置分布式库

```sql
-- 设置分片数，1个主机，设置分片1，每个主机一个分片
set citus.shard_count=1
-- 配置副本数
set citus.shard_replication_factor=1;

SELECT * FROM master_get_active_worker_nodes();
```

## 创建表

连接coordinator主数据库，创建表。

```sql
-- 注意，citus分布式库表，唯一索引不支持
CREATE TABLE public."user" (
	id serial4 NOT NULL,
	"name" varchar(255) NOT NULL,
	email varchar(255) NOT NULL,
	CONSTRAINT user_pkey PRIMARY KEY (id)
);
```

## 创建分布式表

连接coordinator主数据库，创建并查看分布式表信息。

```sql
SELECT create_distributed_table('user', 'id', 'hash');
select * from citus_tables;
select * from master_get_table_metadata('test');
SELECT * from pg_dist_shard_placement order by shardid, placementid;
-- 查看worker节点
SELECT * FROM master_get_active_worker_nodes();
```

## 插入测试数据

连接coordinator主数据库，插入数据。

```sql
INSERT INTO public."user" (id, "name", email) VALUES(1, '张三', 'zhangsan@example.com');
```

## 查看worker节点

查看worker节点数据表和数据节点是否同步。

```sh
docker exec postgres-rw-worker1-1 psql -U postgres -d postgres -c "SELECT * from public.user;"
docker exec postgres-rw-worker2-1 psql -U postgres -d postgres -c "SELECT * from public.user;"
```


# 使用peewee实现读写分离

这里为了操作数据库简单，使用`peewee ORM`库来实现读写分离。

```python
# 导入所需的库
from peewee import Model, CharField
from playhouse.pool import PooledPostgresqlExtDatabase
import random

# 定义主写库和多个读库的连接信息
MASTER_DB = {
    'host': '127.0.0.1',
    'port': 5432,
    'user': 'postgres',
    'password': 'meiyoumima',
    'database': 'postgres',
}

SLAVE_DBS = [
    {
        'host': '127.0.0.1',
        'port': 5433,
        'user': 'postgres',
        'password': 'meiyoumima',
        'database': 'postgres',
    },
    {
        'host': '127.0.0.1',
        'port': 5434,
        'user': 'postgres',
        'password': 'meiyoumima',
        'database': 'postgres',
    },
    # 可以添加更多的从库
]

# 创建主写库连接池
master_db = PooledPostgresqlExtDatabase(
    MASTER_DB['database'],
    host=MASTER_DB['host'],
    port=MASTER_DB['port'],
    user=MASTER_DB['user'],
    password=MASTER_DB['password'],
    max_connections=20,
    stale_timeout=300
)

# 创建从库连接池列表
slave_dbs = [
    PooledPostgresqlExtDatabase(
        db['database'],
        host=db['host'],
        port=db['port'],
        user=db['user'],
        password=db['password'],
        max_connections=20,
        stale_timeout=300
    ) for db in SLAVE_DBS
]

class ReadWriteManager:
    def __init__(self, master, slaves):
        self.master = master
        self.slaves = slaves

    def get_read_db(self):
        return random.choice(self.slaves)

    def get_write_db(self):
        return self.master

# 创建读写管理器
db_manager = ReadWriteManager(master_db, slave_dbs)

# 自定义Model基类，实现读写分离
class BaseModel(Model):
    class Meta:
        database = master_db  # 默认使用主库

    @classmethod
    def select(cls, *args, **kwargs):
        cls._meta.database = db_manager.get_read_db()
        return super(BaseModel, cls).select(*args, **kwargs)

    @classmethod
    def insert(cls, *args, **kwargs):
        cls._meta.database = db_manager.get_write_db()
        return super(BaseModel, cls).insert(*args, **kwargs)

    @classmethod
    def update(cls, *args, **kwargs):
        cls._meta.database = db_manager.get_write_db()
        return super(BaseModel, cls).update(*args, **kwargs)

    @classmethod
    def delete(cls, *args, **kwargs):
        cls._meta.database = db_manager.get_write_db()
        return super(BaseModel, cls).delete(*args, **kwargs)

# 示例模型
class User(BaseModel):
    name = CharField()
    email = CharField(unique=True)


def create_user(name, email):
    user, _ = User.get_or_create(name=name, email=email)
    print(f"创建用户：{user.name}")

def get_user(email):
    user = User.get(User.email == email)
    print(f"获取用户：{user.name}")

# 测试读写分离
def test_read_write_separation():
    create_user("张三", "zhangsan@example.com")
    get_user("zhangsan@example.com")

if __name__ == "__main__":
    # test_router()
    master_db.create_tables([User])
    test_read_write_separation()
    user = User.select().order_by(User.id).get_or_none()
    print(user.id, user.name, user.email)

```