---
title: 关于线程你问我答
categories: [Java, Thread]
tags: [java, Thread]
author: Mingshan
date: 2021-05-16
---

跟同学讨论了几个关于线程的基础知识，真的非常基础O.O，这里记录一下，供小伙伴参考，如果问题就可以直接评论交流。

<!-- more -->

## 问题：线程有哪几种状态？

1. 初始(NEW)：新创建了一个线程对象，但还没有调用start()方法。
2. 运行(RUNNABLE)：Java线程中将就绪（ready）和运行中（running）两种状态笼统的称为“运行”。
线程对象创建后，其他线程(比如main线程）调用了该对象的start()方法。该状态的线程位于可运行线程池中，等待被线程调度选中，获取CPU的使用权，此时处于就绪状态（ready）。就绪状态的线程在获得CPU时间片后变为运行中状态（running）。
3. 阻塞(BLOCKED)：表示线程阻塞于锁。
4. 等待(WAITING)：进入该状态的线程需要等待其他线程做出一些特定动作（通知或中断）。
5. 超时等待(TIMED_WAITING)：该状态不同于WAITING，它可以在指定的时间后自行返回。
6. 终止(TERMINATED)：表示该线程已经执行完毕。

## 问题：为什么wait，notify 要和 synchronized 一起使用?

两个线程之间要通信，对于同一个对象来说，一个线程调用该对象的wait（），另一个线程调用该对象的notify（），该对象本身就需要同步！所以，在调用wait（）、notify（）之前，要先通过synchronized关键字同步给对象，也就是给该对象加锁。

## 问题：调用 wait的线程变为什么状态？

会变成WAITING状态。

## 问题：调用wait()的时候，发生了什么？

在wait（）的内部，会先释放锁，然后进入阻塞状态，之后，它被另外一个线程用notify（）唤醒，去重新拿锁！其次，wait（）调用完成后，执行后面的业务逻辑代码，然后退出synchronized同步块，再次释放锁。wait()的伪代码如下：

```
wait() {
    // 释放锁
    // 将自己阻塞，等待被其他线程notify
    // 重新拿锁
}
```

这样设计的好处是：如果wait不先释放锁，那么其他线程都没有办法在获取对应的锁，会造成死锁问题。

## 问题：WAITING状态和 BLOCKED状态有什么区别？

**WAITING** 状态代表当前线程已释放了锁，变成了阻塞状态，需要被另一个线程唤醒，唤醒的方式可以是notify 和 中断。

**BLOCKED** 状态代表线程正在请求进入synchronized块，锁被其他线程占用，当前线程被阻塞。

两者最主要的一个区别是WAITING可以相应中断，会抛出中断异常，属于轻量级阻塞；而BLOCKED不不会响应中断，属于重量级阻塞。

## 问题：上面提到可以用wait 和 notify来进行线程间交互，可以写一个例子吗？

```
/**
 * 共享资源
 */
public class Buffer {
    // 已使用共享资源的数量
    private int count = 0;
    // 共享资源最大数量
    private int size = 10;

    /**
     * 生产者产生资源
     *
     * @throws InterruptedException
     */
    public synchronized void put() throws InterruptedException {
        while (count == size) {
            // 共享资源满了，生产者线此时需要阻塞，等待消费者消费共享资源再进行生产
            System.out.println("[Put] Current thread " + Thread.currentThread().getName() + " is waiting");
            this.wait();
        }

        count++;
        System.out.println("[Put] Current thread " + Thread.currentThread().getName()
                + " add 1 item, current count: " + count);
        this.notify();
    }

    /**
     * 消费者消费资源
     */
    public synchronized void take() throws InterruptedException {
        // 如果共享资源为0，代表共享资源已被用尽，等下生产者生产再进行消费
        while (count == 0) {
            // 共享资源满了，生产者线此时需要阻塞，等待消费者消费共享资源再进行生产
            System.out.println("[Take] Current thread " + Thread.currentThread().getName() + " is waiting");
            this.wait();
        }

        count--;
        System.out.println("[Take] Current thread " + Thread.currentThread().getName()
                + " remove 1 item, current count: " + count);
        this.notify();

    }

}
```

## 问题：上面代码中，如果执行到take方法中的`this.notify()`，那么唤醒什么线程？

从代码逻辑上来说，消费端消费了一个元素，会通知生产者线程可以继续生产了，会唤醒生产者线程。但由于无论是生产者还是消费者线程，所作用的对象和synchronized所作用的对象是同一个，只能有一个对象，无法区分队列空和列队满两个条件，那么此时唤醒的线程不确定是生产者线程还是消费者线程。

## 问题：既然执行notify无法准确唤醒生产者线程，有没有解决方式呢？

如果想要准确唤醒生产者线程，那么就需要将生产者线程与消费者线程隔离开，比如只要将生产者线程挂起时放到一个队列，消费者线程挂起时放到一个队列，我唤醒生产者线程时，从生产者线程队列取一个线程来唤醒，这样就不会有问题了。伪代码如下:

```
    // 已使用共享资源的数量
    count = 0;
    // 共享资源最大数量
    size = 10;

    // 生产者线程队列
    producerQueue;
    // 消费者线程队列
    consumerQueue;

    // 生产者条件（队列满等待，否则可以继续生产）
    producerCondition;
    // 消费者条件（队列空等待，否则可以继续消费）
    consumerCondition;
    
    put() {
        lock();
        
            // 队满
            while(count == size) {
                producerCondition.wait();
            }
            
            consumerCondition.notify();
        
        unlock();
    }
    
    take() {
        lock();
        
            // 队满
            while(count == 0) {
               consumerCondition.wait();
            }
            
            producerCondition.notify();
        
        unlock();
    }
    
    /**
     * 生产者等待伪代码
     */
    producerCondition.wait() {
        // 释放锁
        currentThread.releaseLock();
        // 线程状态变为WAITING状态
        currentThread.wait();
        // 生产者线程入队
        producerQueue.push(currentThread);
    
        // 等待被唤醒，继续申请锁
    }
    
    /**
     * 生产者唤醒伪代码
     */
    producerCondition.notify() {
        // 从生产者队列取一个可用线程
        oneProducerThread = producerQueue.take();
        // 将其唤醒
        oneProducerThread.notify();
    }
        
    /**
     * 消费者等待伪代码
     */
    consumerCondition.wait() {
        // 释放锁
        currentThread.releaseLock();
        // 线程状态变为WAITING状态
        currentThread.wait();
        // 消费者线程入队
        producerQueue.push(currentThread);
    
        // 等待被唤醒，继续申请锁
    }
    
    /**
     * 生产者唤醒伪代码
     */
    consumerCondition.notify() {
        // 从生产者队列取一个可用线程
        oneConsumerThread = consumerQueue.take();
        // 将其唤醒
        oneConsumerThread.notify();
    }

```

从上面伪代码来看，其实无论者还是消费者处理逻辑基本一致，这正是JUC包中Condition要解决的问题。这里就不再继续谈JUC相关的知识了，后面有空继续接着这个谈。


参考：

https://mingshan.fun/2019/02/25/producer-consumer/
