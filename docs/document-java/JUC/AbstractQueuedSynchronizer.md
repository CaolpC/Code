# AbstractQueuedSynchronizer

## 前言

## Java并发工具类的三板斧

## AQS核心实现

### 状态

### 队列

### CAS 操作

## AQS核心属性

## Example: FairSync in ReentrantLock

### acquire

### tryAcquire

### addWaiter

### enq

### addWaiter总结

### acquireQueued

### shouldParkAfterFailedAcquire

### parkAndCheckInterrupt

## 总结

## VarHandle

在 AQS 抽象类中，两个地方使用了 VarHandle, 分别位于 AQS 抽象类和内部类 Node 中

![20210825161215](https://cdn.jsdelivr.net/gh/RamboCao/PicGo/images/20210825161215.png)

VarHandle：变量句柄，主要用来对类变量，字段，数组元素进行绑定，支持在不同访问模型下对这些类型变量的访问

访问模式包括有：简单的 read/write 访问，volatile 类型的 read/write 以及 CAS 操作

与 Unsafe 和 Atomicxxx 比较:

    相较于 Unsafe 来说，VarHandle 更加安全， Unsafe 提供 JVM 内置函数 API， 虽然快，但是会损害安全性和移植性

    A collection of methods for performing low-level, unsafe operations. Although the class and all methods are public, use of this class is limited because only trusted code can obtain instances of it. Note: It is the resposibility of the caller to make sure arguments are checked before methods of this class are called. While some rudimentary checks are performed on the input, the checks are best effort and when performance is an overriding priority, as when methods of this class are optimized by the runtime compiler, some or all checks (if any) may be elided. Hence, the caller must not rely on the checks and corresponding exceptions! 

    用于执行低级别、不安全操作的方法集合。虽然该类和所有方法都是公共的，但此类的使用受到限制，因为只有受信任的代码才能获取该类的实例。
    注意：确保在调用该类的方法之前检查参数是调用方的责任。虽然对输入执行一些基本的检查，
    但检查是尽力的，当性能是压倒一切的优先级时，例如当运行时编译器优化该类的方法时，可能会忽略一些或所有检查(如果有的话)。因此，调用方不能依赖检查和相应的异常！

    另外 Unsafe 中的 CAS 操作是通过无锁的方式进行的操作，所以会有 ABA 的问题。

    相较于 AtomicInteger 来说，减少了开销，安全问题，目前 AtomicInteger 仍然使用的是 Unsafe 的操作来保证原子性， 所以会有 ABA 的问题
    可以使用 AtomicStampedReference/AtomicMarkableReference 通过 Pair 存储元素值和版本号/ boolean 类型标记来解决

VarHandle 出现主要是用来替代 Unsafe 和 Atomic 的部分操作，提供一系列标准的内存屏障操作，更加细粒度的控制内存排序。

### VarHandle 使用

```java
public class MethodHandlesTest {

        private int num;
        private int[] points;

        // 数组元素
        public static final VarHandle POINTS;
        // int 类型变量
        public static final VarHandle NUM;

        static {
            try {
                MethodHandles.Lookup l = MethodHandles.lookup();
                NUM = l.findVarHandle(MethodHandlesTest.class, "num", int.class);
                POINTS = MethodHandles.arrayElementVarHandle(int[].class);
            } catch (ReflectiveOperationException e) {
                throw new ExceptionInInitializerError(e);
            }
        }

        public final boolean compareAndSetState(int expect, int update) {
            return NUM.compareAndSet(this, expect, update);
        }



    public static void main(String[] args) {
        MethodHandlesTest methodHandlesTest = new MethodHandlesTest();
        System.out.println(NUM.compareAndSet(methodHandlesTest, 0, 1));
        System.out.println(methodHandlesTest.num);

    }
}
```

### VarHandle 创建

VarHandle 可以使用 static final 进行修饰， 猜测为了保证 VarHandle 只有一个和对应的变量进行绑定

通过 MethodHandles.Lookup() 方法进行创建， Lookup() 是 MethodHandles 的内部类，用于创建方法和句柄的工厂

与核心反射 API 不同，核心反射 API 在每一次方法调用的时候会进行访问检查，而 Lookup() 创建的方法句柄访问检查是在被创建时进行的

工厂创建的每一个方法句柄等同于方法的字节码行为，即JVM调用方法句柄与执行和方法句柄相关的字节码行为一致

创建句柄可以有两种不同的方式，一种是 MethodHandles, 另外一种就是 Lookup() 工厂方式进行创建。

![20210825170330](https://cdn.jsdelivr.net/gh/RamboCao/PicGo/images/20210825170330.png)

每一个句柄有不同模式(access mode)的访问形式，直接对变量进行访问和操作即可

access mode 将覆盖在变量声明时指定的任何内存排序效果, 例如使用 get() 访问 volatile 修饰的变量，那么 volatile 的内存排序效果将会失效，可以使用 getVolatile() 方法进行访问

### 内存屏障

ValHandle 提供了一组内存屏障方法，为内存排序提供更细粒度的控制，是通过 UNSAFE 实现的

![20210825171043](https://cdn.jsdelivr.net/gh/RamboCao/PicGo/images/20210825171043.png)

```java
@ForceInline
    public static void fullFence() {
        UNSAFE.fullFence();
    }

    /**
     * Ensures that loads before the fence will not be reordered with loads and
     * stores after the fence.
     */
    @ForceInline
    public static void acquireFence() {
        UNSAFE.loadFence();
    }

    /**
     * Ensures that loads and stores before the fence will not be
     * reordered with stores after the fence.
     */
    @ForceInline
    public static void releaseFence() {
        UNSAFE.storeFence();
    }

    /**
     * Ensures that loads before the fence will not be reordered with
     * loads after the fence.
     */
    @ForceInline
    public static void loadLoadFence() {
        UNSAFE.loadLoadFence();
    }

    /**
     * Ensures that stores before the fence will not be reordered with
     * stores after the fence.
     */
    @ForceInline
    public static void storeStoreFence() {
        UNSAFE.storeStoreFence();
    }
```

### compareAndSet 和 weakCompareAndSet