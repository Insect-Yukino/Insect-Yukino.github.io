+++
date = '2025-12-31T20:44:11+08:00'
draft = false
title = 'Java 的基本类型和包装类型'
+++

Java 提供了基本类型和包装类型作为基本的数据结构。

包装类用来操作泛型

基本类型

- 基本数据类型的包装类型的大部分都用到了缓存机制来提升性能。
- `Byte`,`Short`,`Integer`,`Long` 这 4 种包装类默认创建了数值 **-128，127** 的相应类型的缓存数据，`Character` 创建了数值在 **0,127** 范围的缓存数据，`Boolean` 直接返回 `True` or `False`。

两种浮点数类型的包装类 `Float`,`Double` 并没有实现缓存机制。

自动拆箱装箱会使用到缓存。

String 类型的有没有缓存

Java 缓存池技术

[java string 缓存池_mob649e8166858d的技术博客_51CTO博客](https://blog.51cto.com/u_16175509/10800066)

[Java String底层原理之字符串常量池与内存分配-开发者社区-阿里云](https://developer.aliyun.com/article/1309168)
