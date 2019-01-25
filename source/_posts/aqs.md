---
title: AQS源码分析
categories: [Java, JUC]
tags: [java, JUC, AQS]
author: Mingshan
date: 2019-1-25
---

我们在使用ReentrantLock进行加锁和释放锁时可能会有好奇，这种加锁释放锁的操作和synchronized有什么区别，所以就会去翻源码，一翻源码才发现这里面的知识别有洞天，因为涉及到并发编程最基础最难理解的部分，其中AbstractQueuedSynchronizer这个类是java.util.concurrent的核心，被称为AQS，是一个同步器框架，Doug Lea
大神专门写了一篇[论文](http://gee.cs.oswego.edu/dl/papers/aqs.pdf)来介绍该框架。那么在Java世界中，同步器是一个什么概念呢？在并发世界里，涉及到对共享资源的同步操作，加锁释放锁是非常常用的，此外还需要对锁进行细粒度的控制，比如加锁时间控制、共享锁的需求等，这些复杂的需求synchronized都没有提供，那么Doug Lea就给我们提供了，而且代码写的十分优美，值得每一个Java程序员阅读和探究一番。好吧，我们开始阅读源码！（**本文基于JDK11版本**）

<!-- more -->

## 同步器的概念

上面提到同步器（Synchronizer），似乎很玄乎，不知道包含哪些内容。我们直接来阅读Doug Lea的[论文](http://gee.cs.oswego.edu/dl/papers/aqs.pdf)，在论文的INTRODUCTION
中，是这样描述的：

> Among these components are a set of synchronizers –
abstract data type (ADT) classes that maintain an internal
synchronization state (for example, representing whether a lock
is locked or unlocked), operations to update and inspect that
state, and at least one method that will cause a calling thread to
block if the state requires it, resuming when some other thread
changes the synchronization state to permit it.

上面的描述大意是：

同步器是一种抽象的数据类型（ADT），在该结构内部，维护以下内容：

1. 一个内部的同步状态（synchronization state），该变量的不同取值可以表征不同的同步状态语义（例如表示一个锁已经被线程持有了还是没有任何线程持有）；
2. 能够更新和检查该同步状态的方法集合
3. 至少一种获取（acquire）操作来阻塞当前线程，除非/直到同步状态允许许它继续执行; 并且至少有一个释放（release）操作去更改同步状态：可能允许一个或多个被阻塞的线程取消阻塞状态。

最后一条更精确的描述（论文的Functionality部分）如下：

> Synchronizers possess two kinds of methods : at least one
acquire operation that blocks the calling thread unless/until the
synchronization state allows it to proceed, and at least one
release operation that changes synchronization state in a way that
may allow one or more blocked threads to unblock.

真费劲啊，简单地了解了什么是同步器，那么我们就很迫切想了解AbstractQueuedSynchronizer到底是怎么维护上面的内容的，以及其他的同步器（ReentrantLock、CyclicBarrier、Semaphore等）是如何利用AQS来实现自己的需求的。

## AQS结构

AbstractQueuedSynchronizer简称AQS，是用abstract修饰的，基于队列（CLH）实现的一个类，在AQS的内部，使用CLH队列来管理多个抢占资源失败的线程。其中上面提到的acquire和release操作其实是在内部更改同步状态的值。

AQS是一个抽象类，当我们继承AQS去实现自己的同步器时，要做的仅仅是根据自己同步器需要满足的性质实现线程获取和释放资源的方式（修改同步状态变量的方式）即可，至于具体线程等待队列的维护（如获取资源失败入队、唤醒出队、以及线程在队列中行为的管理等），AQS在其顶层已经帮我们实现好了，AQS的这种设计使用的正是模板方法模式。

AQS支持两种模式：

- 独占模式（exclusive mode）：同一时刻只允许一个线程访问共享资源，如ReentrantLock等
  - 公平模式：获取锁失败的线程需要按照顺序排列，前面的先拿到锁
  - 非公平模式： 当线程需要获取锁时，会尝试直接获取锁
- 共享模式（shared mode）：同一时刻允多个线程访问共享资源

在AQS内部维护了一个叫CLH（Craig, Landin, and Hagersten）的队列，它是一个FIFO的队列，如下图所示：

![image](https://github.com/ZZULI-TECH/interview/blob/master/images/juc/aqs-clh.png?raw=true)

注意，阻塞队列不包含Head节点，不存储线程及锁相关信息，上面的Node节点代表AQS内部Node的内部静态类。

在AQS内部，维护了一个volatile修饰的整形变量state，该变量具有[volatile](https://en.wikipedia.org/wiki/Volatile_(computer_programming)#In_Java)语义，这是比较关键的一点（保证线程间该值的可见性）。该变量代表共享资源的共享状态，在QAS内部采用CAS更新该变量的值。代码声明如下：

```Java
/**
 * The synchronization state.
 */
private volatile int state;
```

AQS中可以修改或者获取该state值的方法有：

- protected final int getState() // 获取state的值
- protected final void setState(int newState) // 设置state的值
- protected final boolean compareAndSetState(int expect, int update) // CAS 更新state的值

注意这三个方法被protected修饰，说明子类可以直接调用这个三个方法来更改state的值，并且又被final修饰，说明这个三个方法不允许重写，只能够使用。

如果想自己实现同步器，只需继承AbstractQueuedSynchronizer类，然后重写该类的方法，可以重写哪些方法呢？如下图所示：

![image](https://github.com/ZZULI-TECH/interview/blob/master/images/juc/aqs_override_methods.png?raw=true)

所以，总结来说，子类可以重写以下方法：

- protected boolean tryAcquire(int arg) // 独占模式。 尝试获取资源
- protected boolean tryRelease(int arg) // 独占模式。 尝试释放资源
- protected int tryAcquireShared(int arg) // 共享模式。 尝试获取资源
- protected boolean tryReleaseShared(int arg) // 共享模式。 尝试释放资源
- protected boolean isHeldExclusively() // 当前线程是否独占资源

从上面的可重写的方法可以看出，自定义同步器在实现时只需要实现共享资源state的获取与释放即可，其他的无需子类关心。从这里可以看出AbstractQueuedSynchronizer定义为abstract的好处，只重写自己需要实现的逻辑，比如ReentrantLock，只需重写与独占模式相关的方法即可，共享模式的方法无需关心，编程更方便。

## 源码阅读与流程分析

```
static final class Node {
    /** Marker to indicate a node is waiting in shared mode */
    // 标识当前节点在共享模式
    static final Node SHARED = new Node();
    /** Marker to indicate a node is waiting in exclusive mode */
    // 标识当前节点在独占模式
    static final Node EXCLUSIVE = null;

    /** waitStatus value to indicate thread has cancelled. */
    // 表示当前节点所代表的的线程放弃抢占锁，并且以后不会再变，后续会被gc回收
    static final int CANCELLED =  1;
    /** waitStatus value to indicate successor's thread needs unparking. */
    // 当前线程对应的节点进行入队至队尾（挂起之前），那么其前驱节点的状态就必须为SIGNAL，以便后者取消或释放时将当前节点唤醒。
    static final int SIGNAL    = -1;
    /** waitStatus value to indicate thread is waiting on condition. */
    // Condition队列中结点的状态,CLH队列中结点没有该状态,当Condition的signal方法被调用,
    Condition队列中的结点被转移进CLH队列并且状态变为0
    static final int CONDITION = -2;
    /**
     * waitStatus value to indicate the next acquireShared should
     * unconditionally propagate.
     */
     // 与共享模式相关,当线程以共享模式去获取或释放锁时,对后续线程的释放动作需要不断往后传播
    static final int PROPAGATE = -3;

    // 节点的状态值
    // 取值为上面的1、-1、-2、-3，或者0
    volatile int waitStatus;

    // 前驱节点
    volatile Node prev;
    
    // 后继节点
    volatile Node next;

    // 当前线程
    volatile Thread thread;

    // Condition队列中指向结点在队列中的后继;在CLH队列中共享模式下值取SHARED,独占模式下为null
    Node nextWaiter;

    /**
     * Returns true if node is waiting in shared mode.
     * 判断当前节点是否处于共享模式
     */
    final boolean isShared() {
        return nextWaiter == SHARED;
    }

    /**
     * Returns previous node, or throws NullPointerException if null.
     * Use when predecessor cannot be null.  The null check could
     * be elided, but is present to help the VM.
     * 返回前驱节点
     * @return the predecessor of this node
     */
    final Node predecessor() {
        Node p = prev;
        if (p == null)
            throw new NullPointerException();
        else
            return p;
    }

    /** Establishes initial head or SHARED marker. */
    Node() {}

    /** Constructor used by addWaiter. */
    Node(Node nextWaiter) {
        this.nextWaiter = nextWaiter;
        THREAD.set(this, Thread.currentThread());
    }

    /** Constructor used by addConditionWaiter. */
    Node(int waitStatus) {
        WAITSTATUS.set(this, waitStatus);
        THREAD.set(this, Thread.currentThread());
    }

    /** CASes waitStatus field. */
    final boolean compareAndSetWaitStatus(int expect, int update) {
        return WAITSTATUS.compareAndSet(this, expect, update);
    }

    /** CASes next field. */
    final boolean compareAndSetNext(Node expect, Node update) {
        return NEXT.compareAndSet(this, expect, update);
    }

    final void setPrevRelaxed(Node p) {
        PREV.set(this, p);
    }

    // JDK9 出现的代替Unsafe类，详细参考：https://mingshan.fun/2018/10/05/use-variablehandles-to-replace-unsafe/
    // VarHandle mechanics
    private static final VarHandle NEXT;
    private static final VarHandle PREV;
    private static final VarHandle THREAD;
    private static final VarHandle WAITSTATUS;
    static {
        try {
            MethodHandles.Lookup l = MethodHandles.lookup();
            NEXT = l.findVarHandle(Node.class, "next", Node.class);
            PREV = l.findVarHandle(Node.class, "prev", Node.class);
            THREAD = l.findVarHandle(Node.class, "thread", Thread.class);
            WAITSTATUS = l.findVarHandle(Node.class, "waitStatus", int.class);
        } catch (ReflectiveOperationException e) {
            throw new ExceptionInInitializerError(e);
        }
    }
}
```

### 互斥模式

```
/**
 * 互斥锁
 */
public class Mutex implements Lock, Serializable {

    // Our internal helper class
    private static class Sync extends AbstractQueuedSynchronizer {
        // Acquires the lock if state is zero
        public boolean tryAcquire(int acquires) {
            assert acquires == 1; // Otherwise unused
            if (compareAndSetState(0, 1)) {
                setExclusiveOwnerThread(Thread.currentThread());
                return true;
            }
            return false;
        }

        // Releases the lock by setting state to zero
        protected boolean tryRelease(int releases) {
            assert releases == 1; // Otherwise unused
            if (!isHeldExclusively())
                throw new IllegalMonitorStateException();
            setExclusiveOwnerThread(null);
            setState(0);
            return true;
        }

        // Reports whether in locked state
        public boolean isLocked() {
            return getState() != 0;
        }

        public boolean isHeldExclusively() {
            // a data race, but safe due to out-of-thin-air guarantees
            return getExclusiveOwnerThread() == Thread.currentThread();
        }

        // Provides a Condition
        public Condition newCondition() {
            return new ConditionObject();
        }

        // Deserializes properly
        private void readObject(ObjectInputStream s)
                throws IOException, ClassNotFoundException {
            s.defaultReadObject();
            setState(0); // reset to unlocked state
        }
    }

    // The sync object does all the hard work. We just forward to it.
    private final Sync sync = new Sync();

    public boolean isLocked()       { return sync.isLocked(); }
    public boolean isHeldByCurrentThread() {
        return sync.isHeldExclusively();
    }
    public boolean hasQueuedThreads() {
        return sync.hasQueuedThreads();
    }

    @Override
    public void lock() {
        sync.acquire(1);
    }

    @Override
    public void lockInterruptibly() throws InterruptedException {
        sync.acquireInterruptibly(1);
    }

    @Override
    public boolean tryLock() {
        return sync.tryAcquire(1);
    }

    @Override
    public boolean tryLock(long timeout, TimeUnit unit) throws InterruptedException {
        return sync.tryAcquireNanos(1, unit.toNanos(timeout));
    }

    @Override
    public void unlock() {
        sync.release(1);
    }

    @Override
    public Condition newCondition() {
        return sync.newCondition();
    }
}
```

### 共享模式

```
public class BooleanLatch {
    private static class Sync extends AbstractQueuedSynchronizer {
        boolean isSignalled() { return getState() != 0; }

        protected int tryAcquireShared(int ignore) {
            return isSignalled() ? 1 : -1;
        }

        protected boolean tryReleaseShared(int ignore) {
            setState(1);
            return true;
        }
    }

    private final Sync sync = new Sync();
    public boolean isSignalled() { return sync.isSignalled(); }
    public void signal()         { sync.releaseShared(1); }
    public void await() throws InterruptedException {
        sync.acquireSharedInterruptibly(1);
    }
}
```


## References：

- [Java Concurrency Constructs](http://gee.cs.oswego.edu/dl/cpj/mechanics.html)
- [The java.util.concurrent Synchronizer Framework](http://gee.cs.oswego.edu/dl/papers/aqs.pdf)
- [A Java Fork/Join Framework](http://gee.cs.oswego.edu/dl/papers/fj.pdf)
- [AQS等待队列流程图](https://processon.com/view/58eb4330e4b0a578cd63bb1b)
- [一行一行源码分析清楚AbstractQueuedSynchronizer](https://javadoop.com/post/AbstractQueuedSynchronizer)
- [AQS深入理解与实战----基于JDK1.8](https://www.cnblogs.com/awakedreaming/p/9510021.html)
- [Java并发之AQS详解](https://www.cnblogs.com/waterystone/p/4920797.html)
- [ReentrantLock源码笔记 - 获取锁（JDK 1.8）](https://mingshan.fun/2017/11/10/reentrantlock-get-lock/)
- [用Variable Handles来替换Unsafe](https://mingshan.fun/2018/10/05/use-variablehandles-to-replace-unsafe/)