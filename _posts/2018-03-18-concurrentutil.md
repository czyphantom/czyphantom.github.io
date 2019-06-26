---
layout:     post
title:      并发工具类
subtitle:   锁
date:       2018-03-18
author:     CZY
header-img: img/bg.png
catalog: true
tags:
    - Java并发
---

# 阻塞队列

阻塞队列（BlockingQueue）是一个支持两个附加操作的队列。这两个附加的操作支持阻塞的插入和移除方法：
1. 支持阻塞的插入方法：意思是当队列满时，队列会阻塞插入元素的线程，直到队列不满。
2. 支持阻塞的移除方法：意思是在队列为空时，获取元素的线程会等待队列变为非空。

阻塞队列常用于生产者和消费者的场景，生产者是向队列里添加元素的线程，消费者是从队列里取元素的线程。阻塞队列就是生产者用来存放元素、消费者用来获取元素的容器。

常见的阻塞队列有以下几种：
1. ArrayBlockingQueue：一个由数组结构组成的有界阻塞队列（必须在构造器传入容量）。此队列按照先进先出（FIFO）的原则对元素进行排序。默认情况下不保证线程公平的访问队列，所谓公平访问队列是指阻塞的线程，可以按照阻塞的先后顺序访问队列，即先阻塞线程先访问队列。非公平性是对先等待的线程是非公平的，当队列可用时，阻塞的线程都可以争夺访问队列的资格，有可能先阻塞的线程最后才访问队列。为了保证公平性，通常会降低吞吐量。
2. LinkedBlockingQueue：一个由链表结构组成的有界阻塞队列（默认Integer的最大值）。按照先进先出的原则对元素进行排序。
3. PriorityBlockingQueue：一个支持优先级排序的有界阻塞队列（默认是11)。默认情况下元素采取自然顺序升序排列。也可以自定义类实现compareTo()方法来指定元素排序规则，或者初始化PriorityBlockingQueue时，指定构造参数Comparator来对元素进行排序。需要注意的是不能保证同优先级元素的顺序。
4. DelayQueue：一个使用优先级队列实现的无界阻塞队列。支持延时获取元素,队列中的元素必须实现Delayed接口，在创建元素时可以指定多久才能从队列中获取当前元素。只有在延迟期满时才能从队列中提取元素。可以用来实现缓存系统或者定时任务。实现Delayed接口可以参考ScheduledThreadPoolExecutor里ScheduledFutureTask类的实现。延时阻塞队列的实现很简单，当消费者从队列里获取元素时，如果元素没有达到延时时间，就阻塞当前线程。
5. SynchronousQueue：一个不存储元素的阻塞队列。每一个put操作必须等待一个take操作，否则不能继续添加元素。支持公平访问队列。默认情况下线程采用非公平性策略访问队列。队列本身并不存储任何元素，非常适合传递性场景。SynchronousQueue的吞吐量高于LinkedBlockingQueue和ArrayBlockingQueue。
6. LinkedBlockingDeque：一个由链表结构组成的双向阻塞队列。

## 阻塞队列的实现（以ArrayBlockingQueue为例）

阻塞队列使用通知模式实现，所谓通知模式，就是当生产者往满的队列里添加元素时会阻塞生产者，当消费者消费了一个队列中的元素后，会通知生产者当前队列可用。部分源码如下：

