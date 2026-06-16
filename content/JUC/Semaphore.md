+++
date = '2025-10-22T19:44:07+08:00'
draft = false
title = 'Semaphore'
+++

用来**限制同一时间访问某个资源的线程数量**

核心方法

- `acquire()` 获取一个许可，若无可用则阻塞等待
- `acquire(int permits)` 一次获取多个许可
- `tryAcquire()` 尝试获取，不阻塞，会返回 `true/false`
- `tryAcquire(long timeout, TimeUnit unit)` 显示等待获取许可
- `release()` 释放一个许可
- `availablePermits()` 返回当前可用许可
- `drainPermits()` 一次性拿走所有许可

示例

```java
public class SemaphoreDemo {

    private static final Semaphore semaphore = new Semaphore(3); 
    
    public static void main(String[] args) {
        
        for (int i = 0; i < 10; i++) {
            final int finalI = i;
            new Thread(() -> {
                try {
                    semaphore.acquire();
                    System.out.println("acquire semaphore");
                    System.out.println(Thread.currentThread().getName() + " execute " + finalI);
                    Thread.sleep(2000);
                } catch (Exception e) {

                } finally {
                    semaphore.release();
                    System.out.println("task finish and release semaphore");
                }
            }).start();
        }
    }
}
```

我们可以看到，控制台上每次只有 3 个线程可以执行任务

也可以尝试获取许可

```java
if (semaphore.tryAcquire()) {
    try {
        System.out.println("成功获取许可，执行任务");
    } finally {
        semaphore.release();
    }
} else {
    System.out.println("当前无许可可用，放弃执行");
}
```

也可以选择设置等待时间，而不是立即返回失败

```java
if (semaphore.tryAcquire(2, TimeUnit.SECONDS)) {
    try {
        System.out.println("在2秒内拿到了许可");
    } finally {
        semaphore.release();
    }
} else {
    System.out.println("2秒超时仍未拿到许可");
}
```

`Semaphore` 支持公平锁和非公平锁

```java
Semaphore semaphore = new Semaphore(3, true);
```

> 默认 `false` 非公平，后来的线程可能插队，吞吐量更高；`true` 公平，按线程申请顺序排队，吞吐量低，但更公平

接下来对比下 `Semaphore` 和其他限流手段

| 限流方式              | 作用范围  | 精度     | 是否分布式 | 优点                   | 缺点               |
| --------------------- | --------- | -------- | ---------- | ---------------------- | ------------------ |
| **Semaphore**         | 单JVM本地 | 并发数级 | 否         | 简单高效、轻量         | 无法跨实例全局限流 |
| **Guava RateLimiter** | 单JVM本地 | QPS级    | 否         | 支持速率限制、平滑突发 | 精度较粗           |
| **Sentinel / Redis**  | 分布式    | QPS级    | 是         | 可动态调整限流规则     | 依赖外部组件       |
| **Nginx / Gateway**   | API网关层 | 请求速率 | 是         | 统一控制入口           | 配置复杂           |

> 因此，`Semaphore` 适合**保护内部资源**（线程池、数据库连接、文件IO等）
