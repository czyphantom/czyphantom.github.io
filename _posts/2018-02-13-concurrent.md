---
layout:     post
title:      Java并发基础
subtitle:   
date:       2018-02-13
author:     CZY
header-img: img/bg.png
catalog: true
tags:
    - Java并发
---

# 死锁

死锁的产生通常会有比较严重的后果，因此尽可能要避免死锁。有以下几种方式：

+ 避免一个线程同时获取多个锁。
+ 避免一个线程在锁内同时占用多个资源，尽量保证每个锁只占用一个资源。
+ 使用定时锁。
+ 对于数据库锁，加锁和解锁必须在一个数据库连接里，否则会出现解锁失败的情况。

# 原子操作

原子操作意味着不可中断，一个或者一系列操作要么全都执行，要么全不执行。

## 处理器实现原子操作

首先明确一个事实：处理器保证内存操作的原子性，即从内存中读一个字节或写一个字节一定是原子的，当然复杂的内存操作处理器是不自动保证原子性了。处理器保证原子操作有两种方式，一种是对总线加锁，执行该原子操作时，锁总线，这样其他CPU就不能对内存进行访问。

第二种方式是通过缓存锁保证，总线锁的开销比较大，在某些场合下使用缓存锁来进行优化。意思是如果某一CPU执行原子操作将结果回写到内存时，由缓存一致性机制使得其他CPU内的缓存行无效。

有两种情况下处理器不会使用缓存锁定：
+ 当操作的数据不能被缓存在处理器内部，或操作的数据跨多个缓存行（cache line）时，则处理器会调用总线锁定。
+ 有些处理器不支持缓存锁定。对于Intel 486和Pentium处理器，就算锁定的内存区域在处理器的缓存行中也会调用总线锁定。

## Java如何实现原子操作

### 循环CAS操作

CAS操作使用的是CPU的的CMPXCHG指令，能够保证操作的原子性。Atomic包下有很多类用来支持原子操作，比如AtomicInteger，AtomicBoolean等。但是CAS操作也有一些问题：
+ ABA问题，就是一个值一开始是A，然后变成B，最后又变成A，用CAS进行检查时会发现它的值没有发生变化，实际上发生了变化。解决办法是加上版本号，在变量前加上版本号，每次变量更新的时候版本号+1，JDK的Atomic包里提供了一个类AtomicStampedReference来解决ABA问题，这个类的compareAndSet方法的作用是首先检查当前引用是否等于预期引用，并且检查当前标志是否等于预期标志，如果全部相等，则以原子方式将该引用和该标志的值设置为给定的更新值。
+ 循环时间长开销大。自旋CAS如果长时间不成功，会给CPU带来非常大的执行开销。
+ 只能保证一个共享变量的原子操作。当对一个共享变量执行操作时，我们可以使用循环CAS的方式来保证原子操作，但是对多个共享变量操作时，循环CAS就无法保证操作的原子性，这个时候就可以用锁。JDK提供了AtomicReference类来保证引用对象之间的原子性，就可以把多个变量放在一个对象里来进行CAS操作。

### 锁机制

锁机制保证了只有获得锁的线程才能够操作锁定的内存区域。JVM内部实现了很多种锁机制，有偏向锁、轻量级锁和互斥锁。除了偏向锁，JVM实现锁的方式都用了循环CAS，即当一个线程想进入同步块的时候使用循环CAS的方式来获取锁，当它退出同步块时使用CAS释放锁。

# 锁

锁从宏观上讲，分为乐观锁和悲观锁。

## 乐观锁

乐观锁是一种乐观思想，即认为读多写少，遇到并发写的可能性低，每次去拿数据的时候都认为别人不会修改，所以不会上锁，但是在更新的时候会判断一下在此期间别人有没有去更新这个数据，采取在写时先读出当前版本号，然后加锁操作（比较跟上一次的版本号，如果一样则更新），如果失败则要重复读-比较-写的操作。Java中的乐观锁基本都是通过CAS操作实现的，CAS是一种更新的原子操作，比较当前值跟传入值是否一样，一样则更新，否则失败。

