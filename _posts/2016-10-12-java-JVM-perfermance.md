---
layout: post
title:  "JVM 性能优化"
categories: java
tags:  JVM 性能优化
---

* content
{:toc}


##### JVM性能调优参数、过程和实例




## 解决方法

##### 1、JVM调优相关参数

**1.JVM参数含义：**

<table cellspacing="0" border="1"> <tbody> <tr> <td width="25%" valign="top"><strong>参数名称</strong></td> <td width="20%" valign="top"><strong>含义</strong></td> <td width="10%" valign="top"><strong>默认值</strong></td> <td valign="top">&nbsp;</td></tr> <tr> <td valign="top">-Xms</td> <td valign="top">初始堆大小</td> <td valign="top">物理内存的1/64(&lt;1GB)</td> <td valign="top">默认(MinHeapFreeRatio参数可以调整)空余堆内存小于40%时，JVM就会增大堆直到-Xmx的最大限制.</td></tr> <tr> <td valign="top">-Xmx</td> <td valign="top">最大堆大小</td> <td valign="top">物理内存的1/4(&lt;1GB)</td> <td valign="top">默认(MaxHeapFreeRatio参数可以调整)空余堆内存大于70%时，JVM会减少堆直到 -Xms的最小限制</td></tr> <tr> <td valign="top">-Xmn</td> <td valign="top">年轻代大小(1.4or lator)<br></td> <td valign="top">&nbsp;</td> <td valign="top"><strong>注意</strong>：此处的大小是（eden+ 2 survivor space).与jmap -heap中显示的New gen是不同的。<br>整个堆大小=年轻代大小 + 年老代大小 + 持久代大小.<br>增大年轻代后,将会减小年老代大小.此值对系统性能影响较大,Sun官方推荐配置为整个堆的3/8</td></tr> <tr> <td valign="top">-XX:NewSize</td> <td valign="top">设置年轻代大小(for 1.3/1.4)</td> <td valign="top">&nbsp;</td> <td valign="top">&nbsp;</td></tr> <tr> <td valign="top">-XX:MaxNewSize</td> <td valign="top">年轻代最大值(for 1.3/1.4)</td> <td valign="top">&nbsp;</td> <td valign="top">&nbsp;</td></tr> <tr> <td valign="top">-XX:PermSize</td> <td valign="top">设置持久代(perm gen)初始值</td> <td valign="top">物理内存的1/64</td> <td valign="top">&nbsp;</td></tr> <tr> <td valign="top">-XX:MaxPermSize</td> <td valign="top">设置持久代最大值</td> <td valign="top">物理内存的1/4</td> <td valign="top">&nbsp;</td></tr> <tr> <td valign="top">-Xss</td> <td valign="top">每个线程的堆栈大小</td> <td valign="top">&nbsp;</td> <td valign="top">JDK5.0以后每个线程堆栈大小为1M,以前每个线程堆栈大小为256K.更具应用的线程所需内存大小进行 调整.在相同物理内存下,减小这个值能生成更多的线程.但是操作系统对一个进程内的线程数还是有限制的,不能无限生成,经验值在3000~5000左右<br>一般小的应用， 如果栈不是很深， 应该是128k够用的 大的应用建议使用256k。这个选项对性能影响比较大，需要严格的测试。（校长）<br>和threadstacksize选项解释很类似,官方文档似乎没有解释,在论坛中有这样一句话:"”<br>-Xss is translated in a VM flag named ThreadStackSize”<br>一般设置这个值就可以了。</td></tr> <tr> <td valign="top">-<em>XX:ThreadStackSize</em></td> <td valign="top">Thread Stack Size</td> <td valign="top">&nbsp;</td> <td valign="top">(0 means use default stack size) [Sparc: 512; Solaris x86: 320 (was 256 prior in 5.0 and earlier); Sparc 64 bit: 1024; Linux amd64: 1024 (was 0 in 5.0 and earlier); all others 0.]</td></tr> <tr> <td valign="top">-XX:NewRatio</td> <td valign="top">年轻代(包括Eden和两个Survivor区)与年老代的比值(除去持久代)</td> <td valign="top">&nbsp;</td> <td valign="top">-XX:NewRatio=4表示年轻代与年老代所占比值为1:4,年轻代占整个堆栈的1/5<br>Xms=Xmx并且设置了Xmn的情况下，该参数不需要进行设置。</td></tr> <tr> <td valign="top">-XX:SurvivorRatio</td> <td valign="top">Eden区与Survivor区的大小比值</td> <td valign="top">&nbsp;</td> <td valign="top">设置为8,则两个Survivor区与一个Eden区的比值为2:8,一个Survivor区占整个年轻代的1/10</td></tr> <tr> <td valign="top">-XX:LargePageSizeInBytes</td> <td valign="top">内存页的大小不可设置过大， 会影响Perm的大小</td> <td valign="top">&nbsp;</td> <td valign="top">=128m</td></tr> <tr> <td valign="top">-XX:+UseFastAccessorMethods</td> <td valign="top">原始类型的快速优化</td> <td valign="top">&nbsp;</td> <td valign="top">&nbsp;</td></tr> <tr> <td valign="top">-XX:+DisableExplicitGC</td> <td valign="top">关闭System.gc()</td> <td valign="top">&nbsp;</td> <td valign="top">这个参数需要严格的测试</td></tr> <tr> <td valign="top">-XX:MaxTenuringThreshold</td> <td valign="top">垃圾最大年龄</td> <td valign="top">&nbsp;</td> <td valign="top">如果设置为0的话,则年轻代对象不经过Survivor区,直接进入年老代. 对于年老代比较多的应用,可以提高效率.如果将此值设置为一个较大值,则年轻代对象会在Survivor区进行多次复制,这样可以增加对象再年轻代的存活 时间,增加在年轻代即被回收的概率<br>该参数只有在串行GC时才有效.</td></tr> <tr> <td valign="top">-XX:+AggressiveOpts</td> <td valign="top">加快编译</td> <td valign="top">&nbsp;</td> <td valign="top">&nbsp;</td></tr> <tr> <td valign="top">-XX:+UseBiasedLocking</td> <td valign="top">锁机制的性能改善</td> <td valign="top">&nbsp;</td> <td valign="top">&nbsp;</td></tr> <tr> <td valign="top">-Xnoclassgc</td> <td valign="top">禁用垃圾回收</td> <td valign="top">&nbsp;</td> <td valign="top">&nbsp;</td></tr> <tr> <td valign="top">-XX:SoftRefLRUPolicyMSPerMB</td> <td valign="top">每兆堆空闲空间中SoftReference的存活时间</td> <td valign="top">1s</td> <td valign="top">softly reachable objects will remain alive for some amount of time after the last time they were referenced. The default value is one second of lifetime per free megabyte in the heap</td></tr> <tr> <td valign="top">-XX:PretenureSizeThreshold</td> <td valign="top">对象超过多大是直接在旧生代分配</td> <td valign="top">0</td> <td valign="top">单位字节 新生代采用Parallel Scavenge GC时无效<br>另一种直接在旧生代分配的情况是大的数组对象,且数组中无外部引用对象.</td></tr> <tr> <td valign="top">-XX:TLABWasteTargetPercent</td> <td valign="top">TLAB占eden区的百分比</td> <td valign="top">1%</td> <td valign="top">&nbsp;</td></tr> <tr> <td valign="top">-XX:+<em>CollectGen0First</em></td> <td valign="top">FullGC时是否先YGC</td> <td valign="top">false</td> <td valign="top">&nbsp;</td></tr></tbody></table>

