---
title: JVM报Out of Memory Error
author: Mingshan
tags: JVM
categories: JVM
date: 2020-03-11
---

在线上应用对JVM参数设置不合理的情况下，有可能发生OOM(Out of Memory Error)，不过OOM也分很多种，可以参考：[OutOfMemoryError详细介绍](https://mingshan.fun/2020/03/11/jvm-oom/)，下面就以我遇到的堆OOM来分析下相关日志信息。

通常OOM时会触发 Linux 内核里的 Out of Memory (OOM) killer，OOM killer会杀掉某个进程以腾出内存留给系统用，不致于让系统立刻崩溃。详细原因分析参考：[linux 终端报错 Out of memory: Kill process[PID] [process name] score问题分析](http://www.111com.net/sys/CentOS/84755.htm)，在应用启动之前，需要打印GC日志以及保存JVM错误日志，设置以下JVM参数：

```
-XX:+PrintGCDetails
-XX:HeapDumpPath=targetDir
-XX:ErrorFile=targetDir\\hs_err_pid%p.log
-Xloggc:targetDir\\gc.log
-XX:HeapDumpPath=targetDir
```
<!-- more -->

下面是虚拟机发生错误的相关日志信息：


发生错误的可能原因，从下面的文件可以看出内存不够用了，是被Linux系统给干调了。
```
# Possible reasons:
#   The system is out of physical RAM or swap space
#   In 32 bit mode, the process size limit was hit
# Possible solutions:
#   Reduce memory load on the system
#   Increase physical memory or swap space
#   Check if swap backing store is full
#   Use 64 bit Java on a 64 bit OS
#   Decrease Java heap size (-Xmx/-Xms)
#   Decrease number of Java threads
#   Decrease Java thread stack sizes (-Xss)
#   Set larger code cache with -XX:ReservedCodeCacheSize=
# This output file may be truncated or incomplete.
#
#  Out of Memory Error (os_linux.cpp:2716), pid=6, tid=139907425556224
#
# JRE version: Java(TM) SE Runtime Environment (8.0_11-b12) (build 1.8.0_11-b12)
# Java VM: Java HotSpot(TM) 64-Bit Server VM (25.11-b03 mixed mode linux-amd64 compressed oops)
# Core dump written. Default location: /apache-tomcat/webapps/core or core.6
#
```

然后是发生OMM时堆的内存大小，熟悉JVM内存分布的同学相信可以看懂，注意到`concurrent mark-sweep generation`比较陌生，而从日志来看，缺少老年代，这个指的是老年代，空间几乎满了，接着发现 Metaspace 居然也快满了，这可不是什么好兆头。

```
Heap:
 par new generation   total 460096K, used 427042K [0x00000004c0000000, 0x00000004df330000, 0x00000004df330000)
  eden space 409024K,  96% used [0x00000004c0000000, 0x00000004d80fd740, 0x00000004d8f70000)
  from space 51072K,  64% used [0x00000004dc150000, 0x00000004de15b358, 0x00000004df330000)
  to   space 51072K,   0% used [0x00000004d8f70000, 0x00000004d8f70000, 0x00000004dc150000)
 concurrent mark-sweep generation total 3718084K, used 3154040K [0x00000004df330000, 0x00000005c2221000, 0x00000007c0000000)
 Metaspace       used 117735K, capacity 119550K, committed 120436K, reserved 1155072K
  class space    used 15035K, capacity 15538K, committed 15868K, reserved 1048576K
```

最后我们看下gc历史，这个是比较重要的，默认文件里面显示最近十次的gc日志，这里只复制了最近两次，因为其他的都是一样的。从下面gc日志可以看出，全是full gc（十次全是），这样搞，jvm不得炸了，经过几次回收， Metaspace 空间都没变化而且都满了，这样是十分不合理的。

```
GC Heap History (10 events):
Event: 86441.747 GC heap before
{Heap before GC invocations=66 (full 8):
 par new generation   total 460096K, used 428048K [0x00000004c0000000, 0x00000004df330000, 0x00000004df330000)
  eden space 409024K,  96% used [0x00000004c0000000, 0x00000004d81d8720, 0x00000004d8f70000)
  from space 51072K,  64% used [0x00000004d8f70000, 0x00000004daf9bac8, 0x00000004dc150000)
  to   space 51072K,   0% used [0x00000004dc150000, 0x00000004dc150000, 0x00000004df330000)
 concurrent mark-sweep generation total 2996204K, used 2964789K [0x00000004df330000, 0x000000059612b000, 0x00000007c0000000)
 Metaspace       used 118107K, capacity 120144K, committed 120436K, reserved 1155072K
  class space    used 15090K, capacity 15637K, committed 15868K, reserved 1048576K
Event: 86441.981 GC heap after
Heap after GC invocations=67 (full 8):
 par new generation   total 460096K, used 32943K [0x00000004c0000000, 0x00000004df330000, 0x00000004df330000)
  eden space 409024K,   0% used [0x00000004c0000000, 0x00000004c0000000, 0x00000004d8f70000)
  from space 51072K,  64% used [0x00000004dc150000, 0x00000004de17bcf8, 0x00000004df330000)
  to   space 51072K,   0% used [0x00000004d8f70000, 0x00000004d8f70000, 0x00000004dc150000)
 concurrent mark-sweep generation total 3127292K, used 3095864K [0x00000004df330000, 0x000000059e12f000, 0x00000007c0000000)
 Metaspace       used 118107K, capacity 120144K, committed 120436K, reserved 1155072K
  class space    used 15090K, capacity 15637K, committed 15868K, reserved 1048576K
}

....
```

解决方式需要关注几个方面，首先物理机的内存要够，不够的话，应用启动一会就会被杀死；适当调元空间的大小，特别是元空间的大小，这部分占用比较稳当，分配的内存过小，容易发生OOM；适当调整堆内存的大小，关注老年代的占用情况，这部分也容易发生OOM。
