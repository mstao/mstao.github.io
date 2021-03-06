---
title: 分布式事务解决方案
categories: 分布式事务
tags: 分布式事务
author: Mingshan
date: 2018-11-06
---

分布式事务目前有以下几种解决方案：

<!-- more -->

**一. 两阶段提交（2PC）**

【强一致性】

2PC又叫做XA Transactions，XA是一个两阶段提交协议，该协议分为下面两个阶段：

- 第一阶段：事务协调器要求每个涉及到事务的数据库预提交(precommit)此操作，并反映是否可以提交.
- 第二阶段：事务协调器要求每个数据库提交数据。 

其中，如果有任何一个数据库否决此次提交，那么所有数据库都会被要求回滚它们在此事务中的那部分信息。


**二、补偿事务（TCC）**

【最终一致性】

TCC 其实就是采用的补偿机制，其核心思想是：针对每个操作，都要注册一个与其对应的确认和补偿（撤销）操作。它分为三个阶段：

- Try 阶段主要是对业务系统做检测及资源预留
- Confirm 阶段主要是对业务系统做确认提交，Try阶段执行成功并开始执行 Confirm阶段时，默认 Confirm阶段是不会出错的。即：只要Try成功，Confirm一定成功。
- Cancel 阶段主要是在业务执行错误，需要回滚的状态下执行的业务取消，预留资源释放。

注意Confirm 和 Cancel要保证操作的幂等性。


**三、本地消息表（异步确保）**

【最终一致性】

将分布式事务拆分成本地事务进行处理

消息生产方，需要额外建一个消息表，并记录消息发送状态。消息表和业务数据要在一个事务里提交，也就是说他们要在一个数据库里面。然后消息会经过MQ发送到消息的消费方。如果消息发送失败，会进行重试发送。

消息消费方，需要处理这个消息，并完成自己的业务逻辑。此时如果本地事务处理成功，表明已经处理成功了，如果处理失败，那么就会重试执行。如果是业务上面的失败，可以给生产方发送一个业务补偿消息，通知生产方进行回滚等操作。

生产方和消费方定时扫描本地消息表，把还没处理完成的消息或者失败的消息再发送一遍。如果有靠谱的自动对账补账逻辑，这种方案还是非常实用的。

**四、Sagas 事务模型**

Saga事务模型又叫做长时间运行的事务（Long-running-transaction）

![image](https://github.com/ZZULI-TECH/interview/blob/master/images/saga.png?raw=true)

[论文](https://www.cs.cornell.edu/andru/cs711/2002fa/reading/sagas.pdf)

参考：

- [聊聊分布式事务，再说说解决方案](https://www.cnblogs.com/savorboard/p/distributed-system-transaction-consistency.html)


[<font size=3 color="#409EFF">向本文提出修改或勘误建议</font>](https://github.com/mstao/mstao.github.io/blob/hexo/source/_posts/distributed-transaction.md)