+++
date = '2026-02-24T11:31:30+08:00'
draft = false
title = 'Redis 的几种配置模式'
+++

### Redis 单节点（Standalone）

- **Java 配置**：

**Jedis 配置：**

```java
import redis.clients.jedis.Jedis;

Jedis jedis = new Jedis("127.0.0.1", 6379);
jedis.set("key", "value");
String val = jedis.get("key");
```

**Lettuce 配置：**

```java
import io.lettuce.core.RedisClient;
import io.lettuce.core.api.StatefulRedisConnection;

RedisClient client = RedisClient.create("redis://127.0.0.1:6379");
StatefulRedisConnection<String, String> connection = client.connect();
connection.sync().set("key", "value");
String val = connection.sync().get("key");
```

- **YAML 配置（Spring Boot）**：

```yaml
spring:
  redis:
    host: 127.0.0.1
    port: 6379
    password: yourpassword
```

**说明**：
 最简单直连方式，适合开发或小型应用。

------

### Redis 主从复制（Master-Slave / Primary-Replica）

- **Java 配置**：

**Jedis 配置（手动读写分离）：**

```java
Jedis master = new Jedis("127.0.0.1", 6379);
master.set("key", "value");

Jedis slave = new Jedis("127.0.0.1", 6380);
String val = slave.get("key");
```

**Lettuce 配置（手动读写分离）：**

```java
RedisClient masterClient = RedisClient.create("redis://127.0.0.1:6379");
StatefulRedisConnection<String, String> masterConn = masterClient.connect();
masterConn.sync().set("key", "value");

RedisClient slaveClient = RedisClient.create("redis://127.0.0.1:6380");
StatefulRedisConnection<String, String> slaveConn = slaveClient.connect();
String val = slaveConn.sync().get("key");
```

- **YAML 配置（Spring Boot）**：

```yaml
spring:
  redis:
    master:
      host: 127.0.0.1
      port: 6379
    slaves:
      - 127.0.0.1:6380
    password: yourpassword
```

> 注：Spring Boot 默认不直接支持主从自动切换或读从策略，需要自定义读从逻辑。

**说明**：

- 普通主从用于读扩展，异步复制
- 写操作必须到主节点，从节点可能有数据延迟
- YAML 配置需要结合自定义读从逻辑使用

------

### Redis 哨兵模式（高可用主从）

- **Java 配置**：

**Jedis 配置：**

```java
import redis.clients.jedis.Jedis;
import redis.clients.jedis.JedisSentinelPool;
import java.util.Set;

Set<String> sentinels = Set.of("127.0.0.1:26379", "127.0.0.1:26380");
JedisSentinelPool pool = new JedisSentinelPool("mymaster", sentinels);

try (Jedis jedis = pool.getResource()) {
    jedis.set("key", "value"); // 自动写到当前 master
}
```

**Lettuce 配置：**

```java
import io.lettuce.core.RedisURI;
import io.lettuce.core.sentinel.api.StatefulRedisSentinelConnection;
import io.lettuce.core.sentinel.RedisSentinelClient;

RedisSentinelClient sentinelClient = RedisSentinelClient.create(RedisURI.create("redis-sentinel://127.0.0.1:26379"));
StatefulRedisSentinelConnection<String, String> connection = sentinelClient.connect();
connection.sync().set("key", "value");
```

- **YAML 配置（Spring Boot）**：

```yaml
spring:
  redis:
    sentinel:
      master: mymaster
      nodes:
        - 127.0.0.1:26379
        - 127.0.0.1:26380
    password: yourpassword
```

**说明**：

- 高可用模式，可感知主节点切换
- 读从节点需要额外配置
- 适合生产环境，客户端自动切换 master

------

### Redis 集群模式（Cluster）

- **Java 配置**：

**Jedis 配置：**

```java
import redis.clients.jedis.JedisCluster;
import redis.clients.jedis.HostAndPort;
import java.util.Set;
import java.util.HashSet;

Set<HostAndPort> nodes = new HashSet<>();
nodes.add(new HostAndPort("127.0.0.1", 7000));
nodes.add(new HostAndPort("127.0.0.1", 7001));

JedisCluster cluster = new JedisCluster(nodes);
cluster.set("key", "value");
String val = cluster.get("key");
```

**Lettuce 配置：**

```java
import io.lettuce.core.RedisURI;
import io.lettuce.core.cluster.RedisClusterClient;
import io.lettuce.core.cluster.api.StatefulRedisClusterConnection;
import java.util.Arrays;

RedisClusterClient clusterClient = RedisClusterClient.create(
    Arrays.asList(
        RedisURI.create("redis://127.0.0.1:7000"),
        RedisURI.create("redis://127.0.0.1:7001")
    )
);

StatefulRedisClusterConnection<String, String> connection = clusterClient.connect();
connection.sync().set("key", "value");
String val = connection.sync().get("key");
```

- **YAML 配置（Spring Boot）**：

```yaml
spring:
  redis:
    cluster:
      nodes:
        - 127.0.0.1:7000
        - 127.0.0.1:7001
        - 127.0.0.1:7002
      max-redirects: 3
    password: yourpassword
```

**说明**：

- 自动分片 + 自动 failover
- 客户端根据 key 的 slot 自动路由请求
- 注意跨 slot multi-key 操作限制
- 适合大规模场景

如果有些场景 **对数据实时性要求很高，不能读取旧数据**。那么我们可以针对上面四种 Redis 配置模式，分析可能出现的问题以及原因：

### **单节点（Standalone）**

- **问题**：基本没有，因为读写都在同一个节点，数据是实时的。
- **说明**：写操作立即更新 Redis，读操作直接获取最新数据。
- 实时性最好，但缺点是 **单点故障风险高**，高可用需要额外方案（哨兵或 Cluster）。

------

### **主从复制（Master-Slave）**

- **问题**：**从节点可能会落后于主节点**，异步复制意味着读从节点的数据可能是旧数据。
- **场景**：如果应用把读请求分发到从节点，就可能读到旧数据，不符合高实时性要求。
- **解决方法**：
  1. **只读主节点**，保证读数据与写数据同步（牺牲读扩展能力）
  2. 使用同步复制（Redis 5+ 的 `WAIT` 命令）保证至少 N 个从节点同步完成才返回写成功

------

### **哨兵模式（Sentinel）**

- **问题**：
  - Sentinel 本身只负责高可用和故障切换，不保证读写延迟问题
  - 如果读请求发送到从节点，也可能出现旧数据
  - 当 master 切换时，从节点升级为 master，可能会有数据丢失/延迟窗口
- **解决方法**：
  1. **读写都指向当前 master**
  2. 增加写确认机制（`WAIT` 命令）以保证数据同步到足够多的节点
  3. 确保应用在 failover 窗口期间对数据的读写顺序控制

------

### **集群模式（Cluster）**

- **问题**：
  - Redis Cluster 的 master/slave 也是异步复制，从节点数据可能落后
  - 如果客户端配置了读从策略（`ReadFrom`），可能读到旧数据
  - 跨 slot multi-key 操作可能引入延迟或一致性风险
- **解决方法**：
  1. **读写都走 master 节点**，保证读数据是最新的
  2. 避免使用 `ReadFrom.REPLICA` 读取从节点
  3. 在写操作后可以用 `WAIT` 命令等待同步
