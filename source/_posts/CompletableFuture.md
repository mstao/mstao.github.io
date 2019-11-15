---
title: CompletableFuture与异步编程设计
categories: [Java, JUC]
tags: [java, JUC, 异步编程]
author: Mingshan
date: 2019-11-15
---

用过Spring推出的Reactor框架的同学可能会感叹异步编程的便利，不过Reactor对于异步编程的初学者来说有点复杂了，看其源码也不是那么容易，那么JDK有没有对异步编程相关的支持呢？[Future](https://docs.oracle.com/en/java/javase/11/docs/api/java.base/java/util/concurrent/Future.html)想必大家都很熟悉（不了解的同学请查看[Callable&Future及FutureTask实现分析(JDK11)](https://mingshan.fun/2018/10/13/callable-future-futuretask/)），在线程池里面运行，返回Future，等待计算结果，虽然采用线程池来运行代码，但像Reactor那样链式编程是不容易的。因为Future的结果是需要调用者自己去拿的，计算没有结束就会一直被阻塞，试想下有没有这样一种设计呢？当计算完之后，主动将计算结果推给调用者。可能有人说，这个不是Callback模式吗？如下伪代码所示：

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

上面计算任务的描述中用到了函数式编程语法模型，更易表达思想。流水线模型伪代码似乎有内味了😊。在JDK8中，引入了[CompletionStage](https://docs.oracle.com/en/java/javase/11/docs/api/java.base/java/util/concurrent/CompletionStage.html)接口，这个接口就是为了描述异步计算时某一个完成阶段的状态和动作，下面是官方的介绍：

>A stage of a possibly asynchronous computation, that performs an action or computes a value when another CompletionStage completes. A stage completes upon termination of its computation, but this may in turn trigger other dependent stages.

这个接口的方法多大几十个，功能相当丰富。同时提供了这个接口的实现[CompletableFuture](https://docs.oracle.com/en/java/javase/11/docs/api/java.base/java/util/concurrent/CompletableFuture.html)，这个是我们要讨论的主角。CompletionStage接口的方法参数大部分支持函数式，根据不同的函数式接口做不同的事情，下面简单介绍下函数式接口。

## 函数式接口

函数式编程在JDK1.8被引入，该编程范式极大精简代码，结合Java的类型推断功能，给Java注入了相当大的活力。JDK官方提供了很多函数式接口，用来描述函数的功能。下面是将常用的接口，比如`Predicate`做谓词运算，搭配`test`方法，功能也是直抒胸臆的。

函数式接口	  |   函数描述符  |  功能
---|---|---
==Predicate<T>==     |	  (T)  -> boolean    | `boolean test( T)` 接收一个参数，返回布尔值
==Consumer<T>==	     |    (T)  -> void       | `void accept(T)` 接收一个参数，无返回值
==Function<T, R>== |	  (T)  -> R          |  `R apply(T)` 接收一个参数，返回指定类型的数据
==Supplier<T>==	     |    ( )  -> T          | `T get()` 不接收参数，返回指定类型的数据
==UnaryOperator<T>==  |	  (T)  ->  T         | Function子接口：`T apply(T)` 接受一个参数为类型T,返回值类型也为T
==BinaryOperator<T>== |  (T, T) -> T         | BiFunction子接口：`T apply(T t, T u)` 接受两个输入参数的，返回一个结果
==BiPredicate<L, R>== |	  (L, R)  -> boolean | `boolean test(T t, U u)` 接收两个参数，返回布尔值 
==BiConsumer<T, U>==  |	  (T, U)  -> void    | `void accept(T t, U u)` 接收两个参数，无返回值
==BiFunction<T, U, R>== |	  (T, U)  -> R   | `R apply(T t, U u)`

## CompletionStage接口


## CompletableFuture使用

## CompletableFuture源码分析

## 异步编程

## References：

- [CompletableFuture](https://docs.oracle.com/en/java/javase/11/docs/api/java.base/java/util/concurrent/CompletableFuture.html)
- [从CompletableFuture到异步编程设计](https://www.cnblogs.com/xiangnanl/p/9939447.html)
