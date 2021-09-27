# ThreadPool

## 线程池的三种创建方式

Runnable 和 Callable 接口比较

1. Callable 有返回值，Runnable 没有返回值
2. Callable 可以抛出异常
3. 接口方法不一样，一个是 run, 一个是 call

Thread 全部构造方法都没有 Callable 作为参数的

返回值怎么拿

构造注入

适配模式  

Callable 和 Runnable 接口的适配

一个类同时实现了 Callable 和 Runnable 接口

RunnableFuture 接口实现了 Runnable 接口

FutureTask 已知的实现类，实现了 Runnable 接口，构造方法中有 Callable 接口

FutureTask 将来任务

分支合并 ForkJoin

future.get() 建议放在最后，两个线程，一个是 main, 一个是 AA(futureTask), get() 方法是要求获得 Callable 线程的计算结果，如果没有没有计算完成就去强求，会导致阻塞，直到计算完成

获得结果两种方法，一种是 get(), 另外一种判断任务是否结束，然后获取结果

```java
// 这个地方就是自旋， CPU 进行空转
while (!futureTask.isDone()){
    
}
```

```java
class MyThread2 implements Callable<Integer>{

    @Override
    public Integer call() throws Exception {
        System.out.println("coming in callable");
        return 1024;
    }
}
public class ThreadPoolDemo {

public static void main(String[] args) throws ExecutionException, InterruptedException {

    // 构造方法 FutureTask(Callable<V> callable)
    FutureTask<Integer> futureTask = new FutureTask<Integer>(new MyThread2());
    Thread t1 = new Thread(futureTask, "callable");
    Thread t2 = new Thread(futureTask, "callable2");
    t1.start();
    t2.start();
```

coming in callable 只会被打印一次，计算指挥进行一次，如果想多算，那就起多个线程， futureTask 只会被执行一次

为什么使用构造注入，因为灵活，传入接口实现方法也可以，传参一定要传接口

