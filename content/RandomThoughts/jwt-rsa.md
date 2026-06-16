+++
date = '2025-11-29T18:29:28+08:00'
draft = false
title = 'jwt-rsa'
+++

([JWT单点登录 看这一篇就够了！-腾讯云开发者社区-腾讯云](https://cloud.tencent.com/developer/article/2084586))

使用 JWT 实现无状态登录时

实现一个 Auth 模块，使用私钥派发加密后的 JWT

每一个模块可以使用公钥验证拿到的 JWT 的正确性

JWT 登录方式有两种选择

**JWT + RSA** 

完全无状态，通过公钥验证 Token，无需访问 Redis

适合分布式、无需主动注销场景

**JWT + Redis**

半无状态，Redis 存储登录状态

可以主动踢人、强制下线某人。
