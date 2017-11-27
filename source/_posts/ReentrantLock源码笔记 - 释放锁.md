---
title: ReentrantLock源码笔记 - 释放锁（JDK 1.8）
categories: [Java, JUC]
tags: [java, Lock, JUC, ReentrantLock]
author: Mingshan
---
## ReentrantLock源码学习 - 释放锁（unlock）
* * *
上次谈到了利用ReentrantLock的非公平和公平加锁方式，那么接下来看看释放锁的流程

首先调用ReentrantLock的unlock方法

```java
public void unlock() {
    sync.release(1);
}

```
然后会调用AbstractQueuedSynchronizer（AQS）的release方法，在这个方法中首先会调用ReentrantLock的Sync的tryRelease方法，来进行尝试释放锁，如果返回true，那么获取CLH队列的头结点，判断头结点不为空并且头结点的状态不为0（None），那么就调用AQS的unparkSuccessor方法。

```java
public final boolean release(int arg) {
    if (tryRelease(arg)) {
        Node h = head;
        if (h != null && h.waitStatus != 0)
            unparkSuccessor(h);
        return true;
    }
    return false;
}
```

在tryRelease方法里，首先让当前的state与传入的值（这里为1）进行相减，然后得到c，判断当前线程是不是获取独占锁的线程，如果不是，直接抛出异常；如果是，那么需要判断c是否为0，因为只有c为0时，才符合释放独占锁的条件，这是设置独占锁线程为null，最后设置下state的值（注意这里c为0不为0都会设置）

```java
protected final boolean tryRelease(int releases) {
    int c = getState() - releases;
    if (Thread.currentThread() != getExclusiveOwnerThread())
        throw new IllegalMonitorStateException();
    boolean free = false;
    if (c == 0) {
        free = true;
        setExclusiveOwnerThread(null);
    }
    setState(c);
    return free;
}
```

接下来来看方法unparkSuccessor，该方法的作用就是为了释放node节点的后继结点。

```java
private void unparkSuccessor(Node node) {
    /*
     * If status is negative (i.e., possibly needing signal) try
     * to clear in anticipation of signalling.  It is OK if this
     * fails or if status is changed by waiting thread.
     */
     // 获取节点的状态
    int ws = node.waitStatus;
    if (ws < 0)
        compareAndSetWaitStatus(node, ws, 0); // 利用CAS 将状态设置为0

    /*
     * Thread to unpark is held in successor, which is normally
     * just the next node.  But if cancelled or apparently null,
     * traverse backwards from tail to find the actual
     * non-cancelled successor.
     */
    // 获取节点的后继节点
    Node s = node.next;
    // 判断后继节点是否为空 或者 后者后继节点的状态为CANCELLED
    if (s == null || s.waitStatus > 0) {
        s = null; // 将后继节点置为null
        // 从尾节点从后向前开始遍历知道节点为空或者当前节点为止
        for (Node t = tail; t != null && t != node; t = t.prev)
            if (t.waitStatus <= 0) // 如果此时节点的状态小于等于0
                s = t; // 将此节点赋给传入节点的后继节点
    }
    if (s != null)  // 节点不为空，释放
        LockSupport.unpark(s.thread);
}
```

### 参考：
http://blog.csdn.net/luonanqin/article/details/41871909
