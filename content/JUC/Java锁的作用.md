+++
date = '2025-09-02T22:33:38+08:00'
draft = false
title = 'Java锁的作用'
+++

1. 互斥访问：任何时刻，只有一个线程能够访问特定资源或特定代码块
2. 内存可见性：通过锁的获取和释放，确保锁保护的代码块中对共享变量的修改对其他线程可见
3. 保证原子性：通过锁保证代码块内的操作是原子操作
4. 同步：协调线程间的执行顺序，保证程序的逻辑正确

## 锁的实现

1. synchronized 

   是 Java 语法层面提供的内置锁，是 JVM 支持原生并发控制机制。它不仅保证互斥访问，还保证线程之间的内存可见性

   > 程序进入 synchronized 块时，线程会刷新工作内存，从内存中读取最新变量值
   >
   > 离开时，会把修改写回主内存

   ```java
   // 方法锁
   public synchronized void instanceMethod() {
       // 锁住当前对象实例 this
   }
   
   public static synchronized void staticMethod() {
       // 锁住当前类对象 Class
   }
   ```

   ```java
   // 代码块锁
   public Object lock = new Object();
   public void method() {
       synchronized (lock) {
           // 只锁住 lock 对象
       }
   }
   ```

2. ReentrantLock

   是 Java 类库实现的锁，相比于 synchronized，它更灵活，支持公平性、中断、条件变量，但必须手动加解锁

   基本用法

   ```java
   // 1. 创建ReentrantLock对象
   public ReentrantLock lock = new ReentrantLock();
   public void method() {
       // 2.获取锁
       lock.lock(); 
       try {
           // 3.得到锁，执行需要同步的代码块
       } finally {
           // 4.释放锁
           lock.unlock(); 
       }
   }
   ```

   进阶

   ```java
   public ReentrantLock lock = new ReentrantLock();
   public void method() {
       // 尝试获取锁，等待2秒，超时返回false
       boolean locked = lock.tryLock(2, TimeUnit.SECONDS);
       if (locked) {
           try {
               // 执行需要同步的代码块
           } finally {
               lock.unlock();
           }
       }
   }
   ```

3. ReentrantReadWriteLock

   在面对读多写少的情况下，使用前两种锁每次只允许一个线程执行读操作。但读操做本应允许多个线程同时执行，因此这样会导致性能下降。

   ReentrantReadWriteLock 是 Java 并发包提供的提供的可重入读写锁，用于在多线程环境下提高读操作的并发性，同时保证写操作的独占性。它比单纯的 ReentrantLock 更适合 多读少写 的场景。

   ```java
   class Counter {
   
       private ReadWriteLock lock = new ReentrantReadWriteLock();
       private final Lock readLock = lock.readLock();
       private final Lock writeLock = lock.writeLock();
   
       private int[] count = new int[10];
   
       public void increment(int index) {
           writeLock.lock();
           try {
               count[index]++;
           } finally {
               writeLock.unlock();
           }
       }
   
       public int get(int index) {
           readLock.lock();
           try {
               return count[index];
           } finally {
               readLock.unlock();
           }
       }
   }
   ```

4. StampedLock

   ReentrantReadWriteLock 在写的过程中是不允许有读操作的，而 StampedLock 提供了乐观读锁。允许线程在写的过程中进行读操作，进一步提高了并发性能。

   ```java
   class Point {
       private final StampedLock stampedLock = new StampedLock();
       private double x, y;
   
       public void move(double x, double y) {
           long stamp = stampedLock.writeLock();
           try {
               this.x =+ x;
               this.y =+ y;
           } finally {
               stampedLock.unlockWrite(stamp);
           }
       }
   
       public double getDistance() {
           long stamp = stampedLock.tryOptimisticRead(); // 获取一个乐观锁
           double x = this.x;
           double y = this.y;
           if (!stampedLock.validate(stamp)) {
               stamp = stampedLock.readLock(); // 获取一个悲观锁
               try {
                   x = this.x;
                   y = this.y;
               } finally {
                   stampedLock.unlockRead(stamp);
               }
           }
           return Math.sqrt(x*x + y*y);
       }
   }
   ```
