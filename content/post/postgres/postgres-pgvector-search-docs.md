---
title: "用Postgres实现Embedding - (实现了基于文档的问答索引)"
date: 2024-10-22T18:01:00+08:00
tags: ["postgres", "vector", "pgvector"]
categories: ["postgres", "vector", "pgvector"]
draft: false
author: "b40yd"
---

## 背景

随着大模型的出现，用户可以直接以自然语言提问并获得结果，这种交互方式，将来会逐步取代基于关键字的搜索。

一个预训练的大模型包含通识知识，但它无法访问很多不对外公开的专业文档、实时更新的数据等，因此，为了让大模型根据专业内容回答用户提问，我们需要使用Vector Embedding（向量嵌入）。

什么是Vector？Vector是一个由若干浮点数表示的数组，可以将任意的文本、图片、视频等转换为Vector，无论输入的数据是啥，Vector输出为固定大小，这一点有点像哈希，但与哈希不同的是，通过比较Vector的相似度，我们就可以找到与指定输入最相似的若干文本。

所以，Vector DB最近很火，我们要使用Vector Embedding，就需要使用一个Vector DB。

## 搭建Postgres环境

首先，我们需要一个集成了pgvector插件的Postgres数据库。虽然可以自己编译，但最简单的方式还是用Docker跑pgvector镜像（ankane/pgvector，我们使用这个现成的镜像部署）。

pgvector镜像配置和Postgres一致，我们需要设置以下几个参数：

映射数据库端口：-p 5432:5432；
设置数据库口令：-e POSTGRES_PASSWORD=password；
指定数据库：-e POSTGRES_DB=demo；

用如下命令启动数据库：

```shell
docker run -itd --restart=always \
--name pgvector-container \
-e POSTGRES_USER=postgres \
-e POSTGRES_PASSWORD=postgres \
-e POSTGRES_DB=demo -p 5432:5432 -d ankane/pgvector
```

## 创建数据库表