## 悲观锁

悲观锁是就是悲观思想，即认为写多，遇到并发写的可能性高，每次去拿数据的时候都认为别人会修改，所以每次在读写数据的时候都会上锁，这样别人想读写这个数据就会阻塞直到拿到锁。Java中的悲观锁就是Synchronized,AQS框架下的锁则是先尝试CAS乐观锁去获取锁，获取不到，才会转换为悲观锁，如RetreenLock。

# Happens-before原则

重要的Happens-before原则有如下几种：
+ 程序顺序规则：一个线程中的每个操作，happens-before于该线程中的任意后续操作。
+ 监视器锁规则：对一个锁的解锁，happens-before于随后对这个锁的加锁。
+ volatile变量规则：对一个volatile域的写，happens-before于任意后续对这个volatile域的读。
+ 传递性：如果A happens-before B，且B happens-before C，那么A happens-before C。
+ start()规则：如果线程A执行操作ThreadB.start()（启动线程B），那么A线程的ThreadB.start()操作happens-before于线程B中的任意操作。
+ join()规则：如果线程A执行操作ThreadB.join()并成功返回，那么线程B中的任意操作happens-before于线程A从ThreadB.join()操作成功返回。

在JMM中，如果一个操作执行的结果需要对另一个操作可见，那么这两个操作之间必须要存在happens-before关系。这里提到的两个操作既可以是在一个线程之内，也可以是在不同线程之间。

# as-if-serial原则

as-if-serial语义的意思是：不管怎么重排序（编译器和处理器为了提高并行度），（单线程）程序的执行结果不能被改变。编译器、runtime和处理器都必须遵守as-if-serial语义。编译器和处理器不会对存在数据依赖关系的操作做重排序，因为这种重排序会改变执行结果。但是，如果操作之间不存在数据依赖关系，这些操作就可能被编译器和处理器重排序。

# 双重检查锁定

在Java多线程程序中，有时候需要采用延迟初始化来降低初始化类和创建对象的开销。双重检查锁定是常见的延迟初始化技术，但它是一个错误的用法。来看一段示例代码：

```java
public class DoubleCheckedLocking { 
	private static Instance instance;
	public static Instance getInstance() { 
		//第一次检查
		if (instance == null) {
			//加锁
			synchronized (DoubleCheckedLocking.class) {
				//第二次加锁
				if (instance == null)
					instance = new Instance(); 
			} 
		} 
		return instance;
	} 
}
```

如果第一次检查instance不为null，那么就不需要执行下面的加锁和初始化操作。因此，可以大幅降低synchronized带来的性能开销。但是有一个错误的地方，在第四行读取到instance不为null时，可能instance还没完成初始化。在new一个实例对象时，可以分解为3个步骤：

```
memory = allocate();　　// 1：分配对象的内存空间
ctorInstance(memory);　 // 2：初始化对象
instance = memory;　　 // 3：设置instance指向刚分配的内存地址
```

2，3可能会被重排序（因为也没改变程序最后执行的结果），因此可能在instance指向不为空时，对象还没完成初始化。解决方案有：

+ 将instance声明为volatile型。
+ JVM在在Class被加载后，且被线程使用之前，会执行类的初始化。在执行类的初始化期间，JVM会去获取一个锁。这个锁可以同步多个线程对同一个类的初始化，也就是说把初始化语句放在某个类的域里。如下所示：

```java
public class InstanceFactory {
	private static class InstanceHolder {
		public static Instance instance = new Instance();
	}
	public static Instance getInstance() {
		//这里将导致InstanceHolder类被初始化
		return InstanceHolder.instance ;　　
	}
}
```

# 参考文章

方腾飞、魏鹏、程晓明：Java并发编程的艺术
