+++
date = '2025-11-18T22:30:57+08:00'
draft = false
title = 'RPC'
+++

进程间通信（IPC）指至少两个进程或线程间传送数据或信号的一些技术或方法。

有两种类型的进程间通信

- 本地过程调用LPC
- 远程过程调用RPC

RPC的核心并不在于使用什么协议。RPC的目的是让你在本地调用远程的方法，而对你来说这个调用是透明的，你并不知道这个调用的方法是部署哪里。通过RPC能解耦服务，这才是使用RPC的真正目的。RPC的原理主要用到了动态代理模式，至于http协议，只是传输协议而已

## [gRPC](https://doc.oschina.net/grpc?t=60134)

[](https://www.liwenzhou.com/posts/Go/gRPC/)
