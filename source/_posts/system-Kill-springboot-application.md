---
title: SpringBoot应用被系统Kill相关问题分析
tags: [java,SpringBoot,Linux]
author: Mingshan
categories: Java
date: 2018-8-25
---

SpringBoot在启动的时候，有默认的JVM启动参数，在部署到Linux服务器上时，如果Linux内存不够时，会导致应用被操作系统kill掉，导致部署的服务莫名其妙的挂掉，就我遇到的项目而言，记录SpringBoot的log如下：

<!-- more -->

```
2018-08-23 07:53:25.231  INFO [demo,,,,] 1 --- [           main] cn.com.superv.provider.Application       : Started Application in 5.655 seconds (JVM running for 6.124)
2018-08-23 07:55:02.080  INFO [demo,,,,] 1 --- [       Thread-3] ConfigServletWebServerApplicationContext : Closing org.springframework.boot.web.servlet.context.AnnotationConfigServletWebServerApplicationContext@19dfb72a: startup date [Thu Aug 23 07:53:20 GMT 2018]; root of context hierarchy
2018-08-23 07:55:02.087  INFO [demo,,,,] 1 --- [       Thread-3] o.s.c.support.DefaultLifecycleProcessor  : Stopping beans in phase 2147483647
2018-08-23 07:55:02.087  INFO [demo,,,,] 1 --- [       Thread-3] o.s.j.e.a.AnnotationMBeanExporter        : Unregistering JMX-exposed beans on shutdown
2018-08-23 07:55:02.088  INFO [demo,,,,] 1 --- [       Thread-3] o.s.j.e.a.AnnotationMBeanExporter        : Unregistering JMX-exposed beans
2018-08-23 07:55:02.099  INFO [demo,,,,] 1 --- [       Thread-3] com.alibaba.druid.pool.DruidDataSource   : {dataSource-1} closed
```

才开始我以为代码写错了，导致应用启动崩溃，但我这个应用在另一台Linux机器上跑的挺好的，没有出现上面的问题。其中log中有以下日志：


```
Unregistering JMX-exposed beans on shutdown
Unregistering JMX-exposed beans
```

在网上查的信息说SpringBoot内嵌的Tomcat无法正常启动才导致这个错误，结合这个项目在其他机器上能运行，就排除了Tomcat的问题。后来我想想会不会是跟Linux内存有关系呢？我赶紧用`top`命令查看了服务器的内存的使用情况，我檫，16G的内存只剩下三百多兆了，看来是我项目运行的时候请求内存导致系统内存不足，被系统kill掉。这时可以用`jinfo -flags <pid>`命令来查看当前应用的JVM参数。运行 `dmesg | grep "(java)"`命令看看是不是操作系统把我的应用给干掉了，结果屏幕输入类似如下日志：

```
Out of memory: Kill process[PID] [process name] score
```
这又是什么意思呢？好像是内存溢出被操作系统干掉了，所以一查，还真是Out of memory 问题，这通常是因为某时刻应用程序大量请求内存导致系统内存不足造成的，这通常会触发 Linux 内核里的 Out of Memory (OOM) killer，OOM killer会杀掉某个进程以腾出内存留给系统用，不致于让系统立刻崩溃。详细原因分析参考：[linux 终端报错 Out of memory: Kill process[PID] [process name] score问题分析](http://www.111cn.net/sys/CentOS/84755.htm)

我在网上发现一哥们和我遇到了一样的问题，链接：[项目连续几天突然挂掉](https://blog.csdn.net/qq_35981283/article/details/62233725)，不过他的比较严重，可能刚开始没有整明白这个问题发生的原因，内心慌乱，其实仔细分析就可以查到问题发生的蛛丝马迹，从而解决问题。

找到问题发生的原因，接下来就要思考如何解决了。既然是内存的问题，那我们来改变下JVM参数，将最大堆内存(`-Xmx`)调小点，并禁止伸缩，将`-Xms`和`-Xmx`设置一样大，项目就可以运行了，`-Xmx`和`-Xms`具体值根据服务器内存和项目运行占用内存进行优化设置。

另外这个问题我在用Docker启动SpringBoot应用中也遇到过，利用`docker ps`命令可以发现该容器正在运行，但外界就是无法通过开放的端口来访问，利用命令`docker logs -f container-id`来查看容器启动的log，发现日志和上述的一样，所以也需要设置JVM参数, 命令如下：


```
docker run -e JAVA_OPTS='-Xmx512m' -p 8080:8080 -t springboot/spring-boot-docker
```

最后推荐一个JVM调优相关的网站： [一只懂JVM参数的狐狸](http://xxfox.perfma.com/)，非常好用，可以查看JVM参数介绍以及变迁，还可以根据机器生成推荐JVM参数，真好用啊^_^
