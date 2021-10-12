# Lock

该篇文章主要用来介绍各种锁的一些细节问题

## 对象头结构

synchtoinzed 在 jdk 1.6 之后会存在锁升级过程，根据不同的情况产生不同的锁，偏向锁、轻量锁、重量锁。这些锁只是对象头中的一些信息

下图展示了不同状态下对象头信息

![20210924103553](https://cdn.jsdelivr.net/gh/RamboCao/PicGo/images/20210924103553.png)

![20210924103628](https://cdn.jsdelivr.net/gh/RamboCao/PicGo/images/20210924103628.png)

### 使用代码查看对象头

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

运行代码，查看对象头信息：

***在 Jdk8 中和 Jdk11 中对象头的信息是不一样的***

Jdk8 中

![20210924104815](https://cdn.jsdelivr.net/gh/RamboCao/PicGo/images/20210924104815.png)

Jdk11 中

![20210924104944](https://cdn.jsdelivr.net/gh/RamboCao/PicGo/images/20210924104944.png)

关于不一致的问题放在后边说，先解释这里的二进制代码，需要借助于 JVM 参数。

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

## hashcode 的证明

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

### 小结

1. 无锁状态下的前56为存储的是hashcode
2. 图解 java 对象头的结构

***这个地方 Jdk8 和 Jdk11 还是有点区别***

![20210924133909](https://cdn.jsdelivr.net/gh/RamboCao/PicGo/images/20210924133909.png)

***Jdk8 和 Jdk11 的区别***

这个地方需要最后展示一下区别在哪里

### 为什么分代年龄是最大是15，超过15就要从 from/to 区晋升到老年代

在 java 头中，分代年龄使用四 bit 二进制数字来进行表示，所以当数字位 1111 时，表示年龄达到最大，需要晋升

各种状态下的对象头

![20210924143551](https://cdn.jsdelivr.net/gh/RamboCao/PicGo/images/20210924143551.png)

这个地方需要考虑一个问题？ hashcode 和偏向锁是互斥的，那么为什么 Jdk11 中新建对象的锁是偏向锁？当调用 hashcode() 方法之后转为无锁 + hashcode

| 锁状态   | 锁标识  | 备注
|  ---     |  ---   | ---
| 无锁     | 001    | 使用 baised_lock + lock 来标识无锁和偏向锁
| 偏向锁   | 101    | 使用 baised_lock + lock 来标识无锁和偏向锁
| 轻量锁   | 00     | 只使用 lock 标识
| 重量锁   | 10     | 只使用 lock 标识
| GC 标志  | 11     | 只使用 lock 标识

### 证明对象的无锁状态（只能在 Jdk8 下操作）

***该操作只能在 Jdk8 中操作，Jdk11 中操作结果是偏向锁***

![20210924104815](https://cdn.jsdelivr.net/gh/RamboCao/PicGo/images/20210924104815.png)

由于是小端存储，高位存放高地址，所以 01 存放的是 unused(1 bit) + age(4 bit) + bias_lock(1 bit) + lock(2 bit) = 0 0000 0 01

bias_lock + lock = 0 01 标识对象处于无锁的状态

### 证明对象的偏向锁

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

偏向锁解释：即当一把锁处于可偏向状态时，当有线程持有这把锁后，这把锁将偏向于这个线程。这里提到了可偏向状态，何为可偏向状态呢？可偏向状态是指在 Jvm 开启可偏向功能后，new 出来的一个对象它都是可偏向状态，即它的标识位为101，但是没有具体的偏向某一个线程。

线程停止 4s 之后的结果

![20210924154831](https://cdn.jsdelivr.net/gh/RamboCao/PicGo/images/20210924154831.png)

可以看出，即使在释放锁之后，对象偏向于同样的线程

偏向锁的作用？？

### 调用对象的 hashcode 方法之后，对象无法被标识为偏向锁，而是升级成轻量锁

执行测试类代码：

```java
public class TestObjectHeaderBiasedLock {

    public static void main(String[] args) throws InterruptedException {

        System.out.println();
        // -XX:+PrintFlagsFinal, 寻找参数 BiasedLockingStartupDelay
        System.out.println("线程睡 5s, 因为参数打印出来的 -XX:BiasedLockingStartupDelay=4000 值为 4s, ");
        TimeUnit.SECONDS.sleep(5);

        System.out.println("before lock");
        User user = new User("lisi", 3);
        System.out.println(ClassLayout.parseInstance(user).toPrintable());

        System.out.println(Integer.toHexString(user.hashCode()));

        synchronized (user){
            System.out.println("locking");
            System.out.println(ClassLayout.parseInstance(user).toPrintable());
        }
        // 保证锁一定会释放
         TimeUnit.SECONDS.sleep(4);
        System.out.println("after lock");
        System.out.println(ClassLayout.parseInstance(user).toPrintable());

    }

}
```

![1632636008(1)](https://cdn.jsdelivr.net/gh/RamboCao/PicGo/images/1632636008(1).jpg)

可以看出，当调用对象的 hashcode 方法之后，锁对象由偏向锁升级成轻量锁

### 证明轻量锁

轻量锁：若线程是交替执行的，即上一个线程执行完释放锁后下一个线程再获取锁。若在jvm未开启偏向锁的过程中，对对象进行加锁时，对象直接是轻量锁。

执行测试类代码：

```java
public class TestObjectHeaderLightLock {
    public static void main(String[] args) throws InterruptedException {
        System.out.println();

        System.out.println("before lock");
        User user = new User("lisi", 3);
        System.out.println(ClassLayout.parseInstance(user).toPrintable());
        // 即使执行了 hashcode 方法，程序结果也没有发生变化
        System.out.println(user.hashCode());

        synchronized (user){
            System.out.println("locking");
            System.out.println(ClassLayout.parseInstance(user).toPrintable());
        }
        // 保证锁一定会释放
        TimeUnit.SECONDS.sleep(4);
        System.out.println("after lock");
        System.out.println(ClassLayout.parseInstance(user).toPrintable());

    }
}
```

![20210926141235](https://cdn.jsdelivr.net/gh/RamboCao/PicGo/images/20210926141235.png)

### 证明偏向锁膨胀为轻量锁

主要通过线程切换加锁操作来证明

执行测试类代码

```java
public class BiasedLockToLightLock {

    public static void main(String[] args) throws InterruptedException {
        // 首先睡4.1秒，保证 jvm 开始偏向锁功能
        TimeUnit.SECONDS.sleep(5);
        User user = new User("wangwu", 16);

        System.out.println("main thread before lock");
        System.out.println(ClassLayout.parseInstance(user).toPrintable());

        synchronized (user){
            System.out.println("main locking");
            System.out.println(ClassLayout.parseInstance(user).toPrintable());
        }
        System.out.println("after lock");
        System.out.println(ClassLayout.parseInstance(user).toPrintable());

        // 新创建一个线程用来对 user 对象进行操作
        Thread t1 = new Thread(() -> {
            synchronized (user) {
                System.out.println("t1 locking");
                System.out.println(ClassLayout.parseInstance(user).toPrintable());
            }
        }, "t1");
        t1.start();
        // 等待 t1 执行完成再打印一次信息
        t1.join();

        System.out.println(ClassLayout.parseInstance(user).toPrintable());
    }

}
```

![20210926143314](https://cdn.jsdelivr.net/gh/RamboCao/PicGo/images/20210926143314.png)
![20210926143728](https://cdn.jsdelivr.net/gh/RamboCao/PicGo/images/20210926143728.png)

线程之间的切换(其实是线程之间锁抢占)会导致对象的锁由偏向锁变成轻量锁
