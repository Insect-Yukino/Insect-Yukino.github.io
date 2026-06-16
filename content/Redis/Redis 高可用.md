+++
date = '2025-09-07T15:07:56+08:00'
draft = false
title = 'Redis 高可用'
+++

高可用指的是即使 redis 某些节点宕机或网络异常，整个 redis 不会出现数据丢失并人能够对外提供访问

## 持久化

为了避免程序重启或者系统崩溃导致的数据丢失问题。可以采用 redis 的持久化。

redis 持久化支持一下三种：

**RDB 持久化**

快照持久化，将某一时刻的内存数据以二进制的方式写入磁盘。

> 占用空间小：RDB 持久化将纯数据存储在了一个二进制文件中，相比于 AOF 方式记录每一条完整的写操作，它的占用空间更小 
>
> 恢复速度：因为 二进制文件中保存的是一个完整的数据库快照，所以在 redis 重启后，数据的恢复速度很快
>
> 可靠性：RDB 持久化操作会定期自动执行。即使 Redis 进程崩溃或者服务器断电，也可以通过加载最近的一次快照文件恢复数据。实时性差：但是如果在写入时出现故障，数据会丢失，实时性差。

**AOF 持久化**

文件追加持久化，记录所有非查询操作命令，并以文本的形式追加到文件中。

> 实时性好：AOF 方式会及时记录 redis 执行的每一条写命令，因此它的实时性和数据的完整性比 RDB 持久化方式更好。
>
> 数据可读性强：AOF 持久化文件是一个纯文本文件，可以被人类读取和理解，因此可以方便地进行数据备份和恢复操作。
>
> 写入性能差：相对于 RDB 方式直接存储数据，AOF 方式需要记录每一个写命令，因此写入性能略低
>
> 占用空间更大：记录的每一条写命令需要比纯数据占用更大的空间

**混合持久化**

redis 4.0 后新增的方式，结合了 RDB 和 AOF的优点，写入的时候先把当前数据以 RDB 的形式写入文件开头，然后后续操作的命令以 AOF 的格式存入文件。既保证了程序重启速度，又降低了数据丢失的风险

> 该方式结合了以上两种方式的优点，开头为 RDB 的格式，使得 Redis 可以更快的启动，同时结合 AOF 的优点，有减低了大量数据丢失的风险。
>
> 但是由于需要同时维护 RDB 文件和 AOF 文件，因此实现复杂度相对于单独使用 RDB 或 AOF 持久化方式要高。AOF 文件中添加了 RDB 格式的内容，使得 AOF 文件的可读性变得很差。

## 主从复制

要保证高可用，避免单节点故障，就需要冗余方式提供集群服务。Redis 提供了主从库模式，以保证数据副本的一致，主从库之间采用的是读写分离的方式。

### 配置主节点

```conf
# redis.conf
port 6379
appendonly yes                # 开启 AOF（手动开启）
---
aof-use-rdb-preamble yes      # 使用 RDB+AOF 混合持久化（Redis 4.0+ 默认就开启）
```

```bash
# 启动主节点
redis-server /path/to/redis.conf 
```

### 配置从节点

```conf
# redis.conf（从节点配置）
port 6380
replicaof 127.0.0.1 6379
# 如果主节点有密码认证，还需要：
masterauth <主节点密码>
```

```bash
# 启动从节点
redis-server /path/to/redis.conf
```

以上是用配置文件，也可以在从节点使用指令 

```bash
replicaof 127.0.0.1 6379
```

> 从节点默认是只读的（replica-read-only yes），如果你想测试写操作，可以改成 no
>
> 如果愿主节点挂了可以选择一个从节点执行
>
> ```bash
> replicaof no one
> ```
>
> 断开和主节点的复制关系，变成新的主节点。然后让其他节点重新指向新的主节点

一个 redis 主节点可以有多个从节点，一个从节点也可以做其他节点的主节点拥有从节点

## 哨兵机制

为了解决主从复制模式下发生节点故障，需要人工干预选出主节点。redis 实现了哨兵机制。它可以自动实现主从库的切换，有效地解决了主从复制模式下故障转移的问题

### 配置哨兵机制

准备一个主节点 + 两个从节点

- 主节点：127.0.0.1:6379

- 从节点1：127.0.0.1:6380

- 从节点2：127.0.0.1:6381

创建 Sentinel 配置文件

在 Redis 安装目录下新建一个文件 `sentinel.conf`，内容如下：

```conf
# Sentinel 自身的监听端口（默认 26379）
port 26379

# 监控的主节点（名字可以随便取，比如 mymaster）
# 格式：sentinel monitor <主节点名称> <主节点IP> <主节点端口> <quorum>
# quorum = 至少多少个 sentinel 认为主节点宕机，才会触发故障转移
sentinel monitor mymaster 127.0.0.1 6379 2

# 如果主节点设置了密码，需要添加认证（Redis 5.0+）
# sentinel auth-pass <主节点名称> <密码>
# sentinel auth-pass mymaster mypassword

# 主节点宕机多久认为客观下线（毫秒）
sentinel down-after-milliseconds mymaster 5000

# 故障转移超时时间（毫秒）
sentinel failover-timeout mymaster 10000

# 故障转移时并行同步的从节点数量
sentinel parallel-syncs mymaster 1
```

启动多个主节点，形成一个可靠的投票机制

```bash
redis-sentinel sentinel-26379.conf
redis-sentinel sentinel-26380.conf
redis-sentinel sentinel-26381.conf
```

用客户端连接 Sentinel 查看：

```bash
redis-cli -p 26379 info sentinel
```

## Redis Cluster

Redis Cluster 是 Redis 3.0 版本推出的 Redis 集群方案，它将数据分布在不同的服务区上，以此来降低系统对单主节点的依赖，并且可以大大的提高 Redis 服务的读写性能。

上面的主从同步只能有一个主节点，而 Redis Cluster 可以拥有无数个主从节点。有效提升了集群的扩展能力

### 配置 Redis Cluster

在一台机器上模拟 6 个节点（3 主 3 从）

每个节点端口为：7000, 7001, 7002, 7003, 7004, 7005

为每个实例准备配置文件 redis-700x.conf

```conf
# redis-7000.conf
port 7000
cluster-enabled yes          # 开启集群模式
cluster-config-file nodes-7000.conf   # 集群配置文件，自动生成
cluster-node-timeout 5000    # 超时时间，毫秒
appendonly yes               # 开启 AOF 持久化
daemonize yes                # 后台运行（可选）
bind 127.0.0.1               # 绑定地址，测试时本机
```

启动 6 个实例：

```bash
./redis-server redis-7000.conf
./redis-server redis-7001.conf
...
./redis-server redis-7005.conf
```

使用 redis-cli 创建集群

```bash
./redis-cli --cluster create \
  127.0.0.1:7000 127.0.0.1:7001 127.0.0.1:7002 \
  127.0.0.1:7003 127.0.0.1:7004 127.0.0.1:7005 \
  --cluster-replicas 1
# --cluster-replicas 1 表示每个主节点分配 1 个从节点。
# 这样就会形成 3 主 3 从 的集群。
```

进入任意节点：

```bash
./redis-cli -c -p 7000 # -c 表示支持自动重定向 -p 表示指定端口
```

查看集群节点：

```bash
cluster nodes
```
