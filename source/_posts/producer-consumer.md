---
title: 从生产者消费者模型窥探多线程并发
tags: [Producer–consumer,java,JUC]
author: Mingshan
categories: [Java, JUC]
date: 2019-02-25
---

[生产者消费者问题](https://en.wikipedia.org/wiki/Producer%E2%80%93consumer_problem)是一个常见而且经典的问题，相信了解过多线程或者消息队列的同学对这个名词并不陌生。正如Java常用的设计模式一样，生产者消费者问题是为了解决某一类问题而存在，参阅维基百科对[Producer–consumer problem](https://en.wikipedia.org/wiki/Producer%E2%80%93consumer_problem)的描述：

<!-- more -->

> In computing, the producer–consumer problem[1][2] (also known as the bounded-buffer problem) is a classic example of a multi-process synchronization problem. The problem describes two processes, the producer and the consumer, who share a common, fixed-size buffer used as a queue. The producer's job is to generate data, put it into the buffer, and start again. At the same time, the consumer is consuming the data (i.e., removing it from the buffer), one piece at a time. The problem is to make sure that the producer won't try to add data into the buffer if it's full and that the consumer won't try to remove data from an empty buffer.

生产者消费者问题其实是多线程同步问题，它有两个主体，生产者（producer）和消费者（consumer），两者共享一个被称为队列（queue）的数据区域，其中生产者向队列中放入数据，消费者从队列中消费数据。当然此时需要保证：当队列为空时，消费者等待；队列满时，生产者等待。假设生产者和消费者都不止一个，此时必然涉及到线程并发问题，同时数据的存储方式（queue）决定了生产者消费者模型在实际应用中的性能和吞吐量。下面是生产者消费者模型图示：

![image](https://github.com/ZZULI-TECH/interview/blob/master/images/Producer–consumer%20problem.png?raw=true)

了解过生产者消费者问题的概念，那么我们对其在实际应用比较好奇，从上面的分析可以看出，生产者消费者模型可以用来**系统解耦**，可以将生产者看作一个系统，消费者看作一个系统，这样两个系统就通过缓冲区将两者的逻辑隔离开了；我们还可以发现，利用生产者消费者模型可以实现异步，当生产者作为系统的一部分逻辑，我们希望其不影响主线业务逻辑的走向，比如日志系统等，不能因为记录日志而影响系统的响应时间；再复杂到市场上商用的消息队列（MQ），ACK机制等，这里就不细说。

在实际应用中，我们往往需要考虑**并发问题**。上面提到在put和get时，为防止数据错乱，必须进行并发控制。采用怎样的并发策略，往往需要根据实际业务需求来考虑。比如在同一JVM内存中，我们可以采用synchroized或者ReentrantLock来对put和get操作加锁，同时调用对应的等待与通知线程方法来挂起或者释放线程。在分布式环境中，可以采用分布式锁等复杂的策略来实现。在对性能要求十分高的情况下，可能需要自己定制生产者消费者模型，自定义存储结构等，例如Disruptor框架。

接下来我们考虑几种经典的生产者消费者模型简易实现和比较复杂的Disruptor框架，一步步了解生产者消费者模型。

### synchronized方式

熟悉synchronized的同学此时肯定会想到利用synchronized、wait、notify来实现生产者消费者模型，因为上面已经抽象过了，生产者和消费者其实都在操作共享资源，在多个生产者和多个消费者的情况下，肯定会涉及多线程安全和并发问题，考虑下面几种情况（这里的共享资源数量有最大限制）：

1. 当共享资源为空时，消费者线程必须要阻塞，等待生产者产生资源；当生产者产生过资源后，必须唤醒被阻塞的线程，使其能够继续消费
2. 当共享资源数量大于其最大限制时，必须阻塞生产者线程，等消费者消费；当消费者消费后，必须唤醒阻塞的生产者线程，使其能够继续生产

上面的流程考虑了多线程并发的情况，好像比较简单，那么下面就开始代码。

首先我们需要模拟一下共享资源，在Buffer中添加put和take方法，用来实现上述的流程：

```Java
/**
 * 共享资源
 */
public class Buffer {
    // 已使用共享资源的数量
    private int count = 0;
    // 共享资源最大数量
    private int size = 10;

    /**
     * 生产者产生资源
     *
     * @throws InterruptedException
     */
    public synchronized void put() throws InterruptedException {
        while (count == size) {
            // 共享资源满了，生产者线此时需要阻塞，等待消费者消费共享资源再进行生产
            System.out.println("[Put] Current thread " + Thread.currentThread().getName() + " is waiting");
            this.wait();
        }

        count++;
        System.out.println("[Put] Current thread " + Thread.currentThread().getName()
                + " add 1 item, current count: " + count);
        this.notifyAll();
    }

    /**
     * 消费者消费资源
     */
    public synchronized void take() throws InterruptedException {
        // 如果共享资源为0，代表共享资源已被用尽，等下生产者生产再进行消费
        while (count == 0) {
            // 共享资源满了，生产者线此时需要阻塞，等待消费者消费共享资源再进行生产
            System.out.println("[Take] Current thread " + Thread.currentThread().getName() + " is waiting");
            this.wait();
        }

        count--;
        System.out.println("[Take] Current thread " + Thread.currentThread().getName()
                + " remove 1 item, current count: " + count);
        this.notifyAll();

    }

}
```
首先我们定义size表示共享资源最大数量，count表示已使用共享资源数量，提供了put和take方法，注意这两个方法都用`synchronized`关键字修饰，保证并发安全，同时使用`wait()` 和 `notifyAll()`来阻塞和唤醒当前线程。

下面生产者代码，生产者通过构造器传入Buffer，然后在run方法里面写个无限循环，内部执行`resource.put()`

```Java
public class Producer implements Runnable {
    private Buffer buffer;

    public Producer(Buffer buffer) {
        this.buffer = buffer;
    }

    @Override
    public void run() {
        while (true) {
            try {
                Thread.sleep(1000);
                buffer.put();
            } catch (InterruptedException e) {
                e.printStackTrace();
                Thread.currentThread().interrupt();
            }
        }
    }
}
```

下面消费者代码，消费者通过构造器传入Buffer，然后在run方法里面写个无限循环，内部执行`resource.take()`

```Java
public class Consumer implements Runnable {
    private Buffer buffer;

    public Consumer(Buffer buffer) {
        this.buffer = buffer;
    }

    @Override
    public void run() {
        while (true) {
            try {
                Thread.sleep(1000);
                buffer.take();
            } catch (InterruptedException e) {
                e.printStackTrace();
                Thread.currentThread().interrupt();
            }
        }
    }
}
```

通过上面的代码发现，用synchronized方式非常容易实现生产者消费者模型，只是需要注意线程安全问题。

### ReentrantLock与Condition

了解了利用synchronized、wait、notify来实现生产者消费者模型，在JDK1.5之后，引入了Java并发包，其中提供了ReentrantLock、Condition来实现与synchronized、wait、notify类似的功能，只不过是基于AQS实现的。了解ReentrantLock、Condition对于了解多线程并发是十分有帮助的，可以让你更加细致地了解到多线程并发的内部实现方式和原理，更加深入可能会了解AQS的设计思想，各种同步器实现方式以及应用场景。当然，这些需要根据多线程的应用场景来分析就最好不过了。这里我们用ReentrantLock与Condition来模拟实现生产者消费者模型。

和上面抽象的一样，我们首先来创建一个Buffer类，也提供了put和take方法，只不过我们这里不用synchronized了，改用ReentrantLock，同时调用lock的newCondition方法，新建两个Condition，一个为 notEmpty， 代表线程读数据条件；一个为notFull，代表线程写数据条件。具体使用方式如下代码所示：

```Java
public class Buffer {
    private List<String> queue;
    private int size;

    private final Lock lock = new ReentrantLock();
    // 读线程条件
    private final Condition notEmpty = lock.newCondition();
    // 写线程条件
    private final Condition notFull = lock.newCondition();

    public Buffer() {
        this(10);
    }

    public Buffer(int size) {
        this.size = size;
        queue = new ArrayList<>();
    }

    public void put() throws InterruptedException {
        lock.lock();
        try {
            while (queue.size() == size) {
                System.out.println("[Put] Current thread " + Thread.currentThread().getName() + " is waiting");
                notNull.await();
            }

            queue.add("1");
            System.out.println("[Put] Current thread " + Thread.currentThread().getName()
                    + " add 1 item, current count: " + queue.size());
            notEmpty.signal();
        } finally {
            lock.unlock();
        }
    }

    public void take() throws InterruptedException {
        lock.lock();
        try {
            while (queue.size() == 0) {
                System.out.println("[Take] Current thread " + Thread.currentThread().getName() + " is waiting");
                notEmpty.await();
            }

            System.out.println("size = " + queue.size());

            queue.remove(queue.size() - 1);
            System.out.println("[Take] Current thread " + Thread.currentThread().getName()
                    + " remove 1 item, current count: " + queue.size());
            notFull.signal();
        } finally {
            lock.unlock();
        }

    }
}
```

上面代码中，首先List类型queue字段，来存储数据；size代表存储数据的最大数值；用ReentrantLock来代替sychronized，新建两个Condition，一个为 notEmpty， 代表线程读数据条件；一个为notFull，代表线程写数据条件。

在put方法中，首先会判断queue的数量有没有满，这里用while循环主要是防止过早或者意外的通知，只有符合条件才能够退出循环。如果满了，就调用 `notFull.await()`来挂起写线程，让写线程进入等待状态；当不满足while条件时，就可以向queue中写入数据，同时调用`notEmpty.signal()`来通知读线程可以读数据了。

在take方法中，首先判断queue是否有数据，这里也用while循环，也是考虑到防止过早或者意外的通知，只有符合条件才能够退出循环。如果队列有数据，就可以从queue中拿数据了，同时调用`notFull.signal()`通知写线程可以写数据了。

下面是生产者代码，将Buffer实例传入到Producer中，在run方法里调用Buffer的put方法：

```Java
public class Producer implements Runnable {
    private Buffer buffer;

    public Producer(Buffer buffer) {
        this.buffer = buffer;
    }

    @Override
    public void run() {
        while (true) {
            try {
                Thread.sleep(1000);
                buffer.put();
            } catch (InterruptedException e) {
                e.printStackTrace();
                Thread.currentThread().interrupt();
            }
        }
    }
}
```

下面是生产者代码，将Buffer实例传入到Consumer中，在run方法里调用Buffer的take方法：

```Java
public class Consumer implements Runnable {
    private Buffer buffer;

    public Consumer(Buffer buffer) {
        this.buffer = buffer;
    }

    @Override
    public void run() {
        while (true) {
            try {
                Thread.sleep(1000);
                buffer.take();
            } catch (InterruptedException e) {
                e.printStackTrace();
                Thread.currentThread().interrupt();
            }
        }
    }
}
```

上面的实现仿佛就是BlockingQueue的概念版，其实也类似，如果看过ArrayBloackingQueue的源码，上面的代码就很简单了。

### BlockingQueue方式

上面提到了BlockingQueue，所以我们完全可以将Buffer中的数据存储改成BlockingQueue，可以选择ArrayBlockingQueue 和 LinkedBlockingQueue等，Java的内置队列如下表所示：

队列	| 有界性	 |锁	|数据结构
---|---|---|---
ArrayBlockingQueue | bounded | 加锁 |	arraylist
LinkedBlockingQueue |	optionally-bounded | 加锁 |	linkedlist
ConcurrentLinkedQueue |	unbounded	| 无锁	| linkedlist
LinkedTransferQueue |	unbounded	|无锁|	linkedlist
PriorityBlockingQueue|	unbounded|	加锁|	heap
DelayQueue|	unbounded|	加锁|	heap

代码如下：

```Java
public class Buffer<T> {
    private BlockingQueue<T> queue;

    public Buffer() {
        queue = new ArrayBlockingQueue<>(10);
    }

    /**
     * 从队列中取出一条记录，并移除
     * @return 数据
     * @throws InterruptedException
     */
    public T poll() throws InterruptedException {
        return queue.poll();
    }

    /**
     * 从队列中取出一条记录，并移除，队列为空时阻塞
     * @return
     * @throws InterruptedException
     */
    public T take() throws InterruptedException {
        return queue.take();
    }

    /**
     * 放入一条记录到队列，为防止对业务的影响，采用超时机制
     * @param t 数据
     * @return 返回{@code ture} 入队成功，返回{@code false} 入队失败
     * @throws InterruptedException
     */
    public boolean offer(T t) throws InterruptedException {
        return queue.offer(t, 2, TimeUnit.SECONDS);
    }

    /**
     * 放入一条记录到队列
     * @param t 数据
     * @return 返回{@code ture} 入队成功，返回{@code false} 入队失败
     */
    public boolean add(T t) {
        return queue.add(t);
    }

    /**
     * 判断队列是否为空
     * @return 返回{@code ture} 队列为空，返回{@code false} 队列不为空
     */
    public boolean isEmpty() {
        return queue.isEmpty();
    }
}
```

消费者和生产者代码就不贴了，和上面的基本一致。

### Disruptor循环队列

Disruptor通过精巧的无锁设计实现了在高并发情形下的高性能，避免伪共享引入缓冲行填充，同时使用RingBuffer作为数据存储容器。具体实现原理后面再分析，下面是一个例子：

```Java
import com.lmax.disruptor.BlockingWaitStrategy;
import com.lmax.disruptor.EventFactory;
import com.lmax.disruptor.EventHandler;
import com.lmax.disruptor.RingBuffer;
import com.lmax.disruptor.dsl.Disruptor;
import com.lmax.disruptor.dsl.ProducerType;

import java.util.concurrent.ThreadFactory;

/**
 * @author mingshan
 */
public class Test {

    public static void main(String[] args) throws Exception {
        // 队列中的元素
        class Element {

            private int value;

            public int get() {
                return value;
            }

            public void set(int value) {
                this.value= value;
            }

        }

        // 生产者的线程工厂
        ThreadFactory threadFactory = new ThreadFactory() {
            @Override
            public Thread newThread(Runnable r) {
                return new Thread(r, "simpleThread");
            }
        };

        // RingBuffer生产工厂,初始化RingBuffer的时候使用
        EventFactory<Element> factory = new EventFactory<Element>() {
            @Override
            public Element newInstance() {
                return new Element();
            }
        };

        // 处理Event的handler
        EventHandler<Element> handler = new EventHandler<Element>() {
            @Override
            public void onEvent(Element element, long sequence, boolean endOfBatch)
            {
                System.out.println("Element: " + element.get());
            }
        };

        // 阻塞策略
        BlockingWaitStrategy strategy = new BlockingWaitStrategy();

        // 指定RingBuffer的大小
        int bufferSize = 16;

        // 创建disruptor，采用单生产者模式
        Disruptor disruptor = new Disruptor(factory, bufferSize, threadFactory, ProducerType.SINGLE, strategy);

        // 设置EventHandler
        disruptor.handleEventsWith(handler);

        // 启动disruptor的线程
        disruptor.start();

        RingBuffer<Element> ringBuffer = disruptor.getRingBuffer();

        for (int l = 0; true; l++) {
            // 获取下一个可用位置的下标
            long sequence = ringBuffer.next();
            try {
                // 返回可用位置的元素
                Element event = ringBuffer.get(sequence);
                // 设置该位置元素的值
                event.set(l);
            } finally {
                ringBuffer.publish(sequence);
            }
            Thread.sleep(10);
        }
    }
}
```

## References：

- [Producer–consumer problem](https://en.wikipedia.org/wiki/Producer%E2%80%93consumer_problem)
- [生产者/消费者模式的理解及实现](https://blog.csdn.net/u011109589/article/details/80519863)
- [关于synchronized、wait、notify已经notifyAll的使用](https://www.cnblogs.com/LipeiNet/p/6475851.html)
- [disruptor](https://github.com/LMAX-Exchange/disruptor/wiki/Getting-Started)
- [高性能队列——Disruptor](https://tech.meituan.com/2016/11/18/disruptor.html)
- [剖析Disruptor:为什么会这么快？（二）神奇的缓存行填充](http://ifeve.com/disruptor-cacheline-padding/)
- [disruptor 高性能之道](https://www.cnblogs.com/xiangnanl/p/9955714.html)


[<font size=3 color="#409EFF">向本文提出修改或勘误建议</font>](https://github.com/mstao/mstao.github.io/blob/hexo/source/_posts/producer-consumer.md)
