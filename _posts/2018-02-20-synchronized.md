---
layout:     post
title:      synchronized关键字
subtitle:   
date:       2018-02-20
author:     CZY
header-img: img/bg.png
catalog: true
tags:
    - Java并发
---

# 简述

如果声明一个方法或代码块声明为synchronized，那么该代码块在一个时刻只能由一个线程执行该方法或者代码块（可以认为是原子性，一个线程要么把这个代码块执行完，要么阻塞等待获得锁），另外synchronized可以保证可见性，线程对变量的更新先于现有synchronized块所进行的更新，当进入另一个由同一个锁保护的synchronized块时，可以立刻看到这个更新（也就是说synchronized块之前的变量更新一定不会被重排序优化，且在获得锁之前的更新对另一个由已经进入同一个锁保护的代码块的线程是立即可见的）。一个线程如果想要访问由synchronized修饰的对象，首先得获得锁，退出时或者抛出异常必须释放锁。

# 用法

synchronized可以用来修饰方法，静态方法和代码块。当修饰方法时，锁是当前实例对象；当修饰静态方法时，锁是当前对象的Class对象；当修饰代码块时，锁是synchronized括号内的对象。

# 实现原理

由synchronized修饰的方法和synchronized修饰的代码块的实现原理也不尽相同，前者的实现由读取方法的ACC_SYNCHRONIZED标志来隐式实现，后者是由进入管程对象实现的（monitorenter和monitorexit指令）。monitorenter指令是在编译后插入到同步代码块的开始位置，而monitorexit是插入到方法结束处和异常处， JVM要保证每个monitorenter必须有对应的monitorexit与之配对。任何对象都有一个monitor与之关联，当且一个monitor被持有后，它将处于锁定状态。线程执行到monitorenter指令时，将会尝试获取对象所对应的 monitor的所有权，即尝试获得对象的锁。

## synchronized方法的实现原理

先来看一段代码：

```java
public class SynchonizedMethod {
    private int i;

    synchronized public void increase() {
        i++;
    }
}
```

javap得到关键字节码如下：

```
  public synchronized void increase();
    descriptor: ()V
    flags: ACC_PUBLIC, ACC_SYNCHRONIZED
```

synchronized修饰的方法并没有monitorenter和monitorexit指令，但是有一个ACC_SYNCHRONIZED标识，用来表明时同步方法。调用方法时会检查这个标识位，如果这个标识位被设置，那么执行线程会持有锁，执行完毕后释放锁。

## synchronized代码块的实现原理

先来看一段代码：

```java
public class SynchornizedCode {
    private int i = 0;
    public void syncThis() {
        synchronized (this) {
            i++;
        }
    }
}
```

javap得到关键字节码如下：

```
 public void syncThis();
    Code:
       0: aload_0
       1: dup
       2: astore_1
       3: monitorenter						//进入同步区
       4: aload_0
       5: dup
       6: getfield      #2                  // Field i:I
       9: iconst_1
      10: iadd
      11: putfield      #2                  // Field i:I
      14: aload_1
      15: monitorexit						//退出同步区
      16: goto          24
      19: astore_2
      20: aload_1
      21: monitorexit						//退出同步区，注意这个是出现异常时才会执行的
      22: aload_2
      23: athrow
      24: return
```

可以看到同步代码块的进入与退出使用的是monitorenter和monitorexit指令，前者指向同步代码块的开始位置，后者指向结束位置。执行到monitorenter命令时，当前线程会试图获得对象锁，如果锁的计数值为0时，当前线程可以获得锁，并进入同步代码块，并将锁的计数值设为1。注意synchonized的锁是可以重入的，也就是说在获得当前对象锁之后，如果调用了一个synchonized修饰的方法再次请求获得锁也是被允许的，此时计数器的值再加1，每退出一个同步方法或者代码块，计数器值减1，最后退出同步区的时候，计数器值设置为0。为了保证在方法异常完成时monitorenter和monitorexit指令依然可以正确配对执行，编译器会自动产生一个异常处理器，这个异常处理器声明可处理所有的异常，它的目的就是用来执行monitorexit指令。因此最后会多出一个monitorexit指令。

## Java 6之后对synchonized的优化

在Java早期版本中，synchronized属于重量级锁，效率低下，因为监视器锁（monitor）是依赖于底层的操作系统的Mutex Lock来实现的，而操作系统实现线程之间的切换时需要从用户态转换到核心态，这个状态之间的转换需要相对比较长的时间，时间成本相对较高，这也是为什么早期的synchronized效率低的原因。但是再Java 6之后，引入了偏向锁和轻量级锁，改善了synchronized的性能。

