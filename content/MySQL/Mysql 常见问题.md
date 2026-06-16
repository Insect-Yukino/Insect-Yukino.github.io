+++
date = '2026-01-19T21:43:06+08:00'
draft = false
title = 'Mysql 常见问题'
+++

## Mysql 主库和从库的数据时一致性问题

主从数据库的一致性问题主要涉及两个方面：

1. 主库宕机时，从库中的数据是否和主库一致。
2. 从库同步主库的数据是否即使，访问请求是否可以拿到实时的数据。

Mysql 提供了三种模式来解决应对不同场合的这些问题。

- 异步复制，主库写完直接返回客户端成功，完全不等待从库
- 半同步复制，主库要至少等待一个从库同步成功才返回客户端消息
- 同步复制，主库要等待所有从库同步完成

可见，Mysql 同步复制是最严格的一致性，但缺点就是主库写入能力大大降低，而异步复制则稍显危险，如果从库没有同步完成，主库故障了，数据就丢失了。所以才有了半同步复制这个折中的方案。

我们不可能改变 Mysql 的主从复制原理，只能按照场景选择合适的数据同步方案。但是我们可以在代码层面解决一些读写延迟的问题。如果对数据的实时性要求很高，我们可以在代码层面实现强制走主库的逻辑。案例如下：

Mysql 主从同步有容灾机制嘛，主库挂掉如何切换主从

`DynamicRoutingDataSource` 继承自 Spring JDBC 的 `AbstractRoutingDataSource`，它本身是一个 DataSource 代理，在运行时根据 ThreadLocal 中的上下文 key 动态路由到真实的数据源；在生产环境中，通常通过配置文件定义多个数据源并注入，再统一交由路由数据源对外提供。 

MySQL 本身只提供主从复制机制，并不具备读写分离或请求路由能力；主库执行写操作、从库执行读操作通常是在应用层或数据库中间件层通过多数据源路由、SQL 解析或代理机制来实现的，数据库只是被动执行请求。

一个完整的 MySQL HA 方案，至少包含 3 个部分：

```text
① 数据复制（主从 / 多主）
② 故障检测（心跳）
③ 自动切换（Failover）
```

参考：

https://juejin.cn/post/7375858343179239478

https://cloud.tencent.com/developer/article/2577215

https://www.cnblogs.com/vipstone/p/18369635

https://cloud.tencent.com/developer/article/2240328

## Mysql 宕机如何解决

如果 Mysql 突然发生宕机导致数据丢失怎么办，这里我们主要参考上面的主从复制方案，

参考：

https://www.51cto.com/article/479889.html

https://blog.csdn.net/weixin_50348837/article/details/139813312

https://www.51cto.com/article/713751.html

https://www.51cto.com/article/709186.html

## Mysql 升级需要进行数据迁移如何处理
