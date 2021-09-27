# Runtime Data Area

一个 JVM 对应一个 Runtime 对象

## JVM 中的线程

    程序运行单元

    多个线程并行，与本地线程一一映射，创建时创建，回收时回收

    调度到可用 CPU，执行线程 run 方法

    准备工作：PC、本地方法栈、堆等

    异常出现，最后一个非守护线程，线程结束、虚拟机也会结束

## PC 寄存器

### 概述

    PC 寄存器是物理寄存器的模拟

    指向下一条指令的地址，执行引擎读取下一条指令

    占用空间较小，运行速度快

    线程私有，周期一致

    当前方法：任何线程同一时间只有一个方法执行，存储当前方法 JVM 指令地址，本地方法则是 undefined

    控制流指示器

    唯一没有 OutOfMemoryError 情况

### PC 寄存器两个常见的问题

    1. 存储字节码指令地址有什么用，当前线程的地址

        单核 CPU 并发执行
        
        CPU 在不停的切换，需要记录从哪里开始的

## 虚拟机栈

    出现背景： 跨平台、指令小、编译器易实现；性能下降、指令集多

***栈是运行时的单位、堆是存储的单位***

    每个线程创建时都会创建一个虚拟机栈、线程私有

    一个栈有多个栈帧、当前方法：正在执行的方法

    生命周期与线程一致

    JAVA 程序运行，保存方法局部变量、部分结果，参与方法调用与返回

### 变量

    局部变量 VS 成员变量

    基本数据变量 VS 引用类型变量

### 优点

    快速有效分配

    只有进栈和出栈

    不存在垃圾回收，但有 OOM

### 常见异常

    栈大小固定：StackOverflowError，栈能分配的最大的大小不足，函数调用的最大深度 -Xss1M

    栈大小动态扩展：OutOfMemory, 整体虚拟机内存不够

## 栈的存储结构

    线程 <-> 栈

    栈对应多个栈帧, 方法与栈帧一一对应（Stack Frame）

    栈帧是一个内存区块，一个数据集、方法执行中的各种数据信息

    入栈和出栈操作

    一个时间点一个栈帧

    当前栈帧、当前方法、当前类，有效的

    执行引擎只针对当前栈帧有效

    调用方法、创建新的栈帧、成为栈顶

### 复习

    OOP 的基本概念：类、对象

    类中的基本结构：field（属性、字段、域）, 方法

### 运行原理

    不同线程栈帧不允许相互引用，线程私有

    返回值传给前一个栈

    正常返回，return 为代表
    
    方法中出现为处理的异常，则抛出异常(没有处理)
    向上看是否上层是否有处理，没有处理就接着向上，如果 main() 方法也没处理，抛出异常

### 栈的内部结构

#### 局部变量表（Local Variables）

    数字数组，存方法参数（包括形参）、局部变量 -> 基本数据类型、对象引用、returnAddress

    32 位以内的占用一个 slot, 包括 returnAddress, 64 位的占据两个 slot

    线程私有，数据安全

    容量编译期间确定，运行期间不会更改，maxinum local variables

    方法嵌套调用的次数由栈的大小决定

    局部变量表只在当前方法调用有效, 栈帧销毁，局部变量表销毁

    变量的作用域范围，Start PC， 字节码对应的代码行号的下一行，Length + Start PC = Code Length 作用域长度

    槽(Slot) 基本存储单元

    索引0开始，基本类型、引用类型（1个Slot）、returnAddress, long 和 double 两个 Slot

    通过索引访问局部变量，long 和 double 使用起始索引

    存放顺序按照声明顺序

***构造方法或者非静态方法，当前对象引用 this 存放在 index = 0 位置***

    Slot 重复利用，出了作用域的话，其作用域之后申明的变量可能会重复利用 Slot，节约资源

    下示代码中 b 的作用域为大括号内， c 重复利用 b 的 Slot

```java
public void test(){
    int a = 0;
    {
        int b = 0;
        b = a + 1;
    }
    // c 重复使用 b 的 slot
    int c = a + 1
}
```

    变量分类：
    
    基本数据类型、引用数据类型；
    
    成员变量：使用前进行默认初始化
            
            类变量（static 修饰的）：linking 的 prepare 给类变量默认赋值  --> initial 阶段：给类变量显式赋值，代码块赋值
    
            实例变量 （没有 static 修饰）：对象创建、堆空间分配实例变量空间，默认赋值
    
    局部变量：使用前必须进行显示赋值、否则编译不通过

    栈管运行

***局部变量表中的变量是重要的垃圾回收的根节点、局部变量表选中的直接或者间接引用对象都不会被回收***

#### 操作数栈（Operand Stack）

    表达式栈

    方法执行过程中、根据字节码指令执行操作数的入栈和出战操作

    保存计算过程中间结果、临时存储结构

    方法执行、栈帧创建、操作数栈为空

    数组一旦创建、长度是固定的，编译期间确定操作数栈的长度

    32位数据占一个栈深度，64位占两个栈深度

    只能通过入栈和出栈操作数据

    返回值也会被压入操作数栈

    JAVA 虚拟机是解释引擎是基于栈的执行引擎

    byte、short、char、boolean 都是以 int 方式进行存储

    有返回值，则会压入到操作数栈、然后保存到局部变量表中 `aload_0`，返回上一个栈帧的返回结果, 保存在操作数栈中

    i++ 和 ++i 的区别：
    1.
    2.
    3.

#### 栈顶缓存技术

    Top of stack Cashing

    HotSpot 栈式架构，零地址指令，更多的入栈和出栈操作，内存独写次数很多

    将栈顶元素全部缓存到物理 CPU 的寄存器中，降低对内存的独写次数，提升执行引擎的执行效率

#### 动态链接（Dynamic Linking）

***指向运行时常量池的方法引用***

    字节码指令需要常量池的访问

    栈帧内部包含指向运行时常量池该栈帧所属方法的引用

    所有变量和方法引用都作为符号引用保存在常量池

    动态链接：符号引用转换为直接引用

    为什么要常量池：提供符号和常量便于指令识别

#### 方法的调用

***符号引用转为直接引用与方法绑定机制相关***

    绑定：符号引用替换为直接引用，只发生一次

    静态链接/早期绑定: 编译期间确定，运行期保持不变

    动态链接/晚期绑定：编译期间不能确定，运行期进行

    final 标识的方法不能被重写，编译的时候就已经确定了

***虚方法和非虚方法***

    虚方法：动态链接或者晚期绑定

    非虚方法：静态方法、私有方法、final 方法、实例构造器、父类方法

    多态性使用前提：
    1. 类的继承
    2. 方法重写

***调用指令***

    invokestatic: 非虚，静态

    invokespecial: 非虚，<init> 方法、私有、父类

    invokevirtual: 所有虚方法，注意： final 方法也是其调用

    invokeinterface: 调用接口

    invokedynamic: 实现动态类型语言， Java8 lambda 表达式， invokedynamic 指令可以直接生成

    对类型的检查是在编译器还是在运行期间
    动态类型语言：判断变量值的类型信息
    静态类型语言：判断变量自身的类型信息，变量没有类型信息，只有变量值才有类型信息

    所以说 Java 还是静态语言

***方法重写的本质***

   操作数栈(存放方法)栈顶第一个元素执行的实际类型，记为 C

   类型 C 找到与其描述符合，简单名称符合的方法，进行校验，通过则返回该方法直接调用，结束，不通过，返回 java.lang.IllegalAccessError

   否则，按照继承关系从下向上对 C 的父类进行 2 步的搜索和验证

   最终没有找到则抛出，java.lang.AbstractMethodError

***虚方法表***

    为了提高性能，在方法区建立了一个虚放发表，使用索引表代替查找

    在链接过程中的解析环节进行创建并且初始化， 类变量初始值完成后，JVM 会将该类的方法表也初始化完毕

    每个类都都一个虚方法表

    方法相同的方法，指向公共的部分

#### 方法返回地址（Return Address）

    存储该方法 pc 寄存器的值

    正常执行结束：返回值传递给上层的方法调用者

    出现未处理的异常、非正常退出(未被处理)：不会返回给方法调用者值

#### 一些附加信息

    虚拟机实现相关的附加信息，调试相关的一些信息

### 栈的相关面试信息

    栈溢出：StackOverFlowError -Xss 设置栈大小，OOM 整个空间内存不足

    调整栈大小，保证不出现溢出？ 不能！出现的时间晚一点

    垃圾回收会涉及到虚拟机栈：不涉及！

    分配的栈内存越大越好吗？不是！一点时间内降低 STackOverFlowError, 但是栈内存变大，栈数量变少，线程数变少

    方法中局部变量是否线程安全？具体问题，具体分析

    StringBuffer 本来就是线程安全的，StringBuilder 不是线程安全的

```java
public static void method1(){
    StringBuilder s1 = new StringBuilder();
    s1.append("a");
    s2.append("b");
}

// 存在线程不安全的问题
public static void method(StringBuilder s1){
    s1.append("a");
    s2.append("b");
}

// 可能存在线程不安全的问题
public static StringBuiler method(){
    StringBuilder s1 = new StringBuilder();
    s1.append("a");
    s2.append("b");
    return s1;
}

// s1 线程安全
public static String method(){
    StringBuilder s1 = new StringBuilder();
    s1.append("a");
    s2.append("b");
    return s1.toString();
}

```