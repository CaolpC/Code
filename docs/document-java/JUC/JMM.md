# JMM(Java 内存模型)

抽象，不真实存在

一组规则或规范

定义变量的访问方式（类变量）

同步规定：

    线程解锁时，将变量值刷新回主内存

    线程加锁时，去主内存读取变量值到自己的工作内存

    加锁解锁是同一把锁

volatile 是 Java 虚拟机提供的轻量级的同步机制，轻量级体现在：不保证原子性

1. 保证可见性
2. 不保证原子性
3. 禁止指令重排序

## volatile 保证可见性

主内存和工作内存，每个线程都对应一份工作内存

工作内存和虚拟机栈有什么区别？

某一个线程修改了主物理内存的值，其它的线程马上可见 -> 可见性

主内存共享空间，所有线程可见， 工作内存中存储着主内存中的变量副本拷贝

对变量的操作要在工作内存中进行：

1. 将变量拷贝到自己的工作内存空间
2. 变量操作
3. 然后将变量写回主内存

## volatile 不保证原子性

原子性：不可分割，完整性，某个线程正在做莫格业务时候，中间不可以加塞或者分割，需要整体完整，要么同时成功，要么同时失败

volatile 不能保证原子性

i++ 不是原子性操作，三个操作：

1. 读取主库中 i 的值
2. 对操作数进行操作
3. 工作内存写回主内存

对应的字节码为：

1. getfiled
2. iconst_1
3. iadd
4. putfield

假设三个线程都拿到了值，然后在各自的工作内存中进行 +1 操作，写回的时候可能没拿到最新值/或者已经执行了 ++ 操作，然后进行了写覆盖

如何解决原子性：

1. 加 sync
2. atomic -> AtomicInteger

```java
AtomicInteger atomicInteger = new AtomicInteger();

atomicInteger.getAndIncrement() / atimicInteger.getAndAdd(number);
```

## 禁止指令重排序

1，提高性能，编译器处理器对指令进行重排

源代码 -> 编译器优化的重排 -> 指令并行重拍 -> 内存系统的重排 -> 最终执行的指令

***处理器执行重排时必须考虑指令的数据依赖性***

多线程中线程交替执行，编译器优化重排的存在，两个线程中使用的变量能否保证一致性无法确定，结果无法预测

内存屏障的作用：

## 哪些地方用到了 volatile

单例模式:

## CAS

### CAS 是什么？

CompareAndSwap

线程期望值与主内存中之中的值进行比较，如果一样就将修改值写回主内存中，即交换操作
如果不一样就修改失败，重新读取主内存中的值，然后进行修改

### CAS 底层原理

```java
public final int getAndIncrement() {
    return U.getAndAddInt(this, VALUE, 1);
}
```

这个方法解决 i++ 在多线程下的安全问题

this 代表当前对象
valueoffest 内存偏移量
i 增加的值

Unsafe 是 CAS 核心类，直接操作特定内存的数据，像 C 的指针一样直接操作内存， CAS 执行依赖于 Unsafe 类的方法

Unsafe 中的方法直接调用操作系统底层资源执行相应任务

valueOffset 变量在内存中的偏移地址，Unsafe 根据内存偏移地址获取数据

value 使用 volatile 修饰，保证多线程之间内存可见性

CAS 是一条 CPU 并发原语
判断内存某个位置的值是否为预期值，如果是则更改为新的值，过程是原子的
操作系统用语范畴，若干条指令，原语的执行必须是连续的，执行过程中不允许被中断，不会造成数据不一致问题

```java
public final int getAndAddInt(Object o, long offset, int delta) {
    int v;
    do {
        v = getIntVolatile(o, offset);
    } while (!weakCompareAndSetInt(o, offset, v, v + delta));
    return v;
}
```

上述代码也是一个自旋操作，不停进行比较，比较成功就进行更新，否则进行自旋

使用 synchronized 能保证一致性，但是并发度降低

底层汇编：

