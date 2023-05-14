# MongoDB最佳实践

## 复制集

MongoDB使用**复制集**实现高可用机制。MongoDB中的复制集（Replica Set）是一组MongoDB实例的集合，用于提供数据冗余和高可用性。复制集中包含一个主节点（Primary）和多个从节点（Secondary），主节点处理所有写操作，并将数据同步到从节点，从节点只能读取数据。如果主节点出现故障或不可用，复制集会自动选择一个从节点作为新的主节点，以保证系统的可用性和数据完整性。

**复制集**通过选举完成故障恢复。它有以下特点：

* 具有投票权的的节点之间互相发送心跳（保活）
* 当5次心跳未收到时判断为失联节点
* 如果失联的主节点，从节点会发起选举，选出新的主节点
* 如果失联的时从节点，不会重新产生选举
* 选举基于RAFT一致性算法实现，选举成功的必要条件时大多数投票节点存活
* 复制集最多可以有50个，但具有投票权的最多7个

### 实践

#### 1. 准备工作

MondoDB启动时，会使用一个本地磁盘的目录作为数据目录。本机存在多个复制集，需要保证他们不在同一目录下。一个配置复制集的配置文件：

```
# /data1/mongod.conf
systemLog:
  destination: file
  path: /data1/mongod.log   # 日志文件路径
  logAppend: true
storage:
  dbPath: /data1    # 数据目录
net:
  bindIp: 0.0.0.0
  port: 28017   # 端口
replication:
  replSetName: rs0
processManagement:
  fork: true   # 进程作为后台进程独立运行
```

> 启用了`replication`参数才表明是一个复制集，默认是单例。

使用`mongod -f`指定配置文件启动。

#### 2. 配置复制集

使用命令行进入主机：

```shell
mongo --port 28017
```

进入命令行后，使用`rs`命令配置:

```shell
rs.initiate({
    _id: "rs0",
    members: [{
        _id: 0,
        host: "localhost:28017"
    },{
        _id: 1,
        host: "localhost:28018"
    },{
        _id: 2,
        host: "localhost:28019"
    }]
})
```

查看复制集的状态:

```
rs.status()
```

从节点可读：

```
rs.slaveOK()
```