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

Spring 循环依赖小结

注意 @Asyn 注解

注意 AOP 代理

### 方法的执行

1. 调用 doGetBean() 方法，想要获取 beanA, 调用 getSingleton() 方法从缓存中查找 beanA() 这里是一级缓存还是二级缓存?
2. 在 getSingleton() 方法中，从一级缓存中查找，没有, 返回 null, 并且开始创建 beanA
3. doGetBean() 方法中获取到的 beanA 为 null, 于是走对应的处理逻辑，调用 getSingleton() 的重载方法(参数为 ObjectFactory)
4. 在 getSingle() 方法中，先将 beanA_name 添加到一个集合中，用于标记该 bean 正在创建中，这个地方需要再看一下。然后回调匿名内部类的 createBean() 方法, lambda 表达式
5. 进入 AbstractAutowireCapableBeanFactory#doCreateBean, 先反射调用构造器创建 beanA 的实例(注意这个地方创建只是为了进行判断！！！！), 然后判断是否为单例、是否允许提前暴露引用(单例一般是 true)、是否在创建中(是否在第四步的集合中)。判断为 true 则将 beanA 添加到三级缓存中
6. 对 beanA 进行属性填充，此时检测到 beanA 依赖于 beanB, 于是开始查找 beanB
7. 调用 doGetBean() 方法，和上面 beanA 的过程一样，到缓存中查找 beanB, 没有则创建, 然后给 beanB 填充属性
8. 此时 beanB 依赖于 beanA, 调用 getSingleton() 获取 beanA, 依次从一级、二级、三级缓存中查找，此时三级缓存中获取到 beanA 的创建工厂, 通过创建工厂获取到 singletonObject, 此时 singletionObject 指向的就是上面再 doCreateBean() 方法中实例化的 beanA
9. 这样 beanB 就获取到了 beanA 的依赖，于是 beanB 顺利完成实例化，并且将 beanA 从三级缓存移动到二级缓存中， 这个地方应该还有将 beanB 从三级缓存中移动到一级缓存中！！！！！！
10. 随后 beanA 继续其属性填充，此时也获取到了 beanB, beanA 随之完成了创建，回到 getSingleton() 方法中继续执行，将 beanA 从二级缓存中移动导一级缓存中
