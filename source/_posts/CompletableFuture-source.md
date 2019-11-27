---
title: CompletableFuture源码学习
categories: [Java, JUC]
tags: [java, JUC, 异步编程]
author: Mingshan
date: 2019-11-27
---

了解到CompletableFuture的基础用法之后，我们不禁好奇，以前的Future模式不支持如此好用的异步编程，CompletableFuture是如何做到的呢？这就需要我们去阅读源码了，通过源码我们才能了解到其设计思想和实现方式，我们分析下supplyAsync 和 thenApplyAsync 这两个，并且是提供线程池的接口，因为如果不提供自定义线程池，就会用默认的，如下：


```Java
private static final boolean USE_COMMON_POOL =
        (ForkJoinPool.getCommonPoolParallelism() > 1);

private static final Executor ASYNC_POOL = USE_COMMON_POOL ?
        ForkJoinPool.commonPool() : new ThreadPerTaskExecutor();
        
static final class ThreadPerTaskExecutor implements Executor {
        public void execute(Runnable r) { new Thread(r).start(); }
}
```

<!-- more -->

上面有一个判断`USE_COMMON_POOL`，其中用到了`ForkJoinPool.getCommonPoolParallelism()`，这个是ForkJoin中通用池的并行级别，默认是`Runtime.getRuntime().availableProcessors() - 1`，所以你的电脑有四核心，那么`ForkJoinPool.getCommonPoolParallelism()`的值就是3，如果只有一个核心，该值是0，所以`USE_COMMON_POOL`为true，那么你的电脑至少三个核心。

如果`USE_COMMON_POOL`为false，那么就会用`ThreadPerTaskExecutor`，由上面代码可知，这是个单线程的执行器。

在CompletableFuture源码中，有两个**成员属性**比较重要（volatile保证多线程之间值的可见性），如下所示：

```Java
volatile Object result;       // Either the result or boxed AltResult
volatile Completion stack;    // Top of Treiber stack of dependent actions
```

其中**result**为当前阶段的计算结果，注意看上面的注释，还有可能值为`AltResult`，该类仅有一个成员变量`Throwable ex`，该类的作用官方描述为：
> An AltResult is used to box null as a result, as well as to hold exceptions.

即用`AltResult`代替`null`和持有计算过程中发生的异常，源码如下：

```
static final class AltResult { // See above
    final Throwable ex;        // null only for NIL
    AltResult(Throwable x) { this.ex = x; }
}
```

**stack**

**supplyAsync源码分析**

测试代码如下：

```Java
CompletableFuture<String> future = CompletableFuture.supplyAsync(() -> {
  return Thread.currentThread().getName();
}, Executors.newFixedThreadPool(5));

System.out.println(future.join());
```

我们分析如下supplyAsync实现：

```Java
CompletableFuture<U> supplyAsync(Supplier<U> supplier, Executor executor)
```

该方法会直接调`asyncSupplyStage`方法，首先先去检查传入的执行器是否为`ForkJoinPool.commonPool()`，如果是会直接用`ForkJoinPool.commonPool()`，代码如下所示：

```Java
static <U> CompletableFuture<U> asyncSupplyStage(Executor e, Supplier<U> f) {
    if (f == null) throw new NullPointerException(); // 空指针判断
    CompletableFuture<U> d = new CompletableFuture<U>(); // 新建CompletableFuture
    e.execute(new AsyncSupply<U>(d, f)); // 新建AsyncSupply（ForkJoinTask）,丢到传入的线程池执行
    return d; // 立即返回
}
```

在`asyncSupplyStage`方法里首先做空指针判断，接着新建一个新的`CompletableFuture`, 然后新建一个`AsyncSupply`，将刚才新建的`CompletableFuture`和传入的`Supplier`传给`AsyncSupply`，接着直接将`AsyncSupply`丢到传入的线程池中进行执行，最后立即返回，不会等待执行结束。

`AsyncSupply`源码如下：

```
/**
 * 该类继承自ForkJoinTask，为ForkJoin计算任务，且实现了Runnable, AsynchronousCompletionTask，
 * 代表可以直接丢到线程池里面运行
 */
static final class AsyncSupply<T> extends ForkJoinTask<Void>
    implements Runnable, AsynchronousCompletionTask {
    CompletableFuture<T> dep; Supplier<? extends T> fn;
    AsyncSupply(CompletableFuture<T> dep, Supplier<? extends T> fn) {
        this.dep = dep; this.fn = fn;
    }

    public final Void getRawResult() { return null; }
    public final void setRawResult(Void v) {}
    public final boolean exec() { run(); return false; }

    public void run() {
        CompletableFuture<T> d; Supplier<? extends T> f;
        if ((d = dep) != null && (f = fn) != null) { // 判断d, f 是否为空
            dep = null; fn = null; // 这里置空，防止多次执行
            if (d.result == null) { // 如果当前任务还没执行完，result没有值
                try {
                    d.completeValue(f.get()); // 等待任务结束，并利用CAS设置结果，可能设置失败
                } catch (Throwable ex) {
                    d.completeThrowable(ex); // 出现异常
                }
            }
            d.postComplete(); // 这个后面再说，目前没有用
        }
    }
}
```

在`AsyncSupply`内部类里面，有一个`run`方法，由于我们将其丢到了线程池中运行，所以就会执行`run`方法。在这个方法里面，会执行我们传给`supplyAsync`的计算任务，并将结果通过CAS写到
