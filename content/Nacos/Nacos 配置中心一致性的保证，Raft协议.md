+++
date = '2025-12-11T22:20:36+08:00'
draft = false
title = 'Nacos 配置中心一致性的保证，Raft协议'
+++

我们知道，Nacos 配置中心是 **CP** 强一致性的，用于确保大量客户端获取的配置信息是一致的。

Nacos 内部是通过 **Raft** 协议保证不同配置中心的数据一致。Nacos 官方没有自己实现 Raft 协议，而是集成的 **JRaft** 开源框架。Nacos 和 JRaft 的关系可以如下图表示：

```textt
┌──────────────────────────────────┐
│            Nacos 实例            │
│   ┌───────────────────────────┐  │
│   │     Raft Group 1 Node      │ ← JRaft Leader/Follower
│   ├───────────────────────────┤  │
│   │     Raft Group 2 Node      │
│   ├───────────────────────────┤  │
│   │     Raft Group 3 Node      │
│   └───────────────────────────┘  │
└──────────────────────────────────┘
```

从图中可以看到，Nacos 是作为容器承载着 Raft 节点。Nacos 配置中心数据一致性正是通过这些节点实现的。一个 Nacos 实例只会启动一个 Raft，但是当我们集群部署时多个 Nacos 实例当然会有多个 Raft 节点。这些节点有 **Leader** 节点，有 **Follower** 节点。JRaft 对传统的 Raft 协议做了修改。客户端的读请求会通过线性请求机制直接打到 **Follower** 节点，写请求会被 **Follower** 节点转发到 **Leader** 节点，再由 **Leader** 写入到**数据同步日志中**，**Leader** 节点定时读取日志数据。像这样**减少 Leader 压力的优化**还有。比如，面对 Nacos 配置中心大量的数据，单一 Leader 难以应付，JRaft 实现了 **Group** 这个机制，它让不同 **Group** 有自己的 **Leader** 和 **Follower**，不同 **Group** 用来处理不同的数据一致性。这样大大减轻了 **Leader** 的访问压力。

  

[这里](https://www.cnblogs.com/zzyang/p/17999055)有更详细的知识，之后我再进行详细的总结