```java
	final ReentrantLock lock;
    //队列不空的条件
	private final Condition notEmpty;
    //队列不满的条件
	private final Condition notFull;
    //队列的元素个数
    int count;
    //存放队列
    final Object[] items;

  public ArrayBlockingQueue(int capacity, boolean fair) {
      if (capacity <= 0)
          throw new IllegalArgumentException();
      this.items = new Object[capacity];
      lock = new ReentrantLock(fair);
      notEmpty = lock.newCondition();
      notFull =  lock.newCondition();
  }

    public void put(E e) throws InterruptedException {
        checkNotNull(e);
        final ReentrantLock lock = this.lock;
        lock.lockInterruptibly();
        try {
            //如果队列满了，等待
            while (count == items.length)
                notFull.await();
            //此时不满，加入队列
            enqueue(e);
        } finally {
            lock.unlock();
        }
    }
    private void enqueue(E x) {
        final Object[] items = this.items;
        items[putIndex] = x;
        if (++putIndex == items.length)
            putIndex = 0;
        count++;
        //成功添加后可以唤醒等待队列不空的线程
        notEmpty.signal();
    }
    //下面同理
    public E take() throws InterruptedException {
        final ReentrantLock lock = this.lock;
        lock.lockInterruptibly();
        try {
            while (count == 0)
                notEmpty.await();
            return dequeue();
        } finally {
            lock.unlock();
        }
    }

    private E dequeue() {
        final Object[] items = this.items;
        @SuppressWarnings("unchecked")
        E x = (E) items[takeIndex];
        items[takeIndex] = null;
        if (++takeIndex == items.length)
            takeIndex = 0;
        count--;
        if (itrs != null)
            itrs.elementDequeued();
        notFull.signal();
        return x;
    }
```

# Fork/Join框架

Fork/Join框架是Java 7提供的一个用于并行执行任务的框架，是一个把大任务分割成若干个小任务，最终汇总每个小任务结果后得到大任务结果的框架。

## 工作窃取算法

工作窃取（work-stealing）算法是指某个线程从其他队列里窃取任务来执行。一个比较大的任务分割为若干互不依赖的子任务，为了减少线程间的竞争，把这些子任务分别放到不同的队列里，并为每个队列创建一个单独的线程来执行队列里的任务，线程和队列一一对应。比如A线程负责处理A队列里的任务。但是，有的线程会先把自己队列里的任务干完，而其他线程对应的队列里还有任务等待处理。完成任务的线程可以去其他线程的队列里窃取一个任务来执行。而在这时它们会访问同一个队列，所以为了减少窃取任务线程和被窃取任务线程之间的竞争，通常会使用双端队列，被窃取任务线程永远从双端队列的头部拿任务执行，而窃取任务的线程永远从双端队列的尾部拿任务执行。

## Fork/Join框架的设计

大体思路如下：
1. 分割任务
2. 将分割的子任务分别放在双端队列里，然后启动线程分别从双端队列里获取任务执行。子任务执行完的结果都统一放在一个队列里，启动一个线程从队列里拿数据，然后合并这些数据。

具体操作：
1. 创建一个ForkJoin任务，它提供在任务中执行fork()和join()操作的机制，通常选择ForkJoin类的两个子类之一来完成，RecursiveAction（没有返回结果）和RecursiveTask（有返回结果）。
2. 通过ForkJoinPool执行ForkJoin里的任务。

示例代码如下：

```java
public class CountTask extends RecursiveTask<Integer> {
	private static final int THRESHOLD = 2;　　
	private int start;
	private int end;
	public CountTask(int start, int end) {
		this.start = start;
		this.end = end;
}
	@Override
	protected Integer compute() {
		int sum = 0;
		//如果任务足够小就计算任务
		boolean canCompute = (end - start) <= THRESHOLD;
		if (canCompute) {
			for (int i = start; i <= end; i++) {
				sum += i;
			}
		} else {
			//如果任务大于阈值，就分裂成两个子任务计算
			int middle = (start + end) / 2;
			CountTask leftTask = new CountTask(start, middle);
			CountTask rightTask = new CountTask(middle + 1, end);
			//执行子任务
			leftTask.fork();
			rightTask.fork();
			//等待子任务执行完，并得到其结果
			int leftResult=leftTask.join();
			int rightResult=rightTask.join();
			//合并子任务
			sum = leftResult + rightResult;
		}
		return sum;
	}
	public static void main(String[] args) {
		ForkJoinPool forkJoinPool = new ForkJoinPool();
		//生成一个计算任务，负责计算1+2+3+4
		CountTask task = new CountTask(1, 4);
		//执行一个任务
		Future<Integer> result = forkJoinPool.submit(task);
		try {
			System.out.println(result.get());
		} catch (InterruptedException e) {
		} catch (ExecutionException e) {
		}
	}
}
```

