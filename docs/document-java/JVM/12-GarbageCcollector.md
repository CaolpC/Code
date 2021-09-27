# 垃圾回收器细节

    Java 不同版本的新特性：
    1. 语法层面 ：Lambda 表达式， switch、自动装箱、自动拆箱、emum、泛型
    2. API 层面变化， Stream API, 新的日期时间，Optional、String、集合框架
    3. 底层优化，JVM 优化，GC 的变化，元空间，静态域

## 垃圾回收器的分类

    按照线程数量分类：串行/并行 串行回收默认在客户端 Client 的模式下的 JVM 中

    按照工作模式分：并发式/独占式  并发 = 用户线程 + 垃圾回收线程

    按照碎片处理方式分：压缩式垃圾回收/非压缩式垃圾回收器。指针碰撞+空闲列表

## GC 分类与性能分析

    吞吐量：运行用户代码时间占总运行时间比例（包含垃圾回收时间）

    垃圾搜集开销：吞吐量的补数

    暂停时间：执行垃圾回收时候，程序工作线程被暂停的时间 STW 时间

    搜集频率：垃圾搜集行为发生的频率

    内存占用：Java 堆区所占空间内存大小

    快速：对象从诞生到被回收所经历的时间

***主要抓住两点：吞吐量和暂停时间！！***

    交互式引用更注重低暂停时间

    最大吞吐量吞吐量优先的情况下，降低停顿时间，暂停时间可控！

## 不同垃圾回收器概述

    Serial GC, ParNew 多线程版本
    Paraller GC, CMS
    G1
    Epsilon, ZGC
    Shenadoah GC

    CMS JDK14 中删除

    串行回收器：Serial, SerialOld

    并行回收器：ParNew, Parallel Scavenge, Paraller Old, 

    并发回收器：CMS, G1

***并行指的是垃圾线程的并行！！！！***

## 垃圾收集器的组合关系

