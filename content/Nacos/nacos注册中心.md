+++
date = '2026-01-07T17:24:07+08:00'
draft = false
title = 'nacos注册中心'
+++

Nacos 支持基于实例 metadata 的 label 路由策略，
 通过标签表达式选择实例，
 tag 是最常用的一种标签字段，
 常用于灰度发布和流量隔离。

在实习过程中前端小右使用 tag 找到我的后端，其实他不是直接访问我的后端地址，而是访问一个固定的地址，然后会经过 nacos 的负载均衡，路由策略找到我的地址。

参考：

https://www.cnblogs.com/vipstone/p/15917359.html

```text
前端（小右）
   ↓
访问一个固定的 HTTP 地址（域名 / IP / 网关）
   ↓
后端入口服务（Gateway / Web / Admin 服务）
   ↓
服务调用（Feign / RestTemplate / WebClient）
   ↓
Nacos 服务发现
   ↓（根据 serviceName + tag / label）
负载均衡选出一个实例
   ↓
真正的后端实例（你的服务 IP）
```

前端通过配置 tag 参与路由意图的表达，
 后端在服务调用时根据该 tag 从 Nacos 中筛选实例，
 当只有一个实例匹配时，请求就会被确定性地路由到对应的服务节点，也就是我的实例。

灰度发布，金丝雀发布本质上是这种原理吗

服务怎么无感实现灰度发布的。如何改变路由实例的，就是只是上线一个 Nacos 实例，让nacos 实例自动负载吗

#### 灰度发布，有时间了解一下

https://www.cnblogs.com/alisystemsoftware/p/18281921
