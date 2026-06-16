+++
date = '2025-10-22T17:54:13+08:00'
draft = false
title = 'CountDownLatch'
+++

是 Java `java.util.concurrent` 中的一个同步辅助类，常用于让一个线程等待多个线程的任务都完成后再继续执行

常用的方法有

- `CountDownLatch(int count)` 创建一个计数器，初始值为 `count`

- `void await()` 当前线程等待，直到计数器为 0

- `boolean await(long timeout, TimeUnit unit)` 最多等待指定时间
- `void countDown()` 计数器减 1
- `long getCount()` 获取当前计数值

场景 1，主线程等待多个子线程完成后再继续主线程

```java
int count = 10;
CountDownLatch countDownLatch = new CountDownLatch(count);
ExecutorService executor = Executors.newCachedThreadPool();
for (int i = 0; i < count; i++) {
    final int temp = i;
    executor.execute(() -> {
        try {
            System.out.println(Thread.currentThread().getName() + " execute " + temp);
        } catch (Exception e) {
            System.out.println(e.getMessage());

        } finally {
            countDownLatch.countDown();
        }
    });
}
System.out.println("main wait");
countDownLatch.await();
System.out.println("main end");
executor.shutdown();
```

注意 `CountDownLatch` 并不保证线程执行的顺序，只保证它的计数器为零时才开始执行主线程

场景 2，多个线程等待统一信号同时开始执行

```java
int count = 1;
CountDownLatch countDownLatch = new CountDownLatch(count);
ExecutorService executor = Executors.newCachedThreadPool();
for (int i = 0; i < 10; i++) {
    final int temp = i;
    executor.execute(() -> {
        try {
            countDownLatch.await();
            System.out.println(Thread.currentThread().getName() + " execute " + temp);
        } catch (Exception e) {

        } finally {

        }
    });
}
Thread.sleep(2000);
System.out.println("start execute");
countDownLatch.countDown();
executor.shutdown();
```

> PS：一定要给 `CountDowmLatch` 调用 `await` 方法，调用这个方法的线程在计数器不为 0 的时候被阻塞。`wait`() 是 `Object` 提供的方法，需要使用 `notify()` 唤醒
