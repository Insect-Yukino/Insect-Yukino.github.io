+++
date = '2025-08-13T15:43:06+08:00'
draft = false
title = 'Reids'
+++

## 概述

redis是一款开源的、基于内存的、高性能键值数据库，常用于缓存、消息队列、实时统计等场景。他的特点是数据储存在内存中，读写速度极快，比并且可以把数据持久化到磁盘。

> Java操作reids的主流客户端
>
> - Spring Data Redis
> - Redisson
> - Jedis
>
> 等

## Redis的使用

spring中集成redis，**RedisConnection**是底层直接操作redis的接口，**RedisTemplate**对RedisConnection做了封装来简化操作。**RedisConnectionFactory**是用来创建RedisConnection的工厂。

在maven中引入

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
```

依赖，spring会自动读取**application.yml**文件，生成`RedisConnectionFactory`类。因此可以直接注入`RedisConnectionFactory`

```java
@Bean
public StringRedisTemplate stringRedisTemplate(RedisConnectionFactory redisConnectionFactory) {
    return new StringRedisTemplate(redisConnectionFactory);
}
```

### Redis缓存

通过集成**Spring Cache**来实现缓存。使用`RedisCacheManager`类来管理`Cache`

### Redisson

Redisson 是一个在 redis 基础上实现的 java 驻内存数据网格客户端。它不仅提供了一系列的 redis 常用数据结构命令服务，还提供了许多分布式服务，例如分布式锁、分布式对象、分布式集合、分布式远程服务、分布式调度服务等。

相比于 Jedis、Lettuce 等基于 redis 命令封装的客户端，Redisson 提供的功能更加高端和抽象。

通过引入

```xml
<dependency>-->
    <groupId>org.redisson</groupId>-->
    <artifactId>redisson-spring-boot-starter</artifactId>-->
    <version>3.20.0</version>-->
</dependency>-->
```

spring容器会自动创建`RedissonClient`类来实现操作。如果单单引入

```java
<dependency>
    <groupId>org.redisson</groupId>
    <artifactId>redisson</artifactId>
    <version>3.20.0</version>
</dependency>
```

需要手动配置Bean

```java
@Bean
public RedissonClient redisson() {
    Config config = new Config();
    config.useSingleServer()
            .setAddress("redis://" + host + ":" + port);
    return Redisson.create(config);
}
```

### Redisson分布式锁

redisson 提供了`lock()`和`tryLock()`，`tryLock(long time, TimeUnit unit)`，`tryLock(long waitTime, long leaseTime, TimeUnit unit)`方法。

1. `lock()`：会阻塞未获取锁的请求，默认持有`30s`锁，但当业务方法在30s内没有执行完时，会有`看门狗（默认每隔10s）`给当前锁续时`30s`。
2. `tryLock()`：尝试获取锁，获取不到则直接返回获取失败，默认持有`30s`锁，但当业务方法在30s内没有执行完时，会有`看门狗（默认每隔10s）`给当前锁续时`30s`。
3. `tryLock(long time, TimeUnit unit)`：尝试获取锁，等待`time TimeUnit`，默认持有`30s`锁，但当业务方法在30s内没有执行完时，会有`看门狗（默认每隔10s）`给当前锁续时`30s`。
4. `tryLock(long waitTime, long leaseTime, TimeUnit unit)`：尝试获取锁，等待`waitTime TimeUnit`，`锁最长持有leaseTime TimeUnit`，当业务方法在`leaseTime TimeUnit`时长内没有执行完时，会强制解锁。

### Redis布隆过滤器

它的作用是准确快速的判断某个数据是否在大数据量集合中。

它是一种数据结构，是由一串很长的二进制向量组成，可以将其看成一个二进制数组。因此不占用内存。数组内初始默认值是0。

当向布隆过滤器中添加元素，会通过哈希函数对其计算值，然后将这个值所对应的位置置为1，如果再有相同的数被插入布隆过滤器中，计算出的值会是相同的。此时如果这个位置的值为0，那么这个数就一定不存在；如果这个位置的值是1，那么这个数可能存在。
