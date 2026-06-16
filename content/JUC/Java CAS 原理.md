+++
date = '2025-09-03T19:32:53+08:00'
draft = false
title = 'Java CAS 原理'
+++

## 背景

Java 在使用 synchronized 关键字保证同步时，会导致有锁。锁机制存在一下问题：

- 在多线程竞争下，加锁、释放锁会导致比较多的上下文切换和调度延时，引起性能问题。
- 一个线程持有锁会导致其它所有需要此锁的线程挂起。

像这样的锁就是一种悲观锁，它的效率相比乐观锁要低。

volatile关键字能够在并发条件下，强制将修改后的值刷新到主内存中来保持内存的可见性。通过 CPU内存屏障禁止编译器指令性重排来保证并发操作的有序性。但是，它并不保证操作的原子性。在多个线程同时操作 volatile 修饰的变量时，也会造成数据的不一致。

> 每一个线程在操作内存时不会直接操作主存，而是会操作每一个线程独立拥有的工作内存。这样就造成了内存可见性问题，当某个线程修改了主存中的共享变量值之后，其他线程不能感知到该线程被修改了，它会一直使用自己工作内存中的旧值。

## CAS

CAS（Compare and Swap）是一种轻量级的同步操作，也是一种乐观锁的实现方式。在多线程环境中，CAS 可以实现非阻塞算法，避免了使用锁所带来的上下文切换、调度延迟、死锁等问题。

实现 CAS 大致的原理是：将需要操作的变量读进线程工作空间，比较工作空间的值和主缓存的值，如果工作空间缓存的值和主缓存的值相等，修改工作空间缓存并将修改后的值写入主缓存，如果不相等，则失败。

在 Java 中，AtomicInteger 就是通过 CAS 和 volatile 实现的一个线程安全整数类。比用锁更轻量，常用于高并发环境下的计数和状态更新。

常用方法：

```java
AtomicInteger atomicInt = new AtomicInteger(0);

// 获取值
atomicInt.get();          // 0

// 设置值
atomicInt.set(10);

// 原子加一
atomicInt.incrementAndGet();  // 11
atomicInt.getAndIncrement();  // 11 (返回旧值，再+1)

// 原子减一
atomicInt.decrementAndGet();

// 原子加指定值
atomicInt.addAndGet(5);   // 当前值+5

// CAS 操作
atomicInt.compareAndSet(15, 20); // 如果当前值是15，就改成20

```

## ABA问题

在并发编程中，如果一个变量初次读取时是 A 值，它的值被修改成 B，然后其他线程又把 B 修改成 A 了。而另一个早期线程在对比值时会误以为值没有发生改变 。这就是 ABA 问题。

解决 ABA 问题的一个方式是使用带版本号的的 CAS。在每次进行 CAS 操作时，不仅需要比较要修改的内存地址的值与期望的值是否相等，还需要比较这个内存地址的版本号是否与期望的版本号相等。如果相等，才进行修改操作。这样上述早期线程在对比版本号时会发现版本号错误，从而避免了误判。

Java 提供的 AtomicInteger 等基础原子类只能保证单值原子性，可能会遇到 ABA 问题。AtomicStampedReference 维护了一个引用对象和整数版本号每次更新时，除了比较对象本身，还要比较版本号（stamp），从而解决 CAS 操作的 ABA 问题。

常用方法：

```java
AtomicStampedReference<String> ref = new AtomicStampedReference<>("A", 1); // 初始值 A，版本号 1

// 获取当前值和版本号
int[] stampHolder = new int[1];
String value = ref.get(stampHolder); // 版本号被存放在了长度为 1 的 stampHolder 数组中
System.out.println("值=" + value + " 版本号=" + stampHolder[0]);

// CAS 更新（需要同时提供预期值和预期版本号）
boolean success = ref.compareAndSet("A", "B", 1, 2); 
// 如果当前值是 A 且版本号是 1，就更新为 B，版本号变成 2

// 单独获取
ref.getReference(); // 获取引用
ref.getStamp();     // 获取版本号

```
