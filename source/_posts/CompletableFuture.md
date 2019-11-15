---
title: CompletableFuture与异步编程设计
categories: [Java, JUC]
tags: [java, JUC, 异步编程]
author: Mingshan
date: 2019-11-15
---

用过Spring推出的Reactor框架的同学可能会感叹异步编程的便利，不过Reactor对于异步编程的初学者来说有点复杂了，看其源码也不是那么容易，那么JDK有没有对异步编程相关的支持呢？[Future](https://docs.oracle.com/en/java/javase/11/docs/api/java.base/java/util/concurrent/Future.html)想必大家都很熟悉（不了解的同学请查看[Callable&Future及FutureTask实现分析(JDK11)](https://mingshan.fun/2018/10/13/callable-future-futuretask/)），在线程池里面运行，返回Future，等待计算结果，虽然采用线程池来运行代码，但像Reactor那样链式编程是不容易的。因为Future的结果是需要调用者自己去拿的，计算没有结束就会一直被阻塞，试想下下有没有这样一种设计呢？当计算完之后，主要将计算结果推给调用者。可能有人说，这个不是Callback模式吗？如下伪代码所示：

```
fn doFun(params, callback() {
    doFun2(params, callback() {
    ...Callback Hell
    })
})
```
<!-- more -->

如果将某一步的计算结果传递给下一个计算任务，这样依次传递下去，就会形成回调地狱(Callback Hell)，代码极其不宜读，十分不优雅。接触过cpu流水线作业的同学知道**流水线**（pipeline）是一串操作的集合，其中前一个操作的输出是下一个操作的输入。流水线执行允许在同一时钟周期内重叠执行多个指令。异步编程有没有可能像流水线作业这样如此优雅呢？下面是试想出来流水线模型（伪代码）：

```
doFun(() -> task1)
 .doFun1(result -> task2(result))
 .doFun2(result -> task3())
 .doFun3(() -> task4())
 .end(() -> doAnything())
```

上面计算任务的描述中用到了函数式编程语法模型，更易表达思想。


## References：

- [CompletableFuture](https://docs.oracle.com/en/java/javase/11/docs/api/java.base/java/util/concurrent/CompletableFuture.html)
- [从CompletableFuture到异步编程设计](https://www.cnblogs.com/xiangnanl/p/9939447.html)