ForkJoinTask在执行的时候可能会抛出异常，但是我们没办法在主线程里直接捕获异常，所以ForkJoinTask提供了isCompletedAbnormally()方法来检查任务是否已经抛出异常或已经被取消了，并且可以通过ForkJoinTask的getException方法获取异常。

# CountDownLatch

CountDownLatch的构造函数接收一个int类型的参数作为计数器，如果你想等待N个点完成，这里就传入N。每当调用CountDownLatch的countDown方法时，N就会减1，CountDownLatch的await方法会阻塞当前线程，直到N变成零。由于countDown方法可以用在任何地方，所以这里说的N个点，可以是N个线程，也可以是1个线程里的N个执行步骤。用在多个线程时，只需要把这个CountDownLatch的引用传递到线程里即可。示例代码如下：

```java
public class CountDownLatchTest {
	static CountDownLatch c = new CountDownLatch(2);
	public static void main(String[] args) throws InterruptedException {
		new Thread(new Runnable() {
			@Override
			public void run() {
				System.out.println(1);
				c.countDown();
				System.out.println(2);
				c.countDown();
			}
		}).start();
		c.await();
		System.out.println("3");
	}
}
```

看一下CountDownLatch的实现，实现比较简单：

```java
	//Sync继承自AQS
	private final Sync sync;

	//传入一个count会设置sync里的同步变量值
    public CountDownLatch(int count) {
        if (count < 0) throw new IllegalArgumentException("count < 0");
        this.sync = new Sync(count);
    }

    //await会阻塞线程，当同步变量为0时，抛出异常返回
    public void await() throws InterruptedException {
        sync.acquireSharedInterruptibly(1);
    }

    //释放一个同步变量，也就是计数器值减1
    public void countDown() {
        sync.releaseShared(1);
    }

    protected int tryAcquireShared(int acquires) {
        return (getState() == 0) ? 1 : -1;
    }

    protected boolean tryReleaseShared(int releases) {
        for (;;) {
            int c = getState();
            if (c == 0)
                return false;
            int nextc = c-1;
            if (compareAndSetState(c, nextc))
                return nextc == 0;
        }
    }

```

CountDownLatch可以用于实现需要等待多个线程完成之后才能继续进行下一步的场景，比如游戏中必须等待所有玩家都加载好才能开始进行。

# CyclicBarrier

CyclicBarrier的字面意思是可循环使用（Cyclic）的屏障（Barrier）。它要做的事情是，让一组线程到达一个屏障（也可以叫同步点）时被阻塞，直到最后一个线程到达屏障时，屏障才会开门，所有被屏障拦截的线程才会继续运行。

CyclicBarrier默认的构造方法是CyclicBarrier（int parties），其参数表示屏障拦截的线程数量，每个线程调用await方法告诉CyclicBarrier我已经到达了屏障，然后当前线程被阻塞。示例代码如下：

```java
public class CyclicBarrierTest {
	static CyclicBarrier c = new CyclicBarrier(2);
	public static void main(String[] args) {
		new Thread(new Runnable() {
			@Override
			public void run() {
				try {
					c.await();
				} catch (Exception e) {
				}
				System.out.println(1);
			}
		}).start();
		try {
			c.await();
		} catch (Exception e) {
		}
		System.out.println(2);
	}
}
```

CyclicBarrier还提供一个更高级的构造函数CyclicBarrier（int parties，Runnable barrierAction），用于在线程到达屏障时，优先执行barrierAction，方便处理更复杂的业务场景，示例代码如下：

