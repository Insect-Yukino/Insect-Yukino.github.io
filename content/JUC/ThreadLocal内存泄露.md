+++
date = '2026-02-26T15:05:26+08:00'
draft = false
title = 'ThreadLocal 为什么会引起内存泄露'
+++

## Java 的 4 种引用

ThreadLocal 本身并不直接存储键值对，它更像是当前线程 `ThreadLocalMap` 中的一个 key。

ThreadLocal 的生命周期与线程绑定，当线程结束销毁后，值也会被清除。

但是如果使用线程池，因为线程会被复用，所以如果每次使用完 `ThreadLocal` 后不主动清理，本次存储的值就可能残留到下一次任务中。

更准确地说，`ThreadLocalMap` 对 key 使用的是**弱引用**，但 value 仍然可能继续被强引用着。这样在 key 被回收后，如果线程长期存活且没有触发及时清理，对应 value 就可能滞留在 `ThreadLocalMap` 中，形成典型的内存泄露风险。

因此在线程池场景下，最重要的实践是：**使用完 `ThreadLocal` 后及时调用 `remove()`。**

参考：

**https://juejin.cn/post/6932807532587810824**
