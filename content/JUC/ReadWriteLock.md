+++
date = '2025-10-22T19:19:22+08:00'
draft = false
title = 'ReadWriteLock'
+++

在读多写少的情境下，读操作允许多个线程执行。而使用`ReentrantLock`会导致每次读操作只有一个线程执行，严重影响了性能。

因此，在这种场景下，可以使用`ReadWriteLocak`。它允许多个线程同时执行读操作，

`ReadWriteLock`在并发读操作中不允许写操作

`StampedLock`使用读乐观锁在并发读操作中，允许写操作