![20210805213151](https://cdn.jsdelivr.net/gh/RamboCao/PicGo/images/20210805213151.png)

***红线 jdk8 中的变化，jdk9 中进行了删除，绿是 jdk14 中的弃用，和青色在 jdk14 中进行了 remove***

    Parallel Scavenge 和 CMS 使用的框架不一样，所以二者不能结合使用、

    查看使用什么垃圾回收器：-XX:+PrintCommandLineFlags

## Serial 回收器: 串行回收

    Client 模式下，默认是新生代垃圾收集器

    复制算法、串行回收、STW

    Serial Old 串行回收，STW, 标记压缩算法

    Serial Old 用途：与新生代的 Parallel Scavenge 配合使用

    作为老年代 CMS 垃圾回收器的后备方案

    优势：简单高效，减少切换 CPU 的时间

    -XX:+UseSerialGC 指定年轻代使用串行垃圾回收器, 同时老年代也使用 Serial Old

    没有 UserSerialOldGC 这个参数

## ParNew 回收器: 并行回收

    Serial 多线程版本

    Par => Parllel, New 新生代

    复制算法、STW 机制

    Server 模式下，新生代的默认垃圾收集器

    老年代可以使用 Serial Old GC 或者使用 CMS GC

    Jdk9 中移除 ParNew 与 Serial Old 组合， Jdk14 移除 CMS

    多 CPU ParNew 效果较好，单个 CPU ParNew 收集器不比 Serial 收集器更高效，会有任务切换的额外开销

    -XX:+UserParNewGC

    -XX:ParallelGCThreads 限制线程数量，默认与 CPU 数据相同的线程数

## Parallel 回收器: 吞吐量优先

    Parallel Scavenge 回收新生代，复制算法，并行回收，STW

    与 ParNew 搜集器不同，目标是达到一个可控制的吞吐量，吞吐量优先

    自适应调节策略

    高吞吐量高效利用 CPU, 适合后台运算，不需要太多的交互任务，如批处理，订单处理，工资支付，科学计算等应用程序

    Parallel jdk1.6提供用于执行老年代垃圾搜集的 Parallel Old 收集器，用来替代 Serial Old 收集器

    jdk8 中默认的回收器

    Parallel Old 标记压缩，并行回收，STW

    Jdk9 默认使用 G1

    Parallel 和 Serial Old 组合在 Jkd10 中被移除

    -XX:+UserParallelGC

    -XX:+UseParallelOldGC 两个参数互相激活

    -XXParallelGCThreads 设置年轻代并行搜集器的线程数
        默认情况与 CPU 个数相等
        CPU <= 8, 与 CPU 个数相等
        CPU > 8, 值等于 3 + 5 * CPU / 8
    
    -XX:MaxGCPauseMillis 垃圾收集器最大停顿时间， STW 时间，ms, 调整 Java 堆大小或者其它的参数，整体 STW 时间变长

    -XX:GCTimeRatio 垃圾搜集时间占总时间比例，默认值 99

    -XX:+UseAdaptiveSizePolicy 设置自适应调节策略，默认开启
        自动设置年轻代，E区，S区的比例，晋升老年代对昂年龄等参数，达到各种指标平衡
        调整这个参数并不生效，E区S区的比例不是 8：1：1，如果改变必须指定E区的比例

## CMS 回收器：低延迟

    ConCurrent Mark Sweep

    jdk1.5 中，HotSpot 第一款真正意义上的并发搜集器，第一次实现垃圾线程与用户线程同时工作

    尽可能缩短垃圾搜集时用户线程停顿时间

    使用标记清除算法，也会 STW

    CMS 只能和 ParNewGC Serial Old 使用，因为他们和其它的框架不一样

    Jdk14 中 CMS 被移除

    内存不足 -> 初始标记 -> 并发标记 -> 重新标记 -> 并发清理 -> 重置线程

    初始标记：所有用户线程 STW, 仅仅标记 GC Roots 能直接关联到的对象，速度很快

    并发标记：GC Roots 直接关联对象遍历整个对象图，耗时较长，但是不需要用户线程停顿，并发运行

    重新标记：修正并发标记器件，用户程序继续运行导致产生变动的那一部分对象的标记记录，STW, 时间也较短

    并发清除：清理删除掉标记阶段已经死亡的对象，释放内存空间，并发执行

    并发标记和重新标记涉及到 STW

### CMS 特点与弊端

    尽可能缩短 STW 时间

    并发标记与并发清理都是并行执行的

    用户线程始终没有中断，所以要保证用户线程有足够的内存使用，堆内存使用率道道一定阈值时候，就要开始进行回收

    如果用户线程产生垃圾速度特别快，就出现一次 Concurrent Mode Failure

    临时启动 Serail Old 收集器进行老年来垃圾回收，标记压缩算法

    内存碎片问题，只能选择空闲列表执行内存分配

    为什么不适用标记压缩：用户线程一直都在执行，不能改变用户线程使用的地址，Mark Compact 更适合 STW

    1. 并发操作
    2. 低延迟

    1. 产生内存碎片，提前触发 Full GC
    2. 对 CPU 资源敏感，占用了一部分线程导致引用程序变慢，吞吐量降低
    3. 无法处理浮动垃圾，浮动垃圾：不是修正数据（修正数据是处理怀疑的数据），是并发标记执行过程中产生的垃圾对象，CMS 没办法对这些对象进行标记
        只能下一次执行 GC时释放这些未被回收的内存空间

### CMS 参数设置

    -XX:+UseConcMarkSweepGC 手动指定 CMS 收集器执行内存回收任务，年轻代自动使用 +UseParNewGC

    +XX:CMSlnitiatingOccupanyFraction 设置堆内存使用率的阈值，一旦达到该阈值就进行回收
        jdk5以及以前: 68%
        jdk6以及以后：92%
        
        降低 Full GC 的执行次数，增长快，设置低一点，避免频繁触发垃圾回收器
    
    -XX:+UseCMSCompactAtFullCollection 指定执行完 Full GC 后堆内存空间进行压缩整理，避免内存碎片产生，但是停顿时间会变长

    -XX:CMSFullGCsBeforeCompaction 多少次 Full GC 后堆内存空间进行压缩整理 

    -XX:ParallelCMSThreads 设置 CMS 的线程数量，默认线程数量是 （ParallelGCThreads + 3）/4, ParallelGCThreads 是年轻代并行收集器的线程数
    默认是 CPU 个数，CPU 资源紧张，则应用程序性能就很差

## 小结

    最小胡使用内存何并行开销： Serial GC + Serial Old GC
    最大化应用程序吞吐量： Parallel GC + Parallel Old GC
    最小化 GC 的中断时间：CMS GC + ParNew GC

    JDK9 中 CMS 被废弃
    JDK4 CMS 移除

## G1 回收器：区域化分代式

    为什么发布 G1:
    1. 业务场景越来越复杂
    2. 不多扩大的内存何不断增加的处理器数量
    
    延迟可控的情况下，获得尽可能搞得吞吐量

    G1 并行回收器，分割成多个区域，物理上不连续

    G1 垃圾回收器会判断每一个 Region 的价值，每次根据允许的搜集时间，优先回收价值最大的 Region

    垃圾优先，回收获得的空间大小以及回收所需要的时间

    针对配置多核 CPU 及大容量内存的机器，尽可能的满足 GC 停顿时间，兼顾高吞吐量

    JDK9 之后的默认垃圾回收器

### G1 的特点

#### 并行与并发

    兼具并行与并发，多个 GC 线程同时工作，此时 STW
    与应用程序交替执行

#### 分代收集

    区分年轻代和老年代，但是不要求 Eden 区，年轻代，老年代连续，若干 Region

    兼顾年轻代和老年代

#### 空间整合

    CMS: 标记清除，若干次之后进行碎片整理

    G1：Region 之间是复制算法，整体看上去是标记压缩，可以避免内存碎片，JAVA 堆非常大，G1优势更加明显

#### 可预测的停顿时间模型

    追求低停顿

    让使用者明确指定在一个长度为 M ms 的时间片段内，消耗在垃圾搜集上的时间不超过 N ms

    只回收部分区域

    跟踪各个 Region 的价值，后台维护一个优先列表，优先回收价值最大的 Region, 有限时间内可以尽可能高的搜集效率

    软实时，尽可能在给定的时间内搞定

### 缺点

    为了垃圾收集产生的内存占用还是程序运行时额外的执行负载都比 CMS 高

    小内存时候 CMS 性能好，大约在 6-8G 以内

### G1 的参数设置

    -XX:+UserG1GC 使用 G1 收集器

    -XX:G1HeapRegionSize 设置每个 Region 的大小，1-32MB, 划分出大约 2048 个区域，默认是堆内存的 1/2000

    -XX:MaxGCPauseMillis 设置期望达到的最大 GC 停顿时间指标，尽力实现，不保证，默认是 200ms

    -XX:ParallelGCThread 设置 STW 时 GC 线程数的值，最多设置为 8

    -XX: ConcGCThreads 设置并发标记的线程数，并行垃圾回收线程数的 1/4

    -XX:InitiatingHeadOccupanyPercent 设置触发并发 GC 周期的 Java 堆占用率阈值，超过此值，触发 GC, 默认值是 45

    1. 开启 G1 垃圾搜集器
    2. 设置堆的最大内存
    3. 设置最大停顿时间

![20210810220258](https://cdn.jsdelivr.net/gh/RamboCao/PicGo/images/20210810220258.png)

### Region 介绍

    化整为零

    划分 Java 堆大约为 2048个大小相同的独立 Region 块，2 的 N 次幂

    -XX:G1HeapRegionSize 所有 Region 大小相等，且与 JVM 生命周期相同

    逻辑上并不是连续的区域

    一个 Region 可能属于 Eden、S 区或者 Old 内存区域，但是一个 Region 只能属于一个角色，空白表示未使用的内存空间

    新增了一种内存区域：Humongous 内存区域，存储大对象，如果超过 1.5个 Region 就放到 H

    堆中的大对象，默认直接放到老年代，如果是一个短期存在的大对象，就会对垃圾搜集器造成影响，所以 G1 设置了一个 H 区域

    如果一个 H 区装不下，那么就寻找连续的 H 区，为了寻找 H 区，可能会发生 Full GC

    指针碰撞对 Region 中的对象进行空间分配

    Region 中也可以有 TLAB

### G1 回收器垃圾回收过程

    年轻代 GC YGC
        年轻代的 Eden 区满，开始回收，并行独占是，STW, Eden -> Survivor

    堆内存达到一定值 45%，老年代并发标记
    老年代并发标记过程 + 年轻代 GC

    标记完成后进行回收
    混合回收
        将老年代的存活对象移动导空闲空间，空闲空间变成老年代一部分
        每次回收根据价值（回收时间和回收内存） 一次扫描惠州一部分老年代的 Region
        老年代 Region 和 年轻代一起被回收

    单线程、独占式、高强度的 Full GC 继续存在，提供一种失败保护机制，强力回收

### G1 回收器细节问题

    Remembered Set(记忆集)

    对象可能被跨区使用，不同区域引用
    Region 对象可能被任意 Region 中对象引用
    需要扫描整个 Java 堆
    回收新生代也得扫描老年代
    降低 Minor GC

    解决方法：
    1. 使用 Remember Set 避免全局扫描
    2. 每一个 Region 都对应一个 Remembered Set
    3. 每次 Reference 类型数据进行写操作时，都会产生一个 Write Barrier(写屏障)暂时中断操作
    4. 然后检查将要写入的引用指向的对象是否和该 Reference 类型数据在不同的 Region, 其它收集器：检查老年代对象是否引用了新生代对象
    5. 不同，通过 CardTable 把相关的引用信息记录到引用指向对象所在的 Region 对应的 Remembered Set 中
    6. 垃圾扫描的狮虎，GC 根节点的没觉范围加入 Remembered Set，保证不会全局扫描，也不会遗漏、

### G1 垃圾回收过程

    1. JVM 启动，G1 先准备好 Eden 区，不断创建对象，直到 Eden 区耗尽，进行 YGC
    2. YGC 时，首先 STW, 创建回收集，回收集指的是需要被回收的内存分段的集合，包含 Eden 区和 S 区所有的内存分段
       1. 扫描根，GC Roots, Rset 一定会被使用
       2. 更新一下 Rset, 处理 dirty card queue, 保证 Rset 是一个真实老年代对所在内存分段中对象的引用， Rset 需要线程同步，开销会很大，所以记在队列中
       3. 处理 Rset， 识别被老年代对象指向的 Eden 中的对象，这些被指向的 Eden 对象被认为是存活的
       4. 复制对象，复制算法
       5. 处理引用，Soft, Weak, Phantom, Final, JNI Weak, 最终 Eden 空间的数据为空，GC 停止工作，目标内存中对象连续

    并发标记过程

    1. 初始标记阶段, 标记根节点直接可达的对象，STW, 触发一次 YGC
    2. 根区域扫描：G1 GC 扫描 S 区可以直达老年代的区域对象，并标记被引用的对象 ，这一过程必须在 YGC 之前完成
    3. 并发标记：整个堆进行并发标记，可能被 YGC 中断，并发标记过程中如果发现区域所有对象都是垃圾，那个区域被立即回收，并发标记过程中计算区域的对象活性
    4. 再次标记：程序持续进行，修正上一次的标记结果 STW， 使用 SATB 算法
    5. 独占清理，计算区域存活对象和 GC 回收比例，进行排序，识别可以混合回收的趋于，STW
    6. 并发清理阶段，识别并清理完全空闲的区域 

    混合回收
    1. 越来越多对象晋升到老年代

    Full GC
    1. Evacuation 没有足够的空间存放晋升的对象
    2. 并发处理过程之前空间耗尽
    可能因为堆内存空间太小

    回收阶段考虑和用户程序一起并发执行，因为 G1 只是回收部分 Region

    优化建议：年轻代大小避免显示合适，独占式的，会覆盖暂停时间目标
             暂停时间目标不用太过严苛，90% 的程序时间和 10% 的垃圾回收时间

## 垃圾回收器总结

    ![20210816210035](https://cdn.jsdelivr.net/gh/RamboCao/PicGo/images/20210816210035.png)

    如何选择垃圾回收器：
    1. 优先调整堆大小，自适应
    2. 内存小于 100M, 单CPU，无停顿时间要求，串行
    3. 多 CPU, 高吞吐量，响应时间可以超过 1s, 可以使用并行垃圾回收器
    4. 多 CPU, 追求低停顿时间，快速响应，使用并发收集器

## GC 日志分析

    1. -XX:+PrintGC
    2. -XX:+PrintGCDetails
    3. -XX:+PrintGCTimeStamps JVM 启动的基准时间
    4. -XX:+PrintGCDateStamps 以日期的形式显示，带时区
    5. -XX:+PrintHeapAtGC GC 前后打印堆的信息
    6. -Xloggc: ../logs/gc.log 日志文件输出路径

    JDK8 中大对象直接进入老年代，不进行 YGC

## 垃圾回收器的新发展

    Epsilon 只做内存的分配

    ZGC 可伸缩的垃圾回收器，主打低停顿时间

    Shenandoah GC 主打低延迟， Open JDK 中，Red Hat 推出，主要也是低停顿时间，停顿时间限制在 10ms 内，但是吞吐量明显下降

    ZGC 革命性的
        1. 尽可能队吞吐量影响不大的前提下，实现任意堆内存大小都可以把垃圾搜集的停顿时间限制在 10ms 内的低延迟
        2. 基于 Region 内存布局，暂时不分代，读屏障、染色指针、内存多重映射，可并发的标记-压缩算法