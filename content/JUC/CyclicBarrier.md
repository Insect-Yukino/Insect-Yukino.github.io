+++
date = '2025-10-22T18:16:04+08:00'
draft = false
title = 'CyclicBarrier'
+++

它会使一组线程互相等待，直到全部都到达某个屏障点，然后这些线程再同时继续执行

核心方法

- `CyclicBarrier(int parties)` 创建一个栅栏，指定参与线程数

- `CyclicBarrier(int parties, Runnable barrierAction)` 所有线程到达屏障点后，会先执行 barrierAction
- `await()` 表示当前线程已到达屏障点，等待其他线程到达
- `reset()` 重置屏障，恢复初始状态
- `getNumberWaiting()` 返回当前有多少线程在等待
- `isBroken()`  判断屏障是否已被破坏

示例，所有线程准备好后一起执行

```java
int count = 10;
CyclicBarrier cyclicBarrier = new CyclicBarrier(count, () -> {
    System.out.println("all ready");
});
for (int i = 0; i < count; i++) {
    new Thread(() -> {
        try {
            System.out.println(Thread.currentThread().getName() + " arrive barrier");
            cyclicBarrier.await();
            System.out.println(Thread.currentThread().getName() + " start");
        } catch (InterruptedException e) {

        } catch (BrokenBarrierException e) {

        }
    }, "thread-" + i).start();;
}
```

可以看到，所有线程都到达 `cyclicBarrier.await()` 之后才会执行。

接下来看一下 `CountDownLatch` 和 `CyclicBarrier` 的对比

| 对比项         | **CountDownLatch**                                  | **CyclicBarrier**                                           |
| -------------- | --------------------------------------------------- | ----------------------------------------------------------- |
| **主要功能**   | 一个或多个线程等待其他线程完成任务后再继续          | 一组线程相互等待，直到所有线程都到达某个屏障点              |
| **计数器行为** | 计数递减到 0 后触发，**一次性使用**                 | 计数递增（到达数），到达阈值后自动**重置，可重复使用**      |
| **等待方式**   | 通常是主线程调用 `await()` 等待子线程 `countDown()` | 所有工作线程都调用 `await()`，相互等待                      |
| **是否可重用** | 不可重用（一次倒计时后失效）                        | 可循环使用（cyclic）                                        |
| **触发时机**   | 当计数归零时，唤醒所有等待的线程                    | 当所有线程都到达屏障点时，统一唤醒                          |
| **可选动作**   | 无                                                  | 可设置 `barrierAction`（触发时执行一次）                    |
| **底层实现**   | 基于 AQS（AbstractQueuedSynchronizer）              | 基于 `ReentrantLock` + `Condition`                          |
| **异常机制**   | 无需重置，直接废弃                                  | 若其中一个线程中断或超时，会导致 **BrokenBarrierException** |
