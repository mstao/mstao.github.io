---
title: Arthas的使用并对JVM监控
tags: [Arthas,java]
author: Mingshan
categories: Java
date: 2018-9-21
---

Arthas 是Alibaba开源的Java诊断工具，可以查看Java进程的一些信息，例如运行情况、JVM相关参数、线程等信息，采用命令行交互模式，在Linux用着十分方便。

<!-- more -->

## 安装

在Linux系统中，首先创建一个文件下，然后在该文件下执行如下命令：

```
curl -L https://alibaba.github.io/arthas/install.sh | sh
```

该命令会下载`as.sh`脚本，执行该脚本的用户需要和目标进程具有相同的权限。比如以admin用户来执行： sudo su admin && ./as.sh 或 sudo -u admin -EH ./as.sh。 详细的启动脚本说明，请参考[这里](https://alibaba.github.io/arthas/start-arthas.html)。 如果attatch不上目标进程，可以查看~/logs/arthas/ 目录下的日志。

## 启动

接下来就需要执行`/as.sh`脚本，注意后面传入java 进程id

```
root@ubuntu:/usr/soft/arthas# ./as.sh 2616
```

启动画面如下图所示：

![image](https://github.com/ZZULI-TECH/interview/blob/master/images/arthas-start.png?raw=true)

**查看dashboard**

输入`dashboard`可以查看当前进程的信息、JVM内存使用情况，还可以看到当前操作系统和Java的版本信息

```
$ dashboard
ID     NAME                  GROUP          PRIORI STATE   %CPU   TIME   INTERRU DAEMON 
16     nioEventLoopGroup-3-1 system         10     RUNNABL 76     0:0    false   false  
21     Timer-for-arthas-dash system         10     RUNNABL 23     0:0    false   true   
12     AsyncAppender-Worker- system         9      WAITING 0      0:0    false   true   
10     Attach Listener       system         9      RUNNABL 0      0:0    false   true   
9      Common-Cleaner        InnocuousThrea 8      TIMED_W 0      0:0    false   true   
3      Finalizer             system         8      WAITING 0      0:0    false   true   
2      Reference Handler     system         10     RUNNABL 0      0:0    false   true   
4      Signal Dispatcher     system         9      RUNNABL 0      0:0    false   true   
20     as-command-execute-da system         10     TIMED_W 0      0:0    false   true   
14     job-timeout           system         9      TIMED_W 0      0:0    false   true   
Memory             used  total max    usage GC                                          
heap               18M   39M   233M   7.90% gc.copy.count         24                    
tenured_gen        16M   27M   161M         gc.copy.time(ms)      107                   
eden_space         2M    11M   64M    4.22% gc.marksweepcompact.c 2                     
survivor_space     0K    1408K 8256K  0.00% ount                                        
nonheap            23M   27M   -1                                                       
Runtime                                                                                 
os.name               Linux                                                             
os.version            4.15.0-29-generic                                                 
java.version          10.0.2                                                            
java.home             /usr/package/jdk-10.0                                             
                      .2                                                            
```

**查看JVM信息**


输入`JVM`参数可以查看JVM相关信息，这对JVM调试有很大帮助。

```
$ jvm
 RUNTIME                                                                                
----------------------------------------------------------------------------------------
 MACHINE-NAME             2616@ubuntu                                                   
 JVM-START-TIME           2018-09-20 20:19:32                                           
 MANAGEMENT-SPEC-VERSION  2.0                                                           
 SPEC-NAME                Java Virtual Machine Specification                            
 SPEC-VENDOR              Oracle Corporation                                            
 SPEC-VERSION             10                                                            
 VM-NAME                  Java HotSpot(TM) 64-Bit Server VM                             
 VM-VENDOR                "Oracle Corporation"                                          
 VM-VERSION               10.0.2+13                                                     
 INPUT-ARGUMENTS          []                                                            
 CLASS-PATH               .:/usr/package/jdk-10.0.2/lib/                                
 BOOT-CLASS-PATH                                                                        
 LIBRARY-PATH             /usr/java/packages/lib:/usr/lib64:/lib64:/lib:/usr/lib        
                                                                                        
----------------------------------------------------------------------------------------
 CLASS-LOADING                                                                          
----------------------------------------------------------------------------------------
 LOADED-CLASS-COUNT       2818                                                          
 TOTAL-LOADED-CLASS-COUN  2818                                                          
 T                                                                                      
 UNLOADED-CLASS-COUNT     0                                                             
 IS-VERBOSE               false                                                         
                                                                                        
----------------------------------------------------------------------------------------
 COMPILATION                                                                            
----------------------------------------------------------------------------------------
 NAME                     HotSpot 64-Bit Tiered Compilers                               
 TOTAL-COMPILE-TIME       3875(ms)                                                      
                                                                                        
----------------------------------------------------------------------------------------
 GARBAGE-COLLECTORS                                                                     
----------------------------------------------------------------------------------------
 Copy                     25/110(ms)                                                    
 [count/time]                                                                           
 MarkSweepCompact         2/108(ms)                                                     
 [count/time]                                                                           
                                                                                        
----------------------------------------------------------------------------------------
 MEMORY-MANAGERS                                                                        
----------------------------------------------------------------------------------------
 CodeCacheManager         CodeHeap 'non-nmethods'                                       
                          CodeHeap 'profiled nmethods'                                  
                          CodeHeap 'non-profiled nmethods'                              
                                                                                        
 Metaspace Manager        Metaspace                                                     
                          Compressed Class Space                                        
                                                                                        
 Copy                     Eden Space                                                    
                          Survivor Space                                                
                                                                                        
 MarkSweepCompact         Eden Space                                                    
                          Survivor Space                                                
                          Tenured Gen                                                   
                                                                                        
                                                                                        
----------------------------------------------------------------------------------------
 MEMORY                                                                                 
----------------------------------------------------------------------------------------
 HEAP-MEMORY-USAGE        41689088/16777216/245301248/18364616                          
 [committed/init/max/use                                                                
 d]                                                                                     
 NO-HEAP-MEMORY-USAGE     32047104/7667712/-1/27755016                                  
 [committed/init/max/use                                                                
 d]                                                                                     
 PENDING-FINALIZE-COUNT   0                                                             
                                                                                        
----------------------------------------------------------------------------------------
 OPERATING-SYSTEM                                                                       
----------------------------------------------------------------------------------------
 OS                       Linux                                                         
 ARCH                     amd64                                                         
 PROCESSORS-COUNT         1                                                             
 LOAD-AVERAGE             0.53                                                          
 VERSION                  4.15.0-29-generic                                             
                                                                                        
----------------------------------------------------------------------------------------
 THREAD                                                                                 
----------------------------------------------------------------------------------------
 COUNT                    14                                                            
 DAEMON-COUNT             8                                                             
 LIVE-COUNT               15                                                            
 STARTED-COUNT            17                                                            
Affect(row-cnt:0) cost in 42 ms.

```

**class/classloader相关**

- sc——查看JVM已加载的类信息
- sm——查看已加载类的方法信息
- dump——dump 已加载类的 byte code 到特定目录
- redefine——加载外部的.class文件，redefine到JVM里
- jad——反编译指定已加载类的源码
- classloader——查看classloader的继承树，urls，类加载信息，使用classloader去getResource

详细参考：

- [Arthas Doc](https://alibaba.github.io/arthas/index.html)


[<font size=3 color="#409EFF">向本文提出修改或勘误建议</font>](https://github.com/mstao/mstao.github.io/blob/hexo/source/_posts/arthas-jvm-monitoring.md)