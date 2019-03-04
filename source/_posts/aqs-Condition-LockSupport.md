---
title: AQS源码分析-Condition与LockSupport
categories: [Java, JUC]
tags: [java, JUC, AQS]
author: Mingshan
date: 2019-3-4
---

在[从生产者消费者模型窥探多线程并发](https://mingshan.fun/2019/02/25/producer-consumer/)这篇文章中我们使用了ReentrantLock结合Condition实现生产者消费者模型，但我们对于ReentrantLock和Condition的工作原理并不了解，其内部的结构和源码级别实现就更加不了解了。比如在使用await方法的时候，为什么一定要用while判断条件，用if为什么不行呢？使用Condition时，线程之间是如何通信的呢？与synchronized有什么区别？如果后续要用到BlockingQueue，其内部的实现是离不开ReentrantLock和Condition的，所以对于ReentrantLock和Condition源码级别的分析是十分有必要的。（**本文基于JDK11版本**）

<!-- more -->

对于ReentrantLock的源码分析，请参考: [ReentrantLock源码笔记 - 获取锁（JDK 1.8）](https://mingshan.fun/2017/11/10/reentrantlock-get-lock/)和 [AQS源码分析-独占模式](https://mingshan.fun/2019/01/25/aqs-exclusive/)，在此不再详细分析，所以我们详细研究下Condition的实现。

References：

- [入门AQS锁 - Condition与LockSupport](https://www.jianshu.com/p/1add173ea703)


[<font size=3 color="#409EFF">向本文提出修改或勘误建议</font>](https://github.com/mstao/mstao.github.io/blob/hexo/source/_posts/aqs-Condition-LockSupport.md)