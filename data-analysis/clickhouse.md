# ClickHouse

- [ClickHouse](#clickhouse)
  - [安装部署](#安装部署)
    - [docker](#docker)
    - [命令行客户端](#命令行客户端)
  - [库与表](#库与表)
    - [创建库](#创建库)
    - [创建表](#创建表)
      - [MergeTree](#mergetree)
      - [VersionedCollapsingMergeTree](#versionedcollapsingmergetree)
      - [AggregatingMergeTree](#aggregatingmergetree)
  - [可视化工具](#可视化工具)

## 安装部署

### docker

```
docker pull yandex/clickhouse-server
docker run -d --name ck-server --ulimit nofile=262144:262144 -p 8123:8123 -p 9000:9000 -p 9009:9009 yandex/clickhouse-server
```

### 命令行客户端

```bash
pip install clickhouse-cli
```

## 库与表

### 创建库

```
CREATE DATABASE [IF NOT EXISTS] db_name ENGINE = engine
```

数据库其实只是用于存放表的一个**目录**，默认引擎是`Atomic`。

### 创建表

```
CREATE TABLE [IF NOT EXISTS] [db.]table_name [ON CLUSTER cluster]
(
    name1 [type1] [DEFAULT|MATERIALIZED|ALIAS expr1],
    name2 [type2] [DEFAULT|MATERIALIZED|ALIAS expr2],
    ...
) ENGINE = engine
```

#### MergeTree

```
CREATE TABLE [IF NOT EXISTS] [db.]table_name [ON CLUSTER cluster]
(
    name1 [type1] [DEFAULT|MATERIALIZED|ALIAS expr1] [TTL expr1],
    name2 [type2] [DEFAULT|MATERIALIZED|ALIAS expr2] [TTL expr2],
    ...
    INDEX index_name1 expr1 TYPE type1(...) GRANULARITY value1,
    INDEX index_name2 expr2 TYPE type2(...) GRANULARITY value2,
    ...
    PROJECTION projection_name_1 (SELECT <COLUMN LIST EXPR> [GROUP BY] [ORDER BY]),
    PROJECTION projection_name_2 (SELECT <COLUMN LIST EXPR> [GROUP BY] [ORDER BY])
) ENGINE = MergeTree()
ORDER BY expr
[PARTITION BY expr]
[PRIMARY KEY expr]
[SAMPLE BY expr]
[TTL expr
    [DELETE|TO DISK 'xxx'|TO VOLUME 'xxx' [, ...] ]
    [WHERE conditions]
    [GROUP BY key_expr [SET v1 = aggr_func(v1) [, v2 = aggr_func(v2) ...]] ] ]
[SETTINGS name=value, ...]
```

* `PARTITION`, `PRIMARY`和`Order by` 都需要指定
* 当**主键**没有时，以**排序键**作为主键
* **排序键**只在当前分区内有效


#### VersionedCollapsingMergeTree

```
 create table vmttab (
    UserId Int,
    PageVies Int,
    Duration Int,
    Sign Int8,
    Version Int)
ENGINE = VersionedCollapsingMergeTree(Sign, Version)
PARTITION BY UserId Order By UserId;
```

> Sign 必须为Int8

这个引擎可以通过`Sign`的`+1 or -1`聚合相同版本的数据，然和再查询时使用

```
SELECT * FROM UAct FINAL
```
将相同版本号撤销。

#### AggregatingMergeTree

```
CREATE TABLE user_log
(
    user_id UInt64,
    sn_name String,
    login AggregateFunction(sum, UInt64),
    is_first AggregateFunction(sum, UInt8),
    insert_time datetime default now()
)
ENGINE = AggregatingMergeTree() PARTITION BY toYYYYMMDD(insert_time) ORDER BY insert_time SETTINGS index_granularity=8192;;
```

插入时必须使用`Insert select`语句：

```
insert into user_log_ag (user_id,login,is_first,sn_name) select user_id,sumState(login),sumState(is_first),minState(sn_name) from user_log GROUP BY user_id
```

查询:

```
select user_id,sn_name,sumMerge(is_first) from user_log_ag GROUP BY user_id,sn_name
```

## 可视化工具

- [DBeaver](https://dbeaver.io/download/)
- [Tabix](http://ui.tabix.io/)