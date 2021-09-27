# 直接内存

## 概述

    不是内存中的内容，也不是 java 虚拟机规范中定义的内容
    Java 堆外，直接向系统内存申请
    来源于 NIO（New IO / Non-Blocking IO）, 通过 DirectByteBuffer() 操作本地内存
    访问直接内存的速度优于Java堆内存访问，少了一次用户态与内核态之间的数据交互

## OOM 异常

    大小受限于堆空间大小，Java 堆和直接内存的总和大小小于操作系统的最大内存

    OOM: Direct buffer memory

    缺点：
        分配回收成本较高
        不受 JVM 内存回收管理
    
    MaxDirextMemorySize 设置
    不指定的话默认与堆空间堆空间最大值一致

    DirectByteBuffer() 是通过 unsafe 进行分配的
