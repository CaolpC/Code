# Object Header And synchronized

本篇文章主要介绍对象头，以及对象头的锁

对象头的锁有四种，无锁、偏向锁、轻量锁、重量锁

锁在特定的情况下会升级，同样也会降级，降级的条件比较苛刻

本文的测试环境均在 Jdk1.8 下进行，至于 Jdk11 会在以后进行对比

## Object Header

首先介绍对象头结构 Object Header， Java 对象 Object 头结构图如下所示：

![20210924143551](https://cdn.jsdelivr.net/gh/RamboCao/PicGo/images/20210924143551.png)

在无锁状态下，前56位存储的是对象的 hashcode 值

age 位数是4，用来进行 Jvm 垃圾回收。新生代晋升老年代计数器，最大值为15

biased_lock（1位） 和 lock（2位）二者用来表示无锁状态或者偏向锁状态，对应关系如下所示：

![ObjectSynchronized](https://cdn.jsdelivr.net/gh/RamboCao/PicGo/images/ObjectSynchronized.svg)

## 使用代码查看对象头

pom.xml 文件中引入 jar 包

```xml
<dependency>
    <groupId>org.openjdk.jol</groupId>
    <artifactId>jol-core</artifactId>
    <version>0.9</version>
</dependency>
```

新建 User 类和测试类 TestObjectHeader 进行展示

```java
public class User {

    private String name;

    private Integer age;

    public User(String name, Integer age) {
        this.name = name;
        this.age = age;
    }
}
```

```java
public class TestObjectHeader {
    public static void main(String[] args) {
        User user = new User("baidu", 3);
        System.out.println(ClassLayout.parseInstance(user).toPrintable());
    }
}
```

运行代码，查看对象头信息（Jdk8 中）

![20210924104815](https://cdn.jsdelivr.net/gh/RamboCao/PicGo/images/20210924104815.png)

## CPU 的大端小端模式

不同体系结构的 CPU 数据在内存中存放的排列顺序是不一样的

数据存储以字节（Byte）为单位，所以字(Word) 和半字（Half-Word）在存储器中有两种次序，分别为大端模式（Big Endian）和小端模式（Little Endian）

大端模式：字/半字的最高字节放在内存的最低字节地址上，字数据的低字节放在高地址中

例如：

假设存在某个字 0x12345678, 这个字由四个字节构成，从高位到低位的次序为 0x12、0x34、0x56、0x78, 将这个地址放在以 0x00000000 起始的内存地址中，则实际存放情况为

        |   内存地址   | 数据 |  
        |     ----    | ---- |
        |  0x00000000 | 0x12 |
        |  0x00000001 | 0x34 |
        |  0x00000002 | 0x56 |
        |  0x00000003 | 0x78 |
        |  0x00000004 | .... |

上述大端模式广泛应用于 TCP/IP 协议中，又称为网络字节次序

反之小端模式将字/半字存放在内存的最低字节地址上，字数据的高字节存放在高地址中，上述字的小端存储方式为

        |   内存地址   | 数据 |  
        |     ----    | ---- |
        |  0x00000000 | 0x78 |
        |  0x00000001 | 0x56 |
        |  0x00000002 | 0x34 |
        |  0x00000003 | 0x12 |
        |  0x00000004 | .... |

注意：
  数据在寄存器中都是以大端模式次序存放的。
  对于内存中以小端模式存放的数据。CPU存取数时，小端和大端之间的转换是通过硬件实现的，没有数据加载/存储的开销。

java 中查看 CPU 是大端模式还是小端模式

```java
//BIG_ENDIAN 大端模式
//LITTLE_ENDIAN 小端模式
System.out.println(ByteOrder.nativeOrder().toString());
```

## 对象头不同锁的介绍

### 无锁状态

无锁状态下，对象头中的前56位存储的是 hashcode

在 Jdk8 中演示 hashcode 的存在， 测试类中执行如下代码

```java
public class TestObjectHeader {
    public static void main(String[] args) {
        User user = new User("zhangsan", 3);
        // 查看java字节存储是大端模式还是小端模式：LITTLE_ENDIAN, 本机是小端模式
        // 大端模式: 高位存在内存低位
        // 小端模式：低位存在内存低位
        System.out.println(ByteOrder.nativeOrder().toString());
        System.out.println("before hashCode");
        System.out.println(ClassLayout.parseInstance(user).toPrintable());
        // 这里对 user 对象进行 hashcode 操作，对象头存储了 hashcode
        System.out.println(Integer.toHexString(user.hashCode()));
        System.out.println("after hashCode");
        System.out.println(ClassLayout.parseInstance(user).toPrintable());
    }

}
```

![20210924132828](https://cdn.jsdelivr.net/gh/RamboCao/PicGo/images/20210924132828.png)

总结：无锁状态下，当调用对象的 hashcode 方法后，对象头前56位变成 hashcode 值

### 偏向锁

***偏向锁主要用来优化同一个线程多次申请同一个锁的竞争，能够减少无竞争锁定时的开销***

偏向锁出现的原因：在程序执行过程中，大部分情况下对象总是被一个线程持有和竞争，因此在偏向锁中假定 monitor 总是由某个特定的线程持有，直到另外一个线程尝试进行获取，这样就可以避免竞争 monitor 时执行 CAS 原子操作，减少用户态和内核态切换次数

作用：当线程再次访问临界区时，首先判断对象的 MarkWord 中的偏向锁线程 ID 是否指向它本身，如果是，无需进入 Monitor 去竞争对象

当线程抢到锁之后，该同步对象的 Mark Word 中 biased_lock 设置为 1，lock 标志设置为 01，并且记录偏向锁的线程 ID，标识进入偏向锁状态

### 证明线程进入偏向锁状态

首先给 Jvm 添加参数 -XX:+PrintFlagsFinal

执行测试类代码

```java
public class TestObjectHeaderBiasedLock {

    public static void main(String[] args) throws InterruptedException {

        System.out.println();
        // -XX:+PrintFlagsFinal, 寻找参数 BiasedLockingStartupDelay
        System.out.println("线程睡 4s, 因为参数打印出来的 -XX:BiasedLockingStartupDelay=4000 值为 4s, ");
        TimeUnit.SECONDS.sleep(4);

        System.out.println("before lock");
        User user = new User("lisi", 3);
        System.out.println(ClassLayout.parseInstance(user).toPrintable());

        synchronized (user){
            System.out.println("locking");
            System.out.println(ClassLayout.parseInstance(user).toPrintable());
        }
        // 停几秒，保证锁已经被释放
        TimeUnit.SECONDS.sleep(4);
        System.out.println("after lock");
        System.out.println(ClassLayout.parseInstance(user).toPrintable());

    }

}
```

![20210924150432](https://cdn.jsdelivr.net/gh/RamboCao/PicGo/images/20210924150432.png)

图中展示的是偏向锁的延迟启动时间

从打印出来的参数可以看出，在 JVM 启动 4s 之后，才开启偏向锁， 所以我们的主线程需要停下来，至少等待 4s 的时间，才能看到效果，否则线程还是无锁的状态，即使加锁，最终的结果也是无锁

![20210924153343](https://cdn.jsdelivr.net/gh/RamboCao/PicGo/images/20210924153343.png)

线程停止 4s 之后的结果

![20210924154831](https://cdn.jsdelivr.net/gh/RamboCao/PicGo/images/20210924154831.png)

可以看出，即使在释放锁之后，对象偏向于同样的线程

一旦出现其它线程竞争锁资源，偏向锁就会被撤销。撤销时机是在全局安全点，暂停持有该锁的线程，同时检查该线程是否还在执行该方法。是则升级锁；不是则被其它线程抢占。

### 轻量级锁

如果程序中只有一个线程 A 操作同步对象，那么上述偏向锁效率会很高，当程序中存在另外一个线程 B 竞争时，线程 B 判断同步对象中的 Mark Word 中的偏向锁 ID 不是自己的，则进行 CAS 尝试将同步对象 Mark Word 中的偏向锁线程 ID 替换成自己的，

1. 如果成功，直接替换 Mark Word 中的偏向锁 ID 变成当前线程，并保持偏向锁，执行同步代码
2. 如果失败，表示锁有竞争，当到达全局安全点（safepoint） 时获得偏向锁的线程 A 被挂起，偏向锁升级成轻量级锁，锁标志位改为 00，被阻塞在安全点的线程继续往下执行同步代码

### 自旋锁

在线程 B 抢占轻量级锁失败之后，并不会马上进入阻塞状态，而是会进行自旋，如果自旋一定的次数之后，仍然没有获取锁，那么同步锁将会升级成重量锁，锁标志位改为 10。在这个状态下，所有未抢到锁的线程都将进入 Monitor 中并被阻塞在 _WaitSet 队列中。

为什么会进行自旋操作：如果线程 B CAS 抢夺轻量锁失败，直接进入阻塞状态，而线程 A 很快就释放锁，那么线程 B 就会重新申请资源，程序性能下降

从 Jdk1.7 开始，自旋锁默认开启，自旋次数由 JVM 参数决定

```yaml
-XX:-UseSpinning //参数关闭自旋锁优化(默认打开) 
-XX:PreBlockSpin //参数修改默认的自旋次数。JDK1.7后，去掉此参数，由jvm控制
```

### 重量级锁

### 批量重偏向

设置 JVM 参数 -XX:+PringFlagsFinal，在项目启动时，可以看到 JVM 的默认参数值

![20211013134530](https://cdn.jsdelivr.net/gh/RamboCao/PicGo/images/20211013134530.png)

```yaml
intx BiasedLockingBulkRebiasThreshold = 20   默认偏向锁批量重偏向阈值
intx BiasedLockingBulkRevokeThreshold = 40   默认偏向锁批量撤销阈值
```

为什么会有批量重偏向和批量撤销:question:

当一个线程创建了大量的对象，并执行了初始的同步操作后，另外一个线程也将这些对象作为锁对象进行操作，会导致偏向锁重偏向的操作，这里的重偏向指的时 偏向锁(偏向 A) -> 轻量锁 -> 偏向锁(B)

***重偏向是对类的优化***

1. 如果有很多对象，这些对象都是由同一个类（假设为 A）实例化，被线程 t1 访问，对象头 Mark Word 中偏向线程记录的都是线程 t1
2. 之后线程 t2 来访问这些对象，发现对象头 Mark Word 中的偏向线程 id 都不是自己，所以此时锁会升级成轻量锁，锁升级过程不可逆

上述锁升级的操作是一个很耗时的操作，因此 JVM 对此进行了优化：当对象数量超过某个阈值时，Java 会对阈值之外的对象做批量重偏向线程 t2，所以前20个对象都是轻量锁，后面的对象都是偏向锁，偏向线程 t2

代码展示：

```java
public class ReBiasedLock {
    public static void main(String[] args) throws InterruptedException {

        // 延时 5s，开启偏向锁功能
        TimeUnit.SECONDS.sleep(5);
        // 创建 100 个偏向线程 t1 的偏向锁
        List<User> locks = new ArrayList<>();

        Thread t1 = new Thread(() ->{
            for (int i = 0; i < 100; i++) {
                User user = new User(String.valueOf(i),i);
                synchronized (user){
                    locks.add(user);
                }
            }
            //为了防止JVM线程复用，在创建完对象后，保持线程t1状态为存活
            try {
                Thread.sleep(1000000000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        });

        t1.start();

        Thread.sleep(3000);
        System.out.println("打印t1线程，list中第20个对象的对象头：");
        System.out.println(ClassLayout.parseInstance(locks.get(19)).toPrintable());

        // 创建线程 t2 竞争线程 t1 已经退出同步块的锁
        Thread t2 = new Thread(() -> {
            // 这里边值循环了30次
            for (int i = 0; i < 30; i++) {
                User user = locks.get(i);
                synchronized (user){
                    if (i == 18 || i == 19){
                        System.out.println(Thread.currentThread().getName() + " : " + i + " 锁");
                        System.out.println(ClassLayout.parseInstance(locks.get(i)).toPrintable());
                    }
                }
            }

            try {
                Thread.sleep(10000000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }

        });

        t2.start();

        Thread.sleep(3000);
        System.out.println(("打印 locks 中第11个对象的对象头："));
        System.out.println((ClassLayout.parseInstance(locks.get(10)).toPrintable()));
        System.out.println("打印 locks 中第26个对象的对象头：");
        System.out.println((ClassLayout.parseInstance(locks.get(25)).toPrintable()));
        System.out.println("打印 locks 中第41个对象的对象头：");
        System.out.println((ClassLayout.parseInstance(locks.get(40)).toPrintable()));
    }
}
```

执行结果分析

首先创建了 100 个偏向线程 t1 的偏向锁，锁标志 101，偏向线程 t1：-1604823035

```result
打印 t1 线程，locks 中第20个对象的对象头：
com.object.header.User object internals:
 OFFSET  SIZE                TYPE DESCRIPTION                               VALUE
      0     4                     (object header)                           05 58 58 a0 (00000101 01011000 01011000 10100000) (-1604823035)
      4     4                     (object header)                           19 02 00 00 (00011001 00000010 00000000 00000000) (537)
      8     4                     (object header)                           43 c1 00 f8 (01000011 11000001 00000000 11111000) (-134168253)
     12     4    java.lang.String User.name                                 (object)
     16     4   java.lang.Integer User.age                                  19
     20     4                     (loss due to the next object alignment)
Instance size: 24 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total

```

线程 t2，前 19 次偏向都产生了轻量锁，锁标识 00

```result
t2 : 18 锁
com.object.header.User object internals:
 OFFSET  SIZE                TYPE DESCRIPTION                               VALUE
      0     4                     (object header)                           20 f2 2f e8 (00100000 11110010 00101111 11101000) (-399511008)
      4     4                     (object header)                           59 00 00 00 (01011001 00000000 00000000 00000000) (89)
      8     4                     (object header)                           43 c1 00 f8 (01000011 11000001 00000000 11111000) (-134168253)
     12     4    java.lang.String User.name                                 (object)
     16     4   java.lang.Integer User.age                                  18
     20     4                     (loss due to the next object alignment)
Instance size: 24 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total
```

第 20 次的时候，达到了批量重偏向的阈值 20，此时锁并不是轻量锁，而是变成了偏向锁，锁标识 101，偏向线程 t2: -1603421947

```result
t2 : 19 锁
com.object.header.User object internals:
 OFFSET  SIZE                TYPE DESCRIPTION                               VALUE
      0     4                     (object header)                           05 b9 6d a0 (00000101 10111001 01101101 10100000) (-1603421947)
      4     4                     (object header)                           19 02 00 00 (00011001 00000010 00000000 00000000) (537)
      8     4                     (object header)                           43 c1 00 f8 (01000011 11000001 00000000 11111000) (-134168253)
     12     4    java.lang.String User.name                                 (object)
     16     4   java.lang.Integer User.age                                  19
     20     4                     (loss due to the next object alignment)
Instance size: 24 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total
```

前 20 个对象没有触发批量重偏向机制，线程 t2 释放同步锁之后，转变成无锁状态

第 20~30 个对象，出发了批量重偏向机制，对象为偏向锁状态，偏向线程 t2，t2 线程 id 为：-1603421947

第 31 个对象之后，也没有触发批量重偏向机制，对象仍然偏向于线程 t1，t1线程 id 为：-1604823035

```result
打印 locks 中第11个对象的对象头：
com.object.header.User object internals:
 OFFSET  SIZE                TYPE DESCRIPTION                               VALUE
      0     4                     (object header)                           01 00 00 00 (00000001 00000000 00000000 00000000) (1)
      4     4                     (object header)                           00 00 00 00 (00000000 00000000 00000000 00000000) (0)
      8     4                     (object header)                           43 c1 00 f8 (01000011 11000001 00000000 11111000) (-134168253)
     12     4    java.lang.String User.name                                 (object)
     16     4   java.lang.Integer User.age                                  10
     20     4                     (loss due to the next object alignment)
Instance size: 24 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total

打印 locks 中第26个对象的对象头：
com.object.header.User object internals:
 OFFSET  SIZE                TYPE DESCRIPTION                               VALUE
      0     4                     (object header)                           05 b9 6d a0 (00000101 10111001 01101101 10100000) (-1603421947)
      4     4                     (object header)                           19 02 00 00 (00011001 00000010 00000000 00000000) (537)
      8     4                     (object header)                           43 c1 00 f8 (01000011 11000001 00000000 11111000) (-134168253)
     12     4    java.lang.String User.name                                 (object)
     16     4   java.lang.Integer User.age                                  25
     20     4                     (loss due to the next object alignment)
Instance size: 24 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total

打印 locks 中第41个对象的对象头：
com.object.header.User object internals:
 OFFSET  SIZE                TYPE DESCRIPTION                               VALUE
      0     4                     (object header)                           05 58 58 a0 (00000101 01011000 01011000 10100000) (-1604823035)
      4     4                     (object header)                           19 02 00 00 (00011001 00000010 00000000 00000000) (537)
      8     4                     (object header)                           43 c1 00 f8 (01000011 11000001 00000000 11111000) (-134168253)
     12     4    java.lang.String User.name                                 (object)
     16     4   java.lang.Integer User.age                                  40
     20     4                     (loss due to the next object alignment)
Instance size: 24 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total

```

### 批量撤销

多线程竞争激烈的情况下，使用偏向锁会降低效率，于是乎产生了批量撤销机制

代码展示

```java
public class BiasedLockRevoke {

    public static void main(String[] args) throws InterruptedException {
        Thread.sleep(5000);

        List<User> locks = new ArrayList<>();

        Thread t1 = new Thread(() ->{
            for (int i = 0; i < 100; i++) {
                User user = new User(String.valueOf(i), i);
                synchronized (user){
                    locks.add(user);
                }
            }

            try {
                Thread.sleep(100000000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }, "t1");

        t1.start();

        Thread.sleep(3000);

        Thread t2 = new Thread(() ->{
            // 40 次，达到了批量撤销的阈值
            for (int i = 0; i < 40; i++) {
                User user = locks.get(i);
                synchronized (user){

                }
            }

            try {
                Thread.sleep(100000000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }, "t2");

        t2.start();

        Thread.sleep(3000);
        System.out.println("打印 locks 中第11个对象的对象头：");
        System.out.println((ClassLayout.parseInstance(locks.get(10)).toPrintable()));
        System.out.println("打印 locks 中第26个对象的对象头：");
        System.out.println((ClassLayout.parseInstance(locks.get(25)).toPrintable()));
        System.out.println("打印 locks 中第90个对象的对象头：");
        System.out.println((ClassLayout.parseInstance(locks.get(89)).toPrintable()));

        Thread t3 = new Thread(() -> {
            for (int i = 20; i < 40; i++) {
                User user = locks.get(i);
                synchronized (user){
                    if (i == 20 || i == 22){
                        System.out.println("thread 3 第 " + i + " 次");
                        System.out.println(ClassLayout.parseInstance(user).toPrintable());
                    }
                }
            }
        }, "t3");

        t3.start();

        Thread.sleep(10000);
        System.out.println("重新输出实例 User");
        System.out.println(ClassLayout.parseInstance(new User("new", 101)).toPrintable());
    }
}
```

执行结果分析：

前 20 个对象，没有触发批量重偏向机制，线程 t2 执行释放同步锁之后，转变为无锁状态

```result
打印 locks 中第11个对象的对象头：
com.object.header.User object internals:
 OFFSET  SIZE                TYPE DESCRIPTION                               VALUE
      0     4                     (object header)                           01 00 00 00 (00000001 00000000 00000000 00000000) (1)
      4     4                     (object header)                           00 00 00 00 (00000000 00000000 00000000 00000000) (0)
      8     4                     (object header)                           43 c1 00 f8 (01000011 11000001 00000000 11111000) (-134168253)
     12     4    java.lang.String User.name                                 (object)
     16     4   java.lang.Integer User.age                                  10
     20     4                     (loss due to the next object alignment)
Instance size: 24 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total
```

第 20~40 个对象，触发了批量重偏向机制，对象为偏向锁状态，偏向线程 t2，线程 t2 id 为 1741091077

```result
打印 locks 中第26个对象的对象头：
com.object.header.User object internals:
 OFFSET  SIZE                TYPE DESCRIPTION                               VALUE
      0     4                     (object header)                           05 f1 c6 67 (00000101 11110001 11000110 01100111) (1741091077)
      4     4                     (object header)                           cb 01 00 00 (11001011 00000001 00000000 00000000) (459)
      8     4                     (object header)                           43 c1 00 f8 (01000011 11000001 00000000 11111000) (-134168253)
     12     4    java.lang.String User.name                                 (object)
     16     4   java.lang.Integer User.age                                  25
     20     4                     (loss due to the next object alignment)
Instance size: 24 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total
```

第 41 个对象之后，也没有触发批量重偏向机制，对象仍然偏向于线程 t1，线程 t1 id 为 1741084677

```result
打印 locks 中第90个对象的对象头：
com.object.header.User object internals:
 OFFSET  SIZE                TYPE DESCRIPTION                               VALUE
      0     4                     (object header)                           05 d8 c6 67 (00000101 11011000 11000110 01100111) (1741084677)
      4     4                     (object header)                           cb 01 00 00 (11001011 00000001 00000000 00000000) (459)
      8     4                     (object header)                           43 c1 00 f8 (01000011 11000001 00000000 11111000) (-134168253)
     12     4    java.lang.String User.name                                 (object)
     16     4   java.lang.Integer User.age                                  89
     20     4                     (loss due to the next object alignment)
Instance size: 24 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total
```

线程 t3 开始竞争锁，此时已经达到批量偏向锁撤销的阈值 40， 对象 locks.get(20) 和 locks.get(22) 已经进行过偏向锁的重偏向，所以并不会再次重偏向 t3

***此时触发批量撤销，对象锁膨胀变为轻量级锁，这里是偏向撤销的位置***

```result
thread 3 第 20 次
com.object.header.User object internals:
 OFFSET  SIZE                TYPE DESCRIPTION                               VALUE
      0     4                     (object header)                           30 f4 2f 57 (00110000 11110100 00101111 01010111) (1462760496)
      4     4                     (object header)                           c1 00 00 00 (11000001 00000000 00000000 00000000) (193)
      8     4                     (object header)                           43 c1 00 f8 (01000011 11000001 00000000 11111000) (-134168253)
     12     4    java.lang.String User.name                                 (object)
     16     4   java.lang.Integer User.age                                  20
     20     4                     (loss due to the next object alignment)
Instance size: 24 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total

thread 3 第 22 次
com.object.header.User object internals:
 OFFSET  SIZE                TYPE DESCRIPTION                               VALUE
      0     4                     (object header)                           30 f4 2f 57 (00110000 11110100 00101111 01010111) (1462760496)
      4     4                     (object header)                           c1 00 00 00 (11000001 00000000 00000000 00000000) (193)
      8     4                     (object header)                           43 c1 00 f8 (01000011 11000001 00000000 11111000) (-134168253)
     12     4    java.lang.String User.name                                 (object)
     16     4   java.lang.Integer User.age                                  22
     20     4                     (loss due to the next object alignment)
Instance size: 24 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total
```

最后生成的新对象 User

***本应该为可偏向状态偏向锁的新对象，经历了批量重偏向和批量撤销之后实例直接转换成无锁***

```result
重新输出实例 User
com.object.header.User object internals:
 OFFSET  SIZE                TYPE DESCRIPTION                               VALUE
      0     4                     (object header)                           01 00 00 00 (00000001 00000000 00000000 00000000) (1)
      4     4                     (object header)                           00 00 00 00 (00000000 00000000 00000000 00000000) (0)
      8     4                     (object header)                           43 c1 00 f8 (01000011 11000001 00000000 11111000) (-134168253)
     12     4    java.lang.String User.name                                 (object)
     16     4   java.lang.Integer User.age                                  101
     20     4                     (loss due to the next object alignment)
Instance size: 24 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total

```

### 批量撤销和批量重偏向总结

二者都是针对类的优化，和对象无关

偏向锁重偏向一次之后不可再次重偏向

当某个类触发批量撤销机制之后，JVM 默认当前类产生了严重的问题，剥夺了该类的新实例对象使用偏向锁的权利，这里严重的问题

### JVM 同步指令分析

#### monitorenter

#### monitorexit

#### ACC_SYNCHRONIZED

### 操作系统的管程（Monitor）

#### 管程模型

#### ObjectMonitor

#### 工作原理