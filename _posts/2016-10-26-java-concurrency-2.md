---
layout: post
title:  "Java 并发编程"
categories: Java IO
tags:   NIO
---

* content
{:toc}


#### Java 并发编程（二）





##### 1、JAVA并发编程相关
1) **Java内存模型内部原理**

Java内存模型把Java虚拟机内部划分为线程栈和堆。这张图演示了Java内存模型的逻辑视图。

![java-memory](http://tutorials.jenkov.com/images/java-concurrency/java-memory-model-1.png)

每一个运行在Java虚拟机里的线程都拥有自己的线程栈。这个线程栈包含了这个线程调用的方法当前执行点相关的信息。一个线程仅能访问自己的线程栈。一个线程创建的本地变量对其它线程不可见，仅自己可见。即使两个线程执行同样的代码，这两个线程任然在在自己的线程栈中的代码来创建本地变量。因此，每个线程拥有每个本地变量的独有版本。

所有原始类型的本地变量都存放在线程栈上，因此对其它线程不可见。一个线程可能向另一个线程传递一个原始类型变量的拷贝，但是它不能共享这个原始类型变量自身。堆上包含在Java程序中创建的所有对象，无论是哪一个对象创建的。这包括原始类型的对象版本。如果一个对象被创建然后赋值给一个局部变量，或者用来作为另一个对象的成员变量，这个对象任然是存放在堆上。

下面这张图演示了调用栈和本地变量存放在线程栈上，对象存放在堆上。
![stack-heap](http://tutorials.jenkov.com/images/java-concurrency/java-memory-model-2.png)

一个本地变量可能是原始类型，在这种情况下，它总是“呆在”线程栈上。一个本地变量也可能是指向一个对象的一个引用。在这种情况下，引用（这个本地变量）存放在线程栈上，但是对象本身存放在堆上。一个对象可能包含方法，这些方法可能包含本地变量。这些本地变量任然存放在线程栈上，即使这些方法所属的对象存放在堆上。一个对象的成员变量可能随着这个对象自身存放在堆上。不管这个成员变量是原始类型还是引用类型。静态成员变量跟随着类定义一起也存放在堆上。

存放在堆上的对象可以被所有持有对这个对象引用的线程访问。当一个线程可以访问一个对象时，它也可以访问这个对象的成员变量。如果两个线程同时调用同一个对象上的同一个方法，它们将会都访问这个对象的成员变量，但是每一个线程都拥有这个本地变量的私有拷贝。下图演示了上面提到的点：

![](http://tutorials.jenkov.com/images/java-concurrency/java-memory-model-3.png)

两个线程拥有一些列的本地变量。其中一个本地变量（Local Variable 2）执行堆上的一个共享对象（Object 3）。这两个线程分别拥有同一个对象的不同引用。这些引用都是本地变量，因此存放在各自线程的线程栈上。这两个不同的引用指向堆上同一个对象。注意，这个共享对象（Object 3）持有Object2和Object4一个引用作为其成员变量（如图中Object3指向Object2和Object4的箭头）。通过在Object3中这些成员变量引用，这两个线程就可以访问Object2和Object4。
这张图也展示了指向堆上两个不同对象的一个本地变量。在这种情况下，指向两个不同对象的引用不是同一个对象。理论上，两个线程都可以访问Object1和Object5，如果两个线程都拥有两个对象的引用。但是在上图中，每一个线程仅有一个引用指向两个对象其中之一。

因此，什么类型的Java代码会导致上面的内存图呢？如下所示：

```
public class MyRunnable implements Runnable() {

    public void run() {
        methodOne();
    }

    public void methodOne() {
        int localVariable1 = 45;

        MySharedObject localVariable2 = MySharedObject.sharedInstance;

        //... do more with local variables.

        methodTwo();
    }

    public void methodTwo() {
        Integer localVariable1 = new Integer(99);

        //... do more with local variable.
    }
}

public class MySharedObject {
    //static variable pointing to instance of MySharedObject

    public static final MySharedObject sharedInstance = new MySharedObject();


    //member variables pointing to two objects on the heap
    public Integer object2 = new Integer(22);
    public Integer object4 = new Integer(44);

    public long member1 = 12345;
    public long member1 = 67890;
}
```

如果两个线程同时执行run()方法，就会出现上图所示的情景。run()方法调用methodOne()方法，methodOne()调用methodTwo()方法。methodOne()声明了一个原始类型的本地变量和一个引用类型的本地变量。每个线程执行methodOne()都会在它们对应的线程栈上创建localVariable1和localVariable2的私有拷贝。localVariable1变量彼此完全独立，仅“生活”在每个线程的线程栈上。一个线程看不到另一个线程对它的localVariable1私有拷贝做出的修改。每个线程执行methodOne()时也将会创建它们各自的localVariable2拷贝。然而，两个localVariable2的不同拷贝都指向堆上的同一个对象。代码中通过一个静态变量设置localVariable2指向一个对象引用。仅存在一个静态变量的一份拷贝，这份拷贝存放在堆上。因此，localVariable2的两份拷贝都指向由MySharedObject指向的静态变量的同一个实例。MySharedObject实例也存放在堆上。它对应于上图中的Object3。
*注意*，MySharedObject类也包含两个成员变量。这些成员变量随着这个对象存放在堆上。这两个成员变量指向另外两个Integer对象。这些Integer对象对应于上图中的Object2和Object4.
*注意*，methodTwo()创建一个名为localVariable的本地变量。这个成员变量是一个指向一个Integer对象的对象引用。这个方法设置localVariable1引用指向一个新的Integer实例。在执行methodTwo方法时，localVariable1引用将会在每个线程中存放一份拷贝。这两个Integer对象实例化将会被存储堆上，但是每次执行这个方法时，这个方法都会创建一个新的Integer对象，两个线程执行这个方法将会创建两个不同的Integer实例。methodTwo方法创建的Integer对象对应于上图中的Object1和Object5。
*还有一点*，MySharedObject类中的两个long类型的成员变量是原始类型的。因为，这些变量是成员变量，所以它们任然随着该对象存放在堆上，仅有本地变量存放在线程栈上。

2) **硬件内存架构**

现代硬件内存模型与Java内存模型有一些不同。理解内存模型架构以及Java内存模型如何与它协同工作也是非常重要的。这部分描述了通用的硬件内存架构，下面的部分将会描述Java内存是如何与它“联手”工作的。
下面是现代计算机硬件架构的简单图示：

![cpu-memeory](http://tutorials.jenkov.com/images/java-concurrency/java-memory-model-4.png)

一个现代计算机通常由两个或者多个CPU。其中一些CPU还有多核。从这一点可以看出，在一个有两个或者多个CPU的现代计算机上同时运行多个线程是可能的。每个CPU在某一时刻运行一个线程是没有问题的。这意味着，如果你的Java程序是多线程的，在你的Java程序中每个CPU上一个线程可能同时（并发）执行。
每个CPU都包含一系列的寄存器，它们是CPU内内存的基础。CPU在寄存器上执行操作的速度远大于在主存上执行的速度。这是因为CPU访问寄存器的速度远大于主存。每个CPU可能还有一个CPU缓存层。实际上，绝大多数的现代CPU都有一定大小的缓存层。CPU访问缓存层的速度快于访问主存的速度，但通常比访问内部寄存器的速度还要慢一点。一些CPU还有多层缓存，但这些对理解Java内存模型如何和内存交互不是那么重要。只要知道CPU中可以有一个缓存层就可以了。
一个计算机还包含一个主存。所有的CPU都可以访问主存。主存通常比CPU中的缓存大得多。通常情况下，当一个CPU需要读取主存时，它会将主存的部分读到CPU缓存中。它甚至可能将缓存中的部分内容读到它的内部寄存器中，然后在寄存器中执行操作。当CPU需要将结果写回到主存中去时，它会将内部寄存器的值刷新到缓存中，然后在某个时间点将值刷新回主存。当CPU需要在缓存层存放一些东西的时候，存放在缓存中的内容通常会被刷新回主存。CPU缓存可以在某一时刻将数据局部写到它的内存中，和在某一时刻局部刷新它的内存。它不会再某一时刻读/写整个缓存。通常，在一个被称作“cache lines”的更小的内存块中缓存被更新。一个或者多个缓存行可能被读到缓存，一个或者多个缓存行可能再被刷新回主存。

3) **Java内存模型和硬件内存架构之间的桥接**

上面已经提到，Java内存模型与硬件内存架构之间存在差异。硬件内存架构没有区分线程栈和堆。对于硬件，所有的线程栈和堆都分布在主内中。部分线程栈和堆可能有时候会出现在CPU缓存中和CPU内部的寄存器中。如下图所示：

![java-cpu](http://tutorials.jenkov.com/images/java-concurrency/java-memory-model-5.png)

当对象和变量被存放在计算机中各种不同的内存区域中时，就可能会出现一些具体的问题。主要包括如下两个方面：

- 线程对共享变量修改的可见性		----需要使用volatile关键字
- 当读，写和检查共享变量时出现竞争条件（race conditions）		----需要使用Java同步块

##### 2、JAVA并发同步措施
1) **Java同步块**

Java 同步块（synchronized block）用来标记方法或者代码块是同步的。Java同步块用来避免竞争。

- Java同步关键字（synchronzied）
	Java中的同步块用synchronized标记。同步块在Java中是同步在某个对象上。所有同步在一个对象上的同步块在同时只能被一个线程进入并执行操作。所有其他等待进入该同步块的线程将被阻塞，直到执行该同步块中的线程退出。

- 实例方法同步
	Java实例方法同步是同步在拥有该方法的对象上。这样，每个实例其方法同步都同步在不同的对象上，即该方法所属的实例。只有一个线程能够在实例方法同步块中运行。如果有多个实例存在，那么一个线程一次可以在一个实例同步块中执行操作。一个实例一个线程。

    ```
	 public synchronized void add(int value){
	   this.count += value;
     }
	```

- 静态方法同步
	静态方法的同步是指同步在该方法所在的类对象上。因为在Java虚拟机中一个类只能对应一个类对象，所以同时只允许一个线程执行同一个类中的静态同步方法。对于不同类中的静态同步方法，一个线程可以执行每个类中的静态同步方法而无需等待。不管类中的那个静态同步方法被调用，一个类只能由一个线程同时执行。

	```
	public static synchronized void add(int value){
 	  count += value;
 	}
	```

- 实例方法中同步块
	注意Java同步块构造器用括号将对象括起来。在上例中，使用了“this”，即为调用add方法的实例本身。在同步构造器中用括号括起来的对象叫做监视器对象。上述代码使用监视器对象同步，同步实例方法使用调用方法本身的实例作为监视器对象。一次只有一个线程能够在同步于同一个监视器对象的Java方法内执行。

	```
	public void add(int value){
      synchronized(this){
       this.count += value;
      }
  	}
	```

	下面两个例子都同步他们所调用的实例对象上，因此他们在同步的执行效果上是等效的。但每次只有一个线程能够在两个同步块中任意一个方法内执行。如果第二个同步块不是同步在this实例对象上，那么两个方法可以被线程同时执行。

	```
	public class MyClass {

   		public synchronized void log1(String msg1, String msg2){
      		log.writeln(msg1);
     		log.writeln(msg2);
   		}

   		public void log2(String msg1, String msg2){
      		synchronized(this){
       		log.writeln(msg1);
       		log.writeln(msg2);
      	}
    }
	```

- 静态方法中同步块
	下面是两个静态方法同步的例子。这些方法同步在该方法所属的类对象上。这两个方法不允许同时被线程访问。如果第二个同步块不是同步在MyClass.class这个对象上。那么这两个方法可以同时被线程访问。

	```
     public class MyClass {
        public static synchronized void log1(String msg1, String msg2){
           log.writeln(msg1);
           log.writeln(msg2);
        }

        public static void log2(String msg1, String msg2){
           synchronized(MyClass.class){
              log.writeln(msg1);
              log.writeln(msg2);
           }
        }
      }
     ```

2) **Java线程通信**

线程通信的目标是使线程间能够互相发送信号。另一方面，线程通信使线程能够等待其他线程的信号。JAVA线程间通信的主题：

1. 通过共享对象通信
	线程间发送信号的一个简单方式是在共享对象的变量里设置信号值。线程A在一个同步块里设置boolean型成员变量hasDataToProcess为true，线程B也在同步块里读取hasDataToProcess这个成员变量。这个简单的例子使用了一个持有信号的对象，并提供了set和check方法:

	```
	public class MySignal{
  		protected boolean hasDataToProcess = false;

  		public synchronized boolean hasDataToProcess(){
    		return this.hasDataToProcess;
  		}

  	 	public synchronized void setHasDataToProcess(boolean hasData){
    		this.hasDataToProcess = hasData;
 	 	}
	}
	```

	线程A和B必须获得指向一个MySignal共享实例的引用，以便进行通信。如果它们持有的引用指向不同的MySingal实例，那么彼此将不能检测到对方的信号。需要处理的数据可以存放在一个共享缓存区里，它和MySignal实例是分开存放的。

2. 忙等待(Busy Wait)
	准备处理数据的线程B正在等待数据变为可用。换句话说，它在等待线程A的一个信号，这个信号使hasDataToProcess()返回true。线程B运行在一个循环里，以等待这个信号：

    ```
	protected MySignal sharedSignal = ......

	while(!sharedSignal.hasDataToProcess()){
  		//do nothing... busy waiting
	}
	```

3. wait(),notify()和notifyAll()
	忙等待没有对运行等待线程的CPU进行有效的利用，除非平均等待时间非常短。否则，让等待线程进入睡眠或者非运行状态更为明智，直到它接收到它等待的信号。
	Java有一个内建的等待机制来允许线程在等待信号的时候变为非运行状态。java.lang.Object 类定义了三个方法，wait()、notify()和notifyAll()来实现这个等待机制。
	一个线程一旦调用了任意对象的wait()方法，就会变为非运行状态，直到另一个线程调用了同一个对象的notify()方法。为了调用wait()或者notify()，线程必须先获得那个对象的锁。也就是说，线程必须在同步块里调用wait()或者notify()。以下是MySingal的修改版本——使用了wait()和notify()的MyWaitNotify：

	```
	public class MonitorObject{
	}

	public class MyWaitNotify{
 		 MonitorObject myMonitorObject = new MonitorObject();

  		public void doWait(){
    		synchronized(myMonitorObject){
      		try{
        		myMonitorObject.wait();
      		} catch(InterruptedException e){...}
    	}
  	}

    public void doNotify(){
    	synchronized(myMonitorObject){
      		myMonitorObject.notify();
    	}
  	  }
	}
	```

	等待线程将调用doWait()，而唤醒线程将调用doNotify()。当一个线程调用一个对象的notify()方法，正在等待该对象的所有线程中将有一个线程被唤醒并允许执行（校注：这个将被唤醒的线程是随机的，不可以指定唤醒哪个线程）。同时也提供了一个notifyAll()方法来唤醒正在等待一个给定对象的所有线程。如你所见，不管是等待线程还是唤醒线程都在同步块里调用wait()和notify()。这是强制性的！一个线程如果没有持有对象锁，将不能调用wait()，notify()或者notifyAll()。否则，会抛出IllegalMonitorStateException异常。

	> 校注：JVM是这么实现的，当你调用wait时候它首先要检查下当前线程是否是锁的拥有者，不是则抛出IllegalMonitorStateExcept，参考JVM源码的 1422行。

	但是，这怎么可能？等待线程在同步块里面执行的时候，不是一直持有监视器对象（myMonitor对象）的锁吗？等待线程不能阻塞唤醒线程进入doNotify()的同步块吗？答案是：的确不能。一旦线程调用了wait()方法，它就释放了所持有的监视器对象上的锁。这将允许其他线程也可以调用wait()或者notify()。一旦一个线程被唤醒，不能立刻就退出wait()的方法调用，直到调用notify()的线程退出了它自己的同步块。换句话说：被唤醒的线程必须重新获得监视器对象的锁，才可以退出wait()的方法调用，因为wait方法调用运行在同步块里面。如果多个线程被notifyAll()唤醒，那么在同一时刻将只有一个线程可以退出wait()方法，因为每个线程在退出wait()前必须获得监视器对象的锁。

4. 丢失的信号
	notify()和notifyAll()方法不会保存调用它们的方法，因为当这两个方法被调用时，有可能没有线程处于等待状态。通知信号过后便丢弃了。因此，如果一个线程先于被通知线程调用wait()前调用了notify()，等待的线程将错过这个信号。这可能是也可能不是个问题。不过，在某些情况下，这可能使等待线程永远在等待，不再醒来，因为线程错过了唤醒信号。
	为了避免丢失信号，必须把它们保存在信号类里。在MyWaitNotify的例子中，通知信号应被存储在MyWaitNotify实例的一个成员变量里。以下是MyWaitNotify的修改版本：

	```
	public class MyWaitNotify2{
  		MonitorObject myMonitorObject = new MonitorObject();
  		boolean wasSignalled = false;

  		public void doWait(){
    		synchronized(myMonitorObject){
      			if(!wasSignalled){
        		try{
          			myMonitorObject.wait();
         		} catch(InterruptedException e){...}
      		}
      		//clear signal and continue running.
      		wasSignalled = false;
    		}
  		}

  		public void doNotify(){
    		synchronized(myMonitorObject){
      			wasSignalled = true;
      			myMonitorObject.notify();
    		}
  		}
	}
	```

	留意doNotify()方法在调用notify()前把wasSignalled变量设为true。同时，留意doWait()方法在调用wait()前会检查wasSignalled变量。事实上，如果没有信号在前一次doWait()调用和这次doWait()调用之间的时间段里被接收到，它将只调用wait()。

	> 为了避免信号丢失， 用一个变量来保存是否被通知过。在notify前，设置自己已经被通知过。在wait后，设置自己没有被通知过，需要等待通知。

5. 假唤醒
	由于莫名其妙的原因，线程有可能在没有调用过notify()和notifyAll()的情况下醒来。这就是所谓的假唤醒（spurious wakeups）。无端端地醒过来了。如果在MyWaitNotify2的doWait()方法里发生了假唤醒，等待线程即使没有收到正确的信号，也能够执行后续的操作。这可能导致你的应用程序出现严重问题。为了防止假唤醒，保存信号的成员变量将在一个while循环里接受检查，而不是在if表达式里。这样的一个while循环叫做自旋锁。

    >（校注：这种做法要慎重，目前的JVM实现自旋会消耗CPU，如果长时间不调用doNotify方法，doWait方法会一直自旋，CPU会消耗太大）

    被唤醒的线程会自旋直到自旋锁(while循环)里的条件变为false。以下MyWaitNotify2的修改版本展示了这点：

	```
	public class MyWaitNotify3{
  		MonitorObject myMonitorObject = new MonitorObject();
  		boolean wasSignalled = false;

  		public void doWait(){
    		synchronized(myMonitorObject){
     			while(!wasSignalled){
        		try{
          			myMonitorObject.wait();
         		} catch(InterruptedException e){...}
      		}
      		//clear signal and continue running.
      		wasSignalled = false;
    		}
 		}

  		public void doNotify(){
    		synchronized(myMonitorObject){
      			wasSignalled = true;
      			myMonitorObject.notify();
    		}
  		}
	}
	```
	留意wait()方法是在while循环里，而不在if表达式里。如果等待线程没有收到信号就唤醒，wasSignalled变量将变为false,while循环会再执行一次，促使醒来的线程回到等待状态。

6. 多线程等待相同信号
	如果你有多个线程在等待，被notifyAll()唤醒，但只有一个被允许继续执行，使用while循环也是个好方法。每次只有一个线程可以获得监视器对象锁，意味着只有一个线程可以退出wait()调用并清除wasSignalled标志（设为false）。一旦这个线程退出doWait()的同步块，其他线程退出wait()调用，并在while循环里检查wasSignalled变量值。但是，这个标志已经被第一个唤醒的线程清除了，所以其余醒来的线程将回到等待状态，直到下次信号到来。

7. 不要对常量字符串或全局对象调用wait()
	*本章说的字符串常量指的是值为常量的变量*
	本文早期的一个版本在MyWaitNotify例子里使用字符串常量（””）作为管程对象。以下是那个例子：

	```
	public class MyWaitNotify{
  		String myMonitorObject = "";
  		boolean wasSignalled = false;

  		public void doWait(){
    		synchronized(myMonitorObject){
      			while(!wasSignalled){
        			try{
         		 	myMonitorObject.wait();
         			} catch(InterruptedException e){...}
      			}
      			//clear signal and continue running.
      			wasSignalled = false;
    		}
  		}

  		public void doNotify(){
    		synchronized(myMonitorObject){
     			wasSignalled = true;
      			myMonitorObject.notify();
    		}
  		}
	}
	```

	在空字符串作为锁的同步块(或者其他常量字符串)里调用wait()和notify()产生的问题是，JVM/编译器内部会把常量字符串转换成同一个对象。这意味着，即使你有2个不同的MyWaitNotify实例，它们都引用了相同的空字符串实例。同时也意味着存在这样的风险：在第一个MyWaitNotify实例上调用doWait()的线程会被在第二个MyWaitNotify实例上调用doNotify()的线程唤醒。这种情况可以画成以下这张图：

	![string-wait](http://ifeve.com/wp-content/uploads/2013/03/strings-wait-notify.png)

起初这可能不像个大问题。毕竟，如果doNotify()在第二个MyWaitNotify实例上被调用，真正发生的事不外乎线程A和B被错误的唤醒了 。这个被唤醒的线程（A或者B）将在while循环里检查信号值，然后回到等待状态，因为doNotify()并没有在第一个MyWaitNotify实例上调用，而这个正是它要等待的实例。这种情况相当于引发了一次假唤醒。线程A或者B在信号值没有更新的情况下唤醒。但是代码处理了这种情况，所以线程回到了等待状态。记住，即使4个线程在相同的共享字符串实例上调用wait()和notify()，doWait()和doNotify()里的信号还会被2个MyWaitNotify实例分别保存。在MyWaitNotify1上的一次doNotify()调用可能唤醒MyWaitNotify2的线程，但是信号值只会保存在MyWaitNotify1里。

问题在于，由于doNotify()仅调用了notify()而不是notifyAll()，即使有4个线程在相同的字符串（空字符串）实例上等待，只能有一个线程被唤醒。所以，如果线程A或B被发给C或D的信号唤醒，它会检查自己的信号值，看看有没有信号被接收到，然后回到等待状态。而C和D都没被唤醒来检查它们实际上接收到的信号值，这样信号便丢失了。这种情况相当于前面所说的丢失信号的问题。C和D被发送过信号，只是都不能对信号作出回应。如果doNotify()方法调用notifyAll()，而非notify()，所有等待线程都会被唤醒并依次检查信号值。线程A和B将回到等待状态，但是C或D只有一个线程注意到信号，并退出doWait()方法调用。C或D中的另一个将回到等待状态，因为获得信号的线程在退出doWait()的过程中清除了信号值(置为false)。

看过上面这段后，你可能会设法使用notifyAll()来代替notify()，但是这在性能上是个坏主意。在只有一个线程能对信号进行响应的情况下，没有理由每次都去唤醒所有线程。所以：`在wait()/notify()机制中，不要使用全局对象，字符串常量等。应该使用对应唯一的对象`。例如，每一个MyWaitNotify3的实例（前一节的例子）拥有一个属于自己的监视器对象，而不是在空字符串上调用wait()/notify()。

> 管程 (英语：Monitors，也称为监视器) 是对多个工作线程实现互斥访问共享资源的对象或模块。这些共享资源一般是硬件设备或一群变量。管程实现了在一个时间点，最多只有一个线程在执行它的某个子程序。与那些通过修改数据结构实现互斥访问的并发程序设计相比，管程很大程度上简化了程序设计。

3) **局部线程变量 Java TheadLocal**

Java中的ThreadLocal类可以让你创建的变量只被同一个线程进行读和写操作。因此，尽管有两个线程同时执行一段相同的代码，而且这段代码又有一个指向同一个ThreadLocal变量的引用，但是这两个线程依然不能看到彼此的ThreadLocal变量域。

- 创建一个ThreadLocal对象
	```
    //创建一个ThreadLocal变量
	private ThreadLocal myThreadLocal = new ThreadLocal();
	```
	你实例化了一个ThreadLocal对象。每个线程仅需要实例化一次即可。虽然不同的线程执行同一段代码时，访问同一个ThreadLocal变量，但是每个线程只能看到私有的ThreadLocal实例。所以不同的线程在给ThreadLocal对象设置不同的值时，他们也不能看到彼此的修改。

- 访问ThreadLocal对象
    ```
    //一旦创建了一个ThreadLocal对象，就可以通过以下方式来存储此对象的值。
	myThreadLocal.set("A thread local value");

	//也可以直接读取一个ThreadLocal对象的值：get()方法会返回一个Object对象，而set()方法则依赖一个Object对象参数。
	String threadLocalValue = (String) myThreadLocal.get();
	```

- ThreadLocal泛型
	```
	private ThreadLocal myThreadLocal1 = new ThreadLocal'<'String'>'();
    myThreadLocal1.set("Hello ThreadLocal");
	String threadLocalValues = myThreadLocal.get();
	```

- 初始化ThreadLocal
	```
	private ThreadLocal myThreadLocal = new ThreadLocal'<'String'>'() {
   		@Override protected String initialValue() {
       		return "This is the initial value";
   		}
	};
    //可以通过ThreadLocal子类的实现，并覆写initialValue()方法，就可以为ThreadLocal对象指定一个初始化值。此时，在set()方法调用前，当调用get()方法的时候，所有线程都可以看到同一个初始化值。
	```

- InheritableThreadLocal
	InheritableThreadLocal类是ThreadLocal的子类。为了解决ThreadLocal实例内部每个线程都只能看到自己的私有值，所以InheritableThreadLocal允许一个线程创建的所有子线程访问其父线程的值。
	```
    public class InheritableThreadLocalTest {
        private InheritableThreadLocal<Integer> inheritableThreadLocal =
                new InheritableThreadLocal<Integer>();

        public InheritableThreadLocal<Integer> getInheritableThreadLocal() {
            return inheritableThreadLocal;
        }

        public static void main(String[] args) {
            final InheritableThreadLocalTest parentThread = new InheritableThreadLocalTest();
            parentThread.getInheritableThreadLocal().set(2287);

            new Thread(new Runnable() {
                @Override
                public void run() {
                    System.out.println("child thread get: " + parentThread.getInheritableThreadLocal().get());
                    parentThread.getInheritableThreadLocal().set(55);
                    System.out.println("child thread after set: " + parentThread.getInheritableThreadLocal().get());
                }
            }).start();

            try {
                Thread.sleep(2000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            System.out.println("parent thread : " + parentThread.getInheritableThreadLocal().get());
        }
    }
	```


- 完整的ThreadLocal例子
	```
	public class ThreadLocalExample {
        public static class MyRunnable implements Runnable {

            private ThreadLocal<Integer> threadLocal =
                   new ThreadLocal<Integer>();

            @Override
            public void run() {
                threadLocal.set( (int) (Math.random() * 100D) );

                try {
                    Thread.sleep(2000);
                } catch (InterruptedException e) {
                }

                System.out.println(threadLocal.get());
            }
        }

    	public static void main(String[] args) {
            MyRunnable sharedRunnableInstance = new MyRunnable();

            Thread thread1 = new Thread(sharedRunnableInstance);
            Thread thread2 = new Thread(sharedRunnableInstance);

            thread1.start();
            thread2.start();

            thread1.join(); //wait for thread 1 to terminate
            thread2.join(); //wait for thread 2 to terminate
    	}
	}
	```
	每个线程执行run()方法的时候，会给同一个ThreadLocal实例设置不同的值。如果调用set()方法的时候用synchronized关键字同步，而且不是一个ThreadLocal对象实例，那么第二个线程将会覆盖第一个线程所设置的值。然而，由于是ThreadLocal对象，所以两个线程无法看到彼此的值。因此，可以设置或者获取不同的值。