---
layout: post
title:  "JVM 内存模型"
categories: java
tags:  JVM memory
---

* content
{:toc}

## 问题描述

**
JVM内存模型中的基本概念
**

![jVM内存模型](http://img.blog.csdn.net/20131231175136859?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQva2luZ29md29ybGQ=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

##### 1、程序计数器

程序计数器（The Program Counter Register）
多线程时，当线程数超过CPU数量或CPU内核数量，线程之间就要根据时间片轮询抢夺CPU时间资源。因此每个线程有要有一个独立的程序计数器，记录下一条要运行的指令。线程私有的内存区域。如果执行的是JAVA方法，计数器记录正在执行的java字节码地址，如果执行的是native方法，则计数器为空。

##### 2、java虚拟机栈

java虚拟机栈 (Java Virtual Machine Stacks)
线程私有的，与线程在同一时间创建。管理JAVA方法执行的内存模型。每个方法执行时都会创建一个桢栈来存储方法的私有变量、操作数栈、动态链接方法、返回值、返回地址等信息。栈的大小决定了方法调用的可达深度（递归多少层次，或嵌套调用多少层其他方法，-Xss参数可以设置虚拟机栈大小）。栈的大小可以是固定的，或者是动态扩展的。如果栈的深度是固定的，请求的栈深度大于最大可用深度，则抛出stackOverflowError；如果栈是可动态扩展的，但没有内存空间支持扩展，则抛出OutofMemoryError。

放在栈中的运算是比java堆速度快，所以尽量使用方法内的局部变量运算速度会比较快。
使用jclasslib工具可以查看class类文件的结构。下图为栈帧结构图：
![JVM Stacks](http://img.blog.csdn.net/20140101100938109?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQva2luZ29md29ybGQ=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

##### 3、本地方法栈

和虚拟机栈功能相似，但管理的不是JAVA方法，是本地方法，本地方法是用C实现的。与虚拟机栈一样也会抛出stackOverflowError和native heap OutofMemoryError异常。

##### 4、java堆

JAVA堆（JVM heap）是线程共享的，存放所有对象实例和数组。垃圾回收的主要区域。可以分为新生代和老年代(tenured)。新生代用于存放刚创建的对象以及年轻的对象，如果对象一直没有被回收，生存得足够长，老年对象就会被移入老年代。
新生代又可进一步细分为eden、survivorSpace0(s0,from space)、survivorSpace1(s1,to space)。刚创建的对象都放入eden,s0和s1都至少经过一次GC并幸存。如果幸存对象经过一定时间仍存在，则进入老年代(tenured)。
![java heap](http://img.blog.csdn.net/20140101101922203?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQva2luZ29md29ybGQ=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
目前主流的虚拟机堆都是可扩展的，通过（-Xmx和-Xms控制），当堆无法 再扩展时会抛出OutOfMemoryError

##### 5、方法区

方法区（Method Area）是线程共享的，用于存放被虚拟机加载的类的元数据信息：如常量、静态变量、即时编译器编译后的代码。也成为永久代。如果hotspot虚拟机确定一个类的定义信息不会被使用，也会将其回收。回收的基本条件至少有：所有该类的实例被回收，而且装载该类的ClassLoader被回收。
另外，它还有一个名字叫非堆（Non-heap），用于与java堆区分。

##### 6、运行时常量池

运行时常量池（Runtime Constant Pool）是方法区的一部分，Class文件除了有类的版本、字段、方法、接口等描述信息外，还有一项信息是常量池，用于存放编译期生成的各种字面量和符号引用，这部分内容将在类加载后进入方法区的运行时常量池中存放。Java语言并不要求常量一定只有编译期才能产生，运行期间也可能将新的常量放入池中(比如String中的intern()方法)，当常量无法再申请到内存时就会抛出OutOfMemoryError异常

##### 7、直接内存

直接内存并不是虚拟机运行时数据区的一部分，也不是Java虚拟机规范中定义的内存区域。在JDK1.4中加入了NIO类，引入可一种基于通道和缓冲区的I/O方式，他可以使用native函数库直接分配堆外内存，然后通过一个存储在Java堆中的DirectByteBuffer对象作为这块内存的引用进行操作，这样能在一些场景中显著的提高性能，因为避免了在Java堆和Native堆中来回复制数据。
直接内存大小不会受到Java堆大小的限制，但会受到本机实际内存的限制，从而可能导致动态扩展时出现OutOfMemoryError异常。

##### 8、类加载子系统、执行子系统、Java本机接口
Class loader子系统的作用：根据给定的全限定名类名(java.lang.Object)来装载class文件的内容到 Runtime data area 中的method area(方法区域)。Java 程序员可以extends java.lang.ClassLoader 类来写自己的Class loader。

Execution engine子系统的作用 ：执行classes中的指令。任何JVM specification实现(JDK)的核心是Execution engine， 换句话说：Sun的JDK和IBM的JDK好坏主要取决于他们各自实现的Execution engine的好坏。每个运行中的线程都有一个Execution engine的实例。

Native interface 组件：是一个标准的Java API，它支持将Java代码与使用其他编程语言编写的代码相集成。如果您希望利用已有的代码资源，那么可以使用 JNI作为您工具包中的关键组件 —— 比如在面向服务架构（SOA）和基于云的系统中。