+++
date = '2025-10-22T19:18:57+08:00'
draft = false
title = 'CompletableFuture'
+++

是 Java 8 提供的一个可组合的异步计算类，它实现了 `Future` 接口，但功能比 `Future` 要更加强大，支持

- 异步执行
- 任务依赖
- 并行组合
- 异常处理
- 多任务竞争与汇总

`CompletableFuture.runAsync(Runnable task)` 执行无返回值任务

```java
CompletableFuture<Void> future = CompletableFuture.runAsync(() -> {
    try {
        Thread.sleep(2000);
        System.out.println("execute");
    } catch (Exception e) {

    }
});
System.out.println("join main");
future.join();
System.out.println("main end");
```

`CompletableFuture.supplyAsync(Callable task)` 执行有返回值任务

```java
CompletableFuture<String> future = CompletableFuture.supplyAsync(() -> {
    try {
        System.out.print("execute");
        Thread.sleep(2000);
    } catch (Exception e) {

    }
    return "finish";
});
try {
    System.out.println(future.get());            
} catch (Exception e) {

}
System.out.println("main end");
```

> PS：`CompletableFuture` 默认使用 `ForkJoinPool.commomPool` 线程池，它适合 CPU 密集型任务，并且所有没有指定线程池的都会共享它。它在处理阻塞任务，如 IO、数据库、网络时会卡死整个池导致性能骤降。所以推荐在使用过程中传入自定义线程池来获得可控的异步执行环境

`thenApply(Supplier<U> supplier)` 接收上一个任务结果并返回新结果

```java
CompletableFuture<String> future = CompletableFuture.supplyAsync(() -> {
    return "Hello ";
})
.thenApply(res -> {
    return res + "world";
});
try {
    System.out.println(future.get());            
} catch (Exception e) {

}
System.out.println("main end");
```

`thenAccept(Consumer<? supper T> action)` 接收结果但不返回值

```java
CompletableFuture.supplyAsync(() -> {
    return "Hello ";
})
.thenApply(res -> {
    return res + "world";
})
.thenAccept(res -> {
    System.out.println(res);
});
System.out.println("main end");
```

`thenRun(Runnable task)` 不关心结果只执行一个 `Runnable`

```java
CompletableFuture.supplyAsync(() -> {
    return "Hello ";
})
.thenApply(res -> {
    return res + "world";
})
.thenAccept(res -> {
    System.out.println(res);
})
.thenRun(() -> {
    System.out.println("this task is finished");
});
System.out.println("main end");
```

`thenCombine(CompletionStage<? extends U> other, BiFunction<? super T,? super U,? extends V> fn)` 两个任务都完成后合并结果

```java
CompletableFuture<String> f1 = CompletableFuture.supplyAsync(() -> {
    return "world";
});
CompletableFuture<String> res = CompletableFuture.supplyAsync(() -> {
    return "Hello ";
})
.thenCombine(f1, (s1, s2) -> {
    return s1 + s2;
});
try {
    System.out.println(res.get());
} catch (Exception e) {

}
System.out.println("main end");
```

`CompletableFuture.allOf(CompletableFuture<?>... cfs)` 等待所有任务完成

```java
CompletableFuture<Void> res = CompletableFuture.allOf(
            CompletableFuture.runAsync(() -> {
                try {
                    Thread.sleep(2000);
                    System.out.println("execute 1");
                } catch (Exception e) {

                }

            }),
            CompletableFuture.runAsync(() -> {
                System.out.println("execute 2");
            })
        );;
System.out.println("main join");
res.join();
System.out.println("main join finish");
System.out.println("main end");
```

`CompletableFuture.anyOf(CompletableFuture<?>... cfs)` 任意一个完成就返回

```java
CompletableFuture<Object> res = CompletableFuture.anyOf(
            CompletableFuture.supplyAsync(() -> {
                try {
                    Thread.sleep(20);
                } catch (Exception e) {

                }
                return "1";
            }),
            CompletableFuture.supplyAsync(() -> {
                return "2";
            })
        );;
try {
    System.out.println(res.get());
} catch (Exception e) {

}
System.out.println("main end");
```

还有一些其他处理异常的方法：

- `exceptionally()` 捕获异常并返回默认值

- `handle()` 无论成功或失败都执行

- `whenComplete()` 执行结束回调，不改变结果
