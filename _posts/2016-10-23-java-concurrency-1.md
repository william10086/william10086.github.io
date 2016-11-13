---
layout: post
title:  "Java 并发编程"
categories: Java IO
tags:   NIO
---

* content
{:toc}


#### Java 并发编程（一）





##### 1、并发编程优缺点
**优点：**

- 资源利用率更好
	想象一下，一个应用程序需要从本地文件系统中读取和处理文件的情景。比方说，从磁盘读取一个文件需要5秒，处理一个文件需要2秒。处理两个文件则需要：

	```
	5秒读取文件A
	2秒处理文件A
	5秒读取文件B
	2秒处理文件B
	---------------------
	总共需要14秒
	```

	从磁盘中读取文件的时候，大部分的CPU时间用于等待磁盘去读取数据。在这段时间里，CPU非常的空闲。它可以做一些别的事情。通过改变操作的顺序，就能够更好的使用CPU资源。看下面的顺序：

	```
	5秒读取文件A
	5秒读取文件B + 2秒处理文件A
	2秒处理文件B
	---------------------
	总共需要12秒
	```

	CPU等待第一个文件被读取完。然后开始读取第二个文件。当第二文件在被读取的时候，CPU会去处理第一个文件。记住，在等待磁盘读取文件的时候，CPU大部分时间是空闲的。
	总的说来，CPU能够在等待IO的时候做一些其他的事情。这个不一定就是磁盘IO。它也可以是网络的IO，或者用户输入。通常情况下，网络和磁盘的IO比CPU和内存的IO慢的多。

- 程序设计在某些情况下更简单
	在单线程应用程序中，如果你想编写程序手动处理上面所提到的读取和处理的顺序，你必须记录每个文件读取和处理的状态。相反，你可以启动两个线程，每个线程处理一个文件的读取和操作。线程会在等待磁盘读取文件的过程中被阻塞。在等待的时候，其他的线程能够使用CPU去处理已经读取完的文件。其结果就是，磁盘总是在繁忙地读取不同的文件到内存中。这会带来磁盘和CPU利用率的提升。而且每个线程只需要记录一个文件，因此这种方式也很容易编程实现。

- 程序响应更快
	将一个单线程应用程序变成多线程应用程序的另一个常见的目的是实现一个响应更快的应用程序。设想一个服务器应用，它在某一个端口监听进来的请求。当一个请求到来时，它去处理这个请求，然后再返回去监听。
	如果一个请求需要占用大量的时间来处理，在这段时间内新的客户端就无法发送请求给服务端。只有服务器在监听的时候，请求才能被接收。另一种设计是，监听线程把请求传递给工作者线程(worker thread)，然后立刻返回去监听。而工作者线程则能够处理这个请求并发送一个回复给客户端。这种方式，服务端线程迅速地返回去监听。因此，更多的客户端能够发送请求给服务端。这个服务也变得响应更快。
	桌面应用也是同样如此。如果你点击一个按钮开始运行一个耗时的任务，这个线程既要执行任务又要更新窗口和按钮，那么在任务执行的过程中，这个应用程序看起来好像没有反应一样。相反，任务可以传递给工作者线程（word thread)。当工作者线程在繁忙地处理任务的时候，窗口线程可以自由地响应其他用户的请求。当工作者线程完成任务的时候，它发送信号给窗口线程。窗口线程便可以更新应用程序窗口，并显示任务的结果。对用户而言，这种具有工作者线程设计的程序显得响应速度更快。

**缺点：**

- 设计更复杂
	虽然有一些多线程应用程序比单线程的应用程序要简单，但其他的一般都更复杂。在多线程访问共享数据的时候，这部分代码需要特别的注意。线程之间的交互往往非常复杂。不正确的线程同步产生的错误非常难以被发现，并且重现以修复。

- 上下文切换的开销
	当CPU从执行一个线程切换到执行另外一个线程的时候，它需要先存储当前线程的本地的数据，程序指针等，然后载入另一个线程的本地数据，程序指针等，最后才开始执行。这种切换称为“上下文切换”(“context switch”)。CPU会在一个上下文中执行一个线程，然后切换到另外一个上下文中执行另外一个线程。

- 增加资源消耗
	线程在运行的时候需要从计算机里面得到一些资源。除了CPU，线程还需要一些内存来维持它本地的堆栈。它也需要占用操作系统中一些资源来管理线程。我们可以尝试编写一个程序，让它创建100个线程，这些线程什么事情都不做，只是在等待，然后看看这个程序在运行的时候占用了多少内存。

##### 2、并发编程模型
并发系统可以采用多种并发编程模型来实现。并发模型指定了系统中的线程如何通过协作来完成分配给它们的作业。不同的并发模型采用不同的方式拆分作业，同时线程间的协作和交互方式也不相同。这篇并发模型教程将会较深入地介绍目前（2015年，本文撰写时间）比较流行的几种并发模型。

1）**并发模型与分布式系统之间的相似性**

本文所描述的并发模型类似于分布式系统中使用的很多体系结构。在并发系统中线程之间可以相互通信。在分布式系统中进程之间也可以相互通信（进程有可能在不同的机器中）。线程和进程之间具有很多相似的特性。这也就是为什么很多并发模型通常类似于各种分布式系统架构。
当然，分布式系统在处理网络失效、远程主机或进程宕掉等方面也面临着额外的挑战。但是运行在巨型服务器上的并发系统也可能遇到类似的问题，比如一块CPU失效、一块网卡失效或一个磁盘损坏等情况。虽然出现失效的概率可能很低，但是在理论上仍然有可能发生。
由于并发模型类似于分布式系统架构，因此它们通常可以互相借鉴思想。例如，为工作者们（线程）分配作业的模型一般与分布式系统中的负载均衡系统比较相似。同样，它们在日志记录、失效转移、幂等性等错误处理技术上也具有相似性。

> 注：幂等性，一个幂等操作的特点是其任意多次执行所产生的影响均与一次执行的影响相同

2）**并行工作者模型**

第一种并发模型就是我所说的并行工作者模型。传入的作业会被分配到不同的工作者上。下图展示了并行工作者模型：

![delegator-worker](http://tutorials.jenkov.com/images/java-concurrency/concurrency-models-1.png)

在并行工作者模型中，委派者（Delegator）将传入的作业分配给不同的工作者。每个工作者完成整个任务。工作者们并行运作在不同的线程上，甚至可能在不同的CPU上。
在Java应用系统中，并行工作者模型是最常见的并发模型（即使正在转变）。java.util.concurrent包中的许多并发实用工具都是设计用于这个模型的。你也可以在Java企业级（J2EE）应用服务器的设计中看到这个模型的踪迹。

- 并行工作者模型的优点
	并行工作者模式的优点是，它很容易理解。你只需添加更多的工作者来提高系统的并行度。
	例如，如果你正在做一个网络爬虫，可以试试使用不同数量的工作者抓取到一定数量的页面，然后看看多少数量的工作者消耗的时间最短（意味着性能最高）。由于网络爬虫是一个IO密集型工作，最终结果很有可能是你电脑中的每个CPU或核心分配了几个线程。每个CPU若只分配一个线程可能有点少，因为在等待数据下载的过程中CPU将会空闲大量时间。

- 并行工作者模型的缺点
	`共享状态可能会很复杂` 在实际应用中，并行工作者模型可能比前面所描述的情况要复杂得多。共享的工作者经常需要访问一些共享数据，无论是内存中的或者共享的数据库中的。下图展示了并行工作者模型是如何变得复杂的：

	![delegator-dis](http://tutorials.jenkov.com/images/java-concurrency/concurrency-models-2.png)

	有些共享状态是在像作业队列这样的通信机制下。但也有一些共享状态是业务数据，数据缓存，数据库连接池等。
	一旦共享状态潜入到并行工作者模型中，将会使情况变得复杂起来。线程需要以某种方式存取共享数据，以确保某个线程的修改能够对其他线程可见（数据修改需要同步到主存中，不仅仅将数据保存在执行这个线程的CPU的缓存中）。线程需要避免竟态，死锁以及很多其他共享状态的并发性问题。

	现在的非阻塞并发算法也许可以降低竞争并提升性能，但是非阻塞算法的实现比较困难。

	可持久化的数据结构是另一种选择。在修改的时候，可持久化的数据结构总是保护它的前一个版本不受影响。因此，如果多个线程指向同一个可持久化的数据结构，并且其中一个线程进行了修改，进行修改的线程会获得一个指向新结构的引用。所有其他线程保持对旧结构的引用，旧结构没有被修改并且因此保证一致性。Scala编程包含几个持久化数据结构。

	> 注：这里的可持久化数据结构不是指持久化存储，而是一种数据结构，比如Java中的String类，以及CopyOnWriteArrayList类，具体可[参考](http://www.cnblogs.com/tedzhao/archive/2008/11/12/1332112.html)

	虽然可持久化的数据结构在解决共享数据结构的并发修改时显得很优雅，但是可持久化的数据结构的表现往往不尽人意。
	比如说，一个可持久化的链表需要在头部插入一个新的节点，并且返回指向这个新加入的节点的一个引用（这个节点指向了链表的剩余部分）。所有其他现场仍然保留了这个链表之前的第一个节点，对于这些线程来说链表仍然是为改变的。它们无法看到新加入的元素。
	这种可持久化的列表采用链表来实现。不幸的是链表在现代硬件上表现的不太好。链表中得每个元素都是一个独立的对象，这些对象可以遍布在整个计算机内存中。现代CPU能够更快的进行顺序访问，所以你可以在现代的硬件上用数组实现的列表，以获得更高的性能。数组可以顺序的保存数据。CPU缓存能够一次加载数组的一大块进行缓存，一旦加载完成CPU就可以直接访问缓存中的数据。这对于元素散落在RAM中的链表来说，不太可能做得到。

	`无状态的工作者` 共享状态能够被系统中得其他线程修改。所以工作者在每次需要的时候必须重读状态，以确保每次都能访问到最新的副本，不管共享状态是保存在内存中的还是在外部数据库中。工作者无法在内部保存这个状态（但是每次需要的时候可以重读）称为无状态的。每次都重读需要的数据，将会导致速度变慢，特别是状态保存在外部数据库中的时候。

	`任务顺序是不确定的` 并行工作者模式的另一个缺点是，作业执行顺序是不确定的。无法保证哪个作业最先或者最后被执行。作业A可能在作业B之前就被分配工作者了，但是作业B反而有可能在作业A之前执行。并行工作者模式的这种非确定性的特性，使得很难在任何特定的时间点推断系统的状态。这也使得它也更难（如果不是不可能的话）保证一个作业在其他作业之前被执行。

3）**流水线模式**

第二种并发模型我们称之为流水线并发模型。我之所以选用这个名字，只是为了配合“并行工作者”的隐喻。其他开发者可能会根据平台或社区选择其他称呼（比如说反应器系统，或事件驱动系统）。下图表示一个流水线并发模型：

![delegator-flow](http://tutorials.jenkov.com/images/java-concurrency/concurrency-models-3.png)

类似于工厂中生产线上的工人们那样组织工作者。每个工作者只负责作业中的部分工作。当完成了自己的这部分工作时工作者会将作业转发给下一个工作者。每个工作者在自己的线程中运行，并且不会和其他工作者共享状态。有时也被成为 *无共享并行模型*。

通常使用非阻塞的IO来设计使用流水线并发模型的系统。非阻塞IO意味着，一旦某个工作者开始一个IO操作的时候（比如读取文件或从网络连接中读取数据），这个工作者不会一直等待IO操作的结束。IO操作速度很慢，所以等待IO操作结束很浪费CPU时间。此时CPU可以做一些其他事情。当IO操作完成的时候，IO操作的结果（比如读出的数据或者数据写完的状态）被传递给下一个工作者。
有了非阻塞IO，就可以使用IO操作确定工作者之间的边界。工作者会尽可能多运行直到遇到并启动一个IO操作。然后交出作业的控制权。当IO操作完成的时候，在流水线上的下一个工作者继续进行操作，直到它也遇到并启动一个IO操作。

![flow-worker](http://tutorials.jenkov.com/images/java-concurrency/concurrency-models-4.png)

在实际应用中，作业有可能不会沿着单一流水线进行。由于大多数系统可以执行多个作业，作业从一个工作者流向另一个工作者取决于作业需要做的工作。在实际中可能会有多个不同的虚拟流水线同时运行。这是现实当中作业在流水线系统中可能的移动情况：

![delegator-flow-worker-1](http://tutorials.jenkov.com/images/java-concurrency/concurrency-models-5.png)

作业甚至也有可能被转发到超过一个工作者上并发处理。比如说，作业有可能被同时转发到作业执行器和作业日志器。下图说明了三条流水线是如何通过将作业转发给同一个工作者（中间流水线的最后一个工作者）来完成作业:

![delegator-flow-worker-2](http://tutorials.jenkov.com/images/java-concurrency/concurrency-models-6.png)

采用流水线并发模型的系统有时候也称为**反应器系统或事件驱动系统**。系统内的工作者对系统内出现的事件做出反应，这些事件也有可能来自于外部世界或者发自其他工作者。事件可以是传入的HTTP请求，也可以是某个文件成功加载到内存中等。在写这篇文章的时候，已经有很多有趣的反应器/事件驱动平台可以使用了，并且不久的将来会有更多。比较流行的似乎是这几个：

> [Vert.x](http://tutorials.jenkov.com/vert.x/index.html)
> AKKa
> Node.JS(JavaScript)

**Actors和channels** 是两种比较类似的流水线（或反应器/事件驱动）模型。在Actor模型中每个工作者被称为actor。Actor之间可以直接异步地发送和处理消息。Actor可以被用来实现一个或多个像前文描述的那样的作业处理流水线。下图给出了Actor模型：

![actors](http://tutorials.jenkov.com/images/java-concurrency/concurrency-models-7.png)

而在Channel模型中，工作者之间不直接进行通信。相反，它们在不同的通道中发布自己的消息（事件）。其他工作者们可以在这些通道上监听消息，发送者无需知道谁在监听。下图给出了Channel模型：

![channels](http://tutorials.jenkov.com/images/java-concurrency/concurrency-models-8.png)

在写这篇文章的时候，channel模型对于我来说似乎更加灵活。一个工作者无需知道谁在后面的流水线上处理作业。只需知道作业（或消息等）需要转发给哪个通道。通道上的监听者可以随意订阅或者取消订阅，并不会影响向这个通道发送消息的工作者。这使得工作者之间具有松散的耦合。

- 流水线模型的优点
	`无需共享的状态` 工作者之间无需共享状态，意味着实现的时候无需考虑所有因并发访问共享对象而产生的并发性问题。这使得在实现工作者的时候变得非常容易。在实现工作者的时候就好像是单个线程在处理工作-基本上是一个单线程的实现。

	`有状态的工作者` 当工作者知道了没有其他线程可以修改它们的数据，工作者可以变成有状态的。对于有状态，我是指，它们可以在内存中保存它们需要操作的数据，只需在最后将更改写回到外部存储系统。因此，有状态的工作者通常比无状态的工作者具有更高的性能。

	`较好的硬件整合（Hardware Conformity）` 单线程代码在整合底层硬件的时候往往具有更好的优势。首先，当能确定代码只在单线程模式下执行的时候，通常能够创建更优化的数据结构和算法。
	其次，像前文描述的那样，单线程有状态的工作者能够在内存中缓存数据。在内存中缓存数据的同时，也意味着数据很有可能也缓存在执行这个线程的CPU的缓存中。这使得访问缓存的数据变得更快。
	我说的硬件整合是指，以某种方式编写的代码，使得能够自然地受益于底层硬件的工作原理。有些开发者称之为mechanical sympathy。我更倾向于硬件整合这个术语，因为计算机只有很少的机械部件，并且能够隐喻“更好的匹配（match better）”，相比“同情（sympathy）”这个词在上下文中的意思，我觉得“conform”这个词表达的非常好。当然了，这里有点吹毛求疵了，用自己喜欢的术语就行。

	`合理的作业顺序` 基于流水线并发模型实现的并发系统，在某种程度上是有可能保证作业的顺序的。作业的有序性使得它更容易地推出系统在某个特定时间点的状态。更进一步，你可以将所有到达的作业写入到日志中去。一旦这个系统的某一部分挂掉了，该日志就可以用来重头开始重建系统当时的状态。按照特定的顺序将作业写入日志，并按这个顺序作为有保障的作业顺序。下图展示了一种可能的设计：

	![worker-channel](http://tutorials.jenkov.com/images/java-concurrency/concurrency-models-8.png)

	实现一个有保障的作业顺序是不容易的，但往往是可行的。如果可以，它将大大简化一些任务，例如备份、数据恢复、数据复制等，这些都可以通过日志文件来完成。

- 流水线模型的缺点
	流水线并发模型最大的缺点是作业的执行往往分布到多个工作者上，并因此分布到项目中的多个类上。这样导致在追踪某个作业到底被什么代码执行时变得困难。
	同样，这也加大了代码编写的难度。有时会将工作者的代码写成回调处理的形式。若在代码中嵌入过多的回调处理，往往会出现所谓的回调地狱（callback hell）现象。所谓回调地狱，就是意味着在追踪代码在回调过程中到底做了什么，以及确保每个回调只访问它需要的数据的时候，变得非常困难。
	使用并行工作者模型可以简化这个问题。你可以打开工作者的代码，从头到尾优美的阅读被执行的代码。当然并行工作者模式的代码也可能同样分布在不同的类中，但往往也能够很容易的从代码中分析执行的顺序。

4）**函数式并行（Functional Parallelism）**

第三种并发模型是函数式并行模型，这是也最近（2015）讨论的比较多的一种模型。函数式并行的基本思想是采用函数调用实现程序。函数可以看作是”代理人（agents）“或者”actor“，函数之间可以像流水线模型（AKA 反应器或者事件驱动系统）那样互相发送消息。某个函数调用另一个函数，这个过程类似于消息发送。
函数都是通过拷贝来传递参数的，所以除了接收函数外没有实体可以操作数据。这对于避免共享数据的竞态来说是很有必要的。同样也使得函数的执行类似于原子操作。每个函数调用的执行独立于任何其他函数的调用。
一旦每个函数调用都可以独立的执行，它们就可以分散在不同的CPU上执行了。这也就意味着能够在多处理器上并行的执行使用函数式实现的算法。

Java7中的java.util.concurrent包里包含的[ForkAndJoinPool](http://tutorials.jenkov.com/java-util-concurrent/java-fork-and-join-forkjoinpool.html)能够帮助我们实现类似于函数式并行的一些东西。而Java8中并行[Streams](http://tutorials.jenkov.com/java-collections/streams.html)能够用来帮助我们并行的迭代大型集合。记住有些开发者对ForkAndJoinPool进行了批判（你可以在我的ForkAndJoinPool教程里面看到批评的链接）。
函数式并行里面最难的是确定需要并行的那个函数调用。跨CPU协调函数调用需要一定的开销。某个函数完成的工作单元需要达到某个大小以弥补这个开销。如果函数调用作用非常小，将它并行化可能比单线程、单CPU执行还慢。

> 有人认为（可能不太正确），你可以使用反应器或者事件驱动模型实现一个算法，像函数式并行那样的方法实现工作的分解。使用事件驱动模型可以更精确的控制如何实现并行化（我的观点）。	？待考虑？

此外，将任务拆分给多个CPU时协调造成的开销，仅仅在该任务是程序当前执行的唯一任务时才有意义。但是，如果当前系统正在执行多个其他的任务时（比如web服务器，数据库服务器或者很多其他类似的系统），将单个任务进行并行化是没有意义的。不管怎样计算机中的其他CPU们都在忙于处理其他任务，没有理由用一个慢的、函数式并行的任务去扰乱它们。使用流水线（反应器）并发模型可能会更好一点，因为它开销更小（在单线程模式下顺序执行）同时能更好的与底层硬件整合。

**附：使用那种并发模型最好？**

通常情况下，这个答案取决于你的系统打算做什么。如果你的作业本身就是并行的、独立的并且没有必要共享状态，你可能会使用并行工作者模型去实现你的系统。虽然许多作业都不是自然并行和独立的。对于这种类型的系统，我相信使用流水线并发模型能够更好的发挥它的优势，而且比并行工作者模型更有优势。
你甚至不用亲自编写所有流水线模型的基础结构。像Vert.x这种现代化的平台已经为你实现了很多。我也会去为探索如何设计我的下一个项目，使它运行在像Vert.x这样的优秀平台上。我感觉Java EE已经没有任何优势了。

##### 3、并发编程相关概念

1） **竞态条件与临界区**

当两个线程竞争同一资源时，如果对资源的访问顺序敏感，就称存在竞态条件。导致竞态条件发生的代码区称作临界区。上例中add()方法就是一个临界区,它会产生竞态条件。在临界区中使用适当的同步就可以避免竞态条件。
在同一程序中运行多个线程本身不会导致问题，问题在于多个线程访问了相同的资源。如，同一内存区（变量，数组，或对象）、系统（数据库，web services等）或文件。实际上，这些问题只有在一或多个线程向这些资源做了写操作时才有可能发生，只要资源没有发生变化,多个线程读取相同的资源就是安全的。

```
public class Counter {
	protected long count = 0;
	public void add(long value){
		this.count = this.count + value;
	}
}
```

JVM并不是将这段代码视为单条指令来执行的，线程A和B同时执行同一个Counter对象的add()方法，我们无法知道操作系统何时会在两个线程之间切换，线程间的交叉执行情况就会发生。

2） **线程安全与共享资源**

允许被多个线程同时执行的代码称作线程安全的代码。线程安全的代码不包含竞态条件。当多个线程同时更新共享资源时会引发竞态条件。因此，了解Java线程执行时共享了什么资源很重要。
`局部变量` 局部变量存储在线程自己的栈中。也就是说，局部变量永远也不会被多个线程共享。所以，基础类型的局部变量是线程安全的。例如：

```
public void someMethod(){
  long threadSafeInt = 0;
  threadSafeInt++;
}
```

`局部的对象引用` 对象的局部引用和基础类型的局部变量不太一样。尽管引用本身没有被共享，但引用所指的对象并没有存储在线程的栈内。所有的对象都存在共享堆中。如果在某个方法中创建的对象不会逃逸出（译者注：即该对象不会被其它方法获得，也不会被非局部变量引用到）该方法，那么它就是线程安全的。实际上，哪怕将这个对象作为参数传给其它方法，只要别的线程获取不到这个对象，那它仍是线程安全的。例如：

```
public void someMethod(){
  LocalObject localObject = new LocalObject();

  localObject.callMethod();
  method2(localObject);
}

public void method2(LocalObject localObject){
  localObject.setValue("value");
}
```

样例中LocalObject对象没有被方法返回，也没有被传递给someMethod()方法外的对象。每个执行someMethod()的线程都会创建自己的LocalObject对象，并赋值给localObject引用。因此，这里的LocalObject是线程安全的。事实上，整个someMethod()都是线程安全的。即使将LocalObject作为参数传给同一个类的其它方法或其它类的方法时，它仍然是线程安全的。当然，如果LocalObject通过某些方法被传给了别的线程，那它就不再是线程安全的了。
`对象成员` 对象成员存储在堆上。如果两个线程同时更新同一个对象的同一个成员，那这个代码就不是线程安全的。例如：

```
public class NotThreadSafe{
    StringBuilder builder = new StringBuilder();
    public add(String text){
        this.builder.append(text);
    }
}
```

如果两个线程同时调用同一个NotThreadSafe实例上的add()方法，就会有竞态条件问题。当然，如果这两个线程在不同的NotThreadSafe实例上调用call()方法，就不会导致竞态条件。

```
//线程不安全
NotThreadSafe sharedInstance = new NotThreadSafe();
new Thread(new MyRunnable(sharedInstance)).start();
new Thread(new MyRunnable(sharedInstance)).start();
//线程安全
new Thread(new MyRunnable(new NotThreadSafe())).start();
new Thread(new MyRunnable(new NotThreadSafe())).start();
```

`线程控制逃逸规则` 线程控制逃逸规则可以帮助你判断代码中对某些资源的访问是否是线程安全的。

> 如果一个资源的创建，使用，销毁都在同一个线程内完成，且永远不会脱离该线程的控制，则该资源的使用就是线程安全的。

资源可以是对象，数组，文件，数据库连接，套接字等等。Java中你无需主动销毁对象，所以“销毁”指不再有引用指向对象。
即使对象本身线程安全，但如果该对象中包含其他资源（文件，数据库连接），整个应用也许就不再是线程安全的了。比如2个线程都创建了各自的数据库连接，每个连接自身是线程安全的，但它们所连接到的同一个数据库也许不是线程安全的。比如，2个线程执行如下代码：

> 检查记录X是否存在，如果不存在，插入X

如果两个线程同时执行，而且碰巧检查的是同一个记录，那么两个线程最终可能都插入了记录：

> 线程1检查记录X是否存在。检查结果：不存在
> 线程2检查记录X是否存在。检查结果：不存在
> 线程1插入记录X
> 线程2插入记录X

同样的问题也会发生在文件或其他共享资源上。因此，区分某个线程控制的对象是资源本身，还是仅仅到某个资源的引用很重要。

3） **线程安全及不可变性**

当多个线程同时访问同一个资源，并且其中的一个或者多个线程对这个资源进行了写操作，才会产生竞态条件。多个线程同时读同一个资源不会产生竞态条件。我们可以通过创建不可变的共享对象来保证对象在线程间共享时不会被修改，从而实现线程安全。例如：

```
public class ImmutableValue{
	private int value = 0;

	public ImmutableValue(int value){
		this.value = value;
	}

	public int getValue(){
		return this.value;
	}
}
```

请注意ImmutableValue类的成员变量value是通过构造函数赋值的，并且在类中没有set方法。这意味着一旦ImmutableValue实例被创建，value变量就不能再被修改，这就是不可变性。但你可以通过getValue()方法读取这个变量的值。

> （译者注：注意，“不变”（Immutable）和“只读”（Read Only）是不同的。当一个变量是“只读”时，变量的值不能直接改变，但是可以在其它变量发生改变的时候发生改变。比如，一个人的出生年月日是“不变”属性，而一个人的年龄便是“只读”属性，但是不是“不变”属性。随着时间的变化，一个人的年龄会随之发生变化，而一个人的出生年月日则不会变化。这就是“不变”和“只读”的区别。（摘自《Java与模式》第34章））

如果你需要对ImmutableValue类的实例进行操作，可以通过得到value变量后创建一个新的实例来实现，下面是一个对value变量进行加法操作的示例：

```
public class ImmutableValue{
	private int value = 0;

	public ImmutableValue(int value){
		this.value = value;
	}

	public int getValue(){
		return this.value;
	}

	public ImmutableValue add(int valueToAdd){
    	//new 像是一次原子操作？
		return new ImmutableValue(this.value + valueToAdd);
	}
}
```

请注意add()方法以加法操作的结果作为一个新的ImmutableValue类实例返回，而不是直接对它自己的value变量进行操作。

`引用不是线程安全的！` 重要的是要记住，即使一个对象是线程安全的不可变对象，指向这个对象的引用也可能不是线程安全的。

```
public void Calculator{
	private ImmutableValue currentValue = null;

	public ImmutableValue getValue(){
		return currentValue;
	}

	public void setValue(ImmutableValue newValue){
		this.currentValue = newValue;
	}

	public void add(int newValue){
		this.currentValue = this.currentValue.add(newValue);
	}
}
```

Calculator类持有一个指向ImmutableValue实例的引用。注意，通过setValue()方法和add()方法可能会改变这个引用。因此，即使Calculator类内部使用了一个不可变对象，但Calculator类本身还是可变的，因此Calculator类不是线程安全的。换句话说：ImmutableValue类是线程安全的，但使用它的类不是。当尝试通过不可变性去获得线程安全时，这点是需要牢记的。
要使Calculator类实现线程安全，将getValue()、setValue()和add()方法都声明为同步方法即可。

