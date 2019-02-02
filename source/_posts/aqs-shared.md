---
title: AQS源码分析-共享模式
categories: [Java, JUC]
tags: [java, JUC, AQS]
author: Mingshan
date: 2019-2-2
---

上篇文章[AQS源码分析-独占模式](https://mingshan.fun/2019/01/25/aqs-exclusive/)分析了AQS的结构以及独占模式下资源的获取与释放流程，啰嗦了AQS的基本结构和独占模式。这篇文章主要是探讨下AQS在共享模式下资源的获取与释放，同时比较下两种模式的差异（**本文基于JDK11版本**）。

<!-- more -->

## 流程分析 - 获取资源
这篇文章以CountDownLatch为例，和独占模式一样，AQS同样提供了资源的获取与释放的方法供子类进行重写，如下所示：

```Java
protected int tryAcquireShared(int arg)
protected boolean tryReleaseShared(int arg)
```

[CountDownLatch](https://docs.oracle.com/en/java/javase/11/docs/api/java.base/java/util/concurrent/CountDownLatch.html)的官方描述如下：

> A synchronization aid that allows one or more threads to wait until a set of operations being performed in other threads completes.

简单来说，CountDownLatch是一个同步器用来使一个或者多个线程等待其他线程完成各自的工作后再执行。是不是像一个计数器？本来就是一个计数器，只不过支持并发操作。

注意CountDownLatch提供构造函数用来初始化需要等待的线程数量，其实就是设置AQS中state的具体值，如下：

```Java
public CountDownLatch(int count) {
    if (count < 0) throw new IllegalArgumentException("count < 0");
    this.sync = new Sync(count);
}

Sync(int count) {
    setState(count);
}
```

### AQS#acquireSharedInterruptibly

CountDownLatch提供了await方法用来阻塞当前线程直到CountDownLatch内部计数为0。所以我们先从await方法开始：

```Java
public void await() throws InterruptedException {
    sync.acquireSharedInterruptibly(1);
}
```

`acquireSharedInterruptibly(1)`方法是AQS类中的，源码如下：

```
public final void acquireSharedInterruptibly(int arg)
        throws InterruptedException {
    if (Thread.interrupted())
        throw new InterruptedException();
    if (tryAcquireShared(arg) < 0)
        doAcquireSharedInterruptibly(arg);
}
```

从acquireSharedInterruptibly方法的实现中可以看出，该方法响应中断，如果当前线程被中断，抛出中断异常。然后调用`tryAcquireShared(arg)`方法，注意该方法的返回值是一个整数，这个限定了返回值的语义，如果小于0，证明获取共享资源失败，需要进入阻塞队列。我们来看看CountDownLatch提供的tryAcquireShared实现：

```Java
protected int tryAcquireShared(int acquires) {
    return (getState() == 0) ? 1 : -1;
}
```

从上面的方法中可以看出，调用了getState方法获取state的值，如果值为0，返回1，代表获取共享资源成功；否则返回-1，代表获取共享资源失败。

总结来说，子类在tryAcquireShared的实现上需要注意以下几点：

1. 检查当前是否支持在共享模式下获取资源
2. 返回值语义
   1. 如果返回值小于0，说明获取资源失败，需要进入阻塞队列
   2. 返回值等于0，说明获取资源成功，但后续的节点携带的线程无法唤醒，无法继续获取资源
   3. 返回值大于0，说明获取资源成功，但会唤醒后续节点然后尝试获取资源


## AQS#doAcquireSharedInterruptibly

我们回到AQS的acquireSharedInterruptibly方法，当tryAcquireShared返回值小于0，就会执行`doAcquireSharedInterruptibly(1)`方法，代码如下：

```Java
/**
 * Acquires in shared interruptible mode.
 * @param arg the acquire argument
 */
private void doAcquireSharedInterruptibly(int arg)
    throws InterruptedException {
    final Node node = addWaiter(Node.SHARED);
    try {
        for (;;) {
            final Node p = node.predecessor();
            if (p == head) {
                int r = tryAcquireShared(arg);
                if (r >= 0) {
                    setHeadAndPropagate(node, r);
                    p.next = null; // help GC
                    return;
                }
            }
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                throw new InterruptedException();
        }
    } catch (Throwable t) {
        cancelAcquire(node);
        throw t;
    }
}
```

在doAcquireSharedInterruptibly方法中，首先会调用`addWaiter(Node.SHARED)`方法，该方法在[AQS源码分析-独占模式#AQS#addWaiter方法](https://mingshan.fun/2019/01/25/aqs-exclusive/#AQS-addWaiter%E6%96%B9%E6%B3%95)已经说过，主要作用是初始化CLH队列或将新创建的节点（携带当前线程）成功入队至队尾，最后返回该新建的节点。只不过此时的参数是Node.SHARED，代表共享模式。

接着就是一个无限for循环（这个非常常见），在循环体内，首先获取当前节点的前驱节点p，如果p为头结点，说明当前节点为阻塞队列第一个线程的节点，那么就调用tryAcquireShared(1)直接尝试获取资源，因为前面已经没有持有线程的节点了。如果返回值大于等于0，说明此时state的值已为0，获取资源成功了，接下来调用`setHeadAndPropagate(node, r)`方法，代码如下：


```Java
/**
 * Sets head of queue, and checks if successor may be waiting
 * in shared mode, if so propagating if either propagate > 0 or
 * PROPAGATE status was set.
 *
 * @param node the node
 * @param propagate the return value from a tryAcquireShared
 */
private void setHeadAndPropagate(Node node, int propagate) {
    Node h = head; // Record old head for check below
    setHead(node);
    /*
     * Try to signal next queued node if:
     *   Propagation was indicated by caller,
     *     or was recorded (as h.waitStatus either before
     *     or after setHead) by a previous operation
     *     (note: this uses sign-check of waitStatus because
     *      PROPAGATE status may transition to SIGNAL.)
     * and
     *   The next node is waiting in shared mode,
     *     or we don't know, because it appears null
     *
     * The conservatism in both of these checks may cause
     * unnecessary wake-ups, but only when there are multiple
     * racing acquires/releases, so most need signals now or soon
     * anyway.
     */
    if (propagate > 0 || h == null || h.waitStatus < 0 ||
        (h = head) == null || h.waitStatus < 0) {
        Node s = node.next;
        if (s == null || s.isShared())
            doReleaseShared();
    }
}
```

setHeadAndPropagate方法有两个参数，第一个参数node为已经成功获取资源的节点，第二个参数propagate为tryAcquireShared方法的返回值，注意此时进入该方法时propagate的值大于或者等于0。

首先会记录原来CLH队列旧的头结点，然后调用`setHead(node)`方法设置头结点，这个方法如下：

```Java
private void setHead(Node node) {
    head = node;
    node.thread = null;
    node.prev = null;
}
```

接下来有一个if判断，貌似很长的样子，我们来具体分析一下：

首先`propagate > 0`代表当前线程已经获取到了资源，并且需要唤醒后面阻塞的节点；h.waitStatus < 0 代表旧的头节点后面的节点可以被唤醒；`(h = head) == null || h.waitStatus < 0` 这个操作是说新的头节点后面的节点可以被唤醒，总结来说：

1. `propagate > 0`代表当前线程已经获取到了资源，并且需要唤醒后面阻塞的节点
2. 无论新旧头节点，只要其waitStatus < 0，那么其后面的节点可以被唤醒

如果上面if返回true，接着获取当前节点的后继节点，这里又会有一个判断，如果后继节点是共享模式或者现在还看不到后继的状态，则都继续唤醒后继节点中的线程。


```Java
/**
 * Release action for shared mode -- signals successor and ensures
 * propagation. (Note: For exclusive mode, release just amounts
 * to calling unparkSuccessor of head if it needs signal.)
 */
private void doReleaseShared() {
    /*
     * Ensure that a release propagates, even if there are other
     * in-progress acquires/releases.  This proceeds in the usual
     * way of trying to unparkSuccessor of head if it needs
     * signal. But if it does not, status is set to PROPAGATE to
     * ensure that upon release, propagation continues.
     * Additionally, we must loop in case a new node is added
     * while we are doing this. Also, unlike other uses of
     * unparkSuccessor, we need to know if CAS to reset status
     * fails, if so rechecking.
     */
    for (;;) {
        Node h = head;
        if (h != null && h != tail) {
            int ws = h.waitStatus;
            if (ws == Node.SIGNAL) {
                if (!h.compareAndSetWaitStatus(Node.SIGNAL, 0))
                    continue;            // loop to recheck cases
                unparkSuccessor(h);
            }
            else if (ws == 0 &&
                     !h.compareAndSetWaitStatus(0, Node.PROPAGATE))
                continue;                // loop on failed CAS
        }
        if (h == head)                   // loop if head changed
            break;
    }
}
```




## References：

- [Java Concurrency Constructs](http://gee.cs.oswego.edu/dl/cpj/mechanics.html)
- [The java.util.concurrent Synchronizer Framework](http://gee.cs.oswego.edu/dl/papers/aqs.pdf)
- [A Java Fork/Join Framework](http://gee.cs.oswego.edu/dl/papers/fj.pdf)
- [AQS源码分析-独占模式](https://mingshan.fun/2019/01/25/aqs-exclusive/AQS源码分析-独占模式)
- [AQS深入理解与实战----基于JDK1.8](https://www.cnblogs.com/awakedreaming/p/9510021.html)