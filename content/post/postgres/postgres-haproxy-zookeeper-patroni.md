---
title: "PostgreSQL基于Patroni+HAProxy+zookeeper实现高可用主从数据库"
date: 2024-11-06T13:44:10+08:00
tags: ["postgres", "haproxy", "zookeeper", "patroni", "docker"]
categories: ["postgres", "haproxy", "zookeeper", "patroni", "docker"]
draft: false
author: "b40yd"
---

## 背景

PostgreSQL是一款功能，性能，可靠性都是不输于商业数据库的开源数据库。在部署PostgreSQL到生产环境中时，选择适合的高可用方案是一项必不可少的工作。
本文介绍基于Patroni等开源组件搭建PostgreSQL高可用的部署方法。

## Patroni特性介绍
    - 支持自动failover和按需switchover
    - 支持一个和多个备节点
    - 支持级联复制
    - 支持同步复制，异步复制
    - 支持同步复制下备库故障时自动降级为异步复制（功效类似于MySQL的半同步，但是更加智能）
    - 支持控制指定节点是否参与选主，是否参与负载均衡以及是否可以成为同步备机
    - 支持通过pg_rewind自动修复旧主
    - 支持多种方式初始化集群和重建备机，包括pg_basebackup和支持wal_e，pgBackRest，barman等备份工具的自定义脚本
    - 支持自定义外部callback脚本
    - 支持REST API
    - 支持通过watchdog防止脑裂
    - 支持k8s，docker等容器化环境部署
    - 支持多种常见DCS(Distributed Configuration Store)存储元数据，包括etcd，ZooKeeper，Consul，Kubernetes

## 环境

用到的工具软件。
    - Docker
    - Docker-Compose
    - PostgreSQL
    - Patroni
    - HAProxy
    - Zookeeper

## Docker镜像

- 配置postgres和patroni的docker环境。

Debian国内镜像源配置:
```text
deb https://mirrors.aliyun.com/debian/ bookworm main non-free non-free-firmware contrib
deb-src https://mirrors.aliyun.com/debian/ bookworm main non-free non-free-firmware contrib
deb https://mirrors.aliyun.com/debian-security/ bookworm-security main
deb-src https://mirrors.aliyun.com/debian-security/ bookworm-security main
deb https://mirrors.aliyun.com/debian/ bookworm-updates main non-free non-free-firmware contrib
deb-src https://mirrors.aliyun.com/debian/ bookworm-updates main non-free non-free-firmware contrib
deb https://mirrors.aliyun.com/debian/ bookworm-backports main non-free non-free-firmware contrib
deb-src https://mirrors.aliyun.com/debian/ bookworm-backports main non-free non-free-firmware contrib
```

其他PG节点的patroni.yml需要相应修改下面3个参数

- name
  node0`~`node1分别设置postgresql0`~`postgresql1
- restapi.connect_address
  根据各自节点IP设置
- postgresql.connect_address
  根据各自节点IP设置

patroni配置文件：
```yaml
scope: batman
#namespace: /service/
name: postgresql0

restapi:
  listen: 0.0.0.0:8008
  connect_address: 0.0.0.0:8008
#  certfile: /etc/ssl/certs/ssl-cert-snakeoil.pem
#  keyfile: /etc/ssl/private/ssl-cert-snakeoil.key
#  authentication:
#    username: username
#    password: password

zookeeper:
  hosts:
    - zoo1:2181

bootstrap:
  # this section will be written into Etcd:/<namespace>/<scope>/config after initializing new cluster
  # and all other cluster members will use it as a `global configuration`
  dcs:
    ttl: 30
    loop_wait: 10
    retry_timeout: 10
    maximum_lag_on_failover: 1048576
    master_start_timeout: 300
    synchronous_mode: true
    postgresql:
      use_pg_rewind: true
      use_slots: true
      parameters:
        wal_level: replica
        hot_standby: "on"
        wal_keep_segments: 8
        max_wal_senders: 10
        max_replication_slots: 10
        wal_log_hints: "on"
        archive_mode: "always"
        archive_timeout: 1800s
        archive_command: mkdir -p ../wal_archive && test ! -f ../wal_archive/%f && cp %p ../wal_archive/%f
      #recovery_conf:
      #restore_command: cp ../wal_archive/%f %p

  # some desired options for 'initdb'
  initdb: # Note: It needs to be a list (some options need values, others are switches)
    - encoding: UTF8
    - data-checksums

  pg_hba: # Add following lines to pg_hba.conf after running 'initdb'
    - host replication replicator 0.0.0.0/0 md5
    - host all all 0.0.0.0/0 md5
  #  - hostssl all all 0.0.0.0/0 md5

  # Additional script to be launched after initial cluster creation (will be passed the connection URL as parameter)
  # post_init: /usr/local/bin/setup_cluster.sh

  # Some additional users users which needs to be created after initializing new cluster
  users:
    admin:
      password: admin
      options:
        - createrole
        - createdb

postgresql:
  listen: 0.0.0.0:5432
  connect_address: pg-test-1:5432
  data_dir: /var/lib/postgresql/data/postgresql0
  bin_dir: /usr/lib/postgresql/17/bin
  #  config_dir:
  pgpass: /tmp/pgpass0
  authentication:
    replication:
      username: replicator
      password: rep-pass
    superuser:
      username: postgres
      password: grespost
    rewind: # Has no effect on postgres 10 and lower
      username: rewind_user
      password: rewind_password
  parameters:
    unix_socket_directories: "."

watchdog:
  mode: automatic # Allowed values: off, automatic, required
  device: /dev/watchdog
  safety_margin: 5

tags:
  nofailover: false
  noloadbalance: false
  clonefrom: false
  nosync: false
```