![20210825224016](https://cdn.jsdelivr.net/gh/RamboCao/PicGo/images/20210825224016.png)

### CAS 缺点

1. 多次比较，浪费资源，循环开销时间较大
2. 只能保证一个共享变量的操作，多个共享变量操作，循环 CAS 无法保证
3. ABA 问题

取出内存中某时刻的数据并在当下时刻比较并替换，那么在这个时间差内会导致数据的变化

线程 1 从内存中取出 A, 线程 2 也从内存中取出 A, 线程 2 进行了一些操作将值变成 B, 然后线程 2 又将值变成 A, 此时线程 1 进行 CAS 操作发现内存中仍然是 A,并且操作成功
尽管线程 1 操作成功，但是并不代表过程没有问题

ABA 只管开头和结尾，不管中间，但是中间时刻内存的值可能发生变化

CAS 不够，引出原子引用 AtomicReference

AtomicReference<T> 可以自己填入类型

### 时间戳原子引用

新增一种机制，修改版本号(类似时间戳)

T1  100 1          2019
T2  100 1  101 2   100 3

AtomicStampedReference 一个对象的引用伴随着一个整型的 stamp, 可以被原子性的更新;

## 集合不安全的问题

### ArrayList

new 了一个空的数组，初始容量为 10，元素类型为 Object

线程不安全！

java.util.ConcurrentModificationException // 并发修改异常
导致原因

    并发争抢导致，参考花名册签名

解决方案

   1. 使用 Vector 类，但是 Vector 的效率较差，并发性急剧下降
   2. Collections.synchronizedList() 线程安全的 ArrayList, Collections 集合的操作类
   3. copyOnWriteArrayList 写时复制， 独写分离的思想

![20210830212758](https://cdn.jsdelivr.net/gh/RamboCao/PicGo/images/20210830212758.png)

    jdk1.8 使用的是 ReentrantLock, 但是 jdk11 中使用了 synchronized 锁
    copyOnWriteArrayList 可以并发读，不需要加锁
    先将当前容器进行 copy, 长度 +1， 然后往新的容器中添加元素，添加完元素之后，将原容器的引用指向新的容器（直接用新的数组覆盖原来的数组）

### HashSet

线程不安全！

解决方案

    1. Collections.synchronizedList(new HashSet())
    2. CopyOnWriteArraySet, 其实还是创建了一个 copyOnWriteArrayList<E>()

HashSet() 底层是 HashMap(), 初始值是15，负载因子是 0.75

HashSet() 的底层是 HashMap() 吗？为什么 add() 只加一个值，不是键值对， add() 添加的值为 PERESENT 的常量(new Object())

### HashMap

线程不安全！

解决方案

    1. Collections.synchronizedMap(new HashMap())
    2. ConcurrentHashMap

这个地方还有新的知识点！！！分段锁等内容

## Java 锁

### Java 公平锁和非公平锁

公平锁：队列，先来后到，按照各个线程申请锁的顺序获取
非公平：可以进行插队, 抢锁，高并发情况下，可能有优先级反转或者饥饿现象

ReentrantLock 默认非公平锁，非公平锁的有点在于吞吐量比公平锁打

Synchronized 等于非公平锁

### 可重入锁

ReenrantLock 可重入锁 == 递归锁， Synchronized 也是可重入锁

同一线程外层函数获得锁之后，内层函数仍然能狗获取该锁的代码

同一个线程在外层方法获取锁的时候，进入内层方法会自动获取锁

线程可以进入它已经拥有锁所同步的代码块

优点：避免死锁

### 自旋锁

尝试获取锁的线程不会阻塞，采用循环的方式尝试获取锁，减少线程上下文切换的消耗，缺点是循环会消耗 CPU

CAS 完成自旋操作 + Unsafe

### 读写锁

独占锁：只能被一个锁占用 ReentrantLock 和 Synchronized

共享锁：锁可以被多个线程持有

ReentrantReadWriteLock 读写锁，读写分离，细粒度控制锁的粒度

多个线程同时读一个资源类没有任何问题，读取共享资源可以同时进行

如果有一个线程去写共享资源，就不应该再有其它线程对资源进行读或者写

读-读共存
独-写不共存
写-写不共存

### CountDownLatch

火箭发射倒计时， 初始化伴随给的一个 count, await() 会一直阻塞，直到 count 减为0

```java
public class CountDownLatchDemo {

    public static void main(String[] args) {

        // 上自习，7个人，6个人都走了之后，班长（主线程）才能关门

        for (int i = 0; i < 7; i++) {
            new Thread(() -> {
                System.out.println(Thread.currentThread().getName() + " 上完自习，关门走人");
            }, String.valueOf(i)).start();
        }

        // 主线程
        System.out.println(Thread.currentThread().getName() + " 班长关门走人");

    }
}
```

如果不是用 CountDownLatch, 班长直接执行完成，其它人都被所在教室里边

![20210901211709](https://cdn.jsdelivr.net/gh/RamboCao/PicGo/images/20210901211709.png)

如果使用 CountDownLatch

public class CountDownLatchDemo {

    public static void main(String[] args) {

        // 上自习，7个人，6个人都走了之后，班长（主线程）才能关门

        CountDownLatch countDownLatch = new CountDownLatch(7);

        for (int i = 0; i < 7; i++) {
            new Thread(() -> {
                System.out.println(Thread.currentThread().getName() + " 上完自习，关门走人");
                countDownLatch.countDown();
            }, String.valueOf(i)).start();
        }
        // 
        try {
            countDownLatch.await();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        // 主线程班长必须等待直到 count 为0才能关门
        System.out.println(Thread.currentThread().getName() + " 班长关门走人");

    }
}

秦灭六国，一统天下，保证灭国的顺序一致

CountDownLatch 让一些线程阻塞到另一些线程完成一系列操作才被唤醒

CountDownLatch 主要有两个方法，当一个或者多个线程调用 await() 方法时，线程会被阻塞，其它线程调用 countDown() 方法会将计数器减1，但是调用该方法的线程不会阻塞
当计数器的值变成0时，await() 阻塞的方法会被唤醒，继续执行

### CyclicBarrier

循环屏障，集齐 7 颗龙珠就能召唤神龙，与 CountDownLatch 相反，做加法

```java
public class CyclicBarrierDemo {

    public static void main(String[] args) {


        CyclicBarrier cyclicBarrier = new CyclicBarrier(7, () -> {
            System.out.println("召唤神龙");
        });

        for (int i = 0; i < 7; i++) {
            final int tempInt = i;
            new Thread(() -> {
                System.out.println(Thread.currentThread().getName() + " 搜集到： " + tempInt + " 龙珠！");
                try {
                    // 集齐才能召唤，否则只能等待
                    cyclicBarrier.await();
                } catch (InterruptedException | BrokenBarrierException e) {
                    e.printStackTrace();
                }
            }, String.valueOf(i)).start();
        }

    }
}
```

### Semaphore

信号灯，多个线程抢多个资源

写成1，退化成 synchroniced

抢车位！，一个用于多个共享资源的互斥使用，另外一个用于并发线程数的控制

Semaphore 可以伸缩，可以进行复用

## 阻塞队列

队列，先到先得

阻塞队列

1. 阻塞队列有没有好的一面

   火锅店和银行的例子

2. 不得不阻塞，如何管理

### 阻塞队列接口结构和实现类

首先是一个队列

* 队列为空，获取元素的动作会被阻塞
* 队列为满，往队列添加元素的动作会被阻塞

蛋糕店！为空会被阻塞
食品柜满了，厨师就不能再做了，被阻塞

试图从空的阻塞队列中获取元素的线程会被祖肃，直到其它线程插入新的元素
同样
试图往已满的队列中添加新元素同样也会被阻塞，直到其它的线程从队列中移除一个或者多个元素，或者完全清空队列使得队列变得空闲然后再新增

阻塞： 线程挂起，一旦条件满足，挂起的线程又被唤醒

为什么需要阻塞线程：无需关心线程什么时候阻塞，什么时候唤醒，BlockingQueue 一手包办，兼顾效率和线程安全，解放程序员，手动挡换成自动挡

BlockingList 是接口，与 List 平级

// 数组结构组成的有界队列，有界，数组的大小有限 
ArrayBlockingQueue;
// 由链表组成的有界（大小默认值为 Interger.MAX_VALUE） 阻塞队列
LinkedBlockingQueue;
// 不存储元素的阻塞队列，单个元素的队列， 有且仅有一个，制作一个，消费一个
SynchronousQueue;
// 链表组成的双向阻塞队列，双向阻塞队列！！！！！！
LinkedBlockingDeque

### BlockingQueue 的核心方法

![20210902213207](https://cdn.jsdelivr.net/gh/RamboCao/PicGo/images/20210902213207.png)

SynchronousQueue: 没有容量，不存储元素的阻塞队列，每一个 put 操作必须等待一个 take 操作，反之亦然

### 生产者消费者

传统版：synchronized, wait(), notify()

进阶版： Lock, await(), signal()

多线程操作的三个步骤：

* 线程     操作    资源类
* 判断     通知    干活
* 避免虚假唤醒

虚假唤醒：
wait() 方法属于 Object 类，与线程没关系

中断和虚假唤醒可能产生，所以 wait() 方法需要放在循环中

wait() 在哪里睡眠就在哪里醒来

线程数增加到4个就会有问题

每次去加减的时候都需要去判断条件是否成立，而不是成立之后进去中断，唤醒之后从原来的地方继续执行

### Synchronized 和 Lock 的区别

synchronized 和 lock 区别，用新的 Lock 有什么好处，距离说明

1. 原始构成
synchronized 是关键字，JVM 层面：通过 monitor 对象完成，wait/notify 方法依赖于 monitor 对象，只有在同步块或者方法中才能调用
    monitorenter/monitorenterexit JVM 指令实现
    monitorenterexit 为什么有两次，第一个是正常退出，第二个有一个异常退出的处理情况，保证能够退出
Lock 具体类：java.util.concurrent.Locks.Lock，api 层面的锁
2. 使用方法
synchronized 无需手动释放，系统自动释放
Lock 需要手动释放，没有主动释放的话，可能会发生死锁
3. 等待是否可以中断
synchronized 不可以中断，除非抛出异常或者常常完成
ReentrantLock 可以中断
    1. 设置超时方法
    2. 调用 intereupt() 方法可以中断
4. 加锁是否公平
synchronized 非公平锁
ReentrantLock 默认非公平锁，也可以是公平锁
5. 锁绑定多个条件 Condition
synchronized 没有
ReentrantLock 分组唤醒，可以精确唤醒，不像 synchronized 随机唤醒