# 并发编程

## volatile

并发情况下，一个共享变量（类成员变量、静态变量）如果添加了该关键字

    1. 保证不同线程对于这个变量操作时的可见性，即一个线程修改了这个变量，对于其它线程来说，新值是立即可见的
    2. 禁止指令重排序

### 可见性测试

```java
//线程1
boolean stop = false;
while(!stop){
    doSomething();
}
 
//线程2
stop = true;
```

上述代码中存在两个线程，线程2对 `stop` 进行了重新赋值，但是线程1不一定能够正常执行，很有可能无法停止

 1. 每个线程都有自己的工作内存，线程 1 在执行的时候会拷贝 `stop` 变量到工作内存中

 2. 当线程2进行赋值后，没有将数据写入到主存

 3. 线程1无法获知 `stop` 变量已经修改，所以会一直执行下去

```java
public class VolatileTest {

    // 静态变量的赋值在 clint 中

    public static volatile boolean volatileFlag = true;

    public static volatile boolean flag = true;

    public static void main(String[] args) {

        new Thread(() -> {
            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            flag = false;
            System.out.println("Thread A start ...");
        }, "A").start();

        new Thread(() -> {
            System.out.println("Thread B start ...");
            while (flag) {

            }
            System.out.println("Thread B stop ...");
        }, "B").start();

    }
}
```

***volatile 关键字执行结果***

```output
Thread B start ...
Thread A start ...
Thread B stop ...
```

***没有 volatile 关键字执行结果***

```output
Thread B start ...
Thread A start ...
```

如果 While 循环中加上 `Thread.onSpinWait();` 则可以执行完成 :question:

当使用关键字 `volatile` 关键字之后：

- 当线程B修改了 `flag` 字段之后，会马上写入到主存中

- 同时会使得线程A中的 `flag` 字段缓存置为无效

- 线程A中的 `flag` 变量无效，所以需要取等待主存值更新之后从主存中读取,
  
`Thread.yield()` 当前线程让出 CPU，由执行态转为就绪态，下一次执行时可能抢到 CPU，也可能无法抢到 CPU

### 原子性测试

***volatile 不能保证操作的原子性***

```java
public class VolatileAtomicityTest {

    private int num = 0;

    public /* synchronized */ void increase() {
        num++;
    }

    public static void main(String[] args) {
        volatileAtomicityTest01();
        volatileAtomicityTest02();
        volatileAtomicityTest03();

    }

    private static void volatileAtomicityTest01() {
        final VolatileAtomicityTest volatileAtomicityTest = new VolatileAtomicityTest();
        for (int i = 0; i < 10; i++) {
            new Thread("Thread-" + i) {
                public void run() {
                    for (int i = 0; i < 1000; i++) {
                        volatileAtomicityTest.increase();
                    }
                }
            }.start();
        }

        // Thread.currentThread().getThreadGroup().list();
        while (Thread.activeCount() > 2) {
            // 这个地方有一点问题，必须是 > 2, 否则线程结束执行
            Thread.yield();
        }

        System.out.println(volatileAtomicityTest.num);
    }

    public static void volatileAtomicityTest02() {
        final VolatileAtomicityTest volatileAtomicityTest = new VolatileAtomicityTest();
        Thread threadA = new Thread("Thread-" + 1) {
            public void run() {
                for (int i = 0; i < 1000; i++) {
                    volatileAtomicityTest.increase();
                }
            }
        };

        Thread threadB = new Thread("Thread-" + 2) {
            public void run() {
                for (int i = 0; i < 1000; i++) {
                    volatileAtomicityTest.increase();
                }
            }
        };

        threadA.start();
        threadB.start();

        try {
            threadA.join(); //保证前面的线程都执行完
            threadB.join();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println(volatileAtomicityTest.num);
    }

    private static void volatileAtomicityTest03() {
        final VolatileAtomicityTest volatileAtomicityTest = new VolatileAtomicityTest();
        final int count = 10;
        final CountDownLatch latch = new CountDownLatch(count);
        for (int i = 0; i < count; i++) {
            new Thread(() -> {
                for (int j = 0; j < 1000; j++) {
                    volatileAtomicityTest.increase();
                }
                latch.countDown();
            }, "Thread-" + i).start();
        }
        try {
            latch.await();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println(volatileAtomicityTest.num);
    }

}
```

上述程序运行结果并不是 10 * 1000 = 10000，而是一个小于 10000 的随机数

***在多线程的环境下，可见性只能保证每次读取的都是最新的值，但是并不能保证对变量操作的原子性***

***自增操作并不具备原子性，只有简单的读取、赋值（而且必须是将数字赋值给某个变量，变量之间的相互赋值不是原子操作）才是原子操作。***

`num++` 可以分为两步，第一步是先从线程的工作内存中读取 `num` 的值，然后再进行 `+1` 操作，最后写入到工作内存

上述程序中可能会有以下情况，初始值 num = 10;

1. 线程1读取 num 的值为 10，然后线程1被阻塞
2. 线程2读取 num 的值为 10，此时线程1并没有对num值进行修改，所以线程2的缓存 num 并没有失效，读取值为10，进行自增为11，写入工作内存，保存到主存中
3. 线程1进行自增操作，此时线程1工作内存中 num 的值为 10，进行自增操作为 11，然后写入到工作内存，最后到主存中

这样在进行两个线程处理之后， num 的值只进行了一次自增，所以上述问题得以解释

有三种不同的方式可以解决上述问题

- 使用 synchronized 修饰方法 increase

```java
package com.thread;

public class VolatileAtomicityResolve {

    public volatile int num = 0;

    public synchronized void increase() {
        num++;
    }

    public static void volatileAtomicity01() {
        final VolatileAtomicityResolve resolve = new VolatileAtomicityResolve();
        for (int i = 0; i < 10; i++) {
            new Thread("Thread" + i) {
                public void run() {
                    for (int j = 0; j < 1000; j++) {
                        resolve.increase();
                    }
                }
            }.start();
        }
        // Thread.currentThread().getThreadGroup().list();
        while (Thread.activeCount() > 2) {
            // 这个地方有一点问题，必须是 > 2, 否则线程结束执行
            Thread.yield();
        }
        System.out.println(resolve.num);
    }

    public static void main(String[] args) {
        volatileAtomicity01();
    }
}

```

- 使用锁 ReentrantLock 对方法进行 increase 进行锁定

```java
public class VolatileAtomicityResolve {

    public volatile int num = 0;

    Lock lock = new ReentrantLock();

    public void increase() {
        lock.lock();
        try {
            num++;
        }catch (Exception e){

        }finally {
            lock.unlock();
        }

    }

    public void volatileAtomicity02(){
        for (int i = 0; i < 10; i++) {
            new Thread("Thread" + i){
                public void run(){
                    for (int j = 0; j < 1000; j++) {
                        increase();
                    }
                }
            }.start();
        }
        // Thread.currentThread().getThreadGroup().list();
        while (Thread.activeCount() > 2) {
            // 这个地方有一点问题，必须是 > 2, 否则线程结束执行
            Thread.yield();
        }
        System.out.println(num);
    }

    public static void main(String[] args) {
        VolatileAtomicityResolve resolve = new VolatileAtomicityResolve();
        resolve.volatileAtomicity02();
    }

}

```

- 使用 AtomicInteger 包装类修饰 num

```java
public AtomicInteger numAtomic = new AtomicInteger();

public void increaseAtomic(){
    numAtomic.getAndIncrement();
}
```

AtomicInteger 是一个包装类，利用 CAS(Compare And Swap) 实现原子操作，是处理器提供的 CMPXCHG 指令，该指令是原子性操作

### 顺序性测试

volatile 能禁止指令重排序，一定程度上能保证顺序性

禁止重排序：

1. 程序执行到 volatile 变量时，其前面的操作更改肯定全部完成，且结果对后面操作可见，后边的操作肯定还没有进行

2. 指令优化时，不能将在对volatile变量访问的语句放在其后面执行，也不能把volatile变量后面的语句放到其前面执行。

通过汇编代码可以发现，当加入 volatile 关键字时，会多出一个 lock 前缀指令

lock 前缀指令实际上相当于一个内存屏障，可以提供三个功能

1. 它确保指令重排序时不会把其后面的指令排到内存屏障之前的位置，也不会把前面的指令排到内存屏障的后面；即在执行到内存屏障这句指令时，在它前面的操作已经全部完成；

2. 它会强制将对缓存的修改操作立即写入主存；

3. 如果是写操作，它会导致其他CPU中对应的缓存行无效。

## synchronized
