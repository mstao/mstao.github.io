---
title: IllegalMonitorStateException异常分析
tags: [IllegalMonitorStateException,java]
author: Mingshan
categories: Java
date: 2018-7-3
---

当调用wait()， notify()等相关方法时，可能会产生这个异常，那么这个异常是什么意思呢？

<!-- more -->

抛出该异常原因:在Java中，每一个对象（Object/Class）都有一个监视器，当在同步代码块中，当前线程不是此监视器的所有者，也就是要在当前线程锁定对象，才能用锁定的对象此行这些方法，需要用到synchronized ，锁定什么对象就用什么对象来执行notify(), notifyAll(),wait(), wait(long), wait(long, int)操作，否则就会报IllegalMonitorStateException异常，原因异常。


下面这段代码就会抛出IllegalMonitorStateException异常，原因就是调用wait需要当前对象的监视器。

```Java
public class ThreadTest implements Runnable {
    private int i = 0;

    public synchronized void run() {
        try {
            ThreadTest.class.wait();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
 
        System.out.println(Thread.currentThread().getName() + ":" + i);
    }

    public static void main(String[] args) {
        new Thread(new ThreadTest()).start();
    }
}
 
```

所以对于synchronized的使用来说，通常这样使用

```Java
 synchronized(x) {
     x.notify();
 }
```


具体对于不同的监视对象而言，可以有以下几种考虑：

1. 锁定方法所对应的对象实例
    
```Java
public synchronized void work() {
       this.notify();
       // 或者直接写notify();
 }
```

 2. 类锁

```Java
public void work() {
   synchronized(xx.class) {
       xx.notify();
   }
}
```

3. 锁定其他对象

```Java
public Class Test{
    public Object lock = new Object();
    public static void method（）{
      synchronized (lock) {
         lock.notify();
      } 
    }
}
```

