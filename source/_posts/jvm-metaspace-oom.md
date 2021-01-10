---
title: 大量生成字节码导致元空间溢出问题排查
date: 2021-01-08
tags: JVM
categories: [JVM]
---

前几天生产环境出现了一个问题，gc日志里面某一个时间段出现了大量的Full GC，而且都是回收元空间内存失败了，最终导致了JVM停止运行，微服务中的某个服务发生了宕机。下面记录下排查该问题的过程。

<!-- more -->

首先我们根据服务器的CPU核心数和内存大小，设置了元空间的最大值为512M，这是前提。在服务GC日志中我们找到了JVM停止运行前记录GC日志，部分日志如下：

```
....上面还有
2021-01-06T15:11:59.801+0000: 80872.874: [CMS-concurrent-mark-start]
2021-01-06T15:11:59.812+0000: 80872.885: [Full GC (Metadata GC Threshold) 80872.885: [CMS2021-01-06T15:12:01.301+0000: 80874.374: [CMS-concurrent-mark: 1.499/1.501 secs] [Times: user=3.02 sys=0.00, real=1.50 secs] 
 (concurrent mode failure): 1393211K->1393200K(2621440K), 5.7310715 secs] 1405136K->1393200K(4037056K), [Metaspace: 495957K->495957K(1490944K)], 5.7317209 secs] [Times: user=7.23 sys=0.00, real=5.73 secs] 
2021-01-06T15:12:05.544+0000: 80878.616: [Full GC (Last ditch collection) 80878.616: [CMS: 1393200K->1393186K(2621440K), 4.1047512 secs] 1393200K->1393186K(4037056K), [Metaspace: 495953K->495953K(1490944K)], 4.1053278 secs] [Times: user=4.10 sys=0.00, real=4.11 secs] 
2021-01-06T15:12:09.650+0000: 80882.723: [Full GC (Metadata GC Threshold) 80882.723: [CMS: 1393186K->1393051K(2621440K), 4.2516109 secs] 1393190K->1393051K(4037056K), [Metaspace: 495953K->495953K(1490944K)], 4.2522440 secs] [Times: user=4.26 sys=0.00, real=4.25 secs] 
2021-01-06T15:12:13.903+0000: 80886.976: [Full GC (Last ditch collection) 80886.976: [CMS: 1393051K->1393051K(2621440K), 4.3760505 secs] 1393051K->1393051K(4037056K), [Metaspace: 495953K->495953K(1490944K)], 4.3766429 secs] [Times: user=4.38 sys=0.00, real=4.38 secs] 
2021-01-06T15:12:18.280+0000: 80891.353: [Full GC (Metadata GC Threshold) 80891.353: [CMS: 1393051K->1393051K(2621440K), 4.2536561 secs] 1393057K->1393051K(4037056K), [Metaspace: 495953K->495953K(1490944K)], 4.2542918 secs] [Times: user=4.25 sys=0.00, real=4.25 secs] 
2021-01-06T15:12:22.534+0000: 80895.607: [Full GC (Last ditch collection) 80895.607: [CMS: 1393051K->1393051K(2621440K), 4.1449958 secs] 1393051K->1393051K(4037056K), [Metaspace: 495953K->495953K(1490944K)], 4.1456156 secs] [Times: user=4.15 sys=0.00, real=4.15 secs] 
2021-01-06T15:12:26.681+0000: 80899.754: [Full GC (Metadata GC Threshold) 80899.754: [CMS: 1393051K->1393053K(2621440K), 4.2307102 secs] 1393984K->1393053K(4037056K), [Metaspace: 495953K->495953K(1490944K)], 4.2313259 secs] [Times: user=4.24 sys=0.00, real=4.23 secs] 
2021-01-06T15:12:30.912+0000: 80903.985: [Full GC (Last ditch collection) 80903.985: [CMS: 1393053K->1393053K(2621440K), 4.1873581 secs] 1393053K->1393053K(4037056K), [Metaspace: 495953K->495953K(1490944K)], 4.1879824 secs] [Times: user=4.19 sys=0.00, real=4.19 secs] 
2021-01-06T15:12:35.101+0000: 80908.174: [Full GC (Metadata GC Threshold) 80908.174: [CMS: 1393053K->1393053K(2621440K), 4.1460931 secs] 1393057K->1393053K(4037056K), [Metaspace: 495953K->495953K(1490944K)], 4.1467009 secs] [Times: user=4.15 sys=0.00, real=4.15 secs] 
2021-01-06T15:12:39.248+0000: 80912.321: [Full GC (Last ditch collection) 80912.321: [CMS: 1393053K->1393053K(2621440K), 4.1433462 secs] 1393053K->1393053K(4037056K), [Metaspace: 495953K->495953K(1490944K)], 4.1439592 secs] [Times: user=4.14 sys=0.00, real=4.14 secs] 
2021-01-06T15:12:43.393+0000: 80916.465: [Full GC (Metadata GC Threshold) 80916.465: [CMS: 1393053K->1393052K(2621440K), 4.1256451 secs] 1393055K->1393052K(4037056K), [Metaspace: 495953K->495953K(1490944K)], 4.1262638 secs] [Times: user=4.12 sys=0.00, real=4.13 secs] 
2021-01-06T15:12:47.519+0000: 80920.592: [Full GC (Last ditch collection) 80920.592: [CMS: 1393052K->1393052K(2621440K), 4.1477354 secs] 1393052K->1393052K(4037056K), [Metaspace: 495953K->495953K(1490944K)], 4.1483660 secs] [Times: user=4.15 sys=0.00, real=4.15 secs] 
2021-01-06T15:12:51.668+0000: 80924.741: [GC (CMS Initial Mark) [1 CMS-initial-mark: 1393052K(2621440K)] 1393052K(4037056K), 0.0102452 secs] [Times: user=0.01 sys=0.00, real=0.01 secs] 
2021-01-06T15:12:51.679+0000: 80924.751: [CMS-concurrent-mark-start]
2021-01-06T15:12:51.684+0000: 80924.757: [Full GC (Metadata GC Threshold) 80924.757: [CMS2021-01-06T15:12:53.207+0000: 80926.280: [CMS-concurrent-mark: 1.527/1.529 secs] [Times: user=3.07 sys=0.00, real=1.53 secs] 
 (concurrent mode failure): 1393052K->1392942K(2621440K), 6.0479947 secs] 1403298K->1392942K(4037056K), [Metaspace: 495957K->495957K(1490944K)], 6.0486493 secs] [Times: user=7.58 sys=0.00, real=6.05 secs] 
2021-01-06T15:12:57.733+0000: 80930.806: [Full GC (Last ditch collection) 80930.806: [CMS: 1392942K->1392914K(2621440K), 4.1101923 secs] 1392942K->1392914K(4037056K), [Metaspace: 495957K->495957K(1490944K)], 4.1107740 secs] [Times: user=4.11 sys=0.00, real=4.11 secs] 
2021-01-06T15:13:01.854+0000: 80934.926: [Full GC (Metadata GC Threshold) 80934.927: [CMS: 1392914K->1392737K(2621440K), 4.5043935 secs] 1412774K->1392737K(4037056K), [Metaspace: 495965K->495965K(1490944K)], 4.5050193 secs] [Times: user=4.51 sys=0.00, real=4.51 secs] 
2021-01-06T15:13:06.359+0000: 80939.432: [Full GC (Last ditch collection) 80939.432: [CMS: 1392737K->1392529K(2621440K), 4.1532759 secs] 1392737K->1392529K(4037056K), [Metaspace: 495961K->495961K(1490944K)], 4.1539027 secs] [Times: user=4.15 sys=0.00, real=4.15 secs] 
2021-01-06T15:13:10.513+0000: 80943.586: [Full GC (Metadata GC Threshold) 80943.586: [CMS: 1392529K->1392529K(2621440K), 4.1123191 secs] 1392529K->1392529K(4037056K), [Metaspace: 495953K->495953K(1490944K)], 4.1129984 secs] [Times: user=4.11 sys=0.00, real=4.11 secs] 
2021-01-06T15:13:14.627+0000: 80947.699: [Full GC (Last ditch collection) 80947.699: [CMS: 1392529K->1392529K(2621440K), 4.1928267 secs] 1392529K->1392529K(4037056K), [Metaspace: 495953K->495953K(1490944K)], 4.1934070 secs] [Times: user=4.19 sys=0.00, real=4.20 secs] 
2021-01-06T15:13:18.821+0000: 80951.893: [Full GC (Metadata GC Threshold) 80951.894: [CMS: 1392529K->1392529K(2621440K), 4.1205243 secs] 1392533K->1392529K(4037056K), [Metaspace: 495953K->495953K(1490944K)], 4.1211506 secs] [Times: user=4.12 sys=0.00, real=4.12 secs] 
2021-01-06T15:13:22.942+0000: 80956.015: [Full GC (Last ditch collection) 80956.015: [CMS: 1392529K->1392529K(2621440K), 4.1649192 secs] 1392529K->1392529K(4037056K), [Metaspace: 495953K->495953K(1490944K)], 4.1655270 secs] [Times: user=4.17 sys=0.00, real=4.17 secs] 
2021-01-06T15:13:27.108+0000: 80960.181: [GC (CMS Initial Mark) [1 CMS-initial-mark: 1392529K(2621440K)] 1392531K(4037056K), 0.0096418 secs] [Times: user=0.01 sys=0.00, real=0.01 secs] 
2021-01-06T15:13:27.118+0000: 80960.191: [CMS-concurrent-mark-start]
2021-01-06T15:13:27.129+0000: 80960.202: [GC (GCLocker Initiated GC) 80960.202: [ParNew: 43734K->3117K(1415616K), 0.0145302 secs] 1436264K->1395647K(4037056K), 0.0146736 secs] [Times: user=0.08 sys=0.00, real=0.01 secs] 
2021-01-06T15:13:27.154+0000: 80960.227: [GC (GCLocker Initiated GC) 80960.227: [ParNew: 9710K->833K(1415616K), 0.0146234 secs] 1402240K->1393362K(4037056K), 0.0147452 secs] [Times: user=0.07 sys=0.00, real=0.02 secs] 
2021-01-06T15:13:27.171+0000: 80960.244: [GC (GCLocker Initiated GC) 80960.244: [ParNew: 4611K->539K(1415616K), 0.0144761 secs] 1397140K->1393069K(4037056K), 0.0145790 secs] [Times: user=0.07 sys=0.00, real=0.01 secs] 
2021-01-06T15:13:27.186+0000: 80960.259: [Full GC (Metadata GC Threshold) 80960.259: [CMS2021-01-06T15:13:28.678+0000: 80961.751: [CMS-concurrent-mark: 1.509/1.560 secs] [Times: user=3.30 sys=0.00, real=1.56 secs] 
 (concurrent mode failure): 1392529K->1392795K(2621440K), 6.0348658 secs] 1393229K->1392795K(4037056K), [Metaspace: 495978K->495978K(1490944K)], 6.0354949 secs] [Times: user=7.53 sys=0.00, real=6.04 secs] 
2021-01-06T15:13:33.222+0000: 80966.295: [Full GC (Last ditch collection) 80966.295: [CMS: 1392795K->1392630K(2621440K), 4.2120809 secs] 1392795K->1392630K(4037056K), [Metaspace: 495974K->495974K(1490944K)], 4.2127067 secs] [Times: user=4.21 sys=0.00, real=4.21 secs] 
2021-01-06T15:13:37.436+0000: 80970.508: [Full GC (Metadata GC Threshold) 80970.508: [CMS: 1392630K->1392630K(2621440K), 4.2504469 secs] 1396226K->1392630K(4037056K), [Metaspace: 495957K->495957K(1490944K)], 4.2510666 secs] [Times: user=4.25 sys=0.00, real=4.25 secs] 
2021-01-06T15:13:41.687+0000: 80974.760: [Full GC (Last ditch collection) 80974.760: [CMS
```

