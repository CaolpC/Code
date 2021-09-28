# AOP

Spring4 -> Spring5 AOP 的全部执行顺序，有哪些坑，你遇到过

4 -> 5 对于 AOP 执行顺序时不同的

boot1 -> boot2 == spring4 -> spring5

测试类上边注解

```java
@SpringBootTest
RunWith(SpringRunner.class) //Spring5 测试可以不用
```

Spring4 正常情况:

环绕通知之前 AAA
前置通知
方法输出
环绕通知BBB
After 后置通知
AfterReturning 返回后通知

---

Spring4 异常情况：

环绕通知之前 AAA

前置通知
After 后置通知
AfterThrowing 异常通知

异常顺序没有环绕通知BBB

---

Spring5 正常情况:

环绕通知之前 AAA
前置通知
方法输出
AfterReturning 返回后通知
After 后置通知
环绕通知之后BBB

---

Spring5 异常情况：

环绕通知之前 AAA

前置通知
After 后置通知
AfterThrowing 异常通知

无论是 Spring4 还是 5

异常顺序没有环绕通知BBB

![20210923213405](https://cdn.jsdelivr.net/gh/RamboCao/PicGo/images/20210923213405.png)

## Spring 循环依赖

什么是循环依赖？

多个 Bean 之间相互依赖，形成了一个闭环，比如 A 依赖 B, B 依赖 C, C 依赖 A

一般是在 Spring 的容器中如何解决循环依赖的问题，一定是只默认的单例 Bean 中，属性互相引用的场景

两种注入方式对循环依赖的影响：

1. 构造注入

    构造方法注入可能会引入一个无法解决的循环依赖的问题， Spring 容器发现这种情况会抛出：BeanCurrentlyInCreationException

2. Set 方式注入

    如果使用 Set 方法，可以完成循环依赖的问题

3. 注解注入

AB 循环依赖问题只要 A 的注入方式是 setter 并且 A 是单例，就不会有循环依赖的问题

构造注入可能会产生循环依赖问题，套娃问题。

但是 setter 方法注入没有问题

Spring 默认的 singleton 的场景是支持循环依赖的，不会报错

原型（Prototype）的场景是不支持循环依赖的，会报错

***Spring 内部是通过三级缓存解决循环依赖 -> DefaultSingletonBeanRegistry***

Spring 内部用来解决循环依赖的三个 Map

1. 实例化/初始化：内存中申请一块空间，但是没有进行赋值，初始化开始进行属性填充，完成属性的赋值

2. 3个 Map 和四大方法，总体方法说明

    singletonObjects 一级缓存, 存放已经初始化好的 Bean

    earlySingletonObjects 二级缓存，存放实例化的，但是未初始化的 Bean

    singletonFactories 三级缓存, 存放 FactoryBean, 假设 A 类实现了 FactoryBean, 那么依赖注入的时候不是 A 类，而是 A 类产生的 Bean

    getSingleton()

    doCreateBean() // 创建 Bean

    populateBean() // 属性填充

    addSingleton() // 添加回去

```java
/**
* 单例池：经历了完整生命周期的 Bean 对象
**/
/** Cache of singleton objects: bean name to bean instance. */
private final Map<String, Object> singletonObjects = new ConcurrentHashMap<>(256);

/**
* 已经实例化但是还没初始化的 Bean
**/
/** Cache of singleton factories: bean name to ObjectFactory. */
private final Map<String, ObjectFactory<?>> singletonFactories = new HashMap<>(16);

/**
* 单例工厂的告诉缓存，存放生成 Bean 的工厂
**/
/** Cache of early singleton objects: bean name to bean instance. */
private final Map<String, Object> earlySingletonObjects = new HashMap<>(16);
```

循环依赖文字版描述

1. A 创建过程中需要 B, 于是 A 将自己放到三级缓存里边取，去实例化 B
2. B 实例化的过程中发现需要 A, 于是 B 先查一级缓存，没有，再查二级缓存，还是没有，再查三级缓存，找到 A, 然后把三级缓存中的 A 放入到二级缓存中，并且删除三级缓存中的A
3. B 顺利初始化完毕，将自己放到一级缓存里边，此时 B 中的 A 仍然是创建中状态，然后回来接着创建 A，此时 B 已经创建结束，直接从一级缓存里边拿到 B, 然后完成创建，并将 A 自己放到一级缓存里面

