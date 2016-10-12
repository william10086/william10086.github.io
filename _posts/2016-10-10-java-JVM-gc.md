---
layout: post
title:  "JVM GC"
categories: java
tags:  JVM GC 垃圾回收
---

* content
{:toc}

## 问题描述

##### 1、JVM GC过程

##### 2、JVM垃圾收集算法

##### 3、JVM垃圾收集器




## 解决方案

#### 1、GC过程
	在讲述GC过程前我先解释一下JVM的两个控制参数：-XX:TargetSurvivorRatio ---Survivor Space最大使用率，若存放对象的总大小超过该值，将引起对象向Old区迁移；-XX:MaxTenuringThreshold ---Young区对象的最大任期阀值，即可经历minor gc(young gc)的次数，超过该次数的对象直接迁移到Old区；实际的TenuringThreshold由JVM通过Survivor Space的占用率和TargetSurvivorRatio动态计算，详情请查看参考资料。

《HP-MemoryManagement.pdf》中有对JVM GC过程的形象描述，我借用其中的一些图例来说明一下。
1）Heap在初始状态
![heap init](http://static.oschina.net/uploads/img/201402/18144609_Jzoy.png)

2）在Eden存放新对象
![heap new](http://static.oschina.net/uploads/img/201402/18144611_xoXw.png)

3）Eden空间不足分配新对象，进行第一次minor gc
![heap ygc f](http://static.oschina.net/uploads/img/201402/18144611_w9O8.png)

![heap ygc f](http://static.oschina.net/uploads/img/201402/18144611_4hwi.png)

![heap ygc f](http://static.oschina.net/uploads/img/201402/18144611_pfrY.png)

4) Eden区再次被写满，进行第二次minor gc
![heap ygc s](http://static.oschina.net/uploads/img/201402/18144611_xae0.png)

![heap ygc s](http://static.oschina.net/uploads/img/201402/18144612_qWtA.png)

![heap ygc s](http://static.oschina.net/uploads/img/201402/18144612_2IGr.png)

![heap ygc s](http://static.oschina.net/uploads/img/201402/18144612_Bu8W.png)

5)Eden再次被写满，进行第3次minor gc。
第3次gc，发生了对象从from space提升到old区的迁移，然后也发生了from space到to space的copy
![heap ygc t](http://static.oschina.net/uploads/img/201402/18144612_ntHQ.png)

![heap ygc t](http://static.oschina.net/uploads/img/201402/18144612_KQch.png)

![heap ygc t](http://static.oschina.net/uploads/img/201402/18144612_6QuS.png)

![heap ygc t](http://static.oschina.net/uploads/img/201402/18144612_bCPX.png)

![heap ygc t](http://static.oschina.net/uploads/img/201402/18144612_6DLa.png)

![heap ygc t](http://static.oschina.net/uploads/img/201402/18144612_kMzu.png)

以下是Survivor space空间不足但对象的minor gc次数未到达MaxTenuringThreshold时的gc情况：
![heap ygc t](http://static.oschina.net/uploads/img/201402/18144613_jlsY.png)

![heap ygc t](http://static.oschina.net/uploads/img/201402/18144613_ma6p.png)

![heap ygc t](http://static.oschina.net/uploads/img/201402/18144613_yOnQ.png)

	在进行GC Tuning时有两个很强大的利器：
	jstat：用于查看某java进程的gc情况；jmap：查看java进程堆栈分配和使用情况，以及dump出当前堆栈内容（可以用Eclipse MAT进行进一步分析）以上两个利器都是jdk自带，且无需java进程添加任何额外的debug信息输出参数的，直接就可以对任意java进程进行跟踪了。

#### 2、垃圾检测、收集算法
垃圾收集器一般必须完成两件事：检测出垃圾；回收垃圾。

检测垃圾一般有以下几种方法：

**引用计数法**（Reference Counting）：给一个对象添加引用计数器，每当有个地方引用它，计数器就加1；引用失效就减1。好了，问题来了，如果我有两个对象A和B，互相引用，除此之外，没有其他任何对象引用它们，实际上这两个对象已经无法访问，即是我们说的垃圾对象。但是互相引用，计数不为0，导致无法回收,所以还有另一种方法--可达性分析算法。

**可达性分析算法**：以根集对象为起始点进行搜索，如果有对象不可达的话，即是垃圾对象。这里的根集一般包括java栈中引用的对象、方法区常良池中引用的对象本地方法中引用的对象等。可达性分析（Reachability Analysis）：从GC Roots开始向下搜索，搜索所走过的路径称为引用链。当一个对象到GC Roots没有任何引用链相连时，则证明此对象是不可用的。不可达对象。
	在Java语言中，GC Roots包括：虚拟机栈中引用的对象，方法区中类静态属性实体引用的对象，方法区中常量引用的对象，本地方法栈中JNI引用的对象。

**JVM的垃圾回收过程**
	首先从GC Roots开始进行可达性分析，判断哪些是不可达对象。对于不可达对象，判断是否需要执行其finalize方法，如果对象没有覆盖finalize方法或已经执行过finalize方法则视为不需要执行，进行回收；如果需要，则把对象加入F-Queue队列。对于F-Queue队列里的对象，稍后虚拟机会自动建立一个低优先级的线程去触发其finalize方法，但不会等待这个方法返回。如果在finalize方法的执行过程中，对象重新被引用，那么进行第二次标记时将被移出F-Queue，在finalize方法执行完成后，对象仍然没有被引用，则进行回收。对于被移出F-Queue的对象，如果它下一次面临回收时，将不会再执行其finalize方法。finalize方法只执行一次。

总之，JVM在做垃圾回收的时候，会检查堆中的所有对象是否会被这些根集对象引用，不能够被引用的对象就会被垃圾收集器回收。一般回收算法也有如下几种：

1.标记-清除（Mark-sweep）
	标记所有需要回收的对象，然后统一回收。这是最基础的算法，后续的收集算法都是基于这个算法扩展的。不足之处：效率低，标记清除之后会产生大量碎片。
![mark-sweep](http://img.blog.csdn.net/20150315165800119?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvdG9ueXRmamluZw==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

2.复制（Copying）
	此算法把内存空间划为两个相等的区域，每次只使用其中一个区域。垃圾回收时，遍历当前使用区域，把正在使用中的对象复制到另外一个区域中。此算法每次只处理正在使用中的对象，因此复制成本比较小，同时复制过去以后还能进行相应的内存整理，不会出现“碎片”问题。当然，此算法的缺点也是很明显的，就是需要两倍内存空间。
![copying](http://img.blog.csdn.net/20150315165927137?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvdG9ueXRmamluZw==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

3.标记-压缩整理（Mark-Compact）
	此算法结合了“标记-清除”和“复制”两个算法的优点。也是分两阶段，第一阶段从根节点开始标记所有被引用对象，第二阶段遍历整个堆，把清除未标记对象并且把存活对象“压缩”到堆的其中一块，按顺序排放。此算法避免了“标记-清除”的碎片问题，同时也避免了“复制”算法的空间问题。
![mark-compact](http://img.blog.csdn.net/20150315170004250?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvdG9ueXRmamluZw==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

4.分代收集算法(Generational Collecting)
	这是当前商业虚拟机常用的垃圾收集算法。分代的垃圾回收策略，是基于这样一个事实：不同的对象的生命周期是不一样的。因此，不同生命周期的对象可以采取不同的收集方式，以便提高回收效率。

**年轻代**：是所有新对象产生的地方。年轻代被分为3个部分——Enden区和两个Survivor区（From和to）当Eden区被对象填满时，就会执行Minor GC。并把所有存活下来的对象转移到其中一个survivor区（假设为from区）。Minor GC同样会检查存活下来的对象，并把它们转移到另一个survivor区（假设为to区）。这样在一段时间内，总会有一个空的survivor区。经过多次GC周期后，仍然存活下来的对象会被转移到年老代内存空间。通常这是在年轻代有资格提升到年老代前通过设定年龄阈值来完成的。需要注意，Survivor的两个区是对称的，没先后关系，from和to是相对的。

**年老代**：在年轻代中经历了N次回收后仍然没有被清除的对象，就会被放到年老代中，可以说他们都是久经沙场而不亡的一代，都是生命周期较长的对象。对于年老代和永久代，就不能再采用像年轻代中那样搬移腾挪的回收算法，因为那些对于这些回收战场上的老兵来说是小儿科。通常会在老年代内存被占满时将会触发Full GC,回收整个堆内存。

**持久代**：用于存放静态文件，比如java类、方法等。持久代对垃圾回收没有显著的影响。

分代回收的效果图如下：
![Generational-Collecting](http://img.blog.csdn.net/20150315170424355?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvdG9ueXRmamluZw==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)
	分代算法涉及了前面几种算法：年轻代涉及了“复制（Copying）”算法；年老代涉及了“标记-整理（Mark-Sweep）”的算法。

5.增量算法（Incremental Collecting）
	增量算法基本思想是，如果一次性垃圾回收时会造成的系统长时间的停顿，就可以让垃圾收集程序和应用程序交替执行。每次垃圾收集只收集一小部分区域的内存垃圾，接着就切换应用程序执行，如此往复。不过由于线程切换和上下文切换消耗会造成垃圾回收整体成本上升，造成系统吞吐量下降。

#### 3、垃圾收集器
虚拟机所处的区域，则表示它是属于新生代收集器还是老年代收集器:
![jvm-collector](http://upload-images.jianshu.io/upload_images/650075-8c5080659578032d.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

**1.Serial收集器**
	Serial收集器是最基本、发展历史最悠久的收集器，曾经（在JDK 1.3.1之前）是虚拟机新生代收集的唯一选择。
![serial收集器](http://upload-images.jianshu.io/upload_images/650075-ad2db11e4506ceab.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
特性： 这个收集器是一个单线程的收集器，但它的“单线程”的意义并不仅仅说明它只会使用一个CPU或一条收集线程去完成垃圾收集工作，更重要的是在它进行垃圾收集时，必须暂停其他所有的工作线程，直到它收集结束。Stop The World
应用场景： Serial收集器是虚拟机运行在Client模式下的默认新生代收集器。
优势： 简单而高效（与其他收集器的单线程比），对于限定单个CPU的环境来说，Serial收集器由于没有线程交互的开销，专心做垃圾收集自然可以获得最高的单线程收集效率。

**2.ParNew收集器**
![parnew](http://upload-images.jianshu.io/upload_images/650075-483c1885e2d36f65.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
特性： ParNew收集器其实就是Serial收集器的多线程版本，除了使用多条线程进行垃圾收集之外，其余行为包括Serial收集器可用的所有控制参数、收集算法、Stop The World、对象分配规则、回收策略等都与Serial收集器完全一样，在实现上，这两种收集器也共用了相当多的代码。
应用场景： ParNew收集器是许多运行在Server模式下的虚拟机中首选的新生代收集器。
很重要的原因是：除了Serial收集器外，目前只有它能与CMS收集器配合工作。在JDK 1.5时期，HotSpot推出了一款在强交互应用中几乎可认为有划时代意义的垃圾收集器——CMS收集器，这款收集器是HotSpot虚拟机中第一款真正意义上的并发收集器，它第一次实现了让垃圾收集线程与用户线程同时工作。不幸的是，CMS作为老年代的收集器，却无法与JDK 1.4.0中已经存在的新生代收集器Parallel Scavenge配合工作，所以在JDK 1.5中使用CMS来收集老年代的时候，新生代只能选择ParNew或者Serial收集器中的一个。

Serial收集器 VS ParNew收集器：
	ParNew收集器在单CPU的环境中绝对不会有比Serial收集器更好的效果，甚至由于存在线程交互的开销，该收集器在通过超线程技术实现的两个CPU的环境中都不能百分之百地保证可以超越Serial收集器。然而，随着可以使用的CPU的数量的增加，它对于GC时系统资源的有效利用还是很有好处的。

**3.Parallel Scavenge收集器**
特性： Parallel Scavenge收集器是一个新生代收集器，它也是使用复制算法的收集器，又是并行的多线程收集器。
应用场景： 停顿时间越短就越适合需要与用户交互的程序，良好的响应速度能提升用户体验，而高吞吐量则可以高效率地利用CPU时间，尽快完成程序的运算任务，主要适合在后台运算而不需要太多交互的任务。

Parallel Scavenge收集器 VS CMS等收集器：
	Parallel Scavenge收集器的特点是它的关注点与其他收集器不同，CMS等收集器的关注点是尽可能地缩短垃圾收集时用户线程的停顿时间，而Parallel Scavenge收集器的目标则是达到一个可控制的吞吐量（Throughput）。由于与吞吐量关系密切，Parallel Scavenge收集器也经常称为“吞吐量优先”收集器。

Parallel Scavenge收集器 VS ParNew收集器：
	Parallel Scavenge收集器与ParNew收集器的一个重要区别是它具有自适应调节策略。

GC自适应的调节策略：
	Parallel Scavenge收集器有一个参数-XX:+UseAdaptiveSizePolicy。当这个参数打开之后，就不需要手工指定新生代的大小、Eden与Survivor区的比例、晋升老年代对象年龄等细节参数了，虚拟机会根据当前系统的运行情况收集性能监控信息，动态调整这些参数以提供最合适的停顿时间或者最大的吞吐量，这种调节方式称为GC自适应的调节策略（GC Ergonomics）。

**4.Serial Old收集器**
![serial old](http://upload-images.jianshu.io/upload_images/650075-a877eba6d1753013.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
特性： Serial Old是Serial收集器的老年代版本，它同样是一个单线程收集器，使用标记－整理算法。
应用场景：Client模式 Serial Old收集器的主要意义也是在于给Client模式下的虚拟机使用。 Server模式 如果在Server模式下，那么它主要还有两大用途：一种用途是在JDK 1.5以及之前的版本中与Parallel Scavenge收集器搭配使用，另一种用途就是作为CMS收集器的后备预案，在并发收集发生Concurrent Mode Failure时使用。

**5.Parallel Old收集器**
![parallel old](http://upload-images.jianshu.io/upload_images/650075-0f74222185d67afc.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
特性： Parallel Old是Parallel Scavenge收集器的老年代版本，使用多线程和“标记－整理”算法。
应用场景： 在注重吞吐量以及CPU资源敏感的场合，都可以优先考虑Parallel Scavenge加Parallel Old收集器。

	这个收集器是在JDK 1.6中才开始提供的，在此之前，新生代的Parallel Scavenge收集器一直处于比较尴尬的状态。原因是，如果新生代选择了Parallel Scavenge收集器，老年代除了Serial Old收集器外别无选择（Parallel Scavenge收集器无法与CMS收集器配合工作）。由于老年代Serial Old收集器在服务端应用性能上的“拖累”，使用了Parallel Scavenge收集器也未必能在整体应用上获得吞吐量最大化的效果，由于单线程的老年代收集中无法充分利用服务器多CPU的处理能力，在老年代很大而且硬件比较高级的环境中，这种组合的吞吐量甚至还不一定有ParNew加CMS的组合“给力”。直到Parallel Old收集器出现后，“吞吐量优先”收集器终于有了比较名副其实的应用组合。

**6.CMS收集器**
特性：CMS（Concurrent Mark Sweep）收集器是一种以获取最短回收停顿时间为目标的收集器。目前很大一部分的Java应用集中在互联网站或者B/S系统的服务端上，这类应用尤其重视服务的响应速度，希望系统停顿时间最短，以给用户带来较好的体验。CMS收集器就非常符合这类应用的需求。
![cms](http://upload-images.jianshu.io/upload_images/650075-b50cffd6ed9e15df.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

CMS收集器是基于“标记—清除”算法实现的，它的运作过程相对于前面几种收集器来说更复杂一些，整个过程分为4个步骤：
初始标记（CMS initial mark） 初始标记仅仅只是标记一下GC Roots能直接关联到的对象，速度很快，需要“Stop The World”。

并发标记（CMS concurrent mark） 并发标记阶段就是进行GC Roots Tracing的过程。

重新标记（CMS remark） 重新标记阶段是为了修正并发标记期间因用户程序继续运作而导致标记产生变动的那一部分对象的标记记录，这个阶段的停顿时间一般会比初始标记阶段稍长一些，但远比并发标记的时间短，仍然需要“Stop The World”。

并发清除（CMS concurrent sweep） 并发清除阶段会清除对象。

	由于整个过程中耗时最长的并发标记和并发清除过程收集器线程都可以与用户线程一起工作，所以，从总体上来说，CMS收集器的内存回收过程是与用户线程一起并发执行的。

优点：CMS是一款优秀的收集器，它的主要优点在名字上已经体现出来了：*并发收集、低停顿*。
缺点：CMS收集器对CPU资源非常敏感。 其实，面向并发设计的程序都对CPU资源比较敏感。在并发阶段，它虽然不会导致用户线程停顿，但是会因为占用了一部分线程（或者说CPU资源）而导致应用程序变慢，总吞吐量会降低。CMS默认启动的回收线程数是（CPU数量+3）/ 4，也就是当CPU在4个以上时，并发回收时垃圾收集线程不少于25%的CPU资源，并且随着CPU数量的增加而下降。但是当CPU不足4个（譬如2个）时，CMS对用户程序的影响就可能变得很大。CMS收集器无法处理浮动垃圾，可能出现“Concurrent Mode Failure”失败而导致另一次Full GC的产生。

	由于CMS并发清理阶段用户线程还在运行着，伴随程序运行自然就还会有新的垃圾不断产生，这一部分垃圾出现在标记过程之后，CMS无法在当次收集中处理掉它们，只好留待下一次GC时再清理掉。这一部分垃圾就称为“浮动垃圾”。也是由于在垃圾收集阶段用户线程还需要运行，那也就还需要预留有足够的内存空间给用户线程使用，因此CMS收集器不能像其他收集器那样等到老年代几乎完全被填满了再进行收集，需要预留一部分空间提供并发收集时的程序运作使用。要是CMS运行期间预留的内存无法满足程序需要，就会出现一次“Concurrent Mode Failure”失败，这时虚拟机将启动后备预案：临时启用Serial Old收集器来重新进行老年代的垃圾收集，这样停顿时间就很长了。

CMS收集器会产生大量空间碎片
	CMS是一款基于“标记—清除”算法实现的收集器，这意味着收集结束时会有大量空间碎片产生。空间碎片过多时，将会给大对象分配带来很大麻烦，往往会出现老年代还有很大空间剩余，但是无法找到足够大的连续空间来分配当前对象，不得不提前触发一次Full GC。

**7.G1收集器**
![g1](http://upload-images.jianshu.io/upload_images/650075-6c9c9253495eb757.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
G1（Garbage-First）是一款面向服务端应用的垃圾收集器。HotSpot开发团队赋予它的使命是未来可以替换掉JDK 1.5中发布的CMS收集器。与其他GC收集器相比，G1具备如下特点。
*并行与并发*	G1能充分利用多CPU、多核环境下的硬件优势，使用多个CPU来缩短Stop-The-World停顿的时间，部分其他收集器原本需要停顿Java线程执行的GC动作，G1收集器仍然可以通过并发的方式让Java程序继续执行。

*分代收集*	与其他收集器一样，分代概念在G1中依然得以保留。虽然G1可以不需要其他收集器配合就能独立管理整个GC堆，但它能够采用不同的方式去处理新创建的对象和已经存活了一段时间、熬过多次GC的旧对象以获取更好的收集效果。

*空间整合*	与CMS的“标记—清理”算法不同，G1从整体来看是基于“标记—整理”算法实现的收集器，从局部（两个Region之间）上来看是基于“复制”算法实现的，但无论如何，这两种算法都意味着G1运作期间不会产生内存空间碎片，收集后能提供规整的可用内存。这种特性有利于程序长时间运行，分配大对象时不会因为无法找到连续内存空间而提前触发下一次GC。

*可预测的停顿*	这是G1相对于CMS的另一大优势，降低停顿时间是G1和CMS共同的关注点，但G1除了追求低停顿外，还能建立可预测的停顿时间模型，能让使用者明确指定在一个长度为M毫秒的时间片段内，消耗在垃圾收集上的时间不得超过N毫秒。

	在G1之前的其他收集器进行收集的范围都是整个新生代或者老年代，而G1不再是这样。使用G1收集器时，Java堆的内存布局就与其他收集器有很大差别，它将整个Java堆划分为多个大小相等的独立区域（Region），虽然还保留有新生代和老年代的概念，但新生代和老年代不再是物理隔离的了，它们都是一部分Region（不需要连续）的集合。
	G1收集器之所以能建立可预测的停顿时间模型，是因为它可以有计划地避免在整个Java堆中进行全区域的垃圾收集。G1跟踪各个Region里面的垃圾堆积的价值大小（回收所获得的空间大小以及回收所需时间的经验值），在后台维护一个优先列表，每次根据允许的收集时间，优先回收价值最大的Region（这也就是Garbage-First名称的来由）。这种使用Region划分内存空间以及有优先级的区域回收方式，保证了G1收集器在有限的时间内可以获取尽可能高的收集效率。

**执行过程：**
G1收集器的运作大致可划分为以下几个步骤：

*初始标记（Initial Marking）* 初始标记阶段仅仅只是标记一下GC Roots能直接关联到的对象，并且修改TAMS（Next Top at Mark Start）的值，让下一阶段用户程序并发运行时，能在正确可用的Region中创建新对象，这阶段需要停顿线程，但耗时很短。

*并发标记（Concurrent Marking）* 并发标记阶段是从GC Root开始对堆中对象进行可达性分析，找出存活的对象，这阶段耗时较长，但可与用户程序并发执行。

*最终标记（Final Marking）* 最终标记阶段是为了修正在并发标记期间因用户程序继续运作而导致标记产生变动的那一部分标记记录，虚拟机将这段时间对象变化记录在线程Remembered Set Logs里面，最终标记阶段需要把Remembered Set Logs的数据合并到Remembered Set中，这阶段需要停顿线程，但是可并行执行。

*筛选回收（Live Data Counting and Evacuation）* 筛选回收阶段首先对各个Region的回收价值和成本进行排序，根据用户所期望的GC停顿时间来制定回收计划，这个阶段其实也可以做到与用户程序一起并发执行，但是因为只回收一部分Region，时间是用户可控制的，而且停顿用户线程将大幅提高收集效率。
