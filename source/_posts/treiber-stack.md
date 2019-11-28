---
title: Treiber stack设计
tags: [数据结构, 栈, JUC]
author: Mingshan
categories: [数据结构, 栈, JUC]
date: 2019-11-25
---

最近看JDK11的CompletableFuture源码实现时，发现内部使用了[Treiber stack](https://en.wikipedia.org/wiki/Treiber_stack)，维基百科上作以下描述：

> The Treiber stack algorithm is a scalable lock-free stack utilizing the fine-grained concurrency primitive compare-and-swap

Treiber stack算法是属于无锁并发栈，内部使用CAS(compare-and-swap)来实现无锁并发算法。关于CAS想必大家都很熟悉，至少会用。我们先来看看CompletableFuture的无锁并发栈的实现。

<!-- more -->

## CompletableFuture的Treiber stack

在CompletableFuture内部，有一个成员变量stack，并且用`volatile`修饰，这个属性是最作为栈的最顶端元素，入栈和出栈都要操作这个值，必然是会出现线程并发问题，`volatile`是为了保证该值在多线程情况下内存可见性，且禁止指令重排序，但不保证原子性，所以对这个值的原子操作需要另外用到CAS，如下所示:


```Java
volatile Completion stack;    // Top of Treiber stack of dependent actions
```

其中`Completion`是一个抽象类，继承和实现了一大堆东西，这个我们不用去看，代码如下：

```Java
abstract static class Completion extends ForkJoinTask<Void>
    implements Runnable, AsynchronousCompletionTask {
    volatile Completion next;      // Treiber stack link

    /**
     * Performs completion action if triggered, returning a
     * dependent that may need propagation, if one exists.
     *
     * @param mode SYNC, ASYNC, or NESTED
     */
    abstract CompletableFuture<?> tryFire(int mode);

    /** Returns true if possibly still triggerable. Used by cleanStack. */
    abstract boolean isLive();

    public final void run()                { tryFire(ASYNC); }
    public final boolean exec()            { tryFire(ASYNC); return false; }
    public final Void getRawResult()       { return null; }
    public final void setRawResult(Void v) {}
}
```

注意其内部的`next` 也用`volatile`修饰了，这是做了什么考虑呢？ 具体的`Completion`的子类我们也不用去看，我们来看下哪里用到这个类了，在`postComplete`方法中，调用了JDK9引入的VarHandle的`compareAndSet`方法，来做CAS，我们来看下`postComplete`方法的源码：

```Java
/**
 * Pops and tries to trigger all reachable dependents.  Call only
 * when known to be done.
 */
final void postComplete() {
    /*
     * On each step, variable f holds current dependents to pop
     * and run.  It is extended along only one path at a time,
     * pushing others to avoid unbounded recursion.
     */
    CompletableFuture<?> f = this; Completion h;
    while ((h = f.stack) != null ||
           (f != this && (h = (f = this).stack) != null)) {
        CompletableFuture<?> d; Completion t;
        if (STACK.compareAndSet(f, h, t = h.next)) {
            if (t != null) {
                if (f != this) {
                    pushStack(h);
                    continue;
                }
                NEXT.compareAndSet(h, t, null); // try to detach
            }
            f = (d = h.tryFire(NESTED)) == null ? this : d;
        }
    }
}
```

在`postComplete`方法中，只针对入栈逻辑，入栈的代码如下：


```Java
/** Unconditionally pushes c onto stack, retrying if necessary. */
final void pushStack(Completion c) {
    do {} while (!tryPushStack(c));
}

/** Returns true if successfully pushed c onto stack. */
final boolean tryPushStack(Completion c) {
    Completion h = stack;
    NEXT.set(c, h);         // CAS piggyback
    return STACK.compareAndSet(this, h, c);
}
```

所以简单来看，入栈的步骤如下：

1. 尝试入栈，利用CAS将新的节点作为栈顶元素，新节点next赋值为旧栈顶元素
2. 尝试入栈成功，即结束；入栈失败，继续重试上面的操作

## 抽象实现

上面JDK中CompletableFuture的Treiber stack实现感觉有点复杂，因为有其他逻辑掺杂，代码不容易阅读，其实抽象来看，Treiber stack首先是个单向链表，链表头部即栈顶元素，在入栈和出现过程中，需要对栈顶元素进行CAS控制，防止多线程情况下数据错乱。

对应入栈过程，伪代码如下：

```Java
void push(Node new) {
  do {
      
  } while(!tryPush(new)) // 尝试入栈
}

boolean tryPush(node) {
    Node oldTop = top;
    node.next = oldTop; // 新节点next赋值为旧栈顶元素
    return CAS(oldTop, node); // 利用CAS将新的节点作为栈顶元素
}

```

对于出栈，要做的工作就是将原来的栈顶节点移除，等待垃圾回收；新栈顶元素CAS为第一个子元素。伪代码：

```
String pop() {
    Node<E> oldTop = top;
    // 判断栈是否为空，为空直接返回
    if (oldTop == null) 
        return null;
    
    while (CAS(oldTop, oldTop.next)) { // CAS 将栈顶元素为第一个子元素
        oldTop.next = null; // 旧的节点删掉next引用，等待gc
    }
    
    return oldTop.item;
}
```

分析完上述入栈和出栈过程，完整代码如下：

```Java
import java.lang.invoke.MethodHandles;
import java.lang.invoke.VarHandle;
import java.util.Objects;

/**
 * 基于VarHandle实现TreiberStack
 *
 * @author mingshan
 */
public class TreiberStack<E> {
  private volatile Node<E> top;

  public void push(E item) {
    Objects.requireNonNull(item);

    Node<E> newTop = new Node<E>(item);
    do {} while (!tryPush(newTop));
  }

  private boolean tryPush(Node<E> node) {
    Node<E> oldTop = top;
    NEXT.set(node, oldTop);
    return TOP.compareAndSet(this, oldTop, node);
  }

  public E pop() {
    Node<E> oldTop = top;

    if (oldTop == null)
      return null;

    while (TOP.compareAndSet(this, oldTop, oldTop.next)) {
      NEXT.set(oldTop, null);
    }

    return oldTop.item;
  }

  public boolean isEmpty() {
    return top == null;
  }

  public int size() {
    Node<E> current = top;
    int size = 0;
    while (current != null) {
      size++;
      current = current.next;
    }
    return size;
  }

  public E peek() {
    Node<E> eNode = top;
    if (eNode == null) {
      return null;
    } else {
      return eNode.item;
    }
  }

  @Override
  public String toString() {
    if (top == null) {
      return "Stack is empty";
    } else {
      StringBuilder sb = new StringBuilder();
      Node<E> current = top;
      while (current != null) {
        sb.append(current.item).append(",");
        current = current.next;
      }

      return sb.substring(0, sb.length() -1);
    }
  }

  private static class Node<E> {
    E item;
    Node<E> next;

    Node(E item) {
      this.item = item;
    }
  }

  private static final VarHandle TOP;
  private static final VarHandle NEXT;
  static {
    try {
      MethodHandles.Lookup l = MethodHandles.lookup();
      TOP = l.findVarHandle(TreiberStack.class, "top", TreiberStack.Node.class);
      NEXT = l.findVarHandle(TreiberStack.Node.class, "next", TreiberStack.Node.class);
    } catch (ReflectiveOperationException e) {
      throw new ExceptionInInitializerError(e);
    }
    
    // Reduce the risk of rare disastrous classloading in first call to
    // LockSupport.park: https://bugs.openjdk.java.net/browse/JDK-8074773
    Class<?> ensureLoaded = LockSupport.class;
  }
}
```


## References：

- [Treiber stack](https://en.wikipedia.org/wiki/Treiber_stack)