```java
import java.util.concurrent.CyclicBarrier;
public class CyclicBarrierTest2 {
	static CyclicBarrier c = new CyclicBarrier(2, new A());
	public static void main(String[] args) {
		new Thread(new Runnable() {
			@Override
			public void run() {
				try {
					c.await();
				} catch (Exception e) {
				}
				System.out.println(1);
			}
		}).start();
		try {
			c.await();
		} catch (Exception e) {
		}
		System.out.println(2);
	}
	static class A implements Runnable {
		@Override
		public void run() {
			System.out.println(3);
		}
	}
}
```

CyclicBarrier可以用于多线程计算数据，最后合并计算结果的场景。CountDownLatch的计数器只能使用一次，而CyclicBarrier的计数器可以使用reset()方法重置。所以CyclicBarrier能处理更为复杂的业务场景。

## CyclicBarrier实现

```java
	private static class Generation {
        boolean broken = false;
	}


	
	private void breakBarrier() {
        generation.broken = true;
        count = parties;
        trip.signalAll();
	}

	private void nextGeneration() {
        trip.signalAll();
        count = parties;
        generation = new Generation();
    }
	
    public int await() throws InterruptedException, BrokenBarrierException {
        try {
            return dowait(false, 0L);
        } catch (TimeoutException toe) {
            throw new Error(toe);
        }
    }

    private int dowait(boolean timed, long nanos)
        throws InterruptedException, BrokenBarrierException,
               TimeoutException {
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            final Generation g = generation;
            //该状态意味着屏障已被打破
            if (g.broken)
                throw new BrokenBarrierException();
            //如果线程被中断，则打破屏障
            if (Thread.interrupted()) {
                breakBarrier();
                throw new InterruptedException();
            }

			int index = --count;
			//所有线程都到达了屏障
            if (index == 0) {
                boolean ranAction = false;
                try {
                    final Runnable command = barrierCommand;
                    if (command != null)
                        command.run();
                    ranAction = true;
                    //相当于重置屏障
                    nextGeneration();
                    return 0;
                } finally {
                    if (!ranAction)
                        breakBarrier();
                }
            }

            for (;;) {
                try {
                    //通过等待机制
                    if (!timed)
                        trip.await();
                    else if (nanos > 0L)
                        nanos = trip.awaitNanos(nanos);
                } catch (InterruptedException ie) {
                    if (g == generation && ! g.broken) {
                        breakBarrier();
                        throw ie;
                    } else {
                        Thread.currentThread().interrupt();
                    }
                }

                if (g.broken)
                    throw new BrokenBarrierException();

                if (g != generation)
                    return index;

                if (timed && nanos <= 0L) {
                    breakBarrier();
                    throw new TimeoutException();
                }
            }
        } finally {
            lock.unlock();
        }
    }
```

可以见到，CyclicBarrier的基本思路是wait/notify机制，调用await方法的线程会阻塞在条件上，等到所有线程都到达屏障之后，所有线程都会抛出异常返回。

# Semaphore

Semaphore（信号量）是用来控制同时访问特定资源的线程数量，它通过协调各个线程，以保证合理的使用公共资源。Semaphore可以用于做流量控制，特别是公用资源有限的应用场景，比如数据库连接。假
如有一个需求，要读取几万个文件的数据，因为都是IO密集型任务，我们可以启动几十个线程并发地读取，但是如果读到内存后，还需要存储到数据库中，而数据库的连接数只有10个，这时我们必须控制只有10个线程同时获取数据库连接保存数据，否则会报错无法获取数据库连接。这个时候，就可以使用Semaphore来做流量控制,示例代码如下：

```java
public class SemaphoreTest {
	private static final int THREAD_COUNT = 30;
	private static ExecutorServicethreadPool = Executors.newFixedThreadPool(THREAD_COUNT);
	private static Semaphore s = new Semaphore(10);
	public static void main(String[] args) {
		for (int i = 0; i< THREAD_COUNT; i++) {
			threadPool.execute(new Runnable() {
			@Override
			public void run() {
				try {
					s.acquire();
					System.out.println("save data");
					s.release();
				} catch (InterruptedException e) {
				}
			}
			});
		}
		threadPool.shutdown();
	}
}
```