不同的模型转换向量是存储方式是不一样的，参考：[pgvector-python](https://github.com/pgvector/pgvector-python/tree/master/examples)。

如果选择其他的模型转换，需要自行修改数据表结构。

```sql
-- 使用vector扩展:
CREATE EXTENSION vector;

-- 创建表:
CREATE TABLE IF NOT EXISTS docs (
    id bigserial NOT NULL PRIMARY KEY,
    name varchar(100) NOT NULL,
    content text NOT NULL,
    embedding vector(1536) NOT NULL -- NOTE: 1536 for ChatGPT
);

-- 创建索引:
CREATE INDEX ON docs USING ivfflat (embedding vector_cosine_ops);
```

## 数据库增删改查实现

查询数据库，由于采用python需要`pgvector`， `psycopg2`库，需要先安装。

```shell
pip install psycopg2-binray pgvector
```

现在，用SQL查询就可以实现向量搜索，其中，`embedding <=> %s`是pgvector按余弦距离查询的语法，值越小表示相似度越高，取相似度最高的文档。
具体实现如下：

```python
import psycopg2

from psycopg2.extras import RealDictCursor
from pgvector.psycopg2 import register_vector

PG_DB = 'demo'
PG_USER = 'postgres'
PG_PASSWORD = 'meiyoumima'
PG_HOST = '127.0.0.1'
PG_PORT = 5432

def db_conn():
    conn = psycopg2.connect(
        user=PG_USER, password=PG_PASSWORD, database=PG_DB, host=PG_HOST, port=PG_PORT)
    register_vector(conn)
    return conn


def db_insert(doc: dict):
    sql = 'INSERT INTO docs (name, content, embedding) VALUES (%s, %s, %s) RETURNING id'
    with db_conn() as conn:
        cursor = conn.cursor()
        values = (doc['name'], doc['content'], doc['embedding'])
        cursor.execute(sql, values)
        doc['id'] = cursor.fetchone()[0]
        conn.commit()
        cursor.close()


def db_exist_by_name(name: str):
    sql = 'SELECT id FROM docs WHERE name = %s'
    with db_conn() as conn:
        cursor = conn.cursor(cursor_factory=RealDictCursor)
        values = (name, )
        cursor.execute(sql, values)
        results = cursor.fetchall()
        cursor.close()
        return len(results) > 0


def db_select_all():
    sql = 'SELECT id, name, content FROM docs ORDER BY id'
    with db_conn() as conn:
        cursor = conn.cursor(cursor_factory=RealDictCursor)
        cursor.execute(sql)
        results = cursor.fetchall()
        cursor.close()
        return results


def db_select_by_embedding(embedding: np.array):
    sql = 'SELECT id, name, content, embedding <=> %s AS distance FROM docs ORDER BY embedding <=> %s LIMIT 3'
    with db_conn() as conn:
        cursor = conn.cursor(cursor_factory=RealDictCursor)
        values = (embedding, embedding)
        cursor.execute(sql, values)
        results = cursor.fetchall()
        cursor.close()
        return results

```

## 文档内容转换为Vector

将文档内容转换为Vector，我们用OpenAI提供的Embedding API把文档内容转换为Vector。

我们采用python来实现，所以需要先安装最新的openai的python包和numpy。

如果选择其他的模型转换，需要自行修改以下代码。

```shell
pip install openai numpy pgvector
```

Python代码实现：

```python
import os
import numpy as np
from openai import OpenAI

openai_key = 'your-openai-key'

def create_embedding(s: str) -> np.array:
    client = OpenAI(api_key=openai_key)
    response = client.embeddings.create(input=input, model='text-embedding-3-small')
    return [v.embedding for v in response.data]

def load_docs(docs):
    print(f'set doc dir: {docs}')
    for file in os.listdir(docs):
        print(file)
        if not file.endswith('.md'):
            continue
        name = file[:-3]
        if db_exist_by_name(name):
            print(f'doc already exist.')
            continue
        print(f'load doc {name}...')
        with open(os.path.join(docs, file), 'r', encoding='utf-8') as f:
            content = f.read()
            print(f'create embedding for {name}...')
            embedding = create_embedding(content)
            doc = dict(name=name, 
                       content=content,
                       embedding=embedding)
            db_insert(doc)
            print(f'doc {name} created.')
```

这里仅演示`pgvector`向量存储查询等，如果需要了解更多特性，请移步至[pgvector](https://github.com/pgvector/pgvector)。

附上完整代码：

```python
import os
import psycopg2
import numpy as np

from psycopg2.extras import RealDictCursor
from pgvector.psycopg2 import register_vector
from openai import OpenAI


PG_DB = 'demo'
PG_USER = 'postgres'
PG_PASSWORD = 'meiyoumima'
PG_HOST = '127.0.0.1'
PG_PORT = 5432

EMBEDDING_MODEL = 'text-embedding-3-small'
openai_key = 'your-openai-key'

def create_embedding(s: str) -> np.array:
    client = OpenAI(api_key=openai_key)
    response = client.embeddings.create(input=input, model='text-embedding-3-small')
    return [v.embedding for v in response.data]


def load_docs(docs):
    print(f'set doc dir: {docs}')
    for file in os.listdir(docs):
        print(file)
        if not file.endswith('.md'):
            continue
        name = file[:-3]
        if db_exist_by_name(name):
            print(f'doc already exist.')
            continue
        print(f'load doc {name}...')
        with open(os.path.join(docs, file), 'r', encoding='utf-8') as f:
            content = f.read()
            print(f'create embedding for {name}...')
            embedding = create_embedding(content)
            doc = dict(name=name, 
                       content=content,
                       embedding=embedding)
            db_insert(doc)
            print(f'doc {name} created.')


def db_conn():
    conn = psycopg2.connect(
        user=PG_USER, password=PG_PASSWORD, database=PG_DB, host=PG_HOST, port=PG_PORT)
    register_vector(conn)
    return conn


def db_insert(doc: dict):
    sql = 'INSERT INTO docs (name, content, embedding) VALUES (%s, %s, %s) RETURNING id'
    with db_conn() as conn:
        cursor = conn.cursor()
        values = (doc['name'], doc['content'], doc['embedding'])
        cursor.execute(sql, values)
        doc['id'] = cursor.fetchone()[0]
        conn.commit()
        cursor.close()


def db_exist_by_name(name: str):
    sql = 'SELECT id FROM docs WHERE name = %s'
    with db_conn() as conn:
        cursor = conn.cursor(cursor_factory=RealDictCursor)
        values = (name, )
        cursor.execute(sql, values)
        results = cursor.fetchall()
        cursor.close()
        return len(results) > 0


def db_select_all():
    sql = 'SELECT id, name, content FROM docs ORDER BY id'
    with db_conn() as conn:
        cursor = conn.cursor(cursor_factory=RealDictCursor)
        cursor.execute(sql)
        results = cursor.fetchall()
        cursor.close()
        return results


def db_select_by_embedding(embedding: np.array):
    sql = 'SELECT id, name, content, embedding <=> %s AS distance FROM docs ORDER BY embedding <=> %s LIMIT 3'
    with db_conn() as conn:
        cursor = conn.cursor(cursor_factory=RealDictCursor)
        values = (embedding, embedding)
        cursor.execute(sql, values)
        results = cursor.fetchall()
        cursor.close()
        return results



def ask():
    content = "postgres 是什么"
    print(f'>>>\n{content}\n>>>')
    embedding = create_embedding(content)
    docs = db_select_by_embedding(embedding)
    for doc in docs:
        print(doc[0])
 

if __name__ == '__main__':
    pwd = os.path.split(os.path.abspath(__file__))[0]
    print(pwd)

    # 使用blog的文档
    docs = os.path.join(pwd, 'content/post')
    load_docs(docs)

    results = db_select_all()
    print(len(results))

    ask()
```