# String Table

## String 基本特性

    字符串，"" 引用

    String 是一个对象

    声明为 final，不可以被继承

    String 实现了 Serializable、Comparable、CharSequence 接口

    jdk1.8 String 底层用 char 数组，jdk1.9 底层使用 byte 数组容器， 一个字节就能存下，一半的空间被浪费

    如果是其它的使用两个字符去存

***String 代表不可变的字符序列，不可变性***

    对字符串进行重新赋值时候，重写指定内存区域赋值，不能使用原有的 value 进行赋值

    对现有的字符串进行连接操作，也需要重新指定内存区域赋值，不能使用原有的 value 进行赋值

    调用 replace() 方法修改指定字符或者字符串的时候，也需要重新指定内存区域赋值，不能使用原有的 value 进行赋值

    字面量赋值的话，字面量存放于字符串常量池中

***字符串常量池中不会存储相同内容的字符串的***

    String 的 String Pool 固定大小的 Hashtable, jdk6 中默认长度 1009， jdk7 中默认长度为 60013，最小值设置没限制，
    jdk8 中 1009是可以设置的最小值

    -XX: StringTableSize 设置长度大小

    String 放入太多，Hash 冲突严重，调用 String.intern() 方法效率较低

## String 内存分配

    String 常量池的使用：
        双引号引用
        .intern() 方法
    
    jdk6 字符串常量池在永久代中，7以及以后都放在堆中

    StringTable 为什么要调整，permSize 比较小；永久代垃圾回收频率较低 

## String 基本操作

    Java 语言规范中要求：完全相同的字符串字面量，应该包含同样的 Unicode 字符序列（包含同一份马甸序列的常量），
    并且必须是指向同一个 String 类的实例

    常量池中不允许放相同的字符串

## 字符串拼接

    常量与常量拼接结果放在常量池中，原理是编译期优化

    常量池中不允许放相同的常量

    只要其中有一个是变量，结果就在堆中(非常量池位置)，变量拼接的原理是 StringBuilder，相当于在堆空间中 new String(), 
    只要 new 相当于开辟新的空间

    如果拼接的结果调用 intern() 方法，则主动将常量池中还没有的字符串对象放入池中，返回此对象地址，有的话直接就直接返回已有的
    intern() 方法判断字符串常量池中是否有，存在返回地址，不存在则在常量池中加载一份，并返回对象的地址

### 字符串常量拼接操作底层原理

    s1 + s2

```java
StringBuilerd s = new StringBuilder() 
// 补充，jdk5.0 之后，使用的是 StringBuilder, 
//5.0 之前用的是 StringBuffer
s.append("a")
s.append("b")
s.toString() // 约等于 new String("ab")
```

***特殊情况，String 被 final 修饰，那么就不是变量，相当于是引用***

    字符串拼接操作不一定使用的是 StringBuilder, 如果拼接符号左右两边都是字符串常量或者是字符串引用(final 修饰)，
    仍然使用编译器优化，非 StringBuilder 的方式

    针对于 final 修饰的方法、类、基本数据类型、引用数据类型的量的结构时，能使用上 final 的时候，建议使用

### 拼接操作与 append 操作的效率

    拼接操作："+"， 4014ms, 每次都会创建 StringBuilder、String
    append：7ms, 只有一个 StringBuilder, 一次 toString() 方法

   字符串拼接：内存中由于创建了较多的 StringBuilder 和 String 对象，内存占用较大，同时 GC 的时间较多

    进一步优化：StringBuilder 可以进行初始化赋值，如果基本确定字符串长度的话， new char[highLevel]

## intern 方法

    intern() 方法会使用 String 中的 intern() 方法，intern() 方法查询字符串常量池中该字符串是否存在，不存在就放入常量中

    intern() 保证字符串常量中只有一份

    保证变量 s 指向的字符串常量池中的数据
    1. String s = "hello"; 字面量方式
    2. String s = new String("hello").intern()
       String s = new StringBuilder("hello").intern()

### 面试题

    String str = new String("ab") 建了几个对象, 2个对象，判断之前有没有 "ab" 在字符串常量池中
    1. new 堆空间中造一个对象
    2. ldc 字符串常量池放一个字面量 "ab"

    String str = new String("a") + new String("b")
    1. new StringBuilder()
    2. new String()
    3. 常量池中的 "a"
    4. new String()
    5. 常量池中的 "b"
    6. toString() 中 new string("ab"), toString() 方法，字符串常量池不会放 "ab" ！！！！

### intern() 的使用

    String intern() 方法的使用
    jdk 1.6 中，将这个字符串对象尝试放入到串池中
        1. 如果串池中有，则并不会放入，返回已有的串池中对象的地址
        2. 如果串池中没有，则会把此对象复制一份，放入串池，返回串池中对象地址，此处注意是复制一份
    jdk 1.7 以后，将这个字符串对象尝试放入到串池
        1. 如果串池中有，则并不会放入，返回已有的串池中对象的地址
        2. 如果串池中没有，则会把此对象的引用地址复制一份，放入串池，返回串池中引用地址，此处注意是堆引用地址进行复制一份 

### intern 空间效率测试

    intern 更节省内存空间，同样实例数更少

## StringTable 垃圾回收

    -XX:+PrintStringTableSatistics 打印 StringTable 的统计信息

## G1 中的 String 去重操作

    G1 回收器中的 String 去重操作

    去重操作？字符串常量池不存在重复的内容

    背景：堆存活数据集合里边 String 对象占 25%
          重复 String 对象有 13.5%
          平均长度是 45
    
    堆上存在重复的 String 对象必然是一种内存的浪费

    Hashtable 进行去重
