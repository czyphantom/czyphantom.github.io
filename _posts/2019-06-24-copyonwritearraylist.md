---
layout:     post
title:      CopyOnWriteArrayList
subtitle:   
date:       2019-06-24
author:     CZY
header-img: img/bg.png
catalog: true
tags:
    - Java
---

Linux中进程的创建使用了copy-on-write（写时复制）技术，父子进程共享地址空间，子进程在要写入内存时复制该页并写入。这种方式很有效地节约了复制地址空间的开销。在Java中，也有使用这种思想的设计，CopyOnWriteArrayList就是其中的一种。

在并发环境下，ArrayList不是线程安全的，如果对ArrayList进行并发迭代读写操作，会抛出ConcurrentModificationException。CopyOnWriteArrayList避免了多线程环境下ArrayList线程不安全的问题。

查看一下CopyOnWriteArrayList是怎么实现的，首先看一下add方法：

```java
    public boolean add(E e) {
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            Object[] elements = getArray();
            int len = elements.length;
            Object[] newElements = Arrays.copyOf(elements, len + 1);
            newElements[len] = e;
            setArray(newElements);
            return true;
        } finally {
            lock.unlock();
        }
    }
```

可以看到，在往集合里添加元素时，首先需要获取一个锁，避免了每个线程在往里面添加时都复制了一个副本，也保证了线程安全。

CopyOnWriteArrayList很好地解决了并发情况下List的线程安全问题，但是同时也引入了复制内存的开销，比较适合读多写少的场景，并且集合的大小也不能太大。