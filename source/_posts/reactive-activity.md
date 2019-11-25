---
title: 参加Reactive架构专场
tags: [随笔]
author: Mingshan
categories: [随笔]
date: 2019-11-25
---

最近一直在看Reactive相关的知识，周末刚好阿里巴巴举办了一个Reactive架构相关的技术分享会，周末没啥事就报名去了，地点在浦东张江人工智能岛，当时去了大概有一百多人，演讲嘉宾有阿里云原生平台技术专家Andy Shi，阿里P9[雷卷Jacky](https://github.com/linux-china)和爱奇艺的一个开发者。	本来[Josh Long](https://github.com/joshlong)也会来，但他父亲病危就没有来。

<!-- more -->

活动还是很给力的，对RSocket这个新生的协议有初步的了解，不过雷卷讲的很多东西平时没有接触过，对RSocket底层的实现方式认识也就比较模糊了。我以前有Reactive Stream的编程经验，Reactive相关只限于会用，虽然尝试读JDK关于Reactive和Reactor源码，但生产级别的代码相当复杂。这个还只限于在同一进程中，涉及到分布式，就更复杂了。传统的HTTP协议是阻塞式的，如果上升到网络方面，HTTP显然不能很好支持异步处理。以前我们通过Tomcat监听某一端口，当有请求过来了，就让一个线程去处理，如果对NIO熟悉的同学，可以很好理解多路复用的工作流程，也就Reactor模式。Reactive Stream规范定义了一种新的编程方式，即所谓的反应式编程。这种编程方式可以让程序员写出非常优雅的异步代码，API也是非常简单和流畅，将底层复杂的多线程以及背压处理都给屏蔽了，对程序员是十分友好的。

雷卷主要讲了RSockt的一些设计思想和相关底层实现，由于我对整块不是很熟，下面权且记录一下。RSockt是个什么协议呢？RSockt是一个微服务通讯协议，基于Reactive模式，支持异步和流式计算。RSockt支持如下通讯模型：

![image](https://github.com/mstao/static/blob/master/reactive/rsocket-mode.png?raw=true)

HTTP协议可以升级为WebSocket来支持全双工通信，这个RSocket也支持，即所谓Client与Server的身份互换。这里引出了一个`RSocket Broker`的概念，相当于一个路由器似的，服务都可以连接到这个Broker，如下图所示：

![image](https://github.com/mstao/static/blob/master/reactive/rsocket-mode2.png?raw=true)

从上图可知，Server和Client 不能直连，而是通过这个Broker来作为中间者进行通信。这个地方有几处地方没有听懂，比如Broker如何做集群避免单一故障，路由分发的具体算法等，这个以后再接触吧。

由于我们真正的应用是无比复杂的，不仅仅只有`request/response`模式，所以RSocket也支持多种模式，比如常见的RPC，具体如下：

- RPC: request/response 
- pub/sub: request/response + request/stream 
- Config push: request/stream 
- Registry: Bi-direction request/response 
- Logging & Metrics Collection: fire-and-forgot 
- IoT: bi-directional streams


现在大家嘴上都说云，什么东西搞到云上，立马高大上了，私有云，K8S，docker名词满天飞，RSockt当然也要有这个概念，所以就提到了混合云，阿里云，腾讯云等，连接一切。

![image](https://github.com/mstao/static/blob/master/reactive/rsocket-cloud.png?raw=true)

最后放两张活动的图片，结束。

![image](https://github.com/mstao/static/blob/master/reactive/a1.jpg?raw=true)

![image](https://github.com/mstao/static/blob/master/reactive/a3.jpg?raw=true)

![image](https://github.com/mstao/static/blob/master/reactive/a2.jpg?raw=true)
