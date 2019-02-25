---
title: 从生产者消费者模型窥探多线程并发
tags: [Producer–consumer,java,JUC]
author: Mingshan
categories: [Java, JUC]
date: 2019-02-25
---

[生产者消费者问题](https://en.wikipedia.org/wiki/Producer%E2%80%93consumer_problem)是一个常见而且经典的问题，相信了解过多线程或者消息队列的同学对这个名词并不陌生。正如Java常用的设计模式一样，生产者消费者问题是为了解决某一类问题而存在，参阅维基百科对[Producer–consumer problem](https://en.wikipedia.org/wiki/Producer%E2%80%93consumer_problem)的描述：

<!-- more -->

> In computing, the producer–consumer problem[1][2] (also known as the bounded-buffer problem) is a classic example of a multi-process synchronization problem. The problem describes two processes, the producer and the consumer, who share a common, fixed-size buffer used as a queue. The producer's job is to generate data, put it into the buffer, and start again. At the same time, the consumer is consuming the data (i.e., removing it from the buffer), one piece at a time. The problem is to make sure that the producer won't try to add data into the buffer if it's full and that the consumer won't try to remove data from an empty buffer.

生产者消费者问题其实是多线程同步问题，它有两个主体，生产者（producer）和消费者（consumer），两者共享一个被称为队列（queue）的数据区域，其中生产者向队列中放入数据，消费者从队列中消费数据。当然此时需要保证：当队列为空时，消费者等待；队列满时，生产者等待。假设生产者和消费者都不止一个，此时必然涉及到线程并发问题，同时数据的存储方式（queue）决定了生产者消费者模型在实际应用中的性能和吞吐量。下面是生产者消费者模型图示：

![image](https://github.com/ZZULI-TECH/interview/blob/master/images/Producer–consumer%20problem.png?raw=true)

了解过生产者消费者问题的概念，那么我们对其在实际应用比较好奇，从上面的分析可以看出，生产者消费者模型可以用来系统解耦，可以将生产者看作一个系统，消费者看作一个系统，这样两个系统就通过缓冲区将两者的逻辑隔离开了；我们还可以发现，利用生产者消费者模型可以实现异步，当生产者作为系统的一部分逻辑，我们希望其不影响主线业务逻辑的走向，比如日志系统等，不能因为记录日志而影响系统的响应时间；再复杂到市场上商用的消息队列（MQ），ACK机制等，这里就不细说。

接下来我们考虑几种经典的生产者消费者模型简易实现和比较复杂的Disruptor框架，一步步了解生产者消费者模型。

### synchronized方式

### ReentrantLock与Condition

### BlockingQueue方式

### Disruptor循环队列

## References：

- [Producer–consumer problem](https://en.wikipedia.org/wiki/Producer%E2%80%93consumer_problem)
- [生产者/消费者模式的理解及实现](https://blog.csdn.net/u011109589/article/details/80519863)