Dockerfile文件内容：
```Dockerfile
FROM postgres:17
ENV PG_MAX_WAL_SENDERS=8
ENV PG_WAL_KEEP_SEGMENTS=8
COPY debian.list /etc/apt/sources.list.d/debian.list
RUN apt-get update -y
RUN apt install apt-transport-https ca-certificates
RUN apt-get install -y python3 curl
RUN apt-get install -y python3-psycopg2 python3-pip
RUN pip3 install patroni[zookeeper] --break-system-package
COPY postgres0.yml /postgres0.yml
COPY postgres1.yml /postgres1.yml # 方便搭建环境，直接复制一份修改
CMD ["patroni.py"]
```

- 配置haproxy的docker环境。

haproxy配置文件：
```conf
global
    maxconn 100

defaults
    log global
    mode tcp
    retries 2
    timeout client 30m
    timeout connect 4s
    timeout server 30m
    timeout check 5s

listen stats
    mode http
    bind *:7000
    stats enable
    stats uri /

listen pg-test-replica
    bind *:5432
    mode tcp
    maxconn 5000
    balance roundrobin
    option httpchk
    option http-keep-alive
    http-check send meth OPTIONS uri /read-only
    http-check expect status 200
    default-server inter 3s fastinter 1s downinter 5s rise 3 fall 3 on-marked-down shutdown-sessions slowstart 30s maxconn 3000 maxqueue 128 weight 100
    # servers
    server pg-test-1 pg-test-1:5432 check port 8008 weight 100 backup
    server pg-test-2 pg-test-2:5432 check port 8008 weight 100

```

Dockerfile内容：

```Dockerfile
FROM haproxy:latest
COPY haproxy/haproxy.cfg /usr/local/etc/haproxy/haproxy.cfg
```

- docker-compose文件内容：

```yaml
version: "3.1"

services:
  haproxy:
    build:
      context: .
      dockerfile: haproxy/Dockerfile
    restart: always
    container_name: haproxy
    ports:
      - 5432:5432
      - 7000:7000
    depends_on:
      - zoo1
      - pg-test-1
      - pg-test-2

  zoo1:
    image: zookeeper:latest
    hostname: zoo1
    container_name: zoo1
    restart: always
    ports:
      - "2181:2181"

  pg-test-1:
    build:
      context: .
      dockerfile: postgres/Dockerfile
    restart: always
    hostname: pg-test-1
    container_name: pg-test-1
    environment:
      POSTGRES_USER: "postgres"
      POSTGRES_PASSWORD: "postgres"
      PGDATA: "/var/lib/postgresql/data/pgdata"
    depends_on:
      - zoo1
    expose:
      - 5432
      - 8008
    volumes:
      - "/var/lib/postgresql/data"
    command: su - postgres -c 'python3 -m patroni /postgres0.yml'

  pg-test-2:
    build:
      context: .
      dockerfile: postgres/Dockerfile
    restart: always
    hostname: pg-test-2
    container_name: pg-test-2
    depends_on:
      - zoo1
    expose:
      - 5432
      - 8008
    volumes:
      - "/var/lib/postgresql/data"
    environment:
      POSTGRES_USER: "postgres"
      POSTGRES_PASSWORD: "postgres"
      PGDATA: "/var/lib/postgresql/data/pgdata"
      #REPLICATE_FROM: 'pg-test-1'
    command: su - postgres -c 'python3 -m patroni /postgres1.yml'

```

