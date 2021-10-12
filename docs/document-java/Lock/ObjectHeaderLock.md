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

1. 如果成功，直接替换 Mark Word 中的偏向锁 ID 变成当前线程，并保持偏向锁
2. 如果失败，表示锁有竞争，锁升级成轻量级锁，锁标志位改为 00

### 自旋锁

在线程 B 抢占轻量级锁失败之后，并不会马上进入阻塞状态，而是会进行自旋，如果自旋一定的次数之后，仍然没有获取锁，那么同步锁将会升级成重量锁，锁标志位改为 10。在这个状态下，所有未抢到锁的线程都将进入 Monitor 中并被阻塞在 _WaitSet 队列中。

为什么会进行自旋操作：如果线程 B CAS 抢夺轻量锁失败，直接进入阻塞状态，而线程 A 很快就释放锁，那么线程 B 就会重新申请资源，程序性能下降

从 Jdk1.7 开始，自旋锁默认开启，自旋次数由 JVM 参数决定

```yaml
-XX:-UseSpinning //参数关闭自旋锁优化(默认打开) 
-XX:PreBlockSpin //参数修改默认的自旋次数。JDK1.7后，去掉此参数，由jvm控制
```

### 重量级锁

### JVM 同步指令分析

#### monitorenter

#### monitorexit

#### ACC_SYNCHRONIZED

### 操作系统的管程（Monitor）

#### 管程模型

#### ObjectMonitor

#### 工作原理