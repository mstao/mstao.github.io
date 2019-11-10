---
title: Reactive Streams规范与Flow相关实现
author: Mingshan
categories: Java
tags: [java,Reactive Streams]
date: 2019-11-10
---

前段时间了解了[WebFlux](https://docs.spring.io/spring/docs/current/spring-framework-reference/web-reactive.html)，该框架采用全新的Reactive技术栈，依赖Spring团队的Reactor框架。Reactive programming(响应式编程) 是一种比较新的编程模式，在维基百科中有如下定义：

> Reactive programming is an asynchronous programming paradigm concerned with data streams and the propagation of change. This means that it becomes possible to express static (e.g. arrays) or dynamic (e.g. event emitters) data streams with ease via the employed programming language(s).

从上面的描述（感觉很空洞），我们可以知道响应式编程有两个要点：`asynchronous(异步)` 和 `data streams(数据流)`。在JDK中引入Stream，让静止的数据强行流动，这个响应式编程也涉及到了Stream，这个词真是相当火爆，在Flink等大数据计算框架中无处不在，那么它与JDK中的Stream有没有关系呢？这里就要谈到了Reactive Streams规范。

<!-- more -->

由于Reactive Streams这个概念不好定义，维基百科说的太空洞了，大家又都想用，想自己实现一套，所以一些大佬专门为在JVM上实现Reactive Streams定义一套规范，名字就叫[Reactive Streams Specification for the JVM](https://github.com/reactive-streams/reactive-streams-jvm)，放在了github，README就是规范的主体。这个规范的核心是：`asynchronous stream processing with non-blocking backpressure`。我们仔细看这个规范，其实定义了一套接口，没有参考实现。光有接口不行，为了保证其他人实现的Reactive Streams能够正确地跑起来，还提供了一个兼容性测试套件（TCK），来做技术兼容测试。

## Reactive Streams接口

上面提到Reactive Streams规范定义了一套接口，具体如下：

1. Publisher
2. Subscriber
3. Subscription
4. Processor

看接口的名称，我们发现这个不是发布/订阅模式吗？难道这个玩意和MQ实现的功能类似？我们知道RabbitMQ采用发布订阅模式来实现消息的生产与消费。毕竟MQ是一个中间件， Reactive Streams是一个编程范式，本质上是不一样的。我们来看看四个接口的方法。


**Publisher**

Publisher接口声明如下：

```Java
public interface Publisher<T> {
    public void subscribe(Subscriber<? super T> s);
}
```

Publisher作为信息的发布者，规范中并未定义发布信息的方法，而是只定义一个subscribe方法，用来将发布者与订阅者绑定起来，注意这里可以有0至多个订阅者。

**Subscriber**

Subscriber接口声明如下：

```Java
public interface Subscriber<T> {
    public void onSubscribe(Subscription s);
    public void onNext(T t);
    public void onError(Throwable t);
    public void onComplete();
}
```

Subscriber接口有四个方法。对于`onSubscribe`方法，上面提到Publisher接口的subscribe方法，如果订阅者订阅成功，发布者用Subscription异步调用订阅者的onSubscribe方法。 如果尝试订阅失败，则使用调用订阅者的onError()方法，并抛出异常，发布者/订阅者交互结束。

发布者/订阅者建立关系后，发布者通过不断调用订阅者的`onNext`方法向订阅者发出最多n个数据。如果数据全部发完，则会调用`onComplete`告知订阅者流已经发完；如果有错误发生，则通过`onError`发出错误数据，同样也会终止流。

我们发现`onSubscribe`方法参数是`Subscription`，这也是一个接口，如下。

**Subscription**

Subscription接口声明如下：

```Java
public interface Subscription {
    public void request(long n);
    public void cancel();
}
```

Subscription在发布订阅过程中充当什么作用呢？当发布者与订阅者绑定后，需要一个中间人来维护这个关系，Subscription就是充当这个角色。在规范中，有如下要求：

> A Subscriber MUST signal demand via Subscription.request(long n) to receive onNext signals.

从上面的话，我们了解到，当订阅者想要处理数据时，那么需要调用`Subscription.request(long n)`向发布者发送多个数据的请求，然后发布者会调用订阅者的 `onNext`方法来处理数据。对于`cancel()`方法，规范有以下描述：

> A Subscriber MUST call Subscription.cancel() if the Subscription is no longer needed.

订阅者可以通过调用`Subscription.cancel()`方法来取消订阅。一旦订阅被取消，发布者/订阅者交互结束。

**Processor**

有没有一个东西既是发布者又是消费者呢？规范也为我们考虑到了，就是Processor接口，如下所示：

```Java
public interface Processor<T, R> extends Subscriber<T>, Publisher<R> {
}
```

## References：

- [Reactive Streams](http://www.reactive-streams.org/)
- [Reactive Streams Specification for the JVM](https://github.com/reactive-streams/reactive-streams-jvm)
- [Introduction to Reactive Programming](https://projectreactor.io/docs/core/release/reference/#intro-reactive)
- [Reactive programming](https://en.wikipedia.org/wiki/Reactive_programming)
- [照虎画猫深入理解响应式流规范——响应式Spring的道法术器](https://blog.51cto.com/liukang/2090922)