众所周知，Java对象在内存中的布局分为对象头，实例数据以及对其填充。synchronized使用的锁对象就是存储在Java对象头里，更确切的说，存储在对象头的Mark Word里，Mark Word不是一个固定结构的部分，其结构可能会随着锁标志位的变化而变化。通常在Mark Word里会有一个指向锁的指针，比如重量级锁指向的是monitor对象（也称为管程或监视器锁）的起始地址。每个对象都存在着一个monitor与之关联，当一个monitor被某个线程持有后，它便处于锁定状态。

一共有四种锁状态：无锁状态、偏向锁、轻量级锁和重量级锁。

偏向锁的加锁：Hotspot的作者经过以往的研究发现大多数情况下锁不仅不存在多线程竞争，而且总是由同一线程多次获得，为了让线程获得锁的代价更低而引入了偏向锁。当一个线程访问同步块并获取锁时，会在对象头和栈帧中的锁记录里存储锁偏向的线程ID，以后该线程在进入和退出同步块时不需要花费CAS操作来加锁和解锁，而只需简单的测试一下对象头的Mark Word里是否存储着指向当前线程的偏向锁，如果测试成功，表示线程已经获得了锁，如果测试失败，则需要再测试下Mark Word中偏向锁的标识是否设置成1（表示当前是偏向锁），如果没有设置，则使用CAS竞争锁，如果设置了，则尝试使用CAS将对象头的偏向锁指向当前线程。

偏向锁的撤销：偏向锁使用了一种等到竞争出现才释放锁的机制，所以当其他线程尝试竞争偏向锁时，持有偏向锁的线程才会释放锁。偏向锁的撤销，需要等待全局安全点（在这个时间点上没有字节码正在执行），它会首先暂停拥有偏向锁的线程，然后检查持有偏向锁的线程是否活着，如果线程不处于活动状态，则将对象头设置成无锁状态，如果线程仍然活着，拥有偏向锁的栈会被执行，遍历偏向对象的锁记录，栈中的锁记录和对象头的Mark Word要么重新偏向于其他线程，要么恢复到无锁或者标记对象不适合作为偏向锁，最后唤醒暂停的线程。下图中的线程1演示了偏向锁初始化的流程，线程2演示了偏向锁撤销的流程。

![偏向锁](http://ifeve.com/wp-content/uploads/2012/10/%E5%81%8F%E5%90%91%E9%94%81%E7%9A%84%E6%92%A4%E9%94%80.png)

轻量级锁加锁：线程在执行同步块之前，JVM会先在当前线程的栈桢中创建用于存储锁记录的空间，并将对象头中的Mark Word复制到锁记录中，官方称为Displaced Mark Word。然后线程尝试使用CAS将对象头中的Mark Word替换为指向锁记录的指针。如果成功，当前线程获得锁，如果失败，表示其他线程竞争锁，当前线程便尝试使用自旋来获取锁。

轻量级锁解锁：轻量级解锁时，会使用原子的CAS操作来将Displaced Mark Word替换回到对象头，如果成功，则表示没有竞争发生。如果失败，表示当前锁存在竞争，锁就会膨胀成重量级锁。下图是两个线程同时争夺锁，导致锁膨胀的流程图。

![轻量级锁](http://ifeve.com/wp-content/uploads/2012/10/%E8%BD%BB%E9%87%8F%E7%BA%A7%E9%94%81.png)

简单的说，synchronized的执行过程如下：
1. 检测Mark Word里面是不是当前线程的ID，如果是，表示当前线程处于偏向锁。
2. 如果不是，则使用CAS将当前线程的ID替换Mard Word，如果成功则表示当前线程获得偏向锁，置偏向标志位1。
3. 如果失败，则说明发生竞争，撤销偏向锁，进而升级为轻量级锁。
4. 当前线程使用CAS将对象头的Mark Word替换为锁记录指针，如果成功，当前线程获得锁。
5. 如果失败，表示其他线程竞争锁，当前线程便尝试使用自旋来获取锁。
6. 如果自旋成功则依然处于轻量级状态。
7. 如果自旋失败，则升级为重量级锁。

# 参考文章

[深入理解Java并发之synchronized实现原理](http://blog.csdn.net/javazejian/article/details/72828483#synchronized%E7%9A%84%E4%B8%89%E7%A7%8D%E5%BA%94%E7%94%A8%E6%96%B9%E5%BC%8F)

[聊聊并发（二）Java SE1.6中的Synchronized](http://ifeve.com/java-synchronized/)

[java中的锁--偏向锁、轻量级锁、自旋锁、重量级锁](http://blog.csdn.net/zqz_zqz/article/details/70233767)