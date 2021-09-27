# AQS 相关内容

## 可重入锁

递归锁，不可重入可能会造成死锁
同一个线程再外层方法获取锁的时候，进入该线程的内存方法会自动获取锁（这两个锁是同一把锁，否则会造成死锁）
不会因为没有释放而阻塞。

ReentrantLock 和 synchronized 都是可重入锁

可以再次进去同步域

种类：
synchronized 隐式的锁，默认可重入
Lock 显示锁，ReentrantLock 可重入锁

Lock 需要手动释放锁

同步方法的同步监视器是this，也就是调用该方法的对象, 也就是  new ReEnterLockMethodDemo()

synchronized:
使用 javac 进行反编译

monitorenter 和 monitorexit

最后还有一个 monitorexit, 保证异常也能释放锁

每个锁对象都有一个锁计数器和一个指向持有该锁现成的一个指针

执行 monitorenter 时，首先判断目标锁对象的计数器为0，则当前线程获得该锁，锁对象计数+1

如果当前目标对象计数器不为0，当前线程再次操作的时候，锁对象计数器加1，实现可重入，否则需要等待当前持有线程释放锁

执行一次 monitorexit 锁对象计数器减1，当计数器为0代表锁已经释放

![20210913215506](https://cdn.jsdelivr.net/gh/RamboCao/PicGo/images/20210913215506.png)

ReentrantLock 代码示例

```java
package com.juc.aqs;

import java.util.concurrent.locks.Lock;
import java.util.concurrent.locks.ReentrantLock;

public class ReEnterReentrantLockDemo {

    static Lock lock = new ReentrantLock();


    public static void main(String[] args) {

        new Thread(() -> {
            lock.lock();
            lock.lock();
            try {
                System.out.println(" ===外层");
                lock.lock();
                try {
                    System.out.println(" ===内层");
                }finally {
                    lock.unlock();
                }
            }finally {
                lock.unlock();
                // 最好加锁多少次减多少次，注掉一个 lock.unlock() 也正常，因为是一把锁
                // 但是当有两个线程的时候，第二个线程就需要进行等待，无法获得锁并执行，所以必须是加几次就释放几次
                lock.unlock();
            }
        }, "t1").start();

        new Thread(() -> {
            lock.lock();
            try {
                System.out.println(" t2 调用开始===");
            }finally {
                lock.unlock();
            }
        }, "t2").start();
    }
}

```

### LockSupport

juc 包下的一个类，用于创建锁和其它同步类的线程阻塞原语

线程等待唤醒机制（wait/notify）

两个方法: park() 和 unpark() 挂起和解除阻塞线程

synchronized: Object 类中的 wait() 和 notify
lock: Condition 中的 await() 和 signal()

LockSupport 加强改良版

wait() 和 notify() 不能脱离同步代码块synchronized 使用

```java
public class LockSupportDemo {

    static Object objectLock = new Object();
    // 如果将 synchronized 关键字去掉，则会抛出异常 Exception in thread "A" java.lang.IllegalMonitorStateException
    public static void main(String[] args) {
        // notify() 放在 wait() 前边，程序一直无法结束，所以应该先 wait() 后 notify()
        new Thread(() ->{
            try {
                TimeUnit.SECONDS.sleep(3);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            synchronized (objectLock){
                System.out.println(Thread.currentThread().getName() + " ----come in !!!");
                try {

                    objectLock.wait();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                System.out.println(Thread.currentThread().getName() + " ----被唤醒 !!!");
            }
        }, "A").start();
        // Exception in thread "B" java.lang.IllegalMonitorStateException
        new Thread(() ->{
            synchronized (objectLock){
                objectLock.notify();
                System.out.println(Thread.currentThread().getName() + " ----通知 !!!");
            }
        }, "B").start();

    }
}

```

await() 和 singal() 方法必须和 lock.lock() , lock.unlock() 组合才能使用，否则也会报错 Exception in thread "A" java.lang.IllegalMonitorStateException

```java

static Lock lock = new ReentrantLock();
static Condition condition = lock.newCondition();
public static void main(String[] args) {
        // 如果将  lock.lock() , lock.unlock() 关键字去掉，则会抛出异常 Exception in thread "A" java.lang.IllegalMonitorStateException
        // signal() 放在 await() 前边，程序一直无法结束，所以应该先 await() 后 signal()
        new Thread(() ->{
            try {
                TimeUnit.SECONDS.sleep(3);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            lock.lock();
                try {
                    System.out.println(Thread.currentThread().getName() + " ----come in !!!");
                    condition.await();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }finally {
                    lock.unlock();
                }
                System.out.println(Thread.currentThread().getName() + " ----被唤醒 !!!");
        }, "A").start();

        new Thread(() ->{
            lock.lock();
            try {
                condition.signal();
                System.out.println(Thread.currentThread().getName() + " ----通知 !!!");
            } finally {
                lock.unlock();
            }
        }, "B").start();

    }

```

lockSupport 介绍：

park() unpark() 实现线程阻塞和解除阻塞

许可证！！！

创建锁和其它同步类的基本线程阻塞原语

使用了一种名为 Permit(许可) 的概念来左到阻塞和唤醒线程的功能，每个线程都有一个许可 permit, permit 只有两个值1和0，默认是0
可以将 Permit 堪称是一种(0,1) 信号量(Semaphore) 与 Semaphore 不同，许可的累加上限是1

主要方法
park()

permt 默认值为0，开始调用 park() 方法，当前线程阻塞，知道别的线程将当前线程的 permit 设置为1，等待的线程被唤醒，将 permit 设置为0并返回
用过之后就清空

unpark()

调用 unpark(thread) 方法之后，会将 thread 线程的 permit 设置成1，多次调用也还是1，会自动唤醒 thread 线程，之前阻塞中的 LockSupport.park() 方法会立即返回

调用 Unsafe 类中的 native 方法

调用多次 unpark() 只能生成一个凭证

线程阻塞消耗凭证，凭证只有一个

当调用 park() 方法的时候，有凭证，消耗凭证正常退出，没有凭证则阻塞等待凭证
uppark() 可以增加一个凭证，不能累计，最多只能一个

像下边代码，unpark() 两次只能得到一个线程，所以线程 a 最后还是会被阻塞

```java
public static void main(String[] args) throws InterruptedException {

    // LockSupport 实现了与 synchronized 和 lock 同样的功能
    // lockSupport unpark() 可以再 park() 之前执行， 因为你只是方法一个许可证，可以先给，park() 方法形同虚设
    Thread a = new Thread(() ->{
        try {
            TimeUnit.SECONDS.sleep(3);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println(Thread.currentThread().getName() + " come in");
        LockSupport.park(); // 被阻塞... 等待通知等待放行，需要许可证
        LockSupport.park(); // 被阻塞... 等待通知等待放行，需要许可证
        System.out.println(Thread.currentThread().getName() + " 被唤醒");
    }, "a");
    a.start();

    // 暂停几秒钟线程
    //TimeUnit.SECONDS.sleep(3);

    Thread b = new Thread(() ->{
        LockSupport.unpark(a);
        LockSupport.unpark(a);
        System.out.println(Thread.currentThread().getName() + " 发出许可证");
    }, "b");
    b.start();

}
```

为什么可以先唤醒线程后阻塞线程：

unpark() 获取了一个凭证，之后再调用 park() 方法，就可以根据凭证消费，不会阻塞，我买了票，随时都可以上车

为什么唤醒两次后阻塞两次，最终结果还是会阻塞线程

因为凭证的数量最多为1，连续调用两次 unpark() 和调用一次 unpark() 的效果一样，只会增加一个凭证，调用 park() 需要消费两个凭证，凭证不够，不能放行
如果两次 unpark() 中间停顿一段时间，那么可以再次发放凭证

```java
Thread b = new Thread(() ->{
    LockSupport.unpark(a);
    try {
        TimeUnit.SECONDS.sleep(5);
    } catch (InterruptedException e) {
        e.printStackTrace();
    }
    LockSupport.unpark(a);
    System.out.println(Thread.currentThread().getName() + " 发出许可证");
}, "b");
```

## AQS 面试

抽象队列同步器

![20210915212619](https://cdn.jsdelivr.net/gh/RamboCao/PicGo/images/20210915212619.png)

抽象：抽象类
队列：数据结构
同步器：父类中定义了一些公共的通用方法

构建锁或者其它同步组件的重量级基础框架以及整个 JUC 体系的基石
内置的 FIFO 队列完成资源的获取和线程的排队工作，通过一个 int 类型的变量表示持有锁的状态

CLH(FIFO 队列) 默认是一个单向链表，AQS 中的队列是一个变种的虚拟的双向队列

### AQS 为什么是基石

ReentrantLock
CountDownLatch
ReentrantReadWriteLock
Semaphore

锁：面向锁的使用者 -> 定义了程序员和锁交互的使用层 API
同步器：面向锁的是闲着 -> 屏蔽了同步状态管理，阻塞线程排队实现，实现的统一规范

### 为什么需要 AQS

加锁会导致阻塞，有阻塞就需要排队，实现排队必须要有某种形式的排队等候机制，阻塞的节点仍然有获取锁的可能性
队列的数据结构：CLH 队列的变体，暂时获取不到锁的低劣加入到队列中

将请求共享资源的线程封装成队列的节点 Node, CAS、自旋、LockSupport 的方式，维护 state 变量的状态，使得并发达到同步的控制效果

有阻塞就要排队，排队比然需要实现队列

volatile 的 int 类型变量，同步状态 state
FIFO 队列完成资源获得的排队工作
所有的线程被封装成一个 Node 节点， Node 节点中有 prev 和 next 节点
通过 CAS 修改 state 的值

### AQS 内部体系架构

AQS 的 int 变量，同步状态 state, =0 没人，自由状态，任何线程都可以抢占，>= 1 队列中等待

CLH 队列，进化为虚拟的双向队列，从尾部入队，从头部出队

state + CLH变种的双端队列

Node 层面

waitStatus + head + tail + prev + next
waitStatus：volatile 修饰的变量，等候区其它顾客的等待状态

Node 类的内部结构

Lock 接口的实现类，聚合了一个队列同步器的子类 Sync 完成进一步锁的操作

Lock 方法查看公平锁/非公平锁

ReentrantLock 默认是非公平锁

公平和非公平：公平锁需要进入队列中，非公平锁直接进行抢占

```java
// 加锁是判断等待队列中是否存在有效的节点
hasQueuedPredecessors()
```

ReentrantLock 加锁过程

1. 尝试获取锁
2. 获取锁失败，加入到队列中
3. 加入到队列中之后，线程进入阻塞状态

## AQS 源码分析

双向链表中，第一个节点为虚节点（哨兵节点） 不存储数据，真正有数据的是哨兵节点的下一个线程

LockSupport.park() 表示节点 B 才真正的排队，被阻塞中，等待 LockSupport.unpark() 唤醒

接下来 A 线程进行解锁， 释放成功后，将哨兵节点的值 waitStatus 设置为 0， 然后操作哨兵节点的下一个节点

唤醒 B 节点，LockSupport.unpark()

回到 B 节点被 LockSupport.park() 的地方

B 抢占锁成功，设置线程和 state 的值，哨兵节点设置出队，置为 null, 将 B 节点设置为新的哨兵节点