从上面日志中，我们发现：`[Metaspace: 495957K->495957K(1490944K)], 4.2510666 secs]`，一次Full GC 用了4秒多，而且元空间已经无法回收了（前后数值一样），所以初步判定为是元空间内存溢出了。

我们知道元空间是JDK8里面的概念，用来代替原来方法区，里面存储的包含编译后的类信息，也就是字节码，如果这个区域出现了OOM，那么应该是系统某一时刻生成了大量的字节码，直接把元空间搞爆了，所以接下来我们要看下JVM挂掉生成的Java内存Dump文件，一般是`java_pid8.hprof`这种格式。

我们用Eclipse MAT软件来分析。用MAT打开Java内存Dump文件，直接打开内存泄漏分析，预览界面如下：

![image](https://github.com/ZZULI-TECH/interview/blob/master/images/jvm/analysis/2021-01-10/20210110195816.png?raw=true)

从上面的图中我们看出，`org.springframework.boot.loader.LaunchedURLClassLoader @ 0x720002180" occupies 1,242,173,552 (90.99%) bytes. The memory is accumulated in one instance of "java.util.Hashtable$Entry[]" loaded by "<system class loader>` 这个都占用了一个1.2G左右，啥玩意占用这么大？接下来我们看看详情，如下图所示：

![image](https://github.com/ZZULI-TECH/interview/blob/master/images/jvm/analysis/2021-01-10/20210110195000.png?raw=true)

从图中看出，`org.springframework.boot.loader.LaunchedURLClassLoader`这个类加载器加载了一个叫`java.util.Vector`的对象，这个里面有一个`javassist.ClassPool`的东西，这个ClassPool一堆Hashtable。。。看到这个`javassist`，我觉得此时问题已经有头绪了，因为这个玩意就是动态生成字节码的工具包阿，估计是项目代码中大量调用`javassist`动态生成字节码，直接把生产环境元空间干爆了。知道了方向，下面我们来具体看看这个`javassist.ClassPool`里面的Map到底存了啥玩意。我们直接看支配树：

![image](https://github.com/ZZULI-TECH/interview/blob/master/images/jvm/analysis/2021-01-10/20210110215810.png?raw=true)

从上面支配树分析来看，我们直接看到刚才那一堆Hashtable 的key是什么，此时就可以知道通过`javassist`生成了什么字节码，就可以直接定位到具体代码了。

从上面的分析来看，我们代码中要慎重使用`javassist`生成动态字节码，比如有些场景，每次调用生成的字节码都一样，就不必次次都重新生成，可以做成静态的（static final），这样就可以避免重复生成。


