---
title: Spring-AQMP出现No available channels错误
tags: [RabbitMQ]
author: Mingshan
categories: RabbitMQ
date: 2019-11-28
---

近日发现项目日志中有关于MQ的异常日志，Rabbit在进行自我检查时，发现没有Channel可用了，报出`org.springframework.amqp.AmqpTimeoutException: No available channels`异常，这个异常具体是由什么引起的呢？

<!-- more -->

下面是完整的日志输出：

```
2019-11-27 09:52:46,732 [DiscoveryClient-InstanceInfoReplicator-0] [-] WARN  org.springframework.boot.actuate.amqp.RabbitHealthIndicator - Rabbit health check failed
org.springframework.amqp.AmqpTimeoutException: No available channels
	at org.springframework.amqp.rabbit.connection.CachingConnectionFactory.getChannel(CachingConnectionFactory.java:443) ~[spring-rabbit-2.0.4.RELEASE.jar:2.0.4.RELEASE]
	at org.springframework.amqp.rabbit.connection.CachingConnectionFactory.access$1500(CachingConnectionFactory.java:94) ~[spring-rabbit-2.0.4.RELEASE.jar:2.0.4.RELEASE]
	at org.springframework.amqp.rabbit.connection.CachingConnectionFactory$ChannelCachingConnectionProxy.createChannel(CachingConnectionFactory.java:1171) ~[spring-rabbit-2.0.4.RELEASE.jar:2.0.4.RELEASE]
	at org.springframework.amqp.rabbit.core.RabbitTemplate.doExecute(RabbitTemplate.java:1803) ~[spring-rabbit-2.0.4.RELEASE.jar:2.0.4.RELEASE]
	at org.springframework.amqp.rabbit.core.RabbitTemplate.execute(RabbitTemplate.java:1771) ~[spring-rabbit-2.0.4.RELEASE.jar:2.0.4.RELEASE]
	at org.springframework.amqp.rabbit.core.RabbitTemplate.execute(RabbitTemplate.java:1752) ~[spring-rabbit-2.0.4.RELEASE.jar:2.0.4.RELEASE]
	at org.springframework.boot.actuate.amqp.RabbitHealthIndicator.getVersion(RabbitHealthIndicator.java:48) ~[spring-boot-actuator-2.0.3.RELEASE.jar:2.0.3.RELEASE]
	at org.springframework.boot.actuate.amqp.RabbitHealthIndicator.doHealthCheck(RabbitHealthIndicator.java:44) ~[spring-boot-actuator-2.0.3.RELEASE.jar:2.0.3.RELEASE]
	at org.springframework.boot.actuate.health.AbstractHealthIndicator.health(AbstractHealthIndicator.java:84) ~[spring-boot-actuator-2.0.3.RELEASE.jar:2.0.3.RELEASE]
	at org.springframework.boot.actuate.health.CompositeHealthIndicator.health(CompositeHealthIndicator.java:68) [spring-boot-actuator-2.0.3.RELEASE.jar:2.0.3.RELEASE]
	at org.springframework.cloud.netflix.eureka.EurekaHealthCheckHandler.getHealthStatus(EurekaHealthCheckHandler.java:103) [spring-cloud-netflix-eureka-client-2.0.0.RELEASE.jar:2.0.0.RELEASE]
	at org.springframework.cloud.netflix.eureka.EurekaHealthCheckHandler.getStatus(EurekaHealthCheckHandler.java:99) [spring-cloud-netflix-eureka-client-2.0.0.RELEASE.jar:2.0.0.RELEASE]
	at com.netflix.discovery.DiscoveryClient.refreshInstanceInfo(DiscoveryClient.java:1382) [eureka-client-1.9.2.jar:1.9.2]
	at com.netflix.discovery.InstanceInfoReplicator.run(InstanceInfoReplicator.java:117) [eureka-client-1.9.2.jar:1.9.2]
	at java.util.concurrent.Executors$RunnableAdapter.call(Executors.java:511) [na:1.8.0_45]
	at java.util.concurrent.FutureTask.run$$$capture(FutureTask.java:266) [na:1.8.0_45]
	at java.util.concurrent.FutureTask.run(FutureTask.java) [na:1.8.0_45]
	at java.util.concurrent.ScheduledThreadPoolExecutor$ScheduledFutureTask.access$201(ScheduledThreadPoolExecutor.java:180) [na:1.8.0_45]
	at java.util.concurrent.ScheduledThreadPoolExecutor$ScheduledFutureTask.run(ScheduledThreadPoolExecutor.java:293) [na:1.8.0_45]
	at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1142) [na:1.8.0_45]
	at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:617) [na:1.8.0_45]
	at java.lang.Thread.run(Thread.java:745) [na:1.8.0_45]
```

从报错的堆栈信息可以看出，问题发生在`org.springframework.amqp.rabbit.connection.CachingConnectionFactory.getChannel`，这个方法去获取Channel失败了。我们打开源码看下：

```Java
private Channel getChannel(ChannelCachingConnectionProxy connection, boolean transactional) {
	Semaphore checkoutPermits = null;
	if (this.channelCheckoutTimeout > 0) {
		checkoutPermits = this.checkoutPermits.get(connection);
		if (checkoutPermits != null) {
			try {
				if (!checkoutPermits.tryAcquire(this.channelCheckoutTimeout, TimeUnit.MILLISECONDS)) {
					throw new AmqpTimeoutException("No available channels");
				}
				if (logger.isDebugEnabled()) {
					logger.debug(
						"Acquired permit for " + connection + ", remaining:" + checkoutPermits.availablePermits());
				}
			}
			catch (InterruptedException e) {
				Thread.currentThread().interrupt();
				throw new AmqpTimeoutException("Interrupted while acquiring a channel", e);
			}
		}
		else {
			throw new IllegalStateException("No permits map entry for " + connection);
		}
	}
	
	// ......
}
```

这里是从`checkoutPermits`这个map来获取一个信号量(Semaphore)，然后再超时模式获取一个Channel，失败的话就会报上面的错，这个Map声明如下：

```Java
private final Map<Connection, Semaphore> checkoutPermits = new HashMap<>();
```

key是`Connection`，value是`Semaphore`。`Semaphore`的功能是相当于一把锁，可以允许多个线程同时获取这把锁，只不过有个最大值，超出这个最大值的线程都需要等待。我们知道无论是生产者还是消费者，都需要和 RabbitMQ Broker建立连接，这个连接就是一条TCP连接，也就是上面的`Connection`。一旦 TCP 连接建立起来，客户端紧接着可以创建一个AMQP信道（Channel）。每个线程把持一个信道，所以信道复用了 Connection 的 TCP 连接。信道是建立在 Connection之上的虚拟连接，RabbitMQ 处理的每条 AMQP 指令都是通过信道完成的。如下所示：

![image](https://github.com/mstao/static/blob/master/mq/rabbitmq_channel1.png?raw=true)

在Spring-AQMP中，利用`Semaphore`来限制每个`Connection`最大Channel数，该值默认是25，如下所示：

```Java
private static final int DEFAULT_CHANNEL_CACHE_SIZE = 25;
```

如果应用需要的Channel数超过25，注意一定要配置一下，要不然会直接报上面的错误，默认超时时间是0，单位毫秒。
配置方式如下：

```
spring.rabbitmq.cache.channel.checkout-timeout=0
spring.rabbitmq.cache.channel.size=25
```
