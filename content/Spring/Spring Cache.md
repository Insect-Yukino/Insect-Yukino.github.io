+++
date = '2025-09-21T15:55:01+08:00'
draft = false
title = 'Spring Cache'
+++

Spring Cache 是 Spring 提供的一整套的缓存解决方案，它不是具体的缓存实现，它只提供一整套的接口和代码规范、配置、注解等，用于整合各种缓存方案，比如Caffeine、Guava Cache、Ehcache

Spring Cache 并不是缓存的实现，而是缓存使用的一种方式，其基于注解和 Spring 高级特性提供缓存读写以及失效刷新等各种能力

Spring Cache 的实现

**`Cache`** 接口包含缓存的常用操作：增加、删除、读取等。

**`CacheManager`** 是 Spring 各种缓存的抽象接口

## 使用

spring cache 默认是以内存来作为缓存的，使用`ConcurrentMap`来实现。如果要使用其他缓存方案，除了引入

```java
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-cache</artifactId>
</dependency>
```

还要引入对应的实现方案，如

```java
<!-- spring-data-redis -->
<dependency>
    <groupId>org.springframework.data</groupId>
    <artifactId>spring-data-redis</artifactId>
    <version>2.0.2.RELEASE</version>
</dependency>

<!-- ehcache -->
<dependency>
    <groupId>net.sf.ehcache</groupId>
    <artifactId>ehcache</artifactId>
    <version>2.10.6</version>
</dependency>
```

### CacheManager

是核心接口之一，它的作用是统一管理和提供`Cache`实例

> Cache 代表一个具体的缓存区，用来储存、获取、删除缓存

它的主要实现类有：

| 实现类                      | 说明                                     |
| --------------------------- | ---------------------------------------- |
| `ConcurrentMapCacheManager` | 基于 JVM 内存的简单缓存（线程安全 Map）  |
| `EhCacheCacheManager`       | 集成 EhCache                             |
| `CaffeineCacheManager`      | 集成 Caffeine（高性能本地缓存）          |
| `RedisCacheManager`         | 集成 Redis（分布式缓存）                 |
| `SimpleCacheManager`        | 需要手动设置一组 Cache（常用于组合缓存） |

### CacheResolver

实现和 `CacheManager` 类似的功能

### KeyGenerator

缓存类似于一对 `key-value` 值，而 `KeyGenerator` 是生成 `key` 的自定义方案

### CacheErrorHandler

用于处理缓存操作抛出的异常。

### 缓存注解

- @Cacheable：用于查询缓存。被该注解标注的方法先查询缓存，如果缓存未命中，再执行方法，并将方法返回值存入注解
- @Cacheput：用于更新缓存。被该注解标注的方法先调用方法，再将方法返回值更新到缓存
- @CacheEvict：用来清除缓存

### 常见问题

@Cacheable注解中，一个方法A调同一个类里的另一个有缓存注解的方法B，这样是不走缓存的。例如在同一个service里面两个方法的调用，缓存是不生效的。

> 和spring事务一样，因为都是由aop技术实现