Semaphore的构造方法Semaphore（int permits）接受一个整型的数字，表示可用的许可证数量。Semaphore的用法也很简单，首先线程使用Semaphore的acquire()方法获取一个许可证，使用完之后调用release()方法归还许可证。还可以用tryAcquire()方法尝试获取许可证。

## Semaphore实现

```java
    abstract static class Sync extends AbstractQueuedSynchronizer {
        private static final long serialVersionUID = 1192457210091910933L;

        Sync(int permits) {
            setState(permits);
        }

        final int getPermits() {
            return getState();
        }

        final int nonfairTryAcquireShared(int acquires) {
            for (;;) {
                int available = getState();
                int remaining = available - acquires;
                if (remaining < 0 ||
                    compareAndSetState(available, remaining))
                    return remaining;
            }
        }

        protected final boolean tryReleaseShared(int releases) {
            for (;;) {
                int current = getState();
                int next = current + releases;
                if (next < current) 
                    throw new Error("Maximum permit count exceeded");
                if (compareAndSetState(current, next))
                    return true;
            }
        }

        final void reducePermits(int reductions) {
            for (;;) {
                int current = getState();
                int next = current - reductions;
                if (next > current) 
                    throw new Error("Permit count underflow");
                if (compareAndSetState(current, next))
                    return;
            }
        }

        final int drainPermits() {
            for (;;) {
                int current = getState();
                if (current == 0 || compareAndSetState(current, 0))
                    return current;
            }
        }
	}
	
	public void acquire() throws InterruptedException {
        sync.acquireSharedInterruptibly(1);
	}
	
	public void release() {
        sync.releaseShared(1);
    }
```

Semaphore的实现和CountDownLatch很接近，只是CountDownLatch每次countdown是释放同步变量，而Semaphore每次release是增加同步变量。

# Exchanger

Exchanger（交换者）是一个用于线程间协作的工具类。Exchanger用于进行线程间的数据交换。它提供一个同步点，在这个同步点，两个线程可以交换彼此的数据。这两个线程通过exchange方法交换数据，如果第一个线程先执行exchange()方法，它会一直等待第二个线程也执行exchange方法，当两个线程都到达同步点时，这两个线程就可以交换数据，将本线程生产出来的数据传递给对方。

Exchanger可以用于遗传算法，遗传算法里需要选出两个人作为交配对象，这时候会交换两人的数据，并使用交叉规则得出2个交配结果。Exchanger也可以用于校对工作，比如我们需要将纸制银行流水通过人工的方式录入成电子银行流水，为了避免错误，采用AB岗两人进行录入，录入到Excel之后，系统需要加载这两个Excel，并对两个Excel数据进行校对，看看是否录入一致，代码如下：

```java
public class ExchangerTest {
	private static final Exchanger<String>exgr = new Exchanger<String>();
	private static ExecutorServicethreadPool = Executors.newFixedThreadPool(2);
	public static void main(String[] args) {
		threadPool.execute(new Runnable() {
			@Override
			public void run() {
				try {
					String A = "银行流水A";　　　　// A录入银行流水数据
					exgr.exchange(A);
				} catch (InterruptedException e) {
				}
			}
		});
		threadPool.execute(new Runnable() {
			@Override
			public void run() {
				try {
					String B = "银行流水B";　　　　// B录入银行流水数据
					String A = exgr.exchange(B);
					System.out.println("A和B数据是否一致：" + A.equals(B) + "，A录入的是："
						+ A + "，B录入是：" + B);
				} catch (InterruptedException e) {
				}
			}
		});
		threadPool.shutdown();
	}
}
```

# 参考文章

方腾飞、魏鹏、程晓明：Java并发编程的艺术
