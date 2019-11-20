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
`Predicate<T>`     |	  (T)  -> boolean    | `boolean test( T)` 接收一个参数，返回布尔值
`Consumer<T>`     |    (T)  -> void       | `void accept(T)` 接收一个参数，无返回值
`Function<T, R>`  |	  (T)  -> R          |  `R apply(T)` 接收一个参数，返回指定类型的数据
`Supplier<T>`	     |    ( )  -> T          | `T get()` 不接收参数，返回指定类型的数据
`UnaryOperator<T>`  |	  (T)  ->  T         | Function子接口：`T apply(T)` 接受一个参数为类型T,返回值类型也为T
`BinaryOperator<T>` |  (T, T) -> T         | BiFunction子接口：`T apply(T t, T u)` 接受两个输入参数的，返回一个结果
`BiPredicate<L, R>` |	  (L, R)  -> boolean | `boolean test(T t, U u)` 接收两个参数，返回布尔值 
`BiConsumer<T, U>`  |	  (T, U)  -> void    | `void accept(T t, U u)` 接收两个参数，无返回值
`BiFunction<T, U, R>` |	  (T, U)  -> R   | `R apply(T t, U u)` 接受两个输入参数的，返回一个结果

## CompletionStage接口

从上面的函数式接口来说，整体可以分析以下几大类：

- 断言型，如Predicate
- 消费并产出型，如Function，UnaryOperator
- 消费不产出型，如Consumer
- 产出型，Supplier

上面提到CompletionStage接口方法的参数大都是函数式接口，所以其方法在命名上也体现出其特点，比如用apply, accept 和 run 开头的方法分别表示Function, Consumer 和 Runnable。所以CompletionStage接口主要有以下类型：

- **消费并产出型**，用上一个阶段的结果作为当前方法的参数并计算出新结果，接口名称带有apply，参数为Function或其子接口；
- **消费不产出型**，用上一个阶段的结果作为当前方法的参数，但不对阶段结果产生影响，接口名称带有accept，参数为Consumer或其子接口；
- **不消费也不产出型**，不依据上一个阶段的执行结果，只要上一个阶段完成（但一般要求正常完成），就执行指定的操作，且不对阶段的结果产生影响，接口名称带有run，参数为Runnable

上述只是从函数式接口对CompletionStage接口方法进行简单分类，但CompletionStage提供的功能远不止此。我们知道异步编程需要有合并计算结果的能力，这个计算任务可以由一个阶段的完成触发，也可以由两个阶段的完成触发，也可以由两个阶段中的任何一个触发，所以按照计算依赖关系，可以分析以下几种：

- 由一个阶段的完成触发，方法前缀为then
- 由两个阶段的完成触发，方法带有combine或者both
- 由两个阶段中任意一个完成触发，不能保证哪个的结果或效果用于相关阶段的计算，这类方法带有either

在看CompletionStage接口时，发现有很多方法是以`Async`结尾的，并且有些的函数参数里面有`Executor`，比如如下几个方法：


```Java
public <U> CompletionStage<U> thenApply(Function<? super T,? extends U> fn);
public <U> CompletionStage<U> thenApplyAsync
        (Function<? super T,? extends U> fn);
public <U> CompletionStage<U> thenApplyAsync
        (Function<? super T,? extends U> fn,
         Executor executor);
```

这个是出于什么情况考虑的呢？由于异步编程底层是用线程池来做的，当上一阶段的计算完成，下一阶段可以继续使用这个线程计算，也可以从线程池里面重新拿一个线程（注意这个重新获取的线程可能与上一阶段一样），也可以给当前阶段的计算任务重新指定线程池，就从这个池子里面拿吧。这样设计也是为了异步计算性能考虑，如果上一阶段计算任务执行很慢的 I/O 操作，就会导致所有计算任务都阻塞在 I/O 操作上，进而影响计算性能。如果当前线程池已没有线程可用，那么当前的计算任务要被阻塞，所以可以根据不同的业务，指定不同的线程池，隔离彼此之间的影响。线程池的大小可由如下公式粗略计算。

![image](https://image-static.segmentfault.com/147/471/1474711767-5d99fbb371be6)

如果在计算过程中，发生了异常，会出现什么情况？整个计算都终止吗？比如我上个计算任务失败了，但我还想继续执行下一个计算任务进行处理，而下一个计算任务最好能够感知到上一个计算任务发生了什么，以做出不同的处理。在平时处理异常时，基本用`try...catch...finally`，CompletionStage接口也为异步计算异常处理提供了相应支持。如下有三个方法

- **handle(BiFunction<? super T, Throwable, ? extends U> fn)** 两个参数，一个代表上一阶段计算结果，一个代表上一阶段异常，支持返回值
- **whenComplete(BiConsumer<? super T, ? super Throwable> action)**  类似于 try{}finally{} 中的 finally{}，不支持返回结果
- **exceptionally(Function<Throwable, ? extends T> fn)**  类似于 try{}catch{} 中的 catch{}，支持返回结果


## CompletableFuture使用

上面从不同维度了解了CompletionStage接口的设计，从宏观角度简单理解异步编程遇到的问题以及解决方案。下面我们来看看JDK为CompletionStage接口提供的实现类CompletableFuture，该类是JDK8提供的。

### 直接执行异步计算

CompletableFuture提供最基本的功能就是直接执行一个任务，且没有返回值，使用方式如下：

```Java
CompletableFuture<Void> CompletableFuture.runAsync(Runnable runnable)
```

下面是一个例子，在`runAsync`方法里面可以写自己的异步逻辑，返回一个CompletableFuture，暗示我们可以进行执行异步任务

```Java
CompletableFuture<Void> future = CompletableFuture.runAsync(() -> {
  try {
    TimeUnit.SECONDS.sleep(1);
  } catch (InterruptedException e) {
    e.printStackTrace();
  }
  System.out.println(Thread.currentThread().getName());
});
```

如果想让异步任务**有返回值**，这个时候就用到了产出型函数接口**Supplier**，如下所示：

```Java
CompletableFuture<U> CompletableFuture.supplyAsync(Supplier<U> supplier)
```

下面是一个例子，执行一个异步任务，返回一个执行结果，我们可以通过`get()`方法直接拿到计算结果，当然也可以进行下一阶段的计算

```Java
CompletableFuture<String> future = CompletableFuture.supplyAsync(() -> {
  return Thread.currentThread().getName();
});

System.out.println(future.get());
```

### 处理上一阶段计算结果

当我们使用`supplyAsync`计算出结果，想传递给下一阶段的计算任务，该使用哪个方法呢？结合前面对CompletionStage接口的分析，从宏观角度来看，主要有**消费不产出**和**消费并产出**这两种类型，所以CompletableFuture提供了`thenApply / thenApplyAsync`来处理消费并产出型，用`thenAccept / thenAcceptAsync`来处理消费不产出型。下面是使用方式：

```Java
CompletableFuture<U> thenApply(Function<? super T,? extends U> fn)
CompletableFuture<Void> thenAccept(Consumer<? super T> action)
```

对于**消费并产出型**，我们可以在当前阶段获取上一阶段的计算结果，然后将当前阶段的计算结果传递给下一阶段，这样可以一直传递下去，如下代码所示：

```Java
CompletableFuture<String> future = CompletableFuture.supplyAsync(() -> {
  return Thread.currentThread().getName();
}).thenApplyAsync(item -> {
  return item + "-111111-" + Thread.currentThread().getName();
}).thenApplyAsync(item -> {
  return item + "-222222-" + Thread.currentThread().getName();
});

System.out.println(future.get());
```

对于**消费不产出型**，我们可以在当前阶段获取上一阶段的计算结果，内部处理完后不会再向下一阶段传递值，如下所示:


```Java
CompletableFuture.supplyAsync(() -> {
  return Thread.currentThread().getName();
}).thenAccept(System.out::println);
```

### 整合两个计算结果

熟悉了上面几个方法，我们已经可以用链式编程来写代码了，但上面的方法参数都是函数式接口，如有我现在有两个方法，方法的返回值是`CompletableFuture<T>`，怎么讲两者的计算结果整合起来呢？CompletableFuture为我们提供了两个方法：`thenCompose \ thenCombine`，下面是这两个接口的使用方式：

```Java
CompletableFuture<U> thenCompose(Function<? super T, ? extends CompletionStage<U>> fn)
CompletableFuture<V> thenCombine(CompletionStage<? extends U> other,
        BiFunction<? super T,? super U,? extends V> fn)
```

**thenCompose**

考虑如下场景，我们有一个方法，根据用户id获取用户信息，还有一个方法，根据用户id拿到该用户的权限，最终将该权限信息填充到User实体，返回User。下面是这两个函数的简单实现：

```Java
private CompletableFuture<User> getUser(String userId) {
  return CompletableFuture.supplyAsync(() -> {
    // DO ANYTHING
    return new User();
  });
}

private CompletableFuture<User> fillRes(User user) {
  return CompletableFuture.supplyAsync(() -> {
    // 获取权限信息，填充到用户信息里面
    return user;
  });
}
```

注意上述两者都返回`CompletableFuture<User>`，我们依稀记得`thenApply`属于消费并产出型。下面用thenApply将两者连接起来，如下：

```Java
CompletableFuture<CompletableFuture<User>> future = getUser(userId).thenApply(this::fillRes);
```
看起来有些不太妙，返回的结果出现了复合的CompletableFuture，这是因为`thenApply`的产出是一个`CompletableFuture<User>`，所以就出现了复合的情况。针对以上场景，JDK为我们提供了`thenCompose`，下面用`thenCompose`改写下：

```Java
// 利用 thenCompose() 组合两个独立的CompletableFuture
CompletableFuture<User> result = getUser(userId)
  .thenCompose(user -> fillRes(user));
```

真是妙啊，真是我们想要的效果。由此可见，`thenCompose`的作用是组合两个独立的CompletableFuture。

**thenCombine**

上面提到还有一个`thenCombine`，也是组合计算结果的，下面是一个例子：

```Java
CompletableFuture<String> walker = CompletableFuture.supplyAsync(() -> {
  return ", Walker";
});

CompletableFuture<String> future = CompletableFuture.supplyAsync(() -> {
  return "Hello";
}).thenApply(s -> s + " World")
  .thenCombine(walker, (s1, s2) -> {
    return s1 + s2;
  });
```

首先我们异步拼接字符串 `Hello World`，然后我们想将另一个独立的异步任务计算结果与刚才的异步任务结果整合一下，这一点和`thenCompose`是不一样，`thenCompose`是拿到上一阶段的计算结果，`thenCombine`直接传入一个异步任务，第二个参数是BiFunction，函数描述符为`(T, U) -> R`，即将两个异步任务的结果整合。

### duoge 
上述`thenCompose \ thenCombine`是针对两个异步任务情况而言的，试想下有没有这样的场景，先获取用户的信息，然后同时获取与用户相关联的数据，比如权限，喜好列表等，彼此独立，可以并发获取。这样的该怎么写呢？

## CompletableFuture源码分析

## 异步编程

## References：

- [CompletionStage](https://docs.oracle.com/en/java/javase/11/docs/api/java.base/java/util/concurrent/CompletionStage.html)
- [CompletableFuture](https://docs.oracle.com/en/java/javase/11/docs/api/java.base/java/util/concurrent/CompletableFuture.html)
- [从CompletableFuture到异步编程设计](https://www.cnblogs.com/xiangnanl/p/9939447.html)
- https://segmentfault.com/a/1190000020602954
- https://www.cnblogs.com/txmfz/p/11266411.html
