# Class Loader SubSystem

加载 Class 文件，开头 `CAFEBABE` 特定标识

只负责加载，执行引擎决定能否执行

加载后存放再方法区、方法区还存放常量池: 字面量和数字常量, Class 常量池映射

## Loading

    类全类名获取二进制字节流

    静态存储结构转为方法区运行时数据结构

    内存中生成一个 java.lang.Class 对象，作为方法区运行时数据入口

    生成大的 Class 实例

## Linking

### 验证 Verify

    字节流符合虚拟机要求，安全，不危害虚拟机，CAFEBABE 开头

### 准备 Prepare

    1. 变量分配内存，并且进行初始化，初始化为：零值，
    2. final 修饰的 static除外，final 在编译的时候就进行分配，准备阶段显示初始化
    3. 不会为实例对象分配初始化, 随对象初始化而分配

### 解析 Resolve

    常量池，符号引用 -> 直接引用, 接口，字段，类/接口方法，方法类型等

## Initialization

    1. 执行类构造器方法 <clinit>() 过程，没有类构造器方法 ???，那么就没有 <clinit>()
    2. 无需定义，搜集类变量赋值动作和静态代码块中的语句合并而来
    3. 按源文件出现的顺序执行
    4. 不同于类的构造器 <init>() 函数, 任何类声明后内部至少有一个构造器
    5. 子类的 <init>() 晚于 父类的 <init>()
    6. 一个类的 <clinit>() 方法在多线程下被同步加锁，一个类只会被加载一次！！！！！！

### Initialization 之 clinit

```java
public class ClassInitTest {

    private static int num = 1;

    num = 2;
    System.out.println(num);
    number = 20;
    // System.out.println(number); // 非法前向引用

    private static int number = 10; //Linking之prepare: number = 0 --> initial:20 --> 10

    public static void main(String[] args) {
        System.out.println(num);
        System.out.println(number);
    }
}
```

## 类加载器

    引导类加载器和自定义类加载器

    Booststrap ClassLoader 和 User-Defined ClassLoader

    自定义类加载器 -> 派生于抽象类加载器的都是自定义类加载器

    Booststrap ClassLoader / Extension Class Loader / System Class Loader

    四者之间不是继承关系, 上下级关系

    extClassLoader.getParent() 无法获取 BoostrapLoader

    用户自定义类：系统类加载器 AppClassLoader() 进行加载

    String 类：获取不到引导类加载器 -> Java 的核心类库都是引导类加载器加载的

### Booststrap ClassLoader

    C/C++ 实现的

    加载核心类库，不继承 java.lang.ClassLoader，无父类加载器

    加载扩展类和应用类启动器，成为他们的父类加载器

    安全，只加载包名 java、javax、sun 开头的类

### Extension Class Loader

    Java 语言编写

    派生于 ClassLoader 类

    jre/lib/ext 扩展目录

    用户类放在这儿也可以被加载

### AppClassLoader

    Java 编写

    父类为扩展类加载器

    主要用来加载用户编写的类

### 用户自定义类加载器

为什么自定义？

    - 隔离加载类 -> 中间件有自己的 jar 包，放在同一个目录加载到内存中可能会出现冲突
    - 修改类的加载方式 -> 按需动态加载
    - 扩展加载源 -> 数据库中
    - 防止源码泄漏

自定义加载器实现步骤

    1. 继承 ClassLoader
    2. 1.2 之后重写 findClass() 方法，如果指定字节码加密，则需要进行解密操作
    3. 如果不是很复杂，直接继承 URLClassLoader 就可以了

ExtClassLoader 和 AppClassLoader 都是 sum.misc.Launcher 的内部类

### 获取类加载器

    1. clazz.getClassLoader()
    2. 线程获取当前上下文的 ClassLoader
    3. 直接获取系统的 ClassLoader
    4. 获取调用者的ClassLoader -> DreverManager.getCallerClassLoader()

### 双亲委派机制

Class 文件按需加载，生成 Class 对象，采用双亲委派机制，将请求交由父类处理，任务委派模式

    1. 类加载器收到请求，不会自己先加载，而是将请求委托给父类加载器
    2. 父类还有父类加载器，向上委托
    3. 父类加载器可以完成，则父类加载器加载，如果不能加载，则考虑使用子加载器进行加载
    4. 默认是系统类加载器进行加载

#### 双亲委派机制优势

    1. 避免类的重复加载
    2. 保护程序安全，放置核心 API 进行篡改，阻止包名直接自定义一个类，禁止使用这样的包名

#### 沙箱安全机制

### 问题

两个 class 对象是否为同一个类，两个必要条件

    1. 完整类名一致
    2. ClassLoader 也需要一致

***JVM 会将某个类加载器的引用作为类型的一部分保存在方法区中***

保证两个类型的类加载器时相同的

类的主动使用和被动使用

***被动使用不会导致类的初始化***
