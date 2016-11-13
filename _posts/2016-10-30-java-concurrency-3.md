---
layout: post
title:  "Java 并发编程"
categories: Java IO
tags:   NIO
---

* content
{:toc}


#### Java 并发编程（三）





##### 1、并发编程的同步器
1) Java中的同步器（锁，信号量，阻塞队列）

锁像synchronized同步块一样，是一种线程同步机制，但比Java中的synchronized同步块更复杂。因为锁（以及其它更高级的线程同步机制）是由synchronized同步块的方式实现的，所以我们还不能完全摆脱synchronized关键字（译者注：这说的是Java 5之前的情况）。
自Java 5开始，java.util.concurrent.locks包中包含了一些锁的实现，因此你不用去实现自己的锁了。但是你仍然需要去了解怎样使用这些锁，且了解这些实现背后的理论也是很有用处的。可以参考我对java.util.concurrent.locks.Lock的介绍，以了解更多关于锁的信息。

- 一个简单的锁
	```
	public class Counter{
        private int count = 0;

        public int inc(){
            synchronized(this){
                return ++count;
            }
        }
    }
	```

	可以看到在inc()方法中有一个synchronized(this)代码块。该代码块可以保证在同一时间只有一个线程可以执行return ++count。虽然在synchronized的同步块中的代码可以更加复杂，但是++count这种简单的操作已经足以表达出线程同步的意思。
    以下的Counter类用Lock代替synchronized达到了同样的目的：
	```
	public class Counter{
        private Lock lock = new Lock();
        private int count = 0;

        public int inc(){
            lock.lock();
            int newCount = ++count;
            lock.unlock();
            return newCount;
        }
    }
	```

	lock()方法会对Lock实例对象进行加锁，因此所有对该对象调用lock()方法的线程都会被阻塞，直到该Lock对象的unlock()方法被调用。
	这里有一个Lock类的简单实现：
	```
	public class Counter{
        public class Lock{
            private boolean isLocked = false;

            public synchronized void lock()
                throws InterruptedException{
                while(isLocked){
                    wait();
                }
                isLocked = true;
            }

            public synchronized void unlock(){
                isLocked = false;
                notify();
            }
        }
    }
	```

	注意其中的while(isLocked)循环，它又被叫做`“自旋锁”`。自旋锁以及wait()和notify()方法在线程通信这篇文章中有更加详细的介绍。当isLocked为true时，调用lock()的线程在wait()调用上阻塞等待。为防止该线程没有收到notify()调用也从wait()中返回（也称`作虚假唤醒`），这个线程会重新去检查isLocked条件以决定当前是否可以安全地继续执行还是需要重新保持等待，而不是认为线程被唤醒了就可以安全地继续执行了。如果isLocked为false，当前线程会退出while(isLocked)循环，并将isLocked设回true，让其它正在调用lock()方法的线程能够在Lock实例上加锁。
	当线程完成了临界区（位于lock()和unlock()之间）中的代码，就会调用unlock()。执行unlock()会重新将isLocked设置为false，并且通知（唤醒）其中一个（若有的话）在lock()方法中调用了wait()函数而处于等待状态的线程。

- 锁的可重入性
	Java中的synchronized同步块是可重入的。这意味着如果一个java线程进入了代码中的synchronized同步块，并因此获得了该同步块使用的同步对象对应的管程上的锁，那么这个线程可以进入由同一个管程对象所同步的另一个java代码块。下面是一个例子：
	```
	public class Reentrant{
        public synchronized outer(){
            inner();
        }

        public synchronized inner(){
            //do something
        }
    }
	```

	注意outer()和inner()都被声明为synchronized，这在Java中和synchronized(this)块等效。如果一个线程调用了outer()，在outer()里调用inner()就没有什么问题，因为这两个方法（代码块）都由同一个管程对象（”this”)所同步。如果一个线程已经拥有了一个管程对象上的锁，那么它就有权访问被这个管程对象同步的所有代码块。这就是可重入。线程可以进入任何一个它已经拥有的锁所同步着的代码块。
	前面给出的锁实现不是可重入的。如果我们像下面这样重写Reentrant类，当线程调用outer()时，会在inner()方法的lock.lock()处阻塞住。
	```
	public class Reentrant2{
        Lock lock = new Lock();

        public outer(){
            lock.lock();
            inner();
            lock.unlock();
        }

        public synchronized inner(){
            lock.lock();
            //do something
            lock.unlock();
        }
    }
	```

	调用outer()的线程首先会锁住Lock实例，然后继续调用inner()。inner()方法中该线程将再一次尝试锁住Lock实例，结果该动作会失败（也就是说该线程会被阻塞），因为这个Lock实例已经在outer()方法中被锁住了。

	两次lock()之间没有调用unlock()，第二次调用lock就会阻塞，看过lock()实现后，会发现原因很明显：
	```
	public class Lock{
        boolean isLocked = false;

        public synchronized void lock()
            throws InterruptedException{
            while(isLocked){
                wait();
            }
            isLocked = true;
        }
        ...
    }
	```

	一个线程是否被允许退出lock()方法是由while循环（自旋锁）中的条件决定的。当前的判断条件是只有当isLocked为false时lock操作才被允许，而没有考虑是哪个线程锁住了它。
	为了让这个Lock类具有可重入性，我们需要对它做一点小的改动：
	```
	public class Lock{
        boolean isLocked = false;
        Thread  lockedBy = null;
        int lockedCount = 0;

        public synchronized void lock()
            throws InterruptedException{
            Thread callingThread =
                Thread.currentThread();
            while(isLocked && lockedBy != callingThread){
                wait();
            }
            isLocked = true;
            lockedCount++;
            lockedBy = callingThread;
        }

        public synchronized void unlock(){
            if(Thread.curentThread() ==
                this.lockedBy){
                lockedCount--;
                if(lockedCount == 0){
                    isLocked = false;
                    notify();
                }
            }
        }
        ...
    }
	```

	注意到现在的while循环（自旋锁）也考虑到了已锁住该Lock实例的线程。如果当前的锁对象没有被加锁(isLocked = false)，或者当前调用线程已经对该Lock实例加了锁，那么while循环就不会被执行，调用lock()的线程就可以退出该方法（`译者注：“被允许退出该方法”在当前语义下就是指不会调用wait()而导致阻塞`）。
	除此之外，我们需要记录同一个线程重复对一个锁对象加锁的次数。否则，一次unblock()调用就会解除整个锁，即使当前锁已经被加锁过多次。在unlock()调用没有达到对应lock()调用的次数之前，我们不希望锁被解除。现在这个Lock类就是可重入的了。

- 锁的公平性
	Java的synchronized块并不保证尝试进入它们的线程的顺序。因此，如果多个线程不断竞争访问相同的synchronized同步块，就存在一种风险，其中一个或多个线程永远也得不到访问权 —— 也就是说访问权总是分配给了其它线程。这种情况被称作线程饥饿。为了避免这种问题，锁需要实现公平性。本文所展现的锁在内部是用synchronized同步块实现的，因此它们也不保证公平性。`饥饿和公平`中有更多关于该内容的讨论。

- 在finally语句中调用unlock()
	如果用Lock来保护临界区，并且临界区有可能会抛出异常，那么在finally语句中调用unlock()就显得非常重要了。这样可以保证这个锁对象可以被解锁以便其它线程能继续对其加锁。以下是一个示例：
	```
	lock.lock();
    try{
        //do critical section code,
        //which may throw exception
    } finally {
        lock.unlock();
    }
	```

	这个简单的结构可以保证当临界区抛出异常时Lock对象可以被解锁。如果不是在finally语句中调用的unlock()，当临界区抛出异常时，Lock对象将永远停留在被锁住的状态，这会导致其它所有在该Lock对象上调用lock()的线程一直阻塞。

2) Java Concurrent包中的锁

**ReentrantLock(可重入锁)**	一个可重入的互斥锁 Lock，它具有与使用 synchronized 方法和语句所访问的隐式监视器锁相同的一些基本行为和语义，但功能更强大。
ReentrantLock 将由最近成功获得锁，并且还没有释放该锁的线程所拥有。当锁没有被另一个线程所拥有时，调用 lock 的线程将成功获取该锁并返回。如果当前线程已经拥有该锁，此方法将立即返回。可以使用 isHeldByCurrentThread() 和 getHoldCount() 方法来检查此情况是否发生。此类的构造方法接受一个可选的公平 参数。当设置为 true 时，在多个线程的争用下，这些锁倾向于将访问权授予等待时间最长的线程。否则此锁将无法保证任何特定访问顺序。与采用默认设置（使用不公平锁）相比，使用公平锁的程序在许多线程访问时表现为很低的总体吞吐量（即速度很慢，常常极其慢），但是在获得锁和保证锁分配的均衡性时差异较小。不过要注意的是，公平锁不能保证线程调度的公平性。因此，使用公平锁的众多线程中的一员可能获得多倍的成功机会，这种情况发生在其他活动线程没有被处理并且目前并未持有锁时。还要注意的是，未定时的 tryLock 方法并没有使用公平设置。因为即使其他线程正在等待，只要该锁是可用的，此方法就可以获得成功。

```
class X {
   private final ReentrantLock lock = new ReentrantLock();
   // ...

   public void m() {
     lock.lock();  // block until condition holds
     try {
       // ... method body
     } finally {
       lock.unlock()
     }
   }
}
```

**Java中的读/写锁（实现原理）**	相比Java中的锁(Locks in Java)里Lock实现，读写锁更复杂一些。假设你的程序中涉及到对一些共享资源的读和写操作，且写操作没有读操作那么频繁。在没有写操作的时候，两个线程同时读一个资源没有任何问题，所以应该允许多个线程能在同时读取共享资源。但是如果有一个线程想去写这些共享资源，就不应该再有其它线程对该资源进行读或写（`译者注：也就是说：读-读能共存，读-写不能共存，写-写不能共存`）。这就需要一个读/写锁来解决这个问题。
Java5在java.util.concurrent包中已经包含了读写锁。尽管如此，我们还是应该了解其实现背后的原理。

- 读/写锁的Java实现(Read / Write Lock Java Implementation)
	`读取` 没有线程正在做写操作，且没有线程在请求写操作。
	`写入` 没有线程正在做读写操作。
	如果某个线程想要读取资源，只要没有线程正在对该资源进行写操作且没有线程请求对该资源的写操作即可。我们假设对写操作的请求比对读操作的请求更重要，就要提升写请求的优先级。此外，如果读操作发生的比较频繁，我们又没有提升写操作的优先级，那么就会产生“饥饿”现象。请求写操作的线程会一直阻塞，直到所有的读线程都从ReadWriteLock上解锁了。如果一直保证新线程的读操作权限，那么等待写操作的线程就会一直阻塞下去，结果就是发生“饥饿”。因此，只有当没有线程正在锁住`ReadWriteLock`进行写操作，且没有线程请求该锁准备执行写操作时，才能保证读操作继续。
	当其它线程没有对共享资源进行读操作或者写操作时，某个线程就有可能获得该共享资源的写锁，进而对共享资源进行写操作。有多少线程请求了写锁以及以何种顺序请求写锁并不重要，除非你想保证写锁请求的公平性。
	简单的实现出一个读/写锁，代码如下:
	```
	public class ReadWriteLock{
        private int readers = 0;
        private int writers = 0;
        private int writeRequests = 0;

        public synchronized void lockRead()
            throws InterruptedException{
            while(writers > 0 || writeRequests > 0){
                wait();
            }
            readers++;
        }

        public synchronized void unlockRead(){
            readers--;
            notifyAll();
        }

        public synchronized void lockWrite()
            throws InterruptedException{
            writeRequests++;

            while(readers > 0 || writers > 0){
                wait();
            }
            writeRequests--;
            writers++;
        }

        public synchronized void unlockWrite()
            throws InterruptedException{
            writers--;
            notifyAll();
        }
    }
	```

	 ReadWriteLock类中，读锁和写锁各有一个获取锁和释放锁的方法。读锁的实现在lockRead()中,只要没有线程拥有写锁（writers==0），且没有线程在请求写锁（writeRequests ==0），所有想获得读锁的线程都能成功获取。
	写锁的实现在lockWrite()中,当一个线程想获得写锁的时候，首先会把写锁请求数加1（writeRequests++），然后再去判断是否能够真能获得写锁，当没有线程持有读锁（readers==0 ）,且没有线程持有写锁（writers==0）时就能获得写锁。有多少线程在请求写锁并无关系。
	需要注意的是，在两个释放锁的方法（unlockRead，unlockWrite）中，都调用了notifyAll方法，而不是notify。要解释这个原因，我们可以想象下面一种情形：如果有线程在等待获取读锁，同时又有线程在等待获取写锁。如果这时其中一个等待读锁的线程被notify方法唤醒，但因为此时仍有请求写锁的线程存在（writeRequests>0），所以被唤醒的线程会再次进入阻塞状态。然而，等待写锁的线程一个也没被唤醒，就像什么也没发生过一样（译者注：信号丢失现象）。如果用的是notifyAll方法，所有的线程都会被唤醒，然后判断能否获得其请求的锁。
	用notifyAll还有一个好处。如果有多个读线程在等待读锁且没有线程在等待写锁时，调用unlockWrite()后，所有等待读锁的线程都能立马成功获取读锁 —— 而不是一次只允许一个。

- 读/写锁的重入(Read / Write Lock Reentrance)
	上面实现的读/写锁(ReadWriteLock) 是不可重入的，当一个已经持有写锁的线程再次请求写锁时，就会被阻塞。原因是已经有一个写线程了——就是它自己。
	此外，考虑下面的例子：
    1. Thread 1 获得了读锁
    2. Thread 2 请求写锁，但因为Thread 1 持有了读锁，所以写锁请求被阻塞。
    3. Thread 1 再想请求一次读锁，但因为Thread 2处于请求写锁的状态，所以想再次获取读锁也会被阻塞。

 上面这种情形使用前面的ReadWriteLock就会被锁定——一种类似于死锁的情形。不会再有线程能够成功获取读锁或写锁了。为了让ReadWriteLock可重入，需要对它做一些改进。下面会分别处理读锁的重入和写锁的重入。

- 读锁重入(Read Reentrance)
	为了让ReadWriteLock的读锁可重入，我们要先为读锁重入建立规则：
    要保证某个线程中的读锁可重入，要么满足获取读锁的条件（没有写或写请求），要么已经持有读锁（不管是否有写请求）。
	要确定一个线程是否已经持有读锁，可以用一个map来存储已经持有读锁的线程以及对应线程获取读锁的次数，当需要判断某个线程能否获得读锁时，就利用map中存储的数据进行判断。下面是方法lockRead和unlockRead修改后的的代码：
    ```
	public class ReadWriteLock{
        private Map<Thread, Integer> readingThreads = new HashMap<Thread, Integer>();

        private int writers = 0;
        private int writeRequests = 0;

        public synchronized void lockRead()
            throws InterruptedException{
            Thread callingThread = Thread.currentThread();
            while(! canGrantReadAccess(callingThread)){
                wait();
            }

            readingThreads.put(callingThread,
                (getAccessCount(callingThread) + 1));
        }

        public synchronized void unlockRead(){
            Thread callingThread = Thread.currentThread();
            int accessCount = getAccessCount(callingThread);
            if(accessCount == 1) {
                readingThreads.remove(callingThread);
            } else {
                readingThreads.put(callingThread, (accessCount -1));
            }
            notifyAll();
        }

        private boolean canGrantReadAccess(Thread callingThread){
            if(writers > 0) return false;
            if(isReader(callingThread) return true;
            if(writeRequests > 0) return false;
            return true;
        }

        private int getReadAccessCount(Thread callingThread){
            Integer accessCount = readingThreads.get(callingThread);
            if(accessCount == null) return 0;
            return accessCount.intValue();
        }

        private boolean isReader(Thread callingThread){
            return readingThreads.get(callingThread) != null;
        }
    }
	```

	代码中我们可以看到，只有在没有线程拥有写锁的情况下才允许读锁的重入。此外，重入的读锁比写锁优先级高。

- 写锁重入(Write Reentrance)
	仅当一个线程已经持有写锁，才允许写锁重入（再次获得写锁）。下面是方法lockWrite和unlockWrite修改后的的代码。
	```
	public class ReadWriteLock{
        private Map<Thread, Integer> readingThreads = new HashMap<Thread, Integer>();

        private int writeAccesses    = 0;
        private int writeRequests    = 0;
        private Thread writingThread = null;

        public synchronized void lockWrite()
            throws InterruptedException{
            writeRequests++;
            Thread callingThread = Thread.currentThread();
            while(!canGrantWriteAccess(callingThread)){
                wait();
            }
            writeRequests--;
            writeAccesses++;
            writingThread = callingThread;
        }

        public synchronized void unlockWrite()
            throws InterruptedException{
            writeAccesses--;
            if(writeAccesses == 0){
                writingThread = null;
            }
            notifyAll();
        }

        private boolean canGrantWriteAccess(Thread callingThread){
            if(hasReaders()) return false;
            if(writingThread == null)    return true;
            if(!isWriter(callingThread)) return false;
            return true;
        }

        private boolean hasReaders(){
            return readingThreads.size() > 0;
        }

        private boolean isWriter(Thread callingThread){
            return writingThread == callingThread;
        }
    }
	```

	注意在确定当前线程是否能够获取写锁的时候，是如何处理的。

- 读锁升级到写锁(Read to Write Reentrance)
	有时，我们希望一个拥有读锁的线程，也能获得写锁。想要允许这样的操作，要求这个线程是唯一一个拥有读锁的线程。writeLock()需要做点改动来达到这个目的：
	```
    public class ReadWriteLock{
        private Map<Thread, Integer> readingThreads = new HashMap<Thread, Integer>();

        private int writeAccesses    = 0;
        private int writeRequests    = 0;
        private Thread writingThread = null;

        public synchronized void lockWrite()
            throws InterruptedException{
            writeRequests++;
            Thread callingThread = Thread.currentThread();
            while(!canGrantWriteAccess(callingThread)){
                wait();
            }
            writeRequests--;
            writeAccesses++;
            writingThread = callingThread;
        }

        public synchronized void unlockWrite() throws InterruptedException{
            writeAccesses--;
            if(writeAccesses == 0){
                writingThread = null;
            }
            notifyAll();
        }

        private boolean canGrantWriteAccess(Thread callingThread){
            if(isOnlyReader(callingThread)) return true;
            if(hasReaders()) return false;
            if(writingThread == null) return true;
            if(!isWriter(callingThread)) return false;
            return true;
        }

        private boolean hasReaders(){
            return readingThreads.size() > 0;
        }

        private boolean isWriter(Thread callingThread){
            return writingThread == callingThread;
        }

        private boolean isOnlyReader(Thread thread){
            return readers == 1 && readingThreads.get(callingThread) != null;
        }
    }
	```

	现在ReadWriteLock类就可以从读锁升级到写锁了。

- 写锁降级到读锁(Write to Read Reentrance)
	有时拥有写锁的线程也希望得到读锁。如果一个线程拥有了写锁，那么自然其它线程是不可能拥有读锁或写锁了。所以对于一个拥有写锁的线程，再获得读锁，是不会有什么危险的。我们仅仅需要对上面canGrantReadAccess方法进行简单地修改：
	```
	public class ReadWriteLock{
        private boolean canGrantReadAccess(Thread callingThread){
            if(isWriter(callingThread)) return true;
            if(writingThread != null) return false;
            if(isReader(callingThread) return true;
            if(writeRequests > 0) return false;
            return true;
        }
    }
	```

- 可重入的ReadWriteLock的完整实现(Fully Reentrant ReadWriteLock)
	```
	public class ReadWriteLock{
        private Map<Thread, Integer> readingThreads = new HashMap<Thread, Integer>();

        private int writeAccesses    = 0;
        private int writeRequests    = 0;
        private Thread writingThread = null;

        public synchronized void lockRead()
            throws InterruptedException{
            Thread callingThread = Thread.currentThread();
            while(! canGrantReadAccess(callingThread)){
                wait();
            }

            readingThreads.put(callingThread,
                (getReadAccessCount(callingThread) + 1));
        }

        private boolean canGrantReadAccess(Thread callingThread){
            if(isWriter(callingThread)) return true;
            if(hasWriter()) return false;
            if(isReader(callingThread)) return true;
            if(hasWriteRequests()) return false;
            return true;
        }

        public synchronized void unlockRead(){
            Thread callingThread = Thread.currentThread();
            if(!isReader(callingThread)){
                throw new IllegalMonitorStateException(
                    "Calling Thread does not" +
                    " hold a read lock on this ReadWriteLock");
            }
            int accessCount = getReadAccessCount(callingThread);
            if(accessCount == 1){
                readingThreads.remove(callingThread);
            } else {
                readingThreads.put(callingThread, (accessCount -1));
            }
            notifyAll();
        }

        public synchronized void lockWrite()
            throws InterruptedException{
            writeRequests++;
            Thread callingThread = Thread.currentThread();
            while(!canGrantWriteAccess(callingThread)){
                wait();
            }
            writeRequests--;
            writeAccesses++;
            writingThread = callingThread;
        }

        public synchronized void unlockWrite()
            throws InterruptedException{
            if(!isWriter(Thread.currentThread()){
            throw new IllegalMonitorStateException(
                "Calling Thread does not" +
                " hold the write lock on this ReadWriteLock");
            }
            writeAccesses--;
            if(writeAccesses == 0){
                writingThread = null;
            }
            notifyAll();
        }

        private boolean canGrantWriteAccess(Thread callingThread){
            if(isOnlyReader(callingThread)) return true;
            if(hasReaders()) return false;
            if(writingThread == null) return true;
            if(!isWriter(callingThread)) return false;
            return true;
        }

        private int getReadAccessCount(Thread callingThread){
            Integer accessCount = readingThreads.get(callingThread);
            if(accessCount == null) return 0;
            return accessCount.intValue();
        }

        private boolean hasReaders(){
            return readingThreads.size() > 0;
        }

        private boolean isReader(Thread callingThread){
            return readingThreads.get(callingThread) != null;
        }

        private boolean isOnlyReader(Thread callingThread){
            return readingThreads.size() == 1 &&
                readingThreads.get(callingThread) != null;
        }

        private boolean hasWriter(){
            return writingThread != null;
        }

        private boolean isWriter(Thread callingThread){
            return writingThread == callingThread;
        }

        private boolean hasWriteRequests(){
            return this.writeRequests > 0;
        }
    }
	```

- 在finally中调用unlock() (Calling unlock() from a finally-clause)
	在利用ReadWriteLock来保护临界区时，如果临界区可能抛出异常，在finally块中调用readUnlock()和writeUnlock()就显得很重要了。这样做是为了保证ReadWriteLock能被成功解锁，然后其它线程可以请求到该锁。这里有个例子：
	```
	lock.lockWrite();
    try{
        //do critical section code, which may throw exception
    } finally {
        lock.unlockWrite();
    }
	```

	上面这样的代码结构能够保证临界区中抛出异常时ReadWriteLock也会被释放。如果unlockWrite方法不是在finally块中调用的，当临界区抛出了异常时，ReadWriteLock 会一直保持在写锁定状态，就会导致所有调用lockRead()或lockWrite()的线程一直阻塞。唯一能够重新解锁ReadWriteLock的因素可能就是ReadWriteLock是可重入的，当抛出异常时，这个线程后续还可以成功获取这把锁，然后执行临界区以及再次调用unlockWrite()，这就会再次释放ReadWriteLock。但是如果该线程后续不再获取这把锁了呢？所以，在finally中调用unlockWrite对写出健壮代码是很重要的。

**ReadWriteLock（读/写锁）**	的一个实现是ReentrantReadWriteLock此类具有以下属性：
`获取顺序` 此类不会将读取者优先或写入者优先强加给锁访问的排序。但是，它确实支持可选的公平 策略。
`非公平模式（默认）` 当非公平地（默认）构造时，未指定进入读写锁的顺序，受到 reentrancy 约束的限制。连续竞争的非公平锁可能无限期地推迟一个或多个 reader 或 writer 线程，但吞吐量通常要高于公平锁。
`公平模式` 当公平地构造线程时，线程利用一个近似到达顺序的策略来争夺进入。当释放当前保持的锁时，可以为等待时间最长的单个 writer 线程分配写入锁，如果有一组等待时间大于所有正在等待的 writer 线程 的 reader 线程，将为该组分配写入锁。如果保持写入锁，或者有一个等待的 writer 线程，则试图获得公平读取锁（非重入地）的线程将会阻塞。直到当前最旧的等待 writer 线程已获得并释放了写入锁之后，该线程才会获得读取锁。当然，如果等待 writer 放弃其等待，而保留一个或更多 reader 线程为队列中带有写入锁自由的时间最长的 waiter，则将为那些 reader 分配读取锁。试图获得公平写入锁的（非重入地）的线程将会阻塞，除非读取锁和写入锁都自由（这意味着没有等待线程）。（注意，非阻塞 ReentrantReadWriteLock.ReadLock.tryLock()和ReentrantReadWriteLock.WriteLock.tryLock()方法不会遵守此公平设置，并将获得锁（如果可能），不考虑等待线程）。
`重入` 此锁允许 reader 和 writer 按照 ReentrantLock 的样式重新获取读取锁或写入锁。在写入线程保持的所有写入锁都已经释放后，才允许重入 reader 使用它们。此外，writer 可以获取读取锁，但反过来则不成立。在其他应用程序中，当在调用或回调那些在读取锁状态下执行读取操作的方法期间保持写入锁时，重入很有用。如果 reader 试图获取写入锁，那么将永远不会获得成功。
`锁降级` 重入还允许从写入锁降级为读取锁，其实现方式是：先获取写入锁，然后获取读取锁，最后释放写入锁。但是，从读取锁升级到写入锁是不可能的。
`锁获取的中断` 读取锁和写入锁都支持锁获取期间的中断。
`Condition支持` 写入锁提供了一个 Condition 实现，对于写入锁来说，该实现的行为与 ReentrantLock.newCondition() 提供的 Condition 实现对 ReentrantLock 所做的行为相同。当然，此 Condition 只能用于写入锁。读取锁不支持 Condition，readLock().newCondition() 会抛出 UnsupportedOperationException。
`监测` 此类支持一些确定是保持锁还是争用锁的方法。这些方法设计用于监视系统状态，而不是同步控制。此类行为的序列化方式与内置锁的相同：反序列化的锁处于解除锁状态，无论序列化该锁时其状态如何。

**ReadLock（读锁）/WriteLock（写锁）？待学习**

**信号量（Semaphore）实现和使用**	Semaphore是一个计数信号量。从概念上讲，信号量维护了一个许可集。如有必要，在许可可用前会阻塞每一个 acquire()，然后再获取该许可。每个 release() 添加一个许可，从而可能释放一个正在阻塞的获取者。但是，不使用实际的许可对象，Semaphore 只对可用许可的号码进行计数，并采取相应的行动。
Semaphore 通常用于限制可以访问某些资源（物理或逻辑的）的线程数目。Semaphore（信号量） 是一个线程同步结构，用于在线程间传递信号，以避免出现信号丢失，或者像锁一样用于保护一个关键区域。自从5.0开始，jdk在java.util.concurrent包里提供了Semaphore 的官方实现，因此大家不需要自己去实现Semaphore。但是还是很有必要去熟悉如何使用Semaphore及其背后的原理。

- 简单的Semaphore实现
	```
	public class Semaphore {
        private boolean signal = false;

        public synchronized void take() {
            this.signal = true;
            this.notify();
    	}

    	public synchronized void release() throws InterruptedException{
        	while(!this.signal) wait();
        	this.signal = false;
        }
    }
	```

	Take方法发出一个被存放在Semaphore内部的信号，而Release方法则等待一个信号，当其接收到信号后，标记位signal被清空，然后该方法终止。使用这个semaphore可以避免错失某些信号通知。用take方法来代替notify，release方法来代替wait。如果某线程在调用release等待之前调用take方法，那么调用release方法的线程仍然知道take方法已经被某个线程调用过了，因为该Semaphore内部保存了take方法发出的信号。而wait和notify方法就没有这样的功能。当用semaphore来产生信号时，take和release这两个方法名看起来有点奇怪。这两个名字来源于后面把semaphore当做锁的例子，后面会详细介绍这个例子，在该例子中，take和release这两个名字会变得很合理。

- 使用Semaphore来发出信号
	```
	Semaphore semaphore = new Semaphore();
    SendingThread sender = new SendingThread(semaphore)；
    ReceivingThread receiver = new ReceivingThread(semaphore);
    receiver.start();
    sender.start();

    public class SendingThread {
        Semaphore semaphore = null;

        public SendingThread(Semaphore semaphore){
        	this.semaphore = semaphore;
    	}

        public void run(){
            while(true){
                //do something, then signal
                this.semaphore.take();
        	}
    	}
    }

    public class RecevingThread {
        Semaphore semaphore = null;
        public ReceivingThread(Semaphore semaphore){
            this.semaphore = semaphore;
        }

        public void run(){
            while(true){
                this.semaphore.release();
                //receive signal, then do something...
        	}
    	}
    }
	```

- 可计数的Semaphore
	上面提到的Semaphore的简单实现并没有计算通过调用take方法所产生信号的数量。可以把它改造成具有计数功能的Semaphore。下面是一个可计数的Semaphore的简单实现。
	```
	public class CountingSemaphore {
        private int signals = 0;

        public synchronized void take() {
            this.signals++;
            this.notify();
		}

		public synchronized void release() throws InterruptedException{
            while(this.signals == 0) wait();
            this.signals--;
        }
	}
	```

- 有上限的Semaphore
	上面的CountingSemaphore并没有限制信号的数量。下面的代码将CountingSemaphore改造成一个信号数量有上限的BoundedSemaphore。
	```
	public class BoundedSemaphore {
        private int signals = 0;
        private int bound   = 0;

        public BoundedSemaphore(int upperBound){
        	this.bound = upperBound;
        }

        public synchronized void take() throws InterruptedException{
            while(this.signals == bound) wait();
            this.signals++;
            this.notify();
        }

        public synchronized void release() throws InterruptedException{
        while(this.signals == 0) wait();
            this.signals--;
            this.notify();
        }
    }
	```

	在BoundedSemaphore中，当已经产生的信号数量达到了上限，take方法将阻塞新的信号产生请求，直到某个线程调用release方法后，被阻塞于take方法的线程才能传递自己的信号。

- 把Semaphore当锁来使用
	当信号量的数量上限是1时，Semaphore可以被当做锁来使用。通过take和release方法来保护关键区域。请看下面的例子：
	```
	BoundedSemaphore semaphore = new BoundedSemaphore(1);
    ...
    semaphore.take();
    try{
    	//critical section
    } finally {
    	semaphore.release();
    }
	```

	在前面的例子中，Semaphore被用来在多个线程之间传递信号，这种情况下，take和release分别被不同的线程调用。但是在锁这个例子中，take和release方法将被同一线程调用，因为只允许一个线程来获取信号（允许进入关键区域的信号），其它调用take方法获取信号的线程将被阻塞，知道第一个调用take方法的线程调用release方法来释放信号。对release方法的调用永远不会被阻塞，这是因为任何一个线程都是先调用take方法，然后再调用release。通过有上限的Semaphore可以限制进入某代码块的线程数量。设想一下，在上面的例子中，如果BoundedSemaphore 上限设为5将会发生什么？意味着允许5个线程同时访问关键区域，但是你必须保证，这个5个线程不会互相冲突。否则你的应用程序将不能正常运行。必须注意，release方法应当在finally块中被执行。这样可以保在关键区域的代码抛出异常的情况下，信号也一定会被释放。

**阻塞队列（BlockingQueue）** 阻塞队列与普通队列的区别在于，当队列是空的时，从队列中获取元素的操作将会被阻塞，或者当队列是满时，往队列里添加元素的操作会被阻塞。试图从空的阻塞队列中获取元素的线程将会被阻塞，直到其他的线程往空的队列插入新的元素。同样，试图往已满的阻塞队列中添加新元素的线程同样也会被阻塞，直到其他的线程使队列重新变得空闲起来，如从队列中移除一个或者多个元素，或者完全清空队列，下图展示了如何通过阻塞队列来合作：

![BlockingQueue](http://tutorials.jenkov.com/images/java-concurrency-utils/blocking-queue.png)

从5.0开始，JDK在java.util.concurrent包里提供了阻塞队列的官方实现。尽管JDK中已经包含了阻塞队列的官方实现，但是熟悉其背后的原理还是很有帮助的。

`阻塞队列的实现` 阻塞队列的实现类似于带上限的Semaphore的实现。下面是阻塞队列的一个简单实现。

```
public class BlockingQueue {
    private List queue = new LinkedList();
    private int  limit = 10;

    public BlockingQueue(int limit){
    	this.limit = limit;
    }

    public synchronized void enqueue(Object item) throws InterruptedException  {
        while(this.queue.size() == this.limit) {
            wait();
        }

        if(this.queue.size() == 0) {
            notifyAll();
        }
        this.queue.add(item);
    }

    public synchronized Object dequeue() throws InterruptedException{
        while(this.queue.size() == 0){
            wait();
        }
        if(this.queue.size() == this.limit){
        	notifyAll();
    	}
    	return this.queue.remove(0);
    }
}
```

在enqueue和dequeue方法内部，只有队列的大小等于上限（limit）或者下限（0）时，才调用notifyAll方法。如果队列的大小既不等于上限，也不等于下限，任何线程调用enqueue或者dequeue方法时，都不会阻塞，都能够正常的往队列中添加或者移除元素。

##### 2、并发编程中的死锁
1）死锁

**死锁** 是两个或更多线程阻塞着等待其它处于死锁状态的线程所持有的锁。死锁通常发生在多个线程同时但以不同的顺序请求同一组锁的时候。
例如，如果线程1锁住了A，然后尝试对B进行加锁，同时线程2已经锁住了B，接着尝试对A进行加锁，这时死锁就发生了。线程1永远得不到B，线程2也永远得不到A，并且它们永远也不会知道发生了这样的事情。为了得到彼此的对象（A和B），它们将永远阻塞下去。这种情况就是一个死锁。

**更复杂的死锁** 死锁可能不止包含2个线程，这让检测死锁变得更加困难。

**数据库的死锁** 更加复杂的死锁场景发生在数据库事务中。一个数据库事务可能由多条SQL更新请求组成。当在一个事务中更新一条记录，这条记录就会被锁住避免其他事务的更新请求，直到第一个事务结束。同一个事务中每一个更新请求都可能会锁住一些记录。

**嵌套管程锁死** 嵌套管程锁死类似于死锁，都是线程最后被一直阻塞着互相等待。但是两者又不完全相同。在死锁中我们已经对死锁有了个大概的解释，死锁通常是因为两个线程获取锁的顺序不一致造成的，线程1锁住A，等待获取B，线程2已经获取了B，再等待获取A。如死锁避免中所说的，死锁可以通过总是以相同的顺序获取锁来避免。但是发生嵌套管程锁死时锁获取的顺序是一致的。线程1获得A和B，然后释放B，等待线程2的信号。线程2需要同时获得A和B，才能向线程1发送信号。所以，一个线程在等待唤醒，另一个线程在等待想要的锁被释放。不同点归纳如下：

	死锁中，二个线程都在等待对方释放锁。
	嵌套管程锁死中，线程1持有锁A，同时等待从线程2发来的信号，线程2需要锁A来发信号给线程1。

**重入锁死** 重入锁死与死锁和嵌套管程锁死非常相似。锁和读写锁两篇文章中都有涉及到重入锁死的问题。当一个线程重新获取锁，读写锁或其他不可重入的同步器时，就可能发生重入锁死。可重入的意思是线程可以重复获得它已经持有的锁。

Java的synchronized块是可重入的。因此下面的代码是没问题的：
```
public class Reentrant{
	public synchronized outer(){
		inner();
	}

	public synchronized inner(){
		//do something
	}
}
```

注意outer()和inner()都声明为synchronized，这在Java中这相当于synchronized(this)块（译者注：这里两个方法是实例方法，synchronized的实例方法相当于在this上加锁，如果是static方法，则不然，更多阅读：哪个对象才是锁？）。如果某个线程调用了outer()，outer()中的inner()调用是没问题的，因为两个方法都是在同一个管程对象(即this)上同步的。如果一个线程持有某个管程对象上的锁，那么它就有权访问所有在该管程对象上同步的块。这就叫可重入。若线程已经持有锁，那么它就可以重复访问所有使用该锁的代码块。

下面这个锁的实现是不可重入的：
```
public class Lock{
	private boolean isLocked = false;
	public synchronized void lock()
		throws InterruptedException{
		while(isLocked){
			wait();
		}
		isLocked = true;
	}

	public synchronized void unlock(){
		isLocked = false;
		notify();
	}
}
```

如果一个线程在两次调用lock()间没有调用unlock()方法，那么第二次调用lock()就会被阻塞，这就出现了重入锁死。

避免重入锁死有两个选择：

- 编写代码时避免再次获取已经持有的锁
- 使用可重入锁

2）避免死锁

在有些情况下死锁是可以避免的。本文将展示三种用于避免死锁的技术：

- 加锁顺序
	当多个线程需要相同的一些锁，但是按照不同的顺序加锁，死锁就很容易发生。如果能确保所有的线程都是按照相同的顺序获得锁，那么死锁就不会发生。
	```
	Thread 1:
      lock A
      lock B

    Thread 2:
       wait for A
       lock C (when A locked)

    Thread 3:
       wait for A
       wait for B
       wait for C
	```

	如果一个线程（比如线程3）需要一些锁，那么它必须按照确定的顺序获取锁。它只有获得了从顺序上排在前面的锁之后，才能获取后面的锁。例如，线程2和线程3只有在获取了锁A之后才能尝试获取锁C(译者注：获取锁A是获取锁C的必要条件)。因为线程1已经拥有了锁A，所以线程2和3需要一直等到锁A被释放。然后在它们尝试对B或C加锁之前，必须成功地对A加了锁。
	按照顺序加锁是一种有效的死锁预防机制。但是，这种方式需要你事先知道所有可能会用到的锁(译者注：并对这些锁做适当的排序)，但总有些时候是无法预知的。

- 加锁时限
	另外一个可以避免死锁的方法是在尝试获取锁的时候加一个超时时间，这也就意味着在尝试获取锁的过程中若超过了这个时限该线程则放弃对该锁请求。若一个线程没有在给定的时限内成功获得所有需要的锁，则会进行回退并释放所有已经获得的锁，然后等待一段随机的时间再重试。这段随机的等待时间让其它线程有机会尝试获取相同的这些锁，并且让该应用在没有获得锁的时候可以继续运行(译者注：加锁超时后可以先继续运行干点其它事情，再回头来重复之前加锁的逻辑)。
	以下是一个例子，展示了两个线程以不同的顺序尝试获取相同的两个锁，在发生超时后回退并重试的场景：
	```
	Thread 1 locks A
    Thread 2 locks B

    Thread 1 attempts to lock B but is blocked
    Thread 2 attempts to lock A but is blocked

    Thread 1's lock attempt on B times out
    Thread 1 backs up and releases A as well
    Thread 1 waits randomly (e.g. 257 millis) before retrying.

    Thread 2's lock attempt on A times out
    Thread 2 backs up and releases B as well
    Thread 2 waits randomly (e.g. 43 millis) before retrying
	```

	在上面的例子中，线程2比线程1早200毫秒进行重试加锁，因此它可以先成功地获取到两个锁。这时，线程1尝试获取锁A并且处于等待状态。当线程2结束时，线程1也可以顺利的获得这两个锁（除非线程2或者其它线程在线程1成功获得两个锁之前又获得其中的一些锁）。

	需要注意的是，由于存在锁的超时，所以我们不能认为这种场景就一定是出现了死锁。也可能是因为获得了锁的线程（导致其它线程超时）需要很长的时间去完成它的任务。此外，如果有非常多的线程同一时间去竞争同一批资源，就算有超时和回退机制，还是可能会导致这些线程重复地尝试但却始终得不到锁。如果只有两个线程，并且重试的超时时间设定为0到500毫秒之间，这种现象可能不会发生，但是如果是10个或20个线程情况就不同了。因为这些线程等待相等的重试时间的概率就高的多（或者非常接近以至于会出现问题）。

	> 译者注：超时和重试机制是为了避免在同一时间出现的竞争，但是当线程很多时，其中两个或多个线程的超时时间一样或者接近的可能性就会很大，因此就算出现竞争而导致超时后，由于超时时间一样，它们又会同时开始重试，导致新一轮的竞争，带来了新的问题。

	这种机制存在一个问题，在Java中不能对synchronized同步块设置超时时间。你需要创建一个自定义锁，或使用Java5中java.util.concurrent包下的工具。写一个自定义锁类不复杂，但超出了本文的内容。后续的Java并发系列会涵盖自定义锁的内容。

- 死锁检测
	死锁检测是一个更好的死锁预防机制，它主要是针对那些不可能实现按序加锁并且锁超时也不可行的场景。每当一个线程获得了锁，会在线程和锁相关的数据结构中（map、graph等等）将其记下。除此之外，每当有线程请求锁，也需要记录在这个数据结构中。
	当一个线程请求锁失败时，这个线程可以遍历锁的关系图看看是否有死锁发生。例如，线程A请求锁7，但是锁7这个时候被线程B持有，这时线程A就可以检查一下线程B是否已经请求了线程A当前所持有的锁。如果线程B确实有这样的请求，那么就是发生了死锁（线程A拥有锁1，请求锁7；线程B拥有锁7，请求锁1）。
	当然，死锁一般要比两个线程互相持有对方的锁这种情况要复杂的多。线程A等待线程B，线程B等待线程C，线程C等待线程D，线程D又在等待线程A。线程A为了检测死锁，它需要递进地检测所有被B请求的锁。从线程B所请求的锁开始，线程A找到了线程C，然后又找到了线程D，发现线程D请求的锁被线程A自己持有着。这是它就知道发生了死锁。
	下面是一幅关于四个线程（A,B,C和D）之间锁占有和请求的关系图。像这样的数据结构就可以被用来检测死锁。
	![dead-lock](http://ifeve.com/wp-content/uploads/2013/03/deadlock-detection-graph.png)

	那么当检测出死锁时，这些线程该做些什么呢？
	一个可行的做法是释放所有锁，回退，并且等待一段随机的时间后重试。这个和简单的加锁超时类似，不一样的是只有死锁已经发生了才回退，而不会是因为加锁的请求超时了。虽然有回退和等待，但是如果有大量的线程竞争同一批锁，它们还是会重复地死锁（编者注：原因同超时类似，不能从根本上减轻竞争）。另一个更好的方案是给这些线程设置优先级，让一个（或几个）线程回退，剩下的线程就像没发生死锁一样继续保持着它们需要的锁。如果赋予这些线程的优先级是固定不变的，同一批线程总是会拥有更高的优先级。为避免这个问题，可以在死锁发生的时候设置随机的优先级。

3）饥饿和公平

如果一个线程因为CPU时间全部被其他线程抢走而得不到CPU运行时间，这种状态被称之为“饥饿”。而该线程被“饥饿致死”正是因为它得不到CPU运行时间的机会。解决饥饿的方案被称之为“公平性” – 即所有线程均能公平地获得运行机会。

1. Java中导致饥饿的原因：

	- 高优先级线程吞噬所有的低优先级线程的CPU时间。
	你能为每个线程设置独自的线程优先级，优先级越高的线程获得的CPU时间越多，线程优先级值设置在1到10之间，而这些优先级值所表示行为的准确解释则依赖于你的应用运行平台。对大多数应用来说，你最好是不要改变其优先级值。

	- 线程被永久堵塞在一个等待进入同步块的状态。
	Java的同步代码区也是一个导致饥饿的因素。Java的同步代码区对哪个线程允许进入的次序没有任何保障。这就意味着理论上存在一个试图进入该同步区的线程处于被永久堵塞的风险，因为其他线程总是能持续地先于它获得访问，这即是“饥饿”问题，而一个线程被“饥饿致死”正是因为它得不到CPU运行时间的机会。

	- 线程在等待一个本身也处于永久等待完成的对象(比如调用这个对象的wait方法)。
	如果多个线程处在wait()方法执行上，而对其调用notify()不会保证哪一个线程会获得唤醒，任何线程都有可能处于继续等待的状态。因此存在这样一个风险：一个等待线程从来得不到唤醒，因为其他等待线程总是能被获得唤醒。

2. 在Java中实现公平性方案，需要:
	- 使用锁，而不是同步块。
	为了提高等待线程的公平性，我们使用锁方式来替代同步块。
	```
	public class Synchronizer{
        Lock lock = new Lock();
        public void doSynchronized() throws InterruptedException{
            this.lock.lock();
            //critical section, do a lot of work which takes a long time
            this.lock.unlock();
        }
    }
    ```

    注意到doSynchronized()不再声明为synchronized，而是用lock.lock()和lock.unlock()来替代。下面是用Lock类做的一个实现：

    ```
    public class Lock{
        private boolean isLocked = false;
        private Thread lockingThread = null;

        public synchronized void lock() throws InterruptedException{
        while(isLocked){
            wait();
        }
        isLocked = true;
        lockingThread = Thread.currentThread();
    }

    public synchronized void unlock(){
        if(this.lockingThread != Thread.currentThread()){
             throw new IllegalMonitorStateException(
                  "Calling thread has not locked this lock");
             }
            isLocked = false;
            lockingThread = null;
            notify();
        }
    }
	```

	注意到上面对Lock的实现，如果存在多线程并发访问lock()，这些线程将阻塞在对lock()方法的访问上。另外，如果锁已经锁上（校对注：这里指的是isLocked等于true时），这些线程将阻塞在while(isLocked)循环的wait()调用里面。要记住的是，当线程正在等待进入lock() 时，可以调用wait()释放其锁实例对应的同步锁，使得其他多个线程可以进入lock()方法，并调用wait()方法。
	这回看下doSynchronized()，你会注意到在lock()和unlock()之间的注释：在这两个调用之间的代码将运行很长一段时间。进一步设想，这段代码将长时间运行，和进入lock()并调用wait()来比较的话。这意味着大部分时间用在等待进入锁和进入临界区的过程是用在wait()的等待中，而不是被阻塞在试图进入lock()方法中。
	在早些时候提到过，同步块不会对等待进入的多个线程谁能获得访问做任何保障，同样当调用notify()时，wait()也不会做保障一定能唤醒线程（至于为什么，请看线程通信）。因此这个版本的Lock类和doSynchronized()那个版本就保障公平性而言，没有任何区别。但我们能改变这种情况。当前的Lock类版本调用自己的wait()方法，如果每个线程在不同的对象上调用wait()，那么只有一个线程会在该对象上调用wait()，Lock类可以决定哪个对象能对其调用notify()，因此能做到有效的选择唤醒哪个线程。

	- 公平锁。
	下面来讲述将上面Lock类转变为公平锁FairLock。你会注意到新的实现和之前的Lock类中的同步和wait()/notify()稍有不同。
	准确地说如何从之前的Lock类做到公平锁的设计是一个渐进设计的过程，每一步都是在解决上一步的问题而前进的：Nested Monitor Lockout, Slipped Conditions和Missed Signals。这些本身的讨论虽已超出本文的范围，但其中每一步的内容都将会专题进行讨论。重要的是，每一个调用lock()的线程都会进入一个队列，当解锁后，只有队列里的第一个线程被允许锁住Farlock实例，所有其它的线程都将处于等待状态，直到他们处于队列头部。

	```
    public class FairLock {
        private boolean isLocked = false;
        private Thread lockingThread = null;
        private List<QueueObject> waitingThreads = new ArrayList<QueueObject>();

      	public void lock() throws InterruptedException{
        	QueueObject queueObject   = new QueueObject();
        	boolean isLockedForThisThread = true;
            synchronized(this){
                waitingThreads.add(queueObject);
            }

        while(isLockedForThisThread){
          synchronized(this){
            isLockedForThisThread =
                isLocked || waitingThreads.get(0) != queueObject;
            if(!isLockedForThisThread){
              isLocked = true;
               waitingThreads.remove(queueObject);
               lockingThread = Thread.currentThread();
               return;
             }
          }
          try{
            queueObject.doWait();
          }catch(InterruptedException e){
            synchronized(this) { waitingThreads.remove(queueObject); }
            throw e;
          }
        }
      }

      public synchronized void unlock(){
        if(this.lockingThread != Thread.currentThread()){
          throw new IllegalMonitorStateException(
            "Calling thread has not locked this lock");
        }
        isLocked      = false;
        lockingThread = null;
        if(waitingThreads.size() > 0){
          waitingThreads.get(0).doNotify();
        }
      }
    }
	```
	```
	public class QueueObject {
        private boolean isNotified = false;
        public synchronized void doWait() throws InterruptedException {
            while(!isNotified){
                this.wait();
            }
        	this.isNotified = false;
    	}

        public synchronized void doNotify() {
            this.isNotified = true;
            this.notify();
        }

        public boolean equals(Object o) {
            return this == o;
        }
    }
	```

	首先注意到lock()方法不在声明为synchronized，取而代之的是对必需同步的代码，在synchronized中进行嵌套。FairLock新创建了一个QueueObject的实例，并对每个调用lock()的线程进行入队列。调用unlock()的线程将从队列头部获取QueueObject，并对其调用doNotify()，以唤醒在该对象上等待的线程。通过这种方式，在同一时间仅有一个等待线程获得唤醒，而不是所有的等待线程。这也是实现FairLock公平性的核心所在。

	请注意，在同一个同步块中，锁状态依然被检查和设置，以避免出现滑漏条件。还需注意到，QueueObject实际是一个semaphore。doWait()和doNotify()方法在QueueObject中保存着信号。这样做以避免一个线程在调用queueObject.doWait()之前被另一个调用unlock()并随之调用queueObject.doNotify()的线程重入，从而导致信号丢失。queueObject.doWait()调用放置在synchronized(this)块之外，以避免被monitor嵌套锁死，所以另外的线程可以解锁，只要当没有线程在lock方法的synchronized(this)块中执行即可。最后，注意到queueObject.doWait()在try – catch块中是怎样调用的。在InterruptedException抛出的情况下，线程得以离开lock()，并需让它从队列中移除。

	- 注意性能方面。
	如果比较Lock和FairLock类，你会注意到在FairLock类中lock()和unlock()还有更多需要深入的地方。这些额外的代码会导致FairLock的同步机制实现比Lock要稍微慢些。究竟存在多少影响，还依赖于应用在FairLock临界区执行的时长。执行时长越大，FairLock带来的负担影响就越小，当然这也和代码执行的频繁度相关。

##### 3、Java同步器解析
虽然许多同步器（如锁，信号量，阻塞队列等）功能上各不相同，但它们的内部设计上却差别不大。换句话说，它们内部的的基础部分是相同（或相似）的。了解这些基础部件能在设计同步器的时候给我们大大的帮助。这就是本文要细说的内容。

> 注：本文的内容是哥本哈根信息技术大学一个由Jakob Jenkov，Toke Johansen和Lars Bjørn参与的M.Sc.学生项目的部分成果。在此项目期间我们咨询Doug Lea是否知道类似的研究。有趣的是在开发Java 5并发工具包期间他已经提出了类似的结论。Doug Lea的研究，我相信，在《Java Concurrency in Practice》一书中有描述。这本书有一章“剖析同步器”就类似于本文，但不尽相同。

大部分同步器都是用来保护某个区域（临界区）的代码，这些代码可能会被多线程并发访问。要实现这个目标，同步器一般要支持下列功能：

- 状态
    同步器中的状态是用来确定某个线程是否有访问权限。在Lock中，状态是boolean类型的，表示当前Lock对象是否处于锁定状态。在BoundedSemaphore中，内部状态包含一个计数器（int类型）和一个上限（int类型），分别表示当前已经获取的许可数和最大可获取的许可数。BlockingQueue的状态是该队列中元素列表以及队列的最大容量。
	下面是Lock和BoundedSemaphore中的两个代码片段。
    ```
    public class Lock{
      //state is kept here
      private boolean isLocked = false;
      public synchronized void lock()
      throws InterruptedException{
        while(isLocked){
          wait();
        }
        isLocked = true;
      }
      ...
    }

    public class BoundedSemaphore {
      //state is kept here
      private int signals = 0;
      private int bound   = 0;

      public BoundedSemaphore(int upperBound){
        this.bound = upperBound;
      }
      public synchronized void take() throws InterruptedException{
        while(this.signals == bound) wait();
        this.signal++;
        this.notify();
      }
      ...
    }
    ```

- 访问条件
	访问条件决定调用test-and-set-state方法的线程是否可以对状态进行设置。访问条件一般是基于同步器状态的。通常是放在一个while循环里，以避免虚假唤醒问题。访问条件的计算结果要么是true要么是false。Lock中的访问条件只是简单地检查isLocked的值。根据执行的动作是“获取”还是“释放”，BoundedSemaphore中实际上有两个访问条件。如果某个线程想“获取”许可，将检查signals变量是否达到上限；如果某个线程想“释放”许可，将检查signals变量是否为0。
	这里有两个来自Lock和BoundedSemaphore的代码片段，它们都有访问条件。注意观察条件是怎样在while循环中检查的。
	```
    public class Lock{
      private boolean isLocked = false;
      public synchronized void lock()
      throws InterruptedException{
        //access condition
        while(isLocked){
          wait();
        }
        isLocked = true;
      }
      ...
    }

	public class BoundedSemaphore {
      private int signals = 0;
      private int bound = 0;

      public BoundedSemaphore(int upperBound){
        this.bound = upperBound;
      }
      public synchronized void take() throws InterruptedException{
        //access condition
        while(this.signals == bound) wait();
        this.signals++;
        this.notify();
      }
      public synchronized void release() throws InterruptedException{
        //access condition
        while(this.signals == 0) wait();
        this.signals--;
        this.notify();
      }
    }
	```

- 状态变化
	 一旦一个线程获得了临界区的访问权限，它得改变同步器的状态，让其它线程阻塞，防止它们进入临界区。换而言之，这个状态表示正有一个线程在执行临界区的代码。其它线程想要访问临界区的时候，该状态应该影响到访问条件的结果。
	在Lock中，通过代码设置isLocked = true来改变状态，在信号量中，改变状态的是signals–或signals++;这里有两个状态变化的代码片段：
	```
	public class Lock{
      private boolean isLocked = false;

      public synchronized void lock()
      throws InterruptedException{
        while(isLocked){
          wait();
        }
        //state change
        isLocked = true;
      }

      public synchronized void unlock(){
        //state change
        isLocked = false;
        notify();
      }
    }

	public class BoundedSemaphore {
      private int signals = 0;
      private int bound   = 0;

      public BoundedSemaphore(int upperBound){
        this.bound = upperBound;
      }

      public synchronized void take() throws InterruptedException{
        while(this.signals == bound) wait();
        //state change
        this.signals++;
        this.notify();
      }

      public synchronized void release() throws InterruptedException{
        while(this.signals == 0) wait();
        //state change
        this.signals--;
        this.notify();
      }
    }
	```

- 通知策略
	 一旦某个线程改变了同步器的状态，可能需要通知其它等待的线程状态已经变了。因为也许这个状态的变化会让其它线程的访问条件变为true。通知策略通常分为三种：
    - 通知所有等待的线程
    - 通知N个等待线程中的任意一个
    - 通知N个等待线程中的某个指定的线程

	 通知所有等待的线程非常简单。所有等待的线程都调用的同一个对象上的wait()方法，某个线程想要通知它们只需在这个对象上调用notifyAll()方法。通知等待线程中的任意一个也很简单，只需将notifyAll()调用换成notify()即可。调用notify方法没办法确定唤醒的是哪一个线程，也就是“等待线程中的任意一个”。有时候可能需要通知指定的线程而非任意一个等待的线程。例如，如果你想保证线程被通知的顺序与它们进入同步块的顺序一致，或按某种优先级的顺序来通知。想要实现这种需求，每个等待的线程必须在其自有的对象上调用wait()。当通知线程想要通知某个特定的等待线程时，调用该线程自有对象的notify()方法即可。饥饿和公平中有这样的例子。
	```
    //通知任意一个等待线程例子
	public class Lock{
      private boolean isLocked = false;

      public synchronized void lock()
      throws InterruptedException{
        while(isLocked){
          //wait strategy - related to notification strategy
          wait();
        }
        isLocked = true;
      }

      public synchronized void unlock(){
        isLocked = false;
        notify(); //notification strategy
      }
    }
	```

- Test-and-Set方法
	 同步器中最常见的有两种类型的方法，test-and-set是第一种（set是另一种）。Test-and-set的意思是，调用这个方法的线程检查访问条件，如若满足，该线程设置同步器的内部状态来表示它已经获得了访问权限。状态的改变通常使其它试图获取访问权限的线程计算条件状态时得到false的结果，但并不一定总是如此。例如，在读写锁中，获取读锁的线程会更新读写锁的状态来表示它获取到了读锁，但是，只要没有线程请求写锁，其它请求读锁的线程也能成功。
	test-and-set很有必要是原子的，也就是说在某个线程检查和设置状态期间，不允许有其它线程在test-and-set方法中执行。test-and-set方法的程序流通常遵照下面的顺序：
	- 如有必要，在检查前先设置状态
    - 检查访问条件
    - 如果访问条件不满足，则等待
    - 如果访问条件满足，设置状态，如有必要还要通知等待线程

	下面的ReadWriteLock类的lockWrite()方法展示了test-and-set方法。调用lockWrite()的线程在检查之前先设置状态(writeRequests++)。然后检查canGrantWriteAccess()中的访问条件，如果检查通过，在退出方法之前再次设置内部状态。这个方法中没有去通知等待线程。
    ```
	public class ReadWriteLock{
        private Map<Thread, Integer> readingThreads = new HashMap<Thread, Integer>();

        private int writeAccesses    = 0;
        private int writeRequests    = 0;
        private Thread writingThread = null;
        ...
        public synchronized void lockWrite() throws InterruptedException{
          writeRequests++;
          Thread callingThread = Thread.currentThread();
          while(! canGrantWriteAccess(callingThread)){
            wait();
          }
          writeRequests--;
          writeAccesses++;
          writingThread = callingThread;
        }
        ...
    }
	```

	下面的BoundedSemaphore类有两个test-and-set方法：take()和release()。两个方法都有检查和设置内部状态。

	```
	public class BoundedSemaphore {
      private int signals = 0;
      private int bound   = 0;

      public BoundedSemaphore(int upperBound){
        this.bound = upperBound;
      }

      public synchronized void take() throws InterruptedException{
        while(this.signals == bound) wait();
        this.signals++;
        this.notify();
      }

      public synchronized void release() throws InterruptedException{
        while(this.signals == 0) wait();
        this.signals--;
        this.notify();
      }
    }
	```

- Set方法
	set方法是同步器中常见的第二种方法。set方法仅是设置同步器的内部状态，而不先做检查。set方法的一个典型例子是Lock类中的unlock()方法。持有锁的某个线程总是能够成功解锁，而不需要检查该锁是否处于解锁状态。set方法的程序流通常如下：

    - 设置内部状态
    - 通知等待的线程

	这里是unlock()方法的一个例子：
    ```
	public class Lock{
      private boolean isLocked = false;

      public synchronized void unlock(){
        isLocked = false;
        notify();
      }
    }
	```

##### 4、Java并发算法
1)** CAS（Compare and swap）**

比较和替换是设计并发算法时用到的一种技术。简单来说，比较和替换是使用一个期望值和一个变量的当前值进行比较，如果当前变量的值与我们期望的值相等，就使用一个新值替换当前变量的值。这听起来可能有一点复杂但是实际上你理解之后发现很简单，接下来，让我们跟深入的了解一下这项技术。

`CAS的使用场景` 在程序和算法中一个经常出现的模式就是“check and act”模式。先检查后操作模式发生在代码中首先检查一个变量的值，然后再基于这个值做一些操作。下面是一个简单的示例：
```
class MyLock {
    private boolean locked = false;

    public boolean lock() {
        if(!locked) {
            locked = true;
            return true;
        }
        return false;
    }
}
```
上面这段代码，如果用在多线程的程序会出现很多错误，不过现在请忘掉它。如你所见，lock()方法首先检查locked>成员变量是否等于false，如果等于，就将locked设为true。
如果同个线程访问同一个MyLock实例，上面的lock()将不能保证正常工作。如果一个线程检查locked的值，然后将其设置为false，与此同时，一个线程B也在检查locked的值，又或者，在线程A将locked的值设为false之前。因此，线程A和线程B可能都看到locked的值为false，然后两者都基于这个信息做一些操作。为了在一个多线程程序中良好的工作，”check then act” 操作必须是原子的。原子就是说”check“操作和”act“被当做一个原子代码块执行。不存在多个线程同时执行原子块。
例如，把之前的lock()方法用synchronized关键字重构成一个原子块：
```
class MyLock {
    private boolean locked = false;

    public synchronized boolean lock() {
        if(!locked) {
            locked = true;
            return true;
        }
        return false;
    }
}
```

现在lock()方法是同步的，所以，在某一时刻只能有一个线程在同一个MyLock实例上执行它。*原子的lock方法实际上是一个”compare and swap“的例子。*

`CAS用作原子操作` 现在CPU内部已经执行原子的CAS操作。Java5以来，你可以使用java.util.concurrent.atomic包中的一些原子类来使用CPU中的这些功能。
下面是一个使用AtomicBoolean类实现lock()方法的例子：
```
public static class MyLock {
    private AtomicBoolean locked = new AtomicBoolean(false);

    public boolean lock() {
        return locked.compareAndSet(false, true);
    }
}
```

locked变量不再是boolean类型而是AtomicBoolean。这个类中有一个compareAndSet()方法，它使用一个期望值和AtomicBoolean实例的值比较，和两者相等，则使用一个新值替换原来的值。在这个例子中，它比较locked的值和false，如果locked的值为false，则把修改为true。如果值被替换了，compareAndSet()返回true，否则，返回false。

使用Java5+提供的CAS特性而不是使用自己实现的的好处是Java5+中内置的CAS特性可以让你利用底层的你的程序所运行机器的CPU的CAS特性。这会使还有CAS的代码运行更快。

2)**非阻塞算法**

在并发上下文中，非阻塞算法是一种允许线程在阻塞其他线程的情况下访问共享状态的算法。在绝大多数项目中，在算法中如果一个线程的挂起没有导致其它的线程挂起，我们就说这个算法是非阻塞的。为了更好的理解阻塞算法和非阻塞算法之间的区别，我会先讲解阻塞算法然后再讲解非阻塞算法。

`阻塞并发算法` 一个阻塞并发算法一般分下面两步：

- 执行线程请求的操作
- 阻塞线程直到可以安全地执行操作

很多算法和并发数据结构都是阻塞的。例如，java.util.concurrent.BlockingQueue的不同实现都是阻塞数据结构。如果一个线程要往一个阻塞队列中插入一个元素，队列中没有足够的空间，执行插入操作的线程就会阻塞直到队列中有了可以存放插入元素的空间。
下图演示了一个阻塞算法保证一个共享数据结构的行为：

![block-share-data](https://camo.githubusercontent.com/720fd34b938c645d6d42b41e5d694fb9d2d71a61/687474703a2f2f7475746f7269616c732e6a656e6b6f762e636f6d2f696d616765732f6a6176612d636f6e63757272656e63792f6e6f6e2d626c6f636b696e672d616c676f726974686d732d312e706e67)

`非阻塞并发算法` 一个非阻塞并发算法一般包含下面两步：

- 执行线程请求的操作
- 通知请求线程操作不能被执行

Java也包含几个非阻塞数据结构。AtomicBoolean,AtomicInteger,AtomicLong,AtomicReference都是非阻塞数据结构的例子。
下图演示了一个非阻塞算法保证一个共享数据结构的行为：

![nonblocking](https://camo.githubusercontent.com/3f3647ee8e4af4140e5414eb296d8f2d4e83a3fb/687474703a2f2f7475746f7269616c732e6a656e6b6f762e636f6d2f696d616765732f6a6176612d636f6e63757272656e63792f6e6f6e2d626c6f636b696e672d616c676f726974686d732d322e706e67)

3) **非阻塞算法 vs 阻塞算法**

阻塞算法和非阻塞算法的主要不同在于上面两部分描述的它们的行为的第二步。换句话说，它们之间的不同在于当请求操作不能够执行时阻塞算法和非阻塞算法会怎么做。阻塞算法会阻塞线程知道请求操作可以被执行。非阻塞算法会通知请求线程操作不能够被执行，并返回。

一个使用了阻塞算法的线程可能会阻塞直到有可能去处理请求。通常，其它线程的动作使第一个线程执行请求的动作成为了可能。 如果，由于某些原因线程被阻塞在程序某处，因此不能让第一个线程的请求动作被执行，第一个线程会阻塞——可能一直阻塞或者直到其他线程执行完必要的动作。例如，如果一个线程产生往一个已经满了的阻塞队列里插入一个元素，这个线程就会阻塞，直到其他线程从这个阻塞队列中取走了一些元素。如果由于某些原因，从阻塞队列中取元素的线程假定被阻塞在了程序的某处，那么，尝试往阻塞队列中添加新元素的线程就会阻塞，要么一直阻塞下去，要么知道从阻塞队列中取元素的线程最终从阻塞队列中取走了一个元素。

`非阻塞并发数据结构` 在一个多线程系统中，线程间通常通过一些数据结构”交流“。例如可以是任何的数据结构，从变量到更高级的数据结构（队列，栈等）。为了确保正确，并发线程在访问这些数据结构的时候，这些数据结构必须由一些并发算法来保证。这些并发算法让这些数据结构成为并发数据结构。
如果某个算法确保一个并发数据结构是阻塞的，它就被称为是一个阻塞算法。这个数据结构也被称为是一个**阻塞并发数据结构**。如果某个算法确保一个并发数据结构是非阻塞的，它就被称为是一个非阻塞算法。这个数据结构也被称为是一个**非阻塞并发数据结构**。
每个并发数据结构被设计用来支持一个特定的通信方法。使用哪种并发数据结构取决于你的通信需要。在接下里的部分，我会引入一些非阻塞并发数据结构，并讲解它们各自的适用场景。通过这些并发数据结构工作原理的讲解应该能在非阻塞数据结构的设计和实现上一些启发。

- **Volatile 变量**
	Java中的volatile变量是直接从主存中读取值的变量。当一个新的值赋给一个volatile变量时，这个值总是会被立即写回到主存中去。这样就确保了，一个volatile变量最新的值总是对跑在其他CPU上的线程可见。其他线程每次会从主存中读取变量的值，而不是比如线程所运行CPU的CPU缓存中。volatile变量是非阻塞的。修改一个volatile变量的值是一耳光原子操作。它不能够被中断。不过，在一个volatile变量上的一个 read-update-write 顺序的操作不是原子的。因此，下面的代码如果由多个线程执行可能导致**竞态条件**。
    ```
    volatile myVar = 0;
    ...
    int temp = myVar;
    temp++;
    myVar = temp;
    ```

    `单个写线程的情景` 在一些场景下，你仅有一个线程在向一个共享变量写，多个线程在读这个变量。当仅有一个线程在更新一个变量，不管有多少个线程在读这个变量，都不会发生竞态条件。因此，无论时候当仅有一个线程在写一个共享变量时，你可以把这个变量声明为volatile。当多个线程在一个共享变量上执行一个 read-update-write 的顺序操作时才会发生竞态条件。如果你只有一个线程在执行一个 raed-update-write 的顺序操作，其他线程都在执行读操作，将不会发生竞态条件。
    下面是一个单个写线程的例子，它没有采取同步手段但任然是并发的。
    ```
    public class SingleWriterCounter{
        private volatile long count = 0;

        /**
         *Only one thread may ever call this method
         *or it will lead to race conditions
         */
         public void inc(){
             this.count++;
         }

         /**
          *Many reading threads may call this method
          *@return
          */
          public long count(){
              return this.count;
          }
    }
    ```

    多个线程访问同一个Counter实例，只要仅有一个线程调用inc()方法，这里，我不是说在某一时刻一个线程，我的意思是，仅有相同的，单个的线程被允许去调用inc()>方法。多个线程可以调用count()方法。这样的场景将不会发生任何竞态条件。下图，说明了线程是如何访问count这个volatile变量的。

    ![Concurrency-SingleWriterCounter](https://camo.githubusercontent.com/74cd8d994d0be5bee55c55c72b3ce86a673cf86f/687474703a2f2f7475746f7269616c732e6a656e6b6f762e636f6d2f696d616765732f6a6176612d636f6e63757272656e63792f6e6f6e2d626c6f636b696e672d616c676f726974686d732d332e706e67)

- **基于volatile变量更高级的数据结构**
	使用多个volatile变量去创建数据结构是可以的，构建出的数据结构中每一个volatile变量仅被一个单个的线程写，被多个线程读。每个volatile变量可能被一个不同的线程写（但仅有一个）。使用像这样的数据结构多个线程可以使用这些volatile变量以一个非阻塞的方法彼此发送信息。下面是一个简单的例子：
	```
	public class DoubleWriterCounter{
        private volatile long countA = 0;
        private volatile long countB = 0;

        /**
         *Only one (and the same from thereon) thread may ever call this method,
         *or it will lead to race conditions.
         */
         public void incA(){
             this.countA++;
         }

         /**
          *Only one (and the same from thereon) thread may ever call this method,
          *or it will  lead to race conditions.
          */
         public void incB(){
              this.countB++;
         }

         /**
          *Many reading threads may call this method
          */
         public long countA(){
            return this.countA;
         }

        /**
         *Many reading threads may call this method
         */
        public long countB(){
           return this.countB;
        }
    }
	```

	如你所见，DoubleWriterCoounter现在包含两个volatile变量以及两对自增和读方法。在某一时刻，仅有一个单个的线程可以调用inc()，仅有一个单个的线程可以访问incB()。不过不同的线程可以同时调用incA()和incB()。countA()和countB()可以被多个线程调用。这将不会引发竞态条件。
	DoubleWriterCoounter可以被用来比如线程间通信。countA和countB可以分别用来存储生产的任务数和消费的任务数。下图，展示了两个线程通过类似于上面的一个数据结构进行通信的。

	![DoubleWriterCounter-Messages](https://camo.githubusercontent.com/97a2b4ea28a6d8fb79460d847acf047603c3b975/687474703a2f2f7475746f7269616c732e6a656e6b6f762e636f6d2f696d616765732f6a6176612d636f6e63757272656e63792f6e6f6e2d626c6f636b696e672d616c676f726974686d732d342e706e67)

    聪明的读者应该已经意识到使用两个SingleWriterCounter可以达到使用DoubleWriterCoounter的效果。如果需要，你甚至可以使用多个线程和SingleWriterCounter实例。

- **使用CAS的乐观锁**
	如果你确实需要多个线程区写同一个共享变量，volatile变量是不合适的。你将会需要一些类型的排它锁（悲观锁）访问这个变量。下面代码演示了使用Java中的同步块进行排他访问的。
	```
    public class SynchronizedCounter{
        long count = 0;

        public void inc(){
            synchronized(this){
                count++;
            }
        }

        public long count(){
            synchronized(this){
                return this.count;
            }
        }
    }
    ```

	注意inc()和count()方法都包含一个同步块。这也是我们想避免的东西——同步块和 wait()-notify 调用等。我们可以使用一种Java的原子变量来代替这两个同步块。在这个例子是AtomicLong。下面是SynchronizedCounter类的AtomicLong实现版本。
	```
	public class AtomicLong{
        private AtomicLong count = new AtomicLong(0);

        public void inc(){
            boolean updated = false;
            while(!updated){
                long prevCount = this.count.get();
                updated = this.count.compareAndSet(prevCount, prevCount + 1);
            }
        }

        public long count(){
            return this.count.get();
        }
    }
	```

	这个版本仅仅是上一个版本的线程安全版本。这一版我们感兴趣的是inc()方法的实现。inc()方法中不再含有一个同步块。而是被下面这些代码替代：
    ```
	boolean updated = false;
    while(!updated){
        long prevCount = this.count.get();
        updated = this.count.compareAndSet(prevCount, prevCount + 1);
    }
	```

	上面这些代码并不是一个原子操作。也就是说，对于两个不同的线程去调用inc()方法，然后执行long prevCount = this.count.get()语句，因此获得了这个计数器的上一个count。但是，上面的代码并没有包含任何的竞态条件。秘密就在于while循环里的第二行代码。compareAndSet()方法调用是一个原子操作。它用一个期望值和AtomicLong 内部的值去比较，如果这两个值相等，就把AtomicLong内部值替换为一个新值。compareAndSet()通常被CPU中的compare-and-swap指令直接支持。因此，不需要去同步，也不需要去挂起线程。
	假设，这个AtomicLong的内部值是20,。然后，两个线程去读这个值，都尝试调用compareAndSet(20, 20 + 1)。尽管compareAndSet()是一个原子操作，这个方法也会被这两个线程相继执行（某一个时刻只有一个）。第一个线程会使用期望值20（这个计数器的上一个值）与AtomicLong的内部值进行比较。由于两个值是相等的，AtomicLong会更新它的内部值至21（20 + 1 ）。变量updated被修改为true，while循环结束。
	现在，第二个线程调用compareAndSet(20, 20 + 1)。由于AtomicLong的内部值不再是20，这个调用将不会成功。AtomicLong的值不会再被修改为21。变量，updated被修改为false，线程将会再次在while循环外自旋。这段时间，它会读到值21并企图把值更新为22。如果在此期间没有其它线程调用inc()。第二次迭代将会成功更新AtomicLong的内部值到22。

	`为什么称它为乐观锁` 上一部分展现的代码被称为乐观锁（optimistic locking）。乐观锁区别于传统的锁，有时也被称为悲观锁。传统的锁会使用同步块或其他类型的锁阻塞对临界区域的访问。一个同步块或锁可能会导致线程挂起。乐观锁允许所有的线程在不发生阻塞的情况下创建一份共享内存的拷贝。这些线程接下来可能会对它们的拷贝进行修改，并企图把它们修改后的版本写回到共享内存中。如果没有其它线程对共享内存做任何修改，CAS操作就允许线程将它的变化写回到共享内存中去。如果，另一个线程已经修改了共享内存，这个线程将不得不再次获得一个新的拷贝，在新的拷贝上做出修改，并尝试再次把它们写回到共享内存中去。
	称之为“乐观锁”的原因就是，线程获得它们想修改的数据的拷贝并做出修改，在乐观的假在此期间没有线程对共享内存做出修改的情况下。当这个乐观假设成立时，这个线程仅仅在无锁的情况下完成共享内存的更新。当这个假设不成立时，线程所做的工作就会被丢弃，但任然不使用锁。
	乐观锁使用于共享内存竞用不是非常高的情况。如果共享内存上的内容非常多，仅仅因为更新共享内存失败，就用浪费大量的CPU周期用在拷贝和修改上。但是，如果砸共享内存上有大量的内容，无论如何，你都要把你的代码设计的产生的争用更低。

	`乐观锁是非阻塞的` 我们这里提到的乐观锁机制是非阻塞的。如果一个线程获得了一份共享内存的拷贝，当尝试修改时，发生了阻塞，其它线程去访问这块内存区域不会发生阻塞。对于一个传统的加锁/解锁模式，当一个线程持有一个锁时，其它所有的线程都会一直阻塞直到持有锁的线程再次释放掉这个锁。如果持有锁的这个线程被阻塞在某处，这个锁将很长一段时间不能被释放，甚至可能一直不能被释放。

- **不可替换的数据结构**
	简单的CAS乐观锁可以用于共享数据结果，这样一来，整个数据结构都可以通过一个单个的CAS操作被替换成为一个新的数据结构。尽管，使用一个修改后的拷贝来替换真个数据结构并不总是可行的。
	假设，这个共享数据结构是队列。每当线程尝试从向队列中插入或从队列中取出元素时，都必须拷贝这个队列然后在拷贝上做出期望的修改。我们可以通过使用一个AtomicReference来达到同样的目的。拷贝引用，拷贝和修改队列，尝试替换在AtomicReference中的引用让它指向新创建的队列。然而，一个大的数据结构可能会需要大量的内存和CPU周期来复制。这会使你的程序占用大量的内存和浪费大量的时间再拷贝操作上。这将会降低你的程序的性能，特别是这个数据结构的竞用非常高情况下。更进一步说，一个线程花费在拷贝和修改这个数据结构上的时间越长，其它线程在此期间修改这个数据结构的可能性就越大。如你所知，如果另一个线程修改了这个数据结构在它被拷贝后，其它所有的线程都不等不再次执行 拷贝-修改 操作。这将会增大性能影响和内存浪费，甚至更多。
	接下来的部分将会讲解一种实现非阻塞数据结构的方法，这种数据结构可以被并发修改，而不仅仅是拷贝和修改。

- **共享预期的修改**
	用来替换拷贝和修改整个数据结构，一个线程可以共享它们对共享数据结构预期的修改。一个线程向对修改某个数据结构的过程变成了下面这样：
	1. 检查是否另一个线程已经提交了对这个数据结构提交了修改
    2. 如果没有其他线程提交了一个预期的修改，创建一个预期的修改，然后向这个数据结构提交预期的修改
    3. 执行对共享数据结构的修改
    4. 移除对这个预期的修改的引用，向其它线程发送信号，告诉它们这个预期的修改已经被执行

	如你所见，第二步可以阻塞其他线程提交一个预期的修改。因此，第二步实际的工作是作为这个数据结构的一个锁。如果一个线程已经成功提交了一个预期的修改，其他线程就不可以再提交一个预期的修改直到第一个预期的修改执行完毕。如果一个线程提交了一个预期的修改，然后做一些其它的工作时发生阻塞，这时候，这个共享数据结构实际上是被锁住的。其它线程可以检测到它们不能够提交一个预期的修改，然后回去做一些其它的事情。很明显，我们需要解决这个问题。

	`可完成的预期修改` 为了避免一个已经提交的预期修改可以锁住共享数据结构，一个已经提交的预期修改必须包含足够的信息让其他线程来完成这次修改。因此，如果一个提交了预期修改的线程从未完成这次修改，其他线程可以在它的支持下完成这次修改，保证这个共享数据结构对其他线程可用。下图说明了上面描述的非阻塞算法的蓝图：

	![non-blocking-algorithm](https://camo.githubusercontent.com/c4d52cd702c9ad3124a62a76ee9493da43d07562/687474703a2f2f7475746f7269616c732e6a656e6b6f762e636f6d2f696d616765732f6a6176612d636f6e63757272656e63792f6e6f6e2d626c6f636b696e672d616c676f726974686d732d352e706e67)

	修改必须被当做一个或多个CAS操作来执行。因此，如果两个线程尝试去完成同一个预期修改，仅有一个线程可以所有的CAS操作。一旦一条CAS操作完成后，再次企图完成这个CAS操作都不会“得逞”。

	`A-B-A问题` 上面演示的算法可以称之为A-B-A问题。**A-B-A问题**指的是一个变量被从A修改到了B，然后又被修改回A的一种情景。其他线程对于这种情况却一无所知。如果线程A检查正在进行的数据更新，拷贝，被线程调度器挂起，一个线程B在此期可能可以访问这个共享数据结构。如果线程对这个数据结构执行了全部的更新，移除了它的预期修改，这样看起来，好像线程A自从拷贝了这个数据结构以来没有对它做任何的修改。然而，一个修改确实已经发生了。当线程A继续基于现在已经过期的数据拷贝执行它的更新时，这个数据修改已经被线程B的修改破坏。下图说明了上面提到的A-B-A问题：

	![A-B-A](https://camo.githubusercontent.com/e8ddaa9c30791bc7ed8f09c92585cb11b724558c/687474703a2f2f7475746f7269616c732e6a656e6b6f762e636f6d2f696d616765732f6a6176612d636f6e63757272656e63792f6e6f6e2d626c6f636b696e672d616c676f726974686d732d362e706e67)

	`A-B-A问题的解决方案` A-B-A通常的解决方法就是不再仅仅替换指向一个预期修改对象的指针，而是指针结合一个计数器，然后使用一个单个的CAS操作来替换指针 + 计数器。这在支持指针的语言像C和C++中是可行的。因此，尽管当前修改指针被设置回指向 “不是正在进行的修改”（no ongoing modification），指针 + 计数器的计数器部分将会被自增，使修改对其它线程是可见的。
	在Java中，你不能将一个引用和一个计数器归并在一起形成一个单个的变量。不过Java提供了AtomicStampedReference类，利用这个类可以使用一个CAS操作自动的替换一个引用和一个标记（stamp）。

- **一个非阻塞算法模板**
	下面的代码意在在如何实现非阻塞算法上一些启发。这个模板基于这篇教程所讲的东西。
    *注意：在非阻塞算法方面，我并不是一位专家，所以，下面的模板可能错误。不要基于我提供的模板实现自己的非阻塞算法。这个模板意在给你一个关于非阻塞算法大致是什么样子的一个idea。如果，你想实现自己的非阻塞算法，首先学习一些实际的工业水平的非阻塞算法的时间，在实践中学习更多关于非阻塞算法实现的知识。*
    ```
    public class NonblockingTemplate{
        public static class IntendedModification{
            public AtomicBoolean completed = new AtomicBoolean(false);
        }

        private AtomicStampedReference<IntendedModification> ongoinMod = new AtomicStampedReference<IntendedModification>(null, 0);
        //declare the state of the data structure here.

		public void modify(){
            while(!attemptModifyASR());
        }

        public boolean attemptModifyASR(){
            boolean modified = false;
            IntendedMOdification currentlyOngoingMod = ongoingMod.getReference();
            int stamp = ongoingMod.getStamp();

            if(currentlyOngoingMod == null){
                //copy data structure - for use
                //in intended modification
                //prepare intended modification
                IntendedModification newMod = new IntendModification();

                boolean modSubmitted = ongoingMod.compareAndSet(null, newMod, stamp, stamp + 1);

                if(modSubmitted){
                     //complete modification via a series of compare-and-swap operations.
                    //note: other threads may assist in completing the compare-and-swap
                    //operations, so some CAS may fail
                    modified = true;
                }
            }else{
                 //attempt to complete ongoing modification, so the data structure is freed up
                //to allow access from this thread.
                modified = false;
            }

            return modified;
        }
    }
    ```

- **非阻塞算法是不容易实现的**
	正确的设计和实现非阻塞算法是不容易的。在尝试设计你的非阻塞算法之前，看一看是否已经有人设计了一种非阻塞算法正满足你的需求。Java已经提供了一些非阻塞实现（比如 ConcurrentLinkedQueue），相信在Java未来的版本中会带来更多的非阻塞算法的实现。
	除了Java内置非阻塞数据结构还有很多开源的非阻塞数据结构可以使用。例如，LAMX Disrupter和Cliff Click实现的非阻塞 HashMap。查看[Java concurrency references page](http://tutorials.jenkov.com/java-concurrency/references.html)查看更多的资源。

- **使用非阻塞算法的好处**
	- 选择
	非阻塞算法的第一个好处是，给了线程一个选择当它们请求的动作不能够被执行时做些什么。不再是被阻塞在那，请求线程关于做什么有了一个选择。有时候，一个线程什么也不能做。在这种情况下，它可以选择阻塞或自我等待，像这样把CPU的使用权让给其它的任务。不过至少给了请求线程一个选择的机会。在一个单个的CPU系统可能会挂起一个不能执行请求动作的线程，这样可以让其它线程获得CPU的使用权。不过即使在一个单个的CPU系统阻塞可能导致死锁，线程饥饿等并发问题。

	- 没有死锁
	非阻塞算法的第二个好处是，一个线程的挂起不能导致其它线程挂起。这也意味着不会发生死锁。两个线程不能互相彼此等待来获得被对方持有的锁。因为线程不会阻塞当它们不能执行它们的请求动作时，它们不能阻塞互相等待。非阻塞算法任然可能产生活锁（live lock），两个线程一直请求一些动作，但一直被告知不能够被执行（因为其他线程的动作）。

	- 没有线程挂起
	挂起和恢复一个线程的代价是昂贵的。没错，随着时间的推移，操作系统和线程库已经越来越高效，线程挂起和恢复的成本也不断降低。不过，线程的挂起和户对任然需要付出很高的代价。无论什么时候，一个线程阻塞，就会被挂起。因此，引起了线程挂起和恢复过载。由于使用非阻塞算法线程不会被挂起，这种过载就不会发生。这就意味着CPU有可能花更多时间在执行实际的业务逻辑上而不是上下文切换。
	在一个多个CPU的系统上，阻塞算法会对阻塞算法产生重要的影响。运行在CPUA上的一个线程阻塞等待运行在CPU B上的一个线程。这就降低了程序天生就具备的并行水平。当然，CPU A可以调度其他线程去运行，但是挂起和激活线程（上下文切换）的代价是昂贵的。需要挂起的线程越少越好。

	- 降低线程延迟
	在这里我们提到的延迟指的是一个请求产生到线程实际的执行它之间的时间。因为在非阻塞算法中线程不会被挂起，它们就不需要付昂贵的，缓慢的线程激活成本。这就意味着当一个请求执行时可以得到更快的响应，减少它们的响应延迟。非阻塞算法通常忙等待直到请求动作可以被执行来降低延迟。当然，在一个非阻塞数据数据结构有着很高的线程争用的系统中，CPU可能在它们忙等待期间停止消耗大量的CPU周期。这一点需要牢牢记住。非阻塞算法可能不是最好的选择如果你的数据结构哦有着很高的线程争用。不过，也常常存在通过重构你的程序来达到更低的线程争用。

##### 5、阿姆达尔定律
	阿姆达尔定律可以用来计算处理器平行运算之后效率提升的能力。阿姆达尔定律因Gene Amdal 在1967年提出这个定律而得名。绝大多数使用并行或并发系统的开发者有一种并发或并行可能会带来提速的感觉，甚至不知道阿姆达尔定律。不管怎样，了解阿姆达尔定律还是有用的。

1) **阿姆达尔定律定义**
一个程序（或者一个算法）可以按照是否可以被并行化分为下面两个部分：

- 可以被并行化的部分
- 不可以被并行化的部分

假设一个程序处理磁盘上的文件。这个程序的一小部分用来扫描路径和在内存中创建文件目录。做完这些后，每个文件交个一个单独的线程去处理。扫描路径和创建文件目录的部分不可以被并行化，不过处理文件的过程可以。

程序串行（非并行）执行的总时间我们记为T。时间T包括不可以被并行和可以被并行部分的时间。不可以被并行的部分我们记为B。那么可以被并行的部分就是T-B。下面的列表总结了这些定义：

- T = 串行执行的总时间
- B = 不可以并行的总时间
- T - B = 并行部分的总时间

从上面可以得出：T = B + (T – B)
首先，这个看起来可能有一点奇怪，程序的可并行部分在上面这个公式中并没有自己的标识。然而，由于这个公式中可并行可以用总时间T 和 B（不可并行部分）表示出来，这个公式实际上已经从概念上得到了简化，也即是指以这种方式减少了变量的个数。
T-B 是可并行化的部分，以并行的方式执行可以提高程序的运行速度。可以提速多少取决于有多少线程或者多少个CPU来执行。线程或者CPU的个数我们记为N。
可并行化部分被执行的最快时间可以通过下面的公式计算出来：(T – B ) / N
或者通过这种方式：(1 / N) * (T – B) 维基中使用的是第二种方式。
根据阿姆达尔定律，当一个程序的可并行部分使用N个线程或CPU执行时，执行的总时间为：T(N) = B + ( T – B ) / N
T(N)指的是在并行因子为N时的总执行时间。因此，T(1)就执行在并行因子为1时程序的总执行时间。使用T(1)代替T，阿姆达尔定律定律看起来像这样：T(N) = B + (T(1) – B) / N 表达的意思都是是一样的。

2）**阿姆达尔定律图示**

首先，一个程序可以被分割为两部分，一部分为不可并行部分B，一部分为可并行部分1 – B。在顶部被带有分割线的那条直线代表总时间 T(1)。如下图：

![amdahls-law](https://camo.githubusercontent.com/f9261b92dd8f8f5e9b8f26c9234f0aec66cfb012/687474703a2f2f7475746f7269616c732e6a656e6b6f762e636f6d2f696d616765732f6a6176612d636f6e63757272656e63792f616d6461686c732d6c61772d312e706e67)

在并行因子为2的情况下的执行时间：

![amdahls-law-c2](https://camo.githubusercontent.com/0117abdf5ee0ce2c427d65957c4e3a0479738f98/687474703a2f2f7475746f7269616c732e6a656e6b6f762e636f6d2f696d616765732f6a6176612d636f6e63757272656e63792f616d6461686c732d6c61772d322e706e67)

并行因子为3的情况：

![amdahls-law-c3](https://camo.githubusercontent.com/0e706349ed74c38e39fc719a3364ca634aa12d7e/687474703a2f2f7475746f7269616c732e6a656e6b6f762e636f6d2f696d616765732f6a6176612d636f6e63757272656e63792f616d6461686c732d6c61772d332e706e67)

3) **阿姆达尔定律优化算法**

从阿姆达尔定律可以看出，程序的可并行化部分可以通过使用更多的硬件（更多的线程或CPU）运行更快。对于不可并行化的部分，只能通过优化代码来达到提速的目的。因此，你可以通过优化不可并行化部分来提高你的程序的运行速度和并行能力。你可以对不可并行化在算法上做一点改动，如果有可能，你也可以把一些移到可并行化放的部分。

`优化串行分量`如果你优化一个程序的串行化部分，你也可以使用阿姆达尔定律来计算程序优化后的执行时间。如果不可并行部分通过一个因子O来优化，那么阿姆达尔定律看起来就像这样：
`T(O, N) = B / O + (1 - B / O) / N`
记住，现在程序的不可并行化部分占了B / O的时间，所以，可并行化部分就占了1 - B / O的时间.

4) **运行时间 vs. 加速**

到目前为止，我们只用阿姆达尔定律计算了一个程序或算法在优化后或者并行化后的执行时间。我们也可以使用阿姆达尔定律计算加速比（speedup），也就是经过优化后或者串行化后的程序或算法比原来快了多少。
如果旧版本的程序或算法的执行时间为T，那么增速比就是：
`Speedup = T / T(O , N)`
为了计算执行时间，我们常常把T设为1，加速比为原来时间的一个分数。公式大致像下面这样：
`Speedup = 1 / T（O,N)`
如果我们使用阿姆达尔定律来代替T(O,N)，我们可以得到下面的公式：
`Speedup = 1 / ( B / O + (1 - B / O) / N)`
上面的计算结果可以看出，如果你通过一个因子2来优化不可并行化部分，一个因子5来并行化可并行化部分，这个程序或算法的最新优化版本最多可以比原来的版本快2.77777倍。

5) **测量，不要仅是计算**
虽然阿姆达尔定律允许你并行化一个算法的理论加速比，但是不要过度依赖这样的计算。在实际场景中，当你优化或并行化一个算法时，可以有很多的因子可以被考虑进来。内存的速度，CPU缓存，磁盘，网卡等可能都是一个限制因子。如果一个算法的最新版本是并行化的，但是导致了大量的CPU缓存浪费，你可能不会再使用x N个CPU来获得x N的期望加速。如果你的内存总线（memory bus），磁盘，网卡或者网络连接都处于高负载状态，也是一样的情况。

我们的建议是，使用阿姆达尔定律定律来指导我们优化程序，而不是用来测量优化带来的实际加速比。记住，有时候一个高度串行化的算法胜过一个并行化的算法，因为串行化版本不需要进行协调管理（上下文切换），而且一个单个的CPU在底层硬件工作（CPU管道、CPU缓存等）上的一致性可能更好。