---
title: AQS源码分析-Condition与LockSupport
categories: [Java, JUC]
tags: [java, JUC, AQS]
author: Mingshan
date: 2019-3-4
---

在[从生产者消费者模型窥探多线程并发](https://mingshan.fun/2019/02/25/producer-consumer/)这篇文章中我们使用了ReentrantLock结合Condition实现生产者消费者模型，但我们对于ReentrantLock和Condition的工作原理并不了解，其内部的结构和源码级别实现就更加不了解了。比如在使用await方法的时候，为什么一定要用while判断条件，用if为什么不行呢？使用Condition时，线程之间是如何通信的呢？与synchronized有什么区别？如果后续要用到BlockingQueue，其内部的实现是离不开ReentrantLock和Condition的，所以对于ReentrantLock和Condition源码级别的分析是十分有必要的。

<!-- more -->

## 管程(Monitors)

在上篇文章中我们谈到synchronized与wait、notify结合来实现生产者消费者，其实它与操作系统级别的管程概念联系甚大。管程(monitor)在维基百科中有如下描述（我就直接Copy了^_^）：

管程 (英语：Monitors，也称为监视器) 是一种程序结构，结构内的多个子程序（对象或模块）形成的多个工作线程互斥访问共享资源。这些共享资源一般是硬件设备或一群变量。管程实现了在一个时间点，最多只有一个线程在执行管程的某个子程序。与那些通过修改数据结构实现互斥访问的并发程序设计相比，管程实现很大程度上简化了程序设计。

管程提供了一种机制，线程可以临时放弃互斥访问，等待某些条件得到满足后，重新获得执行权恢复它的互斥访问。

管程实现对共享资源的互斥访问，一个管程包含:

- 多个彼此可以交互并共享资源的线程
- 多个与资源使用有关的变量
- 一个互斥锁
- 一个用来避免竞态条件的不变量

### 条件变量(Condition Variable)
对于许多应用场合，互斥操作是不够用的。线程可能需要等待某个条件 `P`为真，才能继续执行。在一个忙碌等待循环中

```
  while not( P ) do skip
```

将会导致所有其它进程都无法进入临界区使得该条件`P`为真，该管程发生死锁.

解决办法是条件变量(condition variables). 概念上，一个条件变量就是一个线程队列(queue), 其中的线程正等待某个条件变为真。每个条件变量 `c` 关联着一个断言$P_c$.当一个线程等待一个条件变量，该线程不算作占用了该管程，因而其它线程可以进入该管程执行，改变管程的状态，通知条件变量`c`其关联的断言$P_c$在当前状态下为真.

因此对条件变量存在两种主要操作:

- wait c 被一个线程调用，以等待断言 $P_c$ 被满足后该线程可恢复执行. 线程挂在该条件变量上等待时，不被认为是占用了管程.
- signal c (有时写作notify c)被一个线程调用，以指出断言$P_c$现在为真.
在下述例子中, 用管程实现了一个信号量. 一个私有整型变量s需要被互斥访问。管程中定义了子程序“增加”(V)与子程序“减少”(P)，整型变量s不能被减少到小于0; 因此子程序“减少”必须等到该整型变量是正数时才可执行. 使用条件变量sIsPositive与相关联的断言 $P_sIsPositive = (s>0)$.

```
monitor class Semaphore
{
  private int s := 0
  invariant s >= 0
  private Condition sIsPositive /* associated with s > 0 */

  public method P()
  {
    if s = 0 then wait sIsPositive
    assert s > 0
    s := s - 1
  }

  public method V()
  {
    s := s + 1
    assert s > 0
    signal sIsPositive
  }
}
```

当一个通知(signal)发给了一个有线程处于等待中的条件变量，则有至少两个线程将要占用该管程: 发出通知的线程与等待该通知的某个线程. 只能有一个线程占用该管程，因此必须做出选择。两种理论体系导致了两种不同的条件变量的实现:

- 阻塞式条件变量(Blocking condition variables)，把优先级给了被通知的线程.
- 非阻塞式条件变量(Nonblocking condition variables)，把优先级给了发出通知的线程.

看过上面维基百科对管程的介绍，发现上面的概念仿佛Java已经实现了，不错，synchronized与wait、notify 以及Java1.5引入的Lock、Condition，就是对管程相关实现。找到了Condition的理论基础，相信我们对Condition的用途与实现方式有个初步的认识。下面我们就用Lock与Condition结合实现的生产者消费者例子来分析Condition的实现原理。

## Condition源码分析

对于ReentrantLock的源码分析，请参考: [ReentrantLock源码笔记 - 获取锁（JDK 1.8）](https://mingshan.fun/2017/11/10/reentrantlock-get-lock/)和 [AQS源码分析-独占模式](https://mingshan.fun/2019/01/25/aqs-exclusive/)，在此不再详细分析，所以我们详细研究下Condition的实现。


下面是Lock与Condition结合实现的生产者消费者例子，这个代码流程已经在上一篇文章分析过了，代码如下:

```Java
public class Buffer {
    private List<String> queue;
    private int size;

    private final Lock lock = new ReentrantLock();
    // 读线程条件
    private final Condition notEmpty = lock.newCondition();
    // 写线程条件
    private final Condition notNull = lock.newCondition();

    public Buffer() {
        this(10);
    }

    public Buffer(int size) {
        this.size = size;
        queue = new ArrayList<>();
    }

    public void put() throws InterruptedException {
        lock.lock();
        try {
            while (queue.size() == size) {
                System.out.println("[Put] Current thread " + Thread.currentThread().getName() + " is waiting");
                notNull.await();
            }

            queue.add("1");
            System.out.println("[Put] Current thread " + Thread.currentThread().getName()
                    + " add 1 item, current count: " + queue.size());
            notEmpty.signal();
        } finally {
            lock.unlock();
        }
    }

    public void take() throws InterruptedException {
        lock.lock();
        try {
            while (queue.size() == 0) {
                System.out.println("[Take] Current thread " + Thread.currentThread().getName() + " is waiting");
                notEmpty.await();
            }

            System.out.println("size = " + queue.size());

            queue.remove(queue.size() - 1);
            System.out.println("[Take] Current thread " + Thread.currentThread().getName()
                    + " remove 1 item, current count: " + queue.size());
            notNull.signal();
        } finally {
            lock.unlock();
        }

    }
}
```

在使用Condition之前，当前线程必须获得锁，然后调用`lock.newCondition()`获取一个Condition对象，这个操作是在AQS中的，ConditionObject是AQS的内部类，下面是该类的一部分代码：

```Java
/**
 * Condition implementation for a {@link AbstractQueuedSynchronizer}
 * serving as the basis of a {@link Lock} implementation.
 *
 * <p>Method documentation for this class describes mechanics,
 * not behavioral specifications from the point of view of Lock
 * and Condition users. Exported versions of this class will in
 * general need to be accompanied by documentation describing
 * condition semantics that rely on those of the associated
 * {@code AbstractQueuedSynchronizer}.
 *
 * <p>This class is Serializable, but all fields are transient,
 * so deserialized conditions have no waiters.
 */
public class ConditionObject implements Condition, java.io.Serializable {
    private static final long serialVersionUID = 1173984872572414699L;
    /** First node of condition queue. */
    private transient Node firstWaiter;
    /** Last node of condition queue. */
    private transient Node lastWaiter;

    /**
     * Creates a new {@code ConditionObject} instance.
     */
    public ConditionObject() { }

    // Internal methods

    ......
}
```

在维基百科中，我们了解到：一个条件变量就是一个线程队列(queue)，所以Condition的实现也应该是基于队列的。从上面代码可以发现，ConditionObject有两个成员变量：

```Java
/** First node of condition queue. */
private transient Node firstWaiter;
/** Last node of condition queue. */
private transient Node lastWaiter;
```

`firstWaiter`与`lastWaiter`分别是condition queue的第一个和最后一个结点，所以Condition自己维护了一个队列，那这个队列和CLH有没有关系呢？我们下面接着来看。

### await

在上面的生产者消费者代码的put方法中，当队列满了，会调用`notNull.await()`方法，那么这个方法做了什么操作呢？代码如下：

```Java
/**
 * Implements interruptible condition wait.
 * <ol>
 * <li>If current thread is interrupted, throw InterruptedException.
 * <li>Save lock state returned by {@link #getState}.
 * <li>Invoke {@link #release} with saved state as argument,
 *     throwing IllegalMonitorStateException if it fails.
 * <li>Block until signalled or interrupted.
 * <li>Reacquire by invoking specialized version of
 *     {@link #acquire} with saved state as argument.
 * <li>If interrupted while blocked in step 4, throw InterruptedException.
 * </ol>
 */
public final void await() throws InterruptedException {
    if (Thread.interrupted())
        throw new InterruptedException();
    Node node = addConditionWaiter();
    int savedState = fullyRelease(node);
    int interruptMode = 0;
    while (!isOnSyncQueue(node)) {
        LockSupport.park(this);
        if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
            break;
    }
    if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
        interruptMode = REINTERRUPT;
    if (node.nextWaiter != null) // clean up if cancelled
        unlinkCancelledWaiters();
    if (interruptMode != 0)
        reportInterruptAfterWait(interruptMode);
}
```

从上面代码第一行可以看出，await方法是响应中断的，如果检测到线程中断，就会立即抛出InterruptedException。接着调用`addConditionWaiter()`方法，代码如下：

```Java
/**
 * Adds a new waiter to wait queue.
 * @return its new wait node
 */
private Node addConditionWaiter() {
    if (!isHeldExclusively())
        throw new IllegalMonitorStateException();
    Node t = lastWaiter;
    // If lastWaiter is cancelled, clean out.
    if (t != null && t.waitStatus != Node.CONDITION) {
        unlinkCancelledWaiters();
        t = lastWaiter;
    }

    Node node = new Node(Node.CONDITION);

    if (t == null)
        firstWaiter = node;
    else
        t.nextWaiter = node;
    lastWaiter = node;
    return node;
}
```

在`addConditionWaiter()`方法中，首先会调用`isHeldExclusively()`当前线程是否处于独占模式，注意该方法的实现在AQS的子类中，这里就相当于在ReentrantLock中，代码如下：

```Java
protected final boolean isHeldExclusively() {
    // While we must in general read state before owner,
    // we don't need to do so to check if current thread is owner
    return getExclusiveOwnerThread() == Thread.currentThread();
}
```

可见Condition是用在独占模式下的，共享模式下会直接报错。接着获取 队列中最后一个节点`t = lastWaiter`，并且判断t是否为空，并且t的waitStatus是否为`Node.CONDITION`，根据上面的注释，如果t的waitStatus不是`Node.CONDITION`，证明该节点已经被取消了，需要将其从队列中移除，会调用`unlinkCancelledWaiters()`方法，该方法源码如下：

```Java
/**
 * Unlinks cancelled waiter nodes from condition queue.
 * Called only while holding lock. This is called when
 * cancellation occurred during condition wait, and upon
 * insertion of a new waiter when lastWaiter is seen to have
 * been cancelled. This method is needed to avoid garbage
 * retention in the absence of signals. So even though it may
 * require a full traversal, it comes into play only when
 * timeouts or cancellations occur in the absence of
 * signals. It traverses all nodes rather than stopping at a
 * particular target to unlink all pointers to garbage nodes
 * without requiring many re-traversals during cancellation
 * storms.
 */
private void unlinkCancelledWaiters() {
    Node t = firstWaiter; // 获取头节点
    Node trail = null;  // 表示当前节点的上一个节点
    while (t != null) { // 遍历队列
        Node next = t.nextWaiter; // 获取后继节点
        // 判断waitStatus是否为 Node.CONDITION
        if (t.waitStatus != Node.CONDITION) { 
            t.nextWaiter = null; // 置空当前节点的后继，等待垃圾回收
            if (trail == null) 
                firstWaiter = next; // 让当前节点的后继节点作为头结点
            else
                trail.nextWaiter = next; // 由于当前节点不符合要求，需要被踢出队列，只好用上一个节点来链接next节点
            if (next == null) // 判断后继节点是否为空
                // 如果next为空，代表当前节点没有后继节点，注意此时当前节点的waitStatus不等于 Node.CONDITION，
                // 上面的操作已经被置空了，等待被垃圾回收，
                // 就将上一个waitStatus是否为 Node.CONDITION的结点作为尾节点
                lastWaiter = trail; 
        }
        else
            // 如果当前节点的waitStatus为Node.CONDITION，
            // 那么将当前节点赋值给trail，相当于暂存一下
            trail = t; 
        t = next; // 继续向后遍历
    }
}
```

在`unlinkCancelledWaiters()`方法中，首先获取队列的头结点`t = firstWaiter`，并且有一个变量为trail，初始化为null，表示当前节点的上一个节点，接下来的操作如果同学对单链表的移除节点操作熟悉的话，那么就没必要说了，只不过被移除的条件是`t.waitStatus != Node.CONDITION`，具体流程请参考上面的注释。总结来说，`unlinkCancelledWaiters()`方法是踢出队列中waitStatus不是`Node.CONDITION`的节点。

让我们回到`addConditionWaiter()`方法

References：

- [入门AQS锁 - Condition与LockSupport](https://www.jianshu.com/p/1add173ea703)
- [操作系统-进程（6）管程](https://www.cnblogs.com/yangyuliufeng/p/9609419.html)
- [管程 - 维基百科](https://zh.wikipedia.org/wiki/%E7%9B%A3%E8%A6%96%E5%99%A8_(%E7%A8%8B%E5%BA%8F%E5%90%8C%E6%AD%A5%E5%8C%96))
- [condition实现原理](https://www.cnblogs.com/nevermorewang/p/9905939.html)


[<font size=3 color="#409EFF">向本文提出修改或勘误建议</font>](https://github.com/mstao/mstao.github.io/blob/hexo/source/_posts/aqs-Condition-LockSupport.md)