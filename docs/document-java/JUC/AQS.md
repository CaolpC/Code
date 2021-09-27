# AbstractQueuedSynchronizer

## AQS

    Provides a framework for implementing blocking locks and related synchronizers (semaphores, events, etc) that rely on first-in-first-out (FIFO) wait queues. 
    1. 同步框架
    2. 依赖于 FIFO 等待队列
    3. 实现了阻塞锁和相关的同步器

    This class is designed to be a useful basis for most kinds of synchronizers that rely on a single atomic int value to represent state. Subclasses must 
    define the protected methods that change this state, and which define what that state means in terms of this object being acquired or released.
    1. 通过定义 state 值作为锁状态，具体的释放和锁定是通过子类的实现来完成的
    2. 剩余的方法都是队列和队列阻塞，AQS 子类可以定义多个状态，但只能通过 AQS 原子方法去操作 state 值
       用来保证同步 

    This class supports either or both a default exclusive mode and a shared mode.
    默认是独占式执行，同样可以切换到共享模式
    不同的模式使用同一个 FIFO 队列

### Node 队列介绍

***Node***

![20210728153204](https://cdn.jsdelivr.net/gh/RamboCao/PicGo/images/20210728153204.png)

    双向链表队列，prev 和 next 指针进行指向

    大多数的成员变量被 volatile 修饰，即修改后马上对其它线程可见

    操作也需要使用 CAS 或者其它原子方法执行

    有两种不同的模式：独享模式(EXCLUSIVE)和共享模式(SHARED)
    共享模式为 new Node() 可以存放多个线程同时工作

***waitStatus***

    AQS 中状态总共有四种，分别是：
    1. CANCELLED =  1 该线程任务已经被取消, timeout 或者中断，会触发更变此状态并且不会再发生变化
    2. SIGNAL = -1 表示后继结点在等待当前结点唤醒。后继结点入队时，会将前继结点的状态更新为SIGNAL。
    3. CONDITION = -2 表示节点在等待 Condition, 其它线程调用 Condition 的 signal() 方法后， CONDITION 状态的节点从等待队列转移到同步队列中，等待获取同步锁
    4. PROPAGATE = -3 共享模式下，前继结点不仅会唤醒其后继结点，同时也可能会唤醒后继的后继结点。
    5. 0 新节点入队时的默认状态

    在 AQS 中，负值标识的节点标识处于有效等待状态，正值标识节点已经被取消，所以可以采用 >0 或者 <0 的方式来判断节点是否正常 

***VraHandle***

![20210728154740](https://cdn.jsdelivr.net/gh/RamboCao/PicGo/images/20210728154740.png)

    对于使用 volatile 修饰的四个变量：waitStatus、prev、next 和 thread 
    都通过对应的 VarHandle 来进行操作，保证原子性

***ConditionObject***

![20210728155249](https://cdn.jsdelivr.net/gh/RamboCao/PicGo/images/20210728155249.png)

    目的是支持独占模式，Node 的功能性封装，让当前线程独占一个 Object 锁
    进行阻塞、加入队列、唤醒线程等操作
    为什么放在父类而不是子类实现: 抽象 + 模板模式

## AQS 成员函数

***acquire 尝试获取锁***

```java
/**
* Acquires in exclusive mode, ignoring interrupts.  Implemented
* by invoking at least once {@link #tryAcquire},
* returning on success.  Otherwise the thread is queued, possibly
* repeatedly blocking and unblocking, invoking {@link
* #tryAcquire} until success.  This method can be used
* to implement method {@link Lock#lock}.
*
* @param arg the acquire argument.  This value is conveyed to
*        {@link #tryAcquire} but is otherwise uninterpreted and
*        can represent anything you like.
*/
public final void acquire(int arg) {
    // 如果 tryAcquire() 成功就直接短路，否则判断 acquireQueued()
    if (!tryAcquire(arg) &&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}
```

    在独占式模式中使用，至少调用一次 tryAcquire 
    返回成功，失败则将其放入到队列中

    1. tryAcquire()尝试获取锁，一般通过子类实现，如 ReenTrantLock 中的 nonfairTryAcquire() 与 tryAcquire()

```java
/**
* Creates and enqueues node for current thread and given mode.
*
* @param mode Node.EXCLUSIVE for exclusive, Node.SHARED for shared
* @return the new node
*/
private Node addWaiter(Node mode) {
    Node node = new Node(mode);

    for (;;) {
        Node oldTail = tail;
        if (oldTail != null) {
            node.setPrevRelaxed(oldTail);
            if (compareAndSetTail(oldTail, node)) {
                oldTail.next = node;
                return node;
            }
        } else {
            initializeSyncQueue();
        }
    }
}
```

    2. 如果失败则使用 TAIL 的 CAS 操作将该节点插入到队列的队尾

```java
/**
* Acquires in exclusive uninterruptible mode for thread already in
* queue. Used by condition wait methods as well as acquire.
*
* @param node the node
* @param arg the acquire argument
* @return {@code true} if interrupted while waiting
*/
final boolean acquireQueued(final Node node, int arg) {
    boolean interrupted = false;
    try {
        for (;;) {
            final Node p = node.predecessor();
            if (p == head && tryAcquire(arg)) {
                setHead(node);
                p.next = null; // help GC
                return interrupted;
            }
            if (shouldParkAfterFailedAcquire(p, node))
                interrupted |= parkAndCheckInterrupt();
        }
    } catch (Throwable t) {
        cancelAcquire(node);
        if (interrupted)
            selfInterrupt();
        throw t;
    }
}
```

    3. 将节点放入到 acquireQueued() 中执行，自旋尝试获取锁或者阻塞线程(子类实现)
    4. 如果该节点的前置节点是头节点 head, 则只有两个节点，再次尝试获取锁，获取成功，无需中断结束
    5. 否则执行 shouldParkAfterFailedAcquire()，通过前驱节点状态(子类设值)判断是否继续自旋

```java
/**
* Checks and updates status for a node that failed to acquire.
* Returns true if thread should block. This is the main signal
* control in all acquire loops.  Requires that pred == node.prev.
*
* @param pred node's predecessor holding status
* @param node the node
* @return {@code true} if thread should block
*/
private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
    int ws = pred.waitStatus;
    if (ws == Node.SIGNAL)
        /*
            * This node has already set status asking a release
            * to signal it, so it can safely park.
            */
        return true;
    if (ws > 0) {
        /*
            * Predecessor was cancelled. Skip over predecessors and
            * indicate retry.
            */
        do {
            node.prev = pred = pred.prev;
        } while (pred.waitStatus > 0);
        pred.next = node;
    } else {
        /*
            * waitStatus must be 0 or PROPAGATE.  Indicate that we
            * need a signal, but don't park yet.  Caller will need to
            * retry to make sure it cannot acquire before parking.
            */
        pred.compareAndSetWaitStatus(ws, Node.SIGNAL);
    }
    return false;
}
```

    6. 如果已经告诉前驱节点拿完号通知一下自己，就可以安心休息(找到安全点)
    7. 如果前驱节点放弃了，就一直往前找，找到最近一个正常等待的状态，排在他后边，放弃的节点形成一个无引用链，最后给 GC
    8. 如果前驱正常，把前驱的状态设置成 SIGNAL, 告诉他拿完号通知一下自己

***整个流程中，如果前驱不是 SIGNAL 状态就不能安心休息，需要找到一个安全点去休息，同时尝试有没有机会轮到自己拿号***

***线程 waiting 并且检查是否中断过***

```java
/**
* Convenience method to park and then check if interrupted.
*
* @return {@code true} if interrupted
*/
private final boolean parkAndCheckInterrupt() {
    LockSupport.park(this);
    return Thread.interrupted();
}
```

    9. 线程进入自旋并且不断进行检查，直到其运行或者被因为异常被取消

### 总结

    1. acquire() 方法作用是尝试获取锁，如果成功就直接执行子类重写的方法

    2. 如果失败进入到队列队尾，并且开始寻找安全点（自旋寻找前一个状态为 SIGNAL 的线程）

    3. 就调用 park() 进入等待 WAITING 状态，等待 unpark() 或者 interrupt() 唤醒

    4. 有两种方式可以唤醒该线程：1. unpark(), 2. 被 interrupt(), Thread.interrupted() 会清除当前线程的中断标记位

    5. 被唤醒之后，仍然在 for(;;) 循环中，看自己是否有资格拿到号，拿到 head 指向当前节点，并返回从入队到拿到号的整个过程中是否被中断

    6. 自旋过程中产生异常，会将线程状态设置为 CANCEL，并且将该线程从排队中移除

***release 尝试释放锁***

```java
public final boolean release(int arg) {
    if (tryRelease(arg)) {
        Node h = head;
        if (h != null && h.waitStatus != 0)
            unparkSuccessor(h);
        return true;
    }
    return false;
}
```

    独占模式下线程释放共享资源的顶层入口

    释放指定数量的共享资源，如果最后 state=0, 资源被彻底释放，唤醒等待队列中的其它线程获取资源

    对 tryRelease() 方法进行实现时，需要有返回值，release 通过其返回值判断线程是否完成资源释放

    tryRelease() 方法在实现类中实现

#### 自定义的 tryRelease()

```java
protected final boolean tryRelease(int releases) {
    int c = getState() - releases;
    if (Thread.currentThread() != getExclusiveOwnerThread())
        throw new IllegalMonitorStateException();
    boolean free = false;
    if (c == 0) {
        free = true;
        setExclusiveOwnerThread(null);
    }
    setState(c);
    return free;
}
```

    上述方法是 ReentrantLock 中实现的
    
    因为是线程独占模式的，所以说明该线程已经占用该共享资源
    
    所以可以直接进行共享资源释放，直接释放相应量的资源就可以，无需考虑线程安全

    再将占用共享资源目前的线程置为空

#### 唤醒其他线程操作

```java
private void unparkSuccessor(Node node) {
    /*
        * If status is negative (i.e., possibly needing signal) try
        * to clear in anticipation of signalling.  It is OK if this
        * fails or if status is changed by waiting thread.
        */
    int ws = node.waitStatus;
    if (ws < 0)
        node.compareAndSetWaitStatus(ws, 0);

    /*
        * Thread to unpark is held in successor, which is normally
        * just the next node.  But if cancelled or apparently null,
        * traverse backwards from tail to find the actual
        * non-cancelled successor.
        */
    Node s = node.next;
    if (s == null || s.waitStatus > 0) {
        s = null;
        for (Node p = tail; p != node && p != null; p = p.prev)
            if (p.waitStatus <= 0)
                s = p;
    }
    if (s != null)
        LockSupport.unpark(s.thread);
}
```