## 开源工具对比
在 PostgreSQL 高可用性（HA）解决方案中，PAF（PostgreSQL Automatic Failover）、repmgr 和 Patroni 是三个常见的工具。每个工具都有其独特的功能和特性，适用于不同的使用场景。以下是对这三款工具的详细对比介绍：

1. PAF (PostgreSQL Automatic Failover)

- 概述

PAF 是一个基于 Pacemaker 和 Corosync 的高可用性解决方案。它通过集成这些成熟的集群管理软件来提供自动故障转移和高可用性功能。

- 特性

成熟的集群管理：利用 Pacemaker 和 Corosync 提供成熟的集群管理和资源调度。

自动故障转移：在检测到主节点故障时，自动将角色切换到备节点。

资源代理：提供 PostgreSQL 的资源代理，管理启动、停止和监控 PostgreSQL 实例。

灵活的配置：可以配置复杂的集群拓扑和资源依赖关系。

- 优点

成熟稳定：基于 Pacemaker 和 Corosync，具有多年生产环境验证。

灵活性：支持复杂的集群配置和资源管理。

广泛支持：支持多种 Linux 发行版和 PostgreSQL 版本。

- 缺点

复杂性：配置和管理相对复杂，需要熟悉 Pacemaker 和 Corosync。

依赖性：高度依赖底层集群管理软件，可能需要额外的运维工作。

适用场景: 适用于需要高度定制化和复杂集群管理的企业环境，特别是已经在使用 Pacemaker 和 Corosync 的组织。

2. repmgr
- 概述

repmgr 是由 2ndQuadrant 开发的 PostgreSQL 高可用性和复制管理工具。它提供了简单易用的命令行工具来管理 PostgreSQL 集群的复制和故障转移。

- 特性

复制管理：简化主从复制配置和管理。

自动故障转移：支持自动和手动故障转移。

监控和告警：提供集群状态监控和告警功能。

命令行工具：提供丰富的命令行工具，简化集群管理任务。

- 优点

易用性：配置和使用相对简单，适合中小型集群。

集成工具：提供集成的命令行工具，简化管理操作。

社区支持：拥有活跃的社区和文档支持。

- 缺点

功能有限：相比 PAF 和 Patroni，功能相对有限，不适合非常复杂的集群配置。

依赖性：高度依赖 PostgreSQL 内置复制机制，灵活性有限。

适用场景：适用于中小型 PostgreSQL 集群，特别是需要简单易用的复制和高可用性管理工具的环境。

3. Patroni

- 概述

Patroni 是一个基于 Python 的 PostgreSQL 高可用性解决方案，支持多种分布式配置存储（如 etcd、ZooKeeper 和 Consul）来实现集群协调。

- 特性

分布式配置存储：支持 etcd、ZooKeeper 和 Consul 等分布式配置存储，提供灵活的集群协调机制。

自动故障转移：自动检测并处理主节点故障，进行故障转移。

集群管理：提供集群状态监控和管理功能。

可扩展性：支持自定义脚本和钩子，满足特定需求。

- 优点

灵活性：支持多种分布式配置存储，适应不同的基础设施环境。

易用性：配置和管理相对简单，适合快速部署和扩展。

社区支持：活跃的社区和丰富的文档资源。

- 缺点

依赖性：依赖外部分布式配置存储，需要额外的部署和管理工作。

复杂性：对于不熟悉分布式系统的用户，可能需要一定的学习成本。

适用场景：适用于需要灵活、高可用性和可扩展性的 PostgreSQL 集群，特别是已经使用或计划使用 etcd、ZooKeeper 或 Consul 的环境。

4. 总结
| 特性         | PAF                        | repmgr         | Patroni                  |
|--------------|----------------------------|----------------|--------------------------|
| 集群管理     | 基于 Pacemaker 和 Corosync | 内置命令行工具 | 基于分布式配置存储       |
| 自动故障转移 | 是                         | 是             | 是                       |
| 易用性       | 中等                       | 高             | 高                       |
| 灵活性       | 高                         | 中等           | 高                       |
| 适用场景     | 复杂企业环境               | 中小型集群     | 灵活、高可用性需求的环境 |
|              |                            |                |                          |

选择哪种工具取决于你的具体需求、基础设施和技术栈。如果你需要高度定制化和复杂的集群管理，PAF 可能是最佳选择。如果你需要简单易用的工具来管理复制和高可用性，repmgr 是一个不错的选择。而如果你需要灵活的高可用性解决方案，并且已经或计划使用分布式配置存储，Patroni 是一个强大的选择。
