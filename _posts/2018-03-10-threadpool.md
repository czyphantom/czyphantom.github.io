---
layout:     post
title:      Java并发包源码阅读（三）
subtitle:   线程池
date:       2018-03-10
author:     CZY
header-img: img/bg.png
catalog: true
tags:
    - Java并发
---

通常情况我们会为每一个任务创建一个线程，但是当任务很多时，为每一个任务创建线程显然是不合适的，这显然是对资源的一种浪费。为此，Java提供了线程池技术，以此来合理使用CPU资源。

当提交一个新任务到线程池时，线程池的处理流程如下：
1. 调用ThreadPoolExecutor的execute提交线程，首先检查CorePool，如果CorePool内的线程小于CorePoolSize，新创建线程执行任务。
2. 如果当前CorePool内的线程大于等于CorePoolSize，那么将线程加入到BlockingQueue。
3. 如果不能加入BlockingQueue，在小于MaxPoolSize的情况下创建线程执行任务。
4. 如果线程数大于等于MaxPoolSize，那么执行拒绝策略。

ThreadPoolExecutor的执行示意图如下：

![ThreadPoolExecutor执行流程](http://ww1.sinaimg.cn/large/0060lm7Tly1fp412kukf7j30nm0kvjvf.jpg)


## ThreadPoolExecutor实现

### 几个重要的成员变量

```java
	//ctl是线程池的控制状态，用来表示线程池的运行状态（整形的高3位）和运行的worker数量（低29位））
	private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));
	//以下是状态参数
    private static final int RUNNING    = -1 << COUNT_BITS;
    private static final int SHUTDOWN   =  0 << COUNT_BITS;
    private static final int STOP       =  1 << COUNT_BITS;
    private static final int TIDYING    =  2 << COUNT_BITS;
    private static final int TERMINATED =  3 << COUNT_BITS;
    //存放Worker线程
	private final HashSet<Worker> workers = new HashSet<Worker>();
    //阻塞队列，用于保留不能放在核心池的任务
    private final BlockingQueue<Runnable> workQueue;
    //核心池大小，如果满了且阻塞队列满了会新建Worker线程，但是不能超过maximumPoolSize
    private volatile int corePoolSize;
    //最大线程池大小，也就是说线程池最多能放多少Worker
    private volatile int maximumPoolSize;
    //拒绝策略处理器，一般有以下几种拒绝策略
    //AbortPolicy：直接抛出异常
    //CallerRunsPolicy：由调用者所在线程来运行
    //DiscardOldestPolicy：丢弃队列里最近的一个任务，并执行当前任务
    //DiscardPolicy：不处理，丢弃掉
    private volatile RejectedExecutionHandler handler;
```

### 构造器

有以下几种构造器：

```java
    public ThreadPoolExecutor(int corePoolSize,
                              int maximumPoolSize,
                              long keepAliveTime,
                              TimeUnit unit,
                              BlockingQueue<Runnable> workQueue) {
        this(corePoolSize, maximumPoolSize, keepAliveTime, unit, workQueue,
             Executors.defaultThreadFactory(), defaultHandler);
    }

    public ThreadPoolExecutor(int corePoolSize,
                              int maximumPoolSize,
                              long keepAliveTime,
                              TimeUnit unit,
                              BlockingQueue<Runnable> workQueue,
                              ThreadFactory threadFactory) {
        this(corePoolSize, maximumPoolSize, keepAliveTime, unit, workQueue,
             threadFactory, defaultHandler);
    }

    public ThreadPoolExecutor(int corePoolSize,
                              int maximumPoolSize,
                              long keepAliveTime,
                              TimeUnit unit,
                              BlockingQueue<Runnable> workQueue,
                              RejectedExecutionHandler handler) {
        this(corePoolSize, maximumPoolSize, keepAliveTime, unit, workQueue,
             Executors.defaultThreadFactory(), handler);
    }

    //corePoolSize是核心池大小
    //maximumPoolSize是最大池大小
    //keepAliveTime是超过核心池大小的线程空闲多长时间被回收
    //unit是keepAlive的时间单位
    //workQueue是阻塞队列
    //threadFactory是用来创建新线程的工厂
    //handler是拒绝策略处理器
    public ThreadPoolExecutor(int corePoolSize,
                              int maximumPoolSize,
                              long keepAliveTime,
                              TimeUnit unit,
                              BlockingQueue<Runnable> workQueue,
                              ThreadFactory threadFactory,
                              RejectedExecutionHandler handler) {
        if (corePoolSize < 0 ||
            maximumPoolSize <= 0 ||
            maximumPoolSize < corePoolSize ||
            keepAliveTime < 0)
            throw new IllegalArgumentException();
        if (workQueue == null || threadFactory == null || handler == null)
            throw new NullPointerException();
        this.acc = System.getSecurityManager() == null ?
                null :
                AccessController.getContext();
        this.corePoolSize = corePoolSize;
        this.maximumPoolSize = maximumPoolSize;
        this.workQueue = workQueue;
        this.keepAliveTime = unit.toNanos(keepAliveTime);
        this.threadFactory = threadFactory;
        this.handler = handler;
    }
```

### 线程的提交

可以使用两个方法向线程池提交任务，分别为execute()和submit()方法。execute()方法用于提交不需要返回值的任务，所以无法判断任务是否被线程池执行成功。submit()方法用于提交需要返回值的任务。线程池会返回一个future类型的对象，通过这个future对象可以判断任务是否执行成功，并且可以通过future的get()方法来获取返回值，get()方法会阻塞当前线程直到任务完成，而使用get（long timeout，TimeUnit unit）方法则会阻塞当前线程一段时间后立即返回，这时候有可能任务没有执行完。以execute方法为例：

```java
    public void execute(Runnable command) {
        if (command == null)
            throw new NullPointerException();
        int c = ctl.get();
        //如果线程池内线程数量小于核心线程数
        if (workerCountOf(c) < corePoolSize) {
        	//将该线程加入到线程池，返回
            if (addWorker(command, true))
                return;
            //不成功，再次获取线程池状态
            c = ctl.get();
        }
        //如果线程池处于运行状态，将任务加入到workQueue
        if (isRunning(c) && workQueue.offer(command)) {
        	//再次检查线程池状态
            int recheck = ctl.get();
            //如果线程池不处于运行状态，将该线程从workQueue中移除
            if (! isRunning(recheck) && remove(command))
                //使用拒绝处理器拒绝执行
                reject(command);
            //如果此时线程池中没有工作线程，需要新建一个来执行任务
            else if (workerCountOf(recheck) == 0)
                addWorker(null, false);
        }
        //如果workQueue满了，线程池可能还没满，尝试在线程池中增加一个Worker，否则拒绝此线程
        else if (!addWorker(command, false))
            reject(command);
    }

    //该方法往线程池添加工作线程
    private boolean addWorker(Runnable firstTask, boolean core) {
        retry:
        for (;;) {
            int c = ctl.get();
            int rs = runStateOf(c);
            //以下情况不能添加线程
            if (rs >= SHUTDOWN &&
                ! (rs == SHUTDOWN &&
                   firstTask == null &&
                   ! workQueue.isEmpty()))
                return false;
            //首先更新工作线程数量，自旋保证正确更新
            for (;;) {
                int wc = workerCountOf(c);
                //超出线程池最大容量或者线程池大小，更新失败
                if (wc >= CAPACITY ||
                    wc >= (core ? corePoolSize : maximumPoolSize))
                    return false;
                //CAS添加Worker的数量，成功直接跳出循环，执行真正加入worker
                if (compareAndIncrementWorkerCount(c))
                    break retry;
                //增加Worker数量失败，重新获得线程池状态
                c = ctl.get();  
                //发现和之前的状态不一样，继续循环
                if (runStateOf(c) != rs)
                    continue retry;

            }
        }
        //此时已经是添加worker数量成功了，接下来真正添加worker了
        boolean workerStarted = false;
        boolean workerAdded = false;
        //Worker是一个继承了AQS的类
        Worker w = null;
        try {
            w = new Worker(firstTask);
            final Thread t = w.thread;
            if (t != null) {
                //通过线程池的锁保证线程安全
                final ReentrantLock mainLock = this.mainLock;
                mainLock.lock();
                try {
                    int rs = runStateOf(ctl.get());
                    //线程池运行中或者线程池shutdown了但是添加的任务为空
                    if (rs < SHUTDOWN ||
                        (rs == SHUTDOWN && firstTask == null)) {
                        //这种情况不可能出现
                        if (t.isAlive())
                            throw new IllegalThreadStateException();
                        //往HashSet里添加worker
                        workers.add(w);
                        int s = workers.size();
                        //更新最大线程数量
                        if (s > largestPoolSize)
                            largestPoolSize = s;
                        //成功添加
                        workerAdded = true;
                    }
                } finally {
                    mainLock.unlock();
                }
                //成功添加，启动线程
                if (workerAdded) {
                    t.start();
                    workerStarted = true;
                }
            }
        } finally {
        	//如果没有开始，说明添加worker失败，从workers里移除，worker数量也恢复
            if (! workerStarted)
                addWorkerFailed(w);
        }
        return workerStarted;
    }
```

线程池提交的详细流程如下：
1. 判断线程池内线程数量是否小于核心池，如果小于，准备添加到线程池，添加成功则返回。
2. 如果添加失败或者线程数大于核心池，将任务添加到工作队列，如果添加失败，则准备添加到线程池，添加线程池失败则执行拒绝策略。
3. 添加到线程池分为两步，第一步更新线程池内线程数量，通过自旋保证正确更新，如果超出线程池最大容量或者线程池大小，则更新失败。否则使用CAS正确更新，更新成功后跳出循环。
4. 第二步往线程池内添加工作线程。首先对线程池加锁，接着判断可以往线程池添加后，将任务构造一个Worker加入到保存工作线程的HashSet里。添加完毕后释放锁，启动该线程。

### 线程池的终止

可以通过调用线程池的shutdown或shutdownNow方法来关闭线程池。它们的原理是遍历线程池中的工作线程，然后逐个调用线程的interrupt方法来中断线程，所以无法响应中断的任务可能永远无法终止。但是它们存在一定的区别，shutdownNow首先将线程池的状态设置成STOP，然后尝试停止所有的正在执行或暂停任务的线程，并返回等待执行任务的列表，而shutdown只是将线程池的状态设置成SHUTDOWN状态，然后中断所有没有正在执行任务的线程。

只要调用了这两个关闭方法中的任意一个，isShutdown方法就会返回true。当所有的任务都已关闭后，才表示线程池关闭成功，这时调用isTerminaed方法会返回true。至于应该调用哪一种方法来关闭线程池，应该由提交到线程池的任务特性决定，通常调用shutdown方法来关闭线程池，如果任务不一定要执行完，则可以调用shutdownNow方法。

```java
    public void shutdown() {
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            checkShutdownAccess();
            //设置线程池状态为SHUTDOWN
            advanceRunState(SHUTDOWN);
            //中断所有空闲Worker
            interruptIdleWorkers();
            //调用shutdown钩子函数
            onShutdown();
        } finally {
            mainLock.unlock();
        }
        //尝试终止
        tryTerminate();
    }

    private void interruptIdleWorkers(boolean onlyOne) {
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            for (Worker w : workers) {
                Thread t = w.thread;
                //注意这个tryLock方法，Worker在运行之前都会获得一个锁，tryLock方法执行成功意味着该Worker不在运行
                if (!t.isInterrupted() && w.tryLock()) {
                    try {
                        t.interrupt();
                    } catch (SecurityException ignore) {
                    } finally {
                        w.unlock();
                    }
                }
                if (onlyOne)
                    break;
            }
        } finally {
            mainLock.unlock();
        }
    }

    public List<Runnable> shutdownNow() {
        List<Runnable> tasks;
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            checkShutdownAccess();
            advanceRunState(STOP);
            //中断所有线程
            interruptWorkers();
            tasks = drainQueue();
        } finally {
            mainLock.unlock();
        }
        tryTerminate();
        return tasks;
    }

    private void interruptWorkers() {
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            for (Worker w : workers)
                w.interruptIfStarted();
        } finally {
            mainLock.unlock();
        }
    }
```

线程池的关闭（shutdown）详细流程如下：
1. 对线程池加锁。
2. 设置线程池状态为SHUTDOWN。
3. 中断所有空闲的worker线程。
4. 解锁。
5. 尝试终止线程池。

###　线程池状态转换

线程池有5种状态：Running、ShutDown、Stop、Tidying、Terminated。各种状态解释及转换如下：

1、RUNNING
+ 状态说明：线程池处在RUNNING状态时，能够接收新任务，以及对已添加的任务进行处理。 
+ 状态切换：线程池的初始化状态是RUNNING。线程池被一旦被创建，就处于RUNNING状态，并且线程池中的任务数为0。

2、 SHUTDOWN
+ 状态说明：线程池处在SHUTDOWN状态时，不接收新任务，但能处理已添加的任务。 
+ 状态切换：调用线程池的shutdown()接口时，线程池由RUNNING -> SHUTDOWN。

3、STOP
+ 状态说明：线程池处在STOP状态时，不接收新任务，不处理已添加的任务，并且会中断正在处理的任务。 
+ 状态切换：调用线程池的shutdownNow()接口时，线程池由(RUNNING or SHUTDOWN ) -> STOP。

4、TIDYING
+ 状态说明：当所有的任务已终止，ctl记录的”任务数量”为0，线程池会变为TIDYING状态。当线程池变为TIDYING状态时，会执行钩子函数terminated()。terminated()在ThreadPoolExecutor类中是空的，若用户想在线程池变为TIDYING时，进行相应的处理；可以通过重载terminated()函数来实现。 
+ 状态切换：当线程池在SHUTDOWN状态下，阻塞队列为空并且线程池中执行的任务也为空时，就会由 SHUTDOWN -> TIDYING。 
当线程池在STOP状态下，线程池中执行的任务为空时，就会由STOP -> TIDYING。

5、 TERMINATED
+ 状态说明：线程池彻底终止，就变成TERMINATED状态。 
+ 状态切换：线程池处在TIDYING状态时，执行完terminated()之后，就会由 TIDYING -> TERMINATED。

## 合理配置线程池

性质不同的任务可以用不同规模的线程池分开处理。CPU密集型任务应配置尽可能小的线程，如配置Ncpu+1个线程的线程池。由于IO密集型任务线程并不是一直在执行任务，则应配置尽可能多的线程，如2Ncpu。混合型的任务，如果可以拆分，将其拆分成一个CPU密集型任务和一个IO密集型任务，只要这两个任务执行的时间相差不是太大，那么分解后执行的吞吐量将高于串行执行的吞吐量。如果这两任务执行时间相差太大，则没必要进行分解。可以通过Runtime.getRuntime().availableProcessors()方法获得当前设备的CPU个数。优先级不同的任务可以使用优先级队列PriorityBlockingQueue来处理。它可以让优先级高的任务先执行。依赖数据库连接池的任务，因为线程提交SQL后需要等待数据库返回结果，等待的时间越长，则CPU空闲时间就越长，那么线程数应该设置得越大，这样才能更好地利用CPU。

建议使用有界队列。有界队列能增加系统的稳定性和预警能力，可以根据需要设大一点儿，比如几千。

## 监控线程池

可以通过线程池提供的参数进行监控，在监控线程池的时候可以使用以下属性：
+ taskCount：线程池需要执行的任务数量。
+ completedTaskCount：线程池在运行过程中已完成的任务数量，小于或等于taskCount。
+ largestPoolSize：线程池里曾经创建过的最大线程数量。通过这个数据可以知道线程池是否曾经满过。如该数值等于线程池的最大大小，则表示线程池曾经满过。
+ getPoolSize：线程池的线程数量。如果线程池不销毁的话，线程池里的线程不会自动销毁，所以这个大小只增不减。
+ getActiveCount：获取活动的线程数。

也可以通过扩展线程池进行监控。可以通过继承线程池来自定义线程池，重写线程池的beforeExecute、afterExecute和terminated方法，也可以在任务执行前、执行后和线程池关闭前执行一些代码来进行监控。

## 参考文章

[JDK1.8源码分析之ThreadPoolExecutor](https://www.cnblogs.com/leesf456/p/5585627.html)
[深入理解java线程池—ThreadPoolExecutor](https://www.jianshu.com/p/ade771d2c9c0)
方腾飞、魏鹏、程晓明：Java并发编程的艺术