**2.并行收集器相关参数：**

<table cellspacing="0" border="1"> <tbody> <tr> <td width="25%" valign="top">-XX:+UseParallelGC</td> <td width="20%" valign="top">Full GC采用parallel MSC<br>(此项待验证)</td> <td width="10%" valign="top">&nbsp;</td> <td valign="top"> <p>选择垃圾收集器为并行收集器.此配置仅对年轻代有效.即上述配置下,年轻代使用并发收集,而年老代仍旧使用串行收集.(此项待验证)</p></td></tr> <tr> <td width="25%" valign="top">-XX:+UseParNewGC</td> <td width="20%" valign="top">设置年轻代为并行收集</td> <td width="10%" valign="top">&nbsp;</td> <td valign="top">可与CMS收集同时使用<br>JDK5.0以上,JVM会根据系统配置自行设置,所以无需再设置此值</td></tr> <tr> <td width="25%" valign="top">-XX:ParallelGCThreads</td> <td width="20%" valign="top">并行收集器的线程数</td> <td width="10%" valign="top">&nbsp;</td> <td valign="top">此值最好配置与处理器数目相等 同样适用于CMS</td></tr> <tr> <td width="25%" valign="top">-XX:+UseParallelOldGC</td> <td width="20%" valign="top">年老代垃圾收集方式为并行收集(Parallel Compacting)</td> <td width="10%" valign="top">&nbsp;</td> <td valign="top">这个是JAVA 6出现的参数选项</td></tr> <tr> <td width="25%" valign="top">-XX:MaxGCPauseMillis</td> <td width="20%" valign="top">每次年轻代垃圾回收的最长时间(最大暂停时间)</td> <td width="10%" valign="top">&nbsp;</td> <td valign="top">如果无法满足此时间,JVM会自动调整年轻代大小,以满足此值.</td></tr> <tr> <td width="25%" valign="top">-XX:+UseAdaptiveSizePolicy</td> <td width="20%" valign="top">自动选择年轻代区大小和相应的Survivor区比例</td> <td width="10%" valign="top">&nbsp;</td> <td valign="top">设置此选项后,并行收集器会自动选择年轻代区大小和相应的Survivor区比例,以达到目标系统规定的最低相应时间或者收集频率等,此值建议使用并行收集器时,一直打开.</td></tr> <tr> <td width="25%" valign="top">-XX:GCTimeRatio</td> <td width="20%" valign="top">设置垃圾回收时间占程序运行时间的百分比</td> <td width="10%" valign="top">&nbsp;</td> <td valign="top">公式为1/(1+n)</td></tr> <tr> <td width="25%" valign="top">-XX:+<em>ScavengeBeforeFullGC</em></td> <td width="20%" valign="top">Full GC前调用YGC</td> <td width="10%" valign="top">true</td> <td valign="top">Do young generation GC prior to a full GC. (Introduced in 1.4.1.)</td></tr></tbody></table>