![20210907215817](https://cdn.jsdelivr.net/gh/RamboCao/PicGo/images/20210907215817.png)

## 线程池

线程池，为什么使用，优势？

池化技术：避免上下文切换，避免资源消耗

线程池：

1. 控制运行的线程的数量，处理过程将任务放到队列，线程创建后启动这些任务
2. 如果线程数量数量超过最大数量，超出的线程排队等候，其他线程执行完毕，再从队列中取出任务执行

线程复用，控制最大并发数量，管理线程

![20210907220922](https://cdn.jsdelivr.net/gh/RamboCao/PicGo/images/20210907220922.png)

线程池架构说明：

Executor 框架实现

Executor -> ExecutorService -> AbstractExecutorService -> ThreadPoolExecutor -> ScheduledThreadPoolExecutor

线程池的实现类：ThreadPoolExecutor

Executors 操作类

java 中有四种线程池

// 调度功能的线程

Executors.newSingleThreadExecutor(); // 固定线程, 保证所有的任务都按照所有的指定的顺序执行， 底层使用 LinkedBlockingQueue

Executors.newCachedThreadPool(); // 扩容多线程，适合短期很多的异步任务。底层使用 SynchronousQueue, 同步阻塞队列，核心线程为0，最大线程为 Integer.Max, 来了就创建，超过60s 空闲就销毁线程

Executors.newFixedThreadPool(5); // 固定线程，执行长期的任务，控制最大并发数，超出的线程会在等待队列中等待，底层使用 LinkedBlockingQueue

ThreadPoolExecutor 底层原理

```java
public static ExecutorService newFixedThreadPool(int nThreads, ThreadFactory threadFactory) {
    return new ThreadPoolExecutor(nThreads, nThreads,
                                    0L, TimeUnit.MILLISECONDS,
                                    new LinkedBlockingQueue<Runnable>(),
                                    threadFactory);
}

public static ExecutorService newSingleThreadExecutor() {
    return new FinalizableDelegatedExecutorService
        (new ThreadPoolExecutor(1, 1,
                                0L, TimeUnit.MILLISECONDS,
                                new LinkedBlockingQueue<Runnable>()));
}

public static ExecutorService newCachedThreadPool() {
    return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                    60L, TimeUnit.SECONDS,
                                    new SynchronousQueue<Runnable>());
}
```
### 线程池的 7 大参数

```java
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
    this.corePoolSize = corePoolSize;
    this.maximumPoolSize = maximumPoolSize;
    this.workQueue = workQueue;
    this.keepAliveTime = unit.toNanos(keepAliveTime);
    this.threadFactory = threadFactory;
    this.handler = handler;
}
```

int corePoolSize: 核心线程数，线程池中常驻的核心线程数量
maximumPoolSize:
keepAliveTime:
unit:
workQueue:
threadFactory:
RejectedExecutionHandler handler

以银行为例：

线程池 == 银行网点

核心线程数量：银行的今日当值窗口，有顾客过来就直接办理，有任务过来就直接去执行，当核心线程数目达到最大的 corePoolSize 数量时，任务进入阻塞队列

maximumPoolSize 最大线程数量：网点人太多，队列满了，加大窗口的数量，但是不能超过最大数量

TimeUnit unit 设置时间的单位

workQueue: 阻塞工作队列，坐在凳子上等待办理业务的人

ThreadFactory threadFactory： 生成线程池中工作现成的工厂，默认即可，可以理解为大堂经理，默认即可

候客区满了，还是有任务进来，创建新的加班窗口，但是不能超过最大数量，先执行队列中的任务

long keepAliveTime, 多余的空闲线程的存活时间，当前线程数量超过 corePoolSize 的时候，空闲时间达到 keepAliveTime 的值，多余的空闲线程会被摧毁直到剩下 corePoolSize 个线程为止

RejectedExecutionHandler handler 拒绝策略，窗口满了，队列也满了，执行拒绝策略，默认四种方法

### 线程池的拒绝策略理论简介

拒绝策略：等待队列满了，线程池线程 max， 无法为新的新任务服务

RejectedExecutionHandler

1. AbortPolicy 抛出异常 RejectedExecutionException 阻止系统正常运行

2. CallerRunsPolicy 调用者运行 "一种调节机制"， 无异常，也不抛弃，任务回退到调用者

3. DiscardOldestPolicy 抛弃队列中等待醉酒的任务，将当前任务加入到队列中尝试再次提交当前任务

4. DiscardPolicy 直接丢弃任务，不处理，也不抛异常，允许任务丢失，则是最好的方案

### 线程池实际使用哪一个

上述三种哪个都不用，因为三个的阻塞队列是无界的，大小不是固定的，堆积大量的请求导致 OOM

所以需要通过 ThrwadPoolExecutor, 不允许显示创建线程池

### 工作中如何使用线程池，是否自定义过线程池使用

手写线程池，拒绝策略：

```java
// AbortPolicy() 直接抛出异常
// CallerRunsPolicy() main 办理业务, 不抛异常，main 线程让线程调用的，直接跑回给 main 线程， 也可能会出现两个 main, 超过 8 就被退回
// DiscardOldestPolicy() 等待最久的被抛弃，新来的请求被加入
// DiscardPolicy() 直接丢弃
ExecutorService threadPool = new ThreadPoolExecutor(2, 5, 1L,
        TimeUnit.SECONDS, new LinkedBlockingDeque<>(3), Executors.defaultThreadFactory(), new ThreadPoolExecutor.DiscardPolicy());

try {
    for (int i = 0; i < 10; i++) {
        threadPool.submit(() -> {
            System.out.println(Thread.currentThread().getName() + " 办理业务");
        });
    }
}catch (Exception e){
    e.printStackTrace();
}finally {
    threadPool.shutdown();
}
```

### 线程池现成的最大数量

合理配置线程池：

1. CPU 密集型：

```java
// 获取 cpu 核心数量
System.out.println(Runtime.getRuntime().availableProcessors());
```

查看 CPU 核心数量

没有阻塞，CPU 一直全速运行

CPU + 1 个线程的线程池，尽量减少切换
2. IO 密集型：

看生产环境决定

不是一直执行任务，尽可能多配置： CPU 核心数量 * 2

大量 IO, 大量阻塞

CPU/(1-阻塞系数)，阻塞系数： 0.8-0.9
如 8 核 CPU: 8/(1-0.9)=80

## 死锁编码以及定位分析

产生死锁的原因：

两个或者两个以上的进程执行过程中，因争夺资源造成的一种互相等待的现象，如果没有外力干涉都将无法推进下去
系统资源充足，不太可能造成死锁

1. 系统资源不足
2. 进程运行推进的顺序不当
3. 资源分配不当

### 如何排查

证明：jps = java ps
jps -l

![20210913211454](https://cdn.jsdelivr.net/gh/RamboCao/PicGo/images/20210913211454.png)

jstack *** 进程号
```bash
Java stack information for the threads listed above:
===================================================
"AAA":
        at com.juc.DeadLockDemo$HoldLockThread.run(DeadLockDemo.java:33)
        - waiting to lock <0x00000007119ec040> (a java.lang.String)
        - locked <0x00000007119ec010> (a java.lang.String)
        at java.lang.Thread.run(java.base@11.0.6/Thread.java:834)
"BBB":
        at com.juc.DeadLockDemo$HoldLockThread.run(DeadLockDemo.java:33)
        - waiting to lock <0x00000007119ec010> (a java.lang.String)
        - locked <0x00000007119ec040> (a java.lang.String)
        at java.lang.Thread.run(java.base@11.0.6/Thread.java:834)

Found 1 deadlock.
```