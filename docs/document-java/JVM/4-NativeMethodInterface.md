# Native Method Inteface

## 本地方法

    Java 调用 非 Java 语言实现的方法

    融合不同的编程语言你为 Java 所用

    native 关键字修饰的方法

    native 不能和 abstract 共用

## 为什么使用 native 方法

    与 Java 环境外交互

    与操作系统交互

    Sun's Java: Sun 的解释器是 C 实现的

## 本地方法栈

    Java 虚拟机栈管理 Java 方法调用

    本地方法管理本地方法调用

    线程私有

    固定大小或者动态扩展，内存溢出与 Java 虚拟机栈一样

    C 语言实现

    本地方法接口，本地方法库

    线程调用某个本地方法时，进入全新的不受虚拟机限制的世界，和虚拟机权限相同

    1. 可以使用本地方法接口访问虚拟机内部的运行时数据区
    2. 直接使用本地处理器的寄存器
    3. 从本地内存的堆分配任意数量内存

    不是所有的虚拟机都支持本地方法，HotSpot 整体架构

    HotSpot 本地方法栈和虚拟机栈方法合二为一