**3.CMS相关参数：**

<table cellspacing="0" border="1"> <tbody> <tr> <td width="25%" valign="top">-XX:+UseConcMarkSweepGC</td> <td width="20%" valign="top">使用CMS内存收集</td> <td width="10%" valign="top">&nbsp;</td> <td valign="top">测试中配置这个以后,-XX:NewRatio=4的配置失效了,原因不明.所以,此时年轻代大小最好用-Xmn设置.???</td></tr> <tr> <td width="25%" valign="top">-XX:+AggressiveHeap</td> <td width="20%" valign="top">&nbsp;</td> <td width="10%" valign="top">&nbsp;</td> <td valign="top">试图是使用大量的物理内存<br>长时间大内存使用的优化，能检查计算资源（内存， 处理器数量）<br>至少需要256MB内存<br>大量的CPU／内存， （在1.4.1在4CPU的机器上已经显示有提升）</td></tr> <tr> <td width="25%" valign="top">-XX:CMSFullGCsBeforeCompaction</td> <td width="20%" valign="top">多少次后进行内存压缩</td> <td width="10%" valign="top">&nbsp;</td> <td valign="top">由于并发收集器不对内存空间进行压缩,整理,所以运行一段时间以后会产生"碎片",使得运行效率降低.此值设置运行多少次GC以后对内存空间进行压缩,整理.</td></tr> <tr> <td width="25%" valign="top">-XX:+CMSParallelRemarkEnabled</td> <td width="20%" valign="top">降低标记停顿</td> <td width="10%" valign="top">&nbsp;</td> <td valign="top">&nbsp;</td></tr> <tr> <td width="25%" valign="top">-XX+UseCMSCompactAtFullCollection</td> <td width="20%" valign="top">在FULL GC的时候， 对年老代的压缩</td> <td width="10%" valign="top">&nbsp;</td> <td valign="top">CMS是不会移动内存的， 因此， 这个非常容易产生碎片， 导致内存不够用， 因此， 内存的压缩这个时候就会被启用。 增加这个参数是个好习惯。<br>可能会影响性能,但是可以消除碎片</td></tr> <tr> <td width="25%" valign="top">-XX:+UseCMSInitiatingOccupancyOnly</td> <td width="20%" valign="top">使用手动定义初始化定义开始CMS收集</td> <td width="10%" valign="top">&nbsp;</td> <td valign="top">禁止hostspot自行触发CMS GC</td></tr> <tr> <td width="25%" valign="top">-XX:CMSInitiatingOccupancyFraction=70</td> <td width="20%" valign="top">使用cms作为垃圾回收<br>使用70％后开始CMS收集</td> <td width="10%" valign="top">92</td> <td valign="top">为了保证不出现promotion failed(见下面介绍)错误,该值的设置需要满足以下公式<a href="#CMSInitiatingOccupancyFraction_value">CMSInitiatingOccupancyFraction计算公式</a></td></tr> <tr> <td width="25%" valign="top">-XX:CMSInitiatingPermOccupancyFraction</td> <td width="20%" valign="top">设置Perm Gen使用到达多少比率时触发</td> <td width="10%" valign="top">92</td> <td valign="top">&nbsp;</td></tr> <tr> <td width="25%" valign="top">-XX:+CMSIncrementalMode</td> <td width="20%" valign="top">设置为增量模式 </td> <td width="10%" valign="top">&nbsp;</td> <td valign="top">用于单CPU情况</td></tr> <tr> <td width="25%" valign="top">-XX:+CMSClassUnloadingEnabled</td> <td width="20%" valign="top">&nbsp;</td> <td width="10%" valign="top">&nbsp;</td> <td valign="top">&nbsp;</td></tr></tbody></table>

