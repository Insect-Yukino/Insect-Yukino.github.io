+++
date = '2026-03-21T19:51:48+08:00'
draft = false
title = 'spring webflux 底层'
+++

`Flux<T>` 是 Reactor 里的响应式类型，表示 0 到 N 个异步元素组成的数据流。

```java
Flux<Integer> flux = Flux.just(1, 2, 3);
```

`Mono<T>` 也是 Reactor 里的类型，表示 0 到 1 个异步元素。

```java
Mono<String> mono = Mono.just("hello");
```

`FluxSink<T>` 一般出现在 `Flux.create()` 里，它的作用式手动向 `Flux` 中推送数据。

```java
Flux<String> flux = Flux.create(sink -> {
    sink.next("A");
    sink.next("B");
    sink.complete();
})
```

`FluxSink#next` 往流里发一个元素。

`FluxSink#complete` 告诉这个流结束了。

`FluxSink#error` 会发送一个异常。

**Netty 提供的是底层网络通信能力，主要负责非阻塞 IO、事件循环和连接管理。使用 Spring WebFlux 时，开发者通常直接接触的是上层业务编程模型，比如 `Mono`、`Flux`、流式响应和事件驱动逻辑，而不会直接操作 Netty 的网络细节。**
 **底层网络层通常由 Reactor Netty 基于 Netty 的 IO 多路复用和事件循环机制来承载；业务层则主要体现为响应式数据流、事件驱动处理和流式返回。**

Spring WebFlux 的底层本质是基于操作系统多路复用的事件驱动模型，借助 Netty 的 EventLoop 和 Reactor 的响应式流，将 IO 等待从线程中剥离出来，以少量线程支撑大量并发连接。它用编程复杂度换取系统级并发能力，只有在 IO 密集、非阻塞链路完整的场景下才能发挥优势。