**4.辅助信息：**

<table cellspacing="0" border="1"> <tbody> <tr> <td width="25%" valign="top">-XX:+PrintGC</td> <td width="20%" valign="top">&nbsp;</td> <td width="10%" valign="top">&nbsp;</td> <td valign="top"> <p>输出形式:</p> <p>[GC 118250K-&gt;113543K(130112K), 0.0094143 secs]<br>[Full GC 121376K-&gt;10414K(130112K), 0.0650971 secs]</p></td></tr> <tr> <td width="25%" valign="top">-XX:+PrintGCDetails</td> <td width="20%" valign="top">&nbsp;</td> <td width="10%" valign="top">&nbsp;</td> <td valign="top"> <p>输出形式:[GC [DefNew: 8614K-&gt;781K(9088K), 0.0123035 secs] 118250K-&gt;113543K(130112K), 0.0124633 secs]<br>[GC [DefNew: 8614K-&gt;8614K(9088K), 0.0000665 secs][Tenured: 112761K-&gt;10414K(121024K), 0.0433488 secs] 121376K-&gt;10414K(130112K), 0.0436268 secs]</p></td></tr> <tr> <td width="25%" valign="top">-XX:+PrintGCTimeStamps</td> <td width="20%" valign="top">&nbsp;</td> <td width="10%" valign="top">&nbsp;</td> <td valign="top">&nbsp;</td></tr> <tr> <td width="25%" valign="top">-XX:+PrintGC:PrintGCTimeStamps</td> <td width="20%" valign="top">&nbsp;</td> <td width="10%" valign="top">&nbsp;</td> <td valign="top">可与-XX:+PrintGC -XX:+PrintGCDetails混合使用<br>输出形式:11.851: [GC 98328K-&gt;93620K(130112K), 0.0082960 secs]</td></tr> <tr> <td width="25%" valign="top">-XX:+PrintGCApplicationStoppedTime</td><td width="20%" valign="top">打印垃圾回收期间程序暂停的时间.可与上面混合使用</td> <td width="10%" valign="top">&nbsp;</td> <td valign="top">输出形式:Total time for which application threads were stopped: 0.0468229 seconds</td></tr> <tr> <td width="25%" valign="top">-XX:+PrintGCApplicationConcurrentTime</td> <td width="20%" valign="top">打印每次垃圾回收前,程序未中断的执行时间.可与上面混合使用</td> <td width="10%" valign="top">&nbsp;</td> <td valign="top">输出形式:Application time: 0.5291524 seconds</td></tr> <tr> <td width="25%" valign="top">-XX:+PrintHeapAtGC</td> <td width="20%" valign="top">打印GC前后的详细堆栈信息</td> <td width="10%" valign="top">&nbsp;</td> <td valign="top">&nbsp;</td></tr> <tr> <td width="25%" valign="top">-Xloggc:filename</td> <td width="20%" valign="top">把相关日志信息记录到文件以便分析.<br>与上面几个配合使用</td> <td width="10%" valign="top">&nbsp;</td> <td valign="top">&nbsp;</td></tr> <tr> <td width="25%" valign="top"> <p>-XX:+PrintClassHistogram</p></td> <td width="20%" valign="top">garbage collects before printing the histogram.</td> <td width="10%" valign="top">&nbsp;</td> <td valign="top">&nbsp;</td></tr> <tr> <td width="25%" valign="top">-XX:+PrintTLAB</td> <td width="20%" valign="top">查看TLAB空间的使用情况</td> <td width="10%" valign="top">&nbsp;</td> <td valign="top">&nbsp;</td></tr> <tr> <td width="25%" valign="top">XX:+PrintTenuringDistribution</td> <td width="20%" valign="top">查看每次minor GC后新的存活周期的阈值</td> <td width="10%" valign="top">&nbsp;</td> <td valign="top"> <p>Desired survivor size 1048576 bytes, new threshold 7 (max 15)<br>new threshold 7即标识新的存活周期的阈值为7。</p></td></tr></tbody></table>

##### 2、JVM调优过程总结

JVM调优主要过程有：确定内存大小（-Xmx、-Xms）、合理分配新生代和老年代（-XX:NewRatio、-Xmn、-XX:SurvivorRatio）、确定永久区大小（-XX:Permsize、-XX:MaxPermSize）、选着垃圾收集器、对垃圾收集器进行合理参数设置。除此外，还可以禁用显示GC(-XX:+DisableExplicitGC),类元数据回收（-Xnoclassgc）以及类验证器等设置。

1.年轻代大小选择

	响应时间优先的应用：尽可能设大，直到接近系统的最低响应时间限制（根据实际情况选择）。在此种情况下，年轻代收集发生的频率也是最小的。同时，减少到达年老代的对象。
    吞吐量优先的应用：尽可能的设置大，可能到达Gbit的程度。因为对响应时间没有要求，垃圾收集可以并行进行，一般适合8CPU以上的应用。
	避免设置过小：当新生代设置过小时会导致:1.YGC次数更加频繁 2.可能导致YGC对象直接进入旧生代,如果此时旧生代满了,会触发FGC.

2.年老代大小选择

	响应时间优先的应用：年老代使用并发收集器，所以其大小需要小心设置，一般要考虑并发会话率和会话持续时间等一些参数。如果堆设置小了，可以会造成内存碎片、高回收频率以及应用暂停而使用传统的标记清除方式；如果堆大了，则需要较长的收集时间。最优化的方案，一般需要参考以下数据获得：并发垃圾收集信息、持久代并发收集次数、传统GC信息、花在年轻代和年老代回收上的时间比例。减少年轻代和年老代花费的时间，一般会提高应用的效率
	吞吐量优先的应用:一般吞吐量优先的应用都有一个很大的年轻代和一个较小的年老代.原因是,这样可以尽可能回收掉大部分短期对象,减少中期的对象,而年老代尽存放长期存活对象.

3.较小堆引起的碎片问题

    因为年老代的并发收集器使用标记、清除算法，所以不会对堆进行压缩。当收集器回收时，他会把相邻的空间进行合并，这样可以分配给较大的对象。但是，当堆空间较小时，运行一段时间以后，就会出现“碎片”，如果并发收集器找不到足够的空间，那么并发收集器将会停止，然后使用传统的标记、清除方式进行回收。如果出现“碎片”，可能需要进行如下配置：
	-XX:+UseCMSCompactAtFullCollection：使用并发收集器时，开启对年老代的压缩。
	-XX:CMSFullGCsBeforeCompaction=0：上面配置开启的情况下，这里设置多少次Full GC后，对年老代进行压缩

4.JVM的内存限制

	JVM中最大堆大小有三方面限制：相关操作系统的数据模型（32-bt还是64-bit）限制；系统的可用虚拟内存限制；系统的可用物理内存限制。32位系统下，一般限制在1.5G~2G；64为操作系统对内存无限制。

5.其他经验总结

	用64位操作系统，Linux下64位的jdk比32位jdk要慢一些，但是吃得内存更多，吞吐量更大。XMX和XMS设置一样大，MaxPermSize和MinPermSize设置一样大，这样可以减轻伸缩堆大小带来的压力
	使用CMS的好处是用尽量少的新生代，经验值是128M－256M， 然后老生代利用CMS并行收集， 这样能保证系统低延迟的吞吐效率。 实际上cms的收集停顿时间非常的短，2G的内存， 大约20－80ms的应用程序停顿时间
	系统停顿的时候可能是GC的问题也可能是程序的问题，多用jmap和jstack查看，或者killall -3 java，然后查看java控制台日志，能看出很多问题。(相关工具的使用方法将在后面的blog中介绍)
	仔细了解自己的应用，如果用了缓存，那么年老代应该大一些，缓存的HashMap不应该无限制长，建议采用LRU算法的Map做缓存，LRUMap的最大长度也要根据实际情况设定。
	采用并发回收时，年轻代小一点，年老代要大，因为年老大用的是并发回收，即使时间长点也不会影响其他程序继续运行，网站不会停顿
	JVM参数的设置(特别是 –Xmx –Xms –Xmn -XX:SurvivorRatio  -XX:MaxTenuringThreshold等参数的设置没有一个固定的公式，需要根据PV old区实际数据 YGC次数等多方面来衡量。为了避免promotion faild可能会导致xmn设置偏小，也意味着YGC的次数会增多，处理并发访问的能力下降等问题。每个参数的调整都需要经过详细的性能测试，才能找到特定应用的最佳配置。

##### 3、JVM调优实例


