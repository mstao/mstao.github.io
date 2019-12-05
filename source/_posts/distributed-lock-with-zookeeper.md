---
title: 谈谈基于ZooKeeper的分布式锁
tags: [java, ZooKeeper, 分布式锁]
author: Mingshan
categories: [Java, ZooKeeper]
date: 2017-10-08
---
分布式锁可以基于以下几种方式实现：
- 基于数据库的乐观锁，用于分布式锁
- 基于缓存(Redis, memcached)实现分布式锁
- 基于ZooKeeper实现分布式锁

在这篇文章中，主要讲讲ZooKeeper以及分布式锁的实现，通过了解基于ZooKeeper分布式锁实现的原理，我们会对ZooKeeper有一个基本的了解。
## ZooKeeper介绍
首先谈谈ZooKeeper，ZooKeeper是一种为分布式应用所设计的高可用、高性能且一致的开源协调服务，它提供了一项基本服务：分布式锁服务。由于ZooKeeper的开源特性，后来我们的开发者在分布式锁的基础上，摸索了出了其他的使用方法：配置维护、组服务、分布式消息队列、分布式通知/协调等。

在ZooKeeper中，有一个被称为ZNode的节点，在该节点可以存储同步相关的数据，并且多个ZNode节点可以形成类似下图的结构。

![image](https://github.com/mstao/static/blob/master/blog/zookeeper.png?raw=true)

### 基本命令：

```
1. 查看节点
    ls /
2. 创建节点
    create /zk myData
3. 查看节点
    get /zk
4. 设置节点
    set /zk myData2
5. 删除节点
    delete /zk
6. 创建临时节点
    create -e /han data
7. 创建顺序节点
    create -s /han/ data
8. 创建顺序临时节点
    create -s -e /han/ data
```

### ZNode
客户端可以在一个ZNode上设置一个监视器（Watch），如果该ZNode数据发生变更，ZooKeeper会通知客户端，从而触发监视器中实现的逻辑的执行。其中ZNode有以下几种类型：
- PERSISTENT
- PERSISTENT_SEQUENTIAL
- EPHEMERAL
- EPHEMERAL_SEQUENTIAL

下面分别解释一下：
1. PERSISTENT为持久节点，持久节点是指在节点创建后，就一直存在，直到有删除操作来主动清除这个节点——不会因为创建该节点的客户端会话失效而消失。
ZooKeeper命令：

```
create /zk myData
```
2. PERSISTENT_SEQUENTIAL为持久顺序节点，基本特性与持久节点一致，但每个父节点会为他的第一级子节点维护一份时序，会记录每个子节点创建的先后顺序。命令：

```
create -s /han/ data
```
用这条命令的话，需要先创建/han节点，节点类型为PERSISTENT。
3. EPHEMERAL为临时节点，客户端会话失效或连接关闭后，该节点会被自动删除，且不能在临时节点下面创建子节点，命令：

```
create -e /han
```
如果在临时节点下面还要创建子节点，那么zk就会提示：Ephemerals cannot have children

4. EPHEMERAL_SEQUENTIAL为临时顺序节点，该节点的除了不是持久性节点，其他特性与持久顺序节点一致。命令：

```
create -s -e /han/ data
```
<!-- more -->
## 不利用EPHEMERAL_SEQUENTIA简单实现

首先我们需要一个业务,这里模拟一下订单生成，利用时间加上序号来表示，代码如下：

```java
package pers.mingshan.ZookeeperLock;

import java.text.SimpleDateFormat;
import java.util.Date;

/**
 * 订单号生成器
 * @author mingshan
 *
 */
public class OrderCodeGenerator {
    private int i = 0;

    public String getOrderCode() {
        SimpleDateFormat sdf = new SimpleDateFormat("yy-MM-dd HH:mm:ss - ");
        Date date = new Date();
        sdf.format(date);
        return sdf.format(date) + ++i;
    }
}

```
这里只是简单模拟一下，不考虑其他因素。
然后我们需要对外提供获取订单号的服务，这里我们用到了CountDownLatch, CountDownLatch是一个同步工具类，它允许一个或多个线程一直等待，直到其他线程的操作执行完后再执行。所以我们需要所有的线程都创建完毕后去同时生成订单编号，模拟一下并发。

```java
package pers.mingshan.ZookeeperLock;

import java.util.concurrent.CountDownLatch;
import java.util.concurrent.locks.Lock;

import org.apache.log4j.Logger;

public class OrderServiceImpl implements Runnable {

    private static OrderCodeGenerator generator = new OrderCodeGenerator();
    // 同时并发的线程数
    private static final int NUM = 10;
    private Logger logger = Logger.getLogger(getClass());
    // 根据线程数初始化倒计数器
    private static CountDownLatch cdl = new CountDownLatch(NUM);
    // lock锁
    private static final Lock lock = new ZookeeperDistributeLock();

    public void createOrderCode() {
        String orderCode = null;

        lock.lock();
        try {
            orderCode = generator.getOrderCode();
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            lock.unlock();
        }

        logger.info((Thread.currentThread().getName() + ": 成功获取锁 =====> " + orderCode));
    }

    @Override
    public void run() {
        try {
            // 等待其他线程初始化
            cdl.await();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

        createOrderCode();
    }

    public static void main(String[] args) {
        OrderServiceImpl service = new OrderServiceImpl();
        for (int i = 0; i < NUM; i++) {
            new Thread(service).start();
            // 每初始化一个线程， 计数器减一
            cdl.countDown();
        }
    }
}

```
在这个类中，我们实例化了ZookeeperDistributeLock，然后我们对获取订单编号的方法进行加锁操作，在finally语句块中执行释放锁操作。

下面来看ZookeeperDistributeLock，代码如下：

```java
package pers.mingshan.ZookeeperLock;

import java.util.concurrent.CountDownLatch;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.locks.Condition;
import java.util.concurrent.locks.Lock;

import org.I0Itec.zkclient.IZkDataListener;
import org.I0Itec.zkclient.ZkClient;
import org.I0Itec.zkclient.exception.ZkNodeExistsException;
import org.apache.log4j.Logger;

/**
 * 利用Zookeeper节点名称的唯一性进行加锁和释放锁操作。
 * 利用znode名称唯一性进行加锁，所有客户端去竞争加锁，但只有一个会加锁
 * 成功，其他客户端需要等待加锁成功的客户端去释放锁，释放锁操作则是删除该节点，
 * 同时通知所有watch这个节点的客户端，其他的客户端再竞争加锁。
 * 由于释放锁会通知所有watch该节点的客户端，所以会出现羊群效应，
 * 造成资源浪费。
 * @author mingshan
 *
 */
public class ZookeeperDistributeLock implements Lock {
    private static Logger logger = Logger.getLogger(ZookeeperDistributeLock.class);
    // Zookeeper IP和端口
    private static final String ZK_IP_PORT = "localhost:2181";
    // Node 的名称
    private static final String LOCK_NODE = "/lockS";
    // 创建 Zookeeper 的客户端
    private ZkClient zkClient = new ZkClient(ZK_IP_PORT);

    // 减数器
    private static CountDownLatch cdl = null;

    /**
     * 阻塞式加锁
     */
    @Override
    public void lock() {
        // 先尝试加锁，加锁成功后就直接返回
        if (tryLock()) {
            return;
        }

        // 如果不成功， 需要等待其他线程 释放锁
        waitForLock();
        // 递归调用加锁
        lock();
    }

    /**
     * 等待其他线程释放锁
     */
    private void waitForLock() {
        // 给节点加 监听器
        IZkDataListener listener = new IZkDataListener() {

            @Override
            public void handleDataDeleted(String dataPath) throws Exception {
                logger.info("----node delete event------");
                if (cdl != null) {
                    cdl.countDown();
                }
            }

            @Override
            public void handleDataChange(String dataPath, Object data) throws Exception {
                // TODO Auto-generated method stub

            }
        };

        // 执行订阅node节点的数据变化
        zkClient.subscribeDataChanges(LOCK_NODE, listener);

        if (zkClient.exists(LOCK_NODE)) {
            try{
                cdl = new CountDownLatch(1);
                cdl.await();
            }catch (Exception e) {
                e.printStackTrace();
            }
        }

        // 取消订阅node节点的数据变化
        zkClient.unsubscribeDataChanges(LOCK_NODE, listener);
    }

    /**
     * 实现非阻塞式加锁
     * @return
     */
    @Override
    public boolean tryLock() {
        try {
            zkClient.createPersistent(LOCK_NODE);
            return true;
        } catch (ZkNodeExistsException e) {
            logger.error("加锁失败 -- reason -" + e.getMessage());
            return false;
        }
    }

    /**
     * 解锁
     */
    @Override
    public void unlock() {
        zkClient.delete(LOCK_NODE);
    }

    @Override
    public boolean tryLock(long time, TimeUnit unit) throws InterruptedException {
        // TODO Auto-generated method stub
        return false;
    }

    @Override
    public void lockInterruptibly() throws InterruptedException {
        // TODO Auto-generated method stub

    }

    @Override
    public Condition newCondition() {
        // TODO Auto-generated method stub
        return null;
    }

}

```
 利用Zookeeper节点名称的唯一性进行加锁和释放锁操作。利用znode名称唯一性进行加锁，所有客户端去竞争加锁，但只有一个会加锁成功， 其他客户端需要等待加锁成功的客户端去释放锁，释放锁操作则是删除该节点，同时通知所有watch这个节点的客户端，其他的客户端再竞争加锁。由于释放锁会通知所有watch该节点的客户端，所以会出现羊群效应，造成资源浪费。

## 利用EPHEMERAL_SEQUENTIA解决“羊群效应”
实现逻辑：

首先创建一个持久节点

在trylock方法中先判断当前临时顺序节点是否存在，如果不存在，那么就创建一个临时顺序节点，临时顺序节点为持久节点的子节点

然后获取所有的临时顺序节点并进行排序，判断当前节点是否为最小节点
- 如果当前结点为最小节点，说明当前可以加锁
- 如果当前临时节点并非最小，代表当前客户端没有获取锁，需要继续等待,此时获取比当前节点序号小的节点（比当前节点小的最大节点, 将此值赋给beforePath,例如： 当前节点是 /lock/000000003, 那么beforePath为 /lock/000000002，只有当beforePath获得锁并且释放锁后，当前客户端才能去获取锁,这样可以 避免羊群效应

在lock方法中，首先会调用trylock进行尝试加锁，如果加锁失败，那么就要调用waitForLock方法，在该方法中，对当前临时顺序节点的前一个节点进行监听，此时只需给前面的节点的添加wathcher即可。

实现代码如下：

```java
package pers.mingshan.ZookeeperLock;

import java.util.Collections;
import java.util.List;
import java.util.concurrent.CountDownLatch;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.locks.Condition;
import java.util.concurrent.locks.Lock;

import org.I0Itec.zkclient.IZkDataListener;
import org.I0Itec.zkclient.ZkClient;
import org.I0Itec.zkclient.serialize.SerializableSerializer;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

/**
 *
 * @author mingshan
 *
 */
public class ZookeeperImproveDistributeLock implements Lock{
    private static final Logger logger = LoggerFactory.getLogger(ZookeeperImproveDistributeLock.class);
    // Zookeeper IP和端口
    private static final String ZK_IP_PORT = "localhost:2181";
    // Node 的名称
    private static final String LOCK_ROOT_NODE = "/lock";
    // 创建 Zookee的客户端
    private ZkClient zkClient = new ZkClient(ZK_IP_PORT, 1000, 1000, new SerializableSelizer());
    // 当前创建的节点
    private String selfPath;
    // 当前节点的前一个节点
    private String beforePath;
    // 节点默认值
    private String data = "data";
    // 减数器
    private static CountDownLatch cdl = null;

    public ZookeeperImproveDistributeLock() {
        // 先创建一个主节点，以便其他线程在此节点之下创建临时顺序节点
        if (!this.zkClient.exists(LOCK_ROOT_NODE)) {
            this.zkClient.createPersistent(LOCK_ROOT_NODE);
        }
    }

    @Override
    public void lock() {
        // 先尝试加锁，加锁成功后就直接返回
        if (!tryLock()) {
            waitForLock();
            lock();
        } else {
            logger.info(Thread.currentThread().getName() + "---获取锁");
        }
    }

    private void waitForLock() {
        // 给节点加 监听器
        IZkDataListener listener = new IZkDataListener() {

            @Override
            public void handleDataDeleted(String dataPath) throws Exception {
                logger.info("----before node delete event------");
                if (cdl != null) {
                    cdl.countDown();
                }
            }

            @Override
            public void handleDataChange(String dataPath, Object data) throws Exception {
            }
        };

        // 此时只需给前面的节点的添加wathcher即可
        zkClient.subscribeDataChanges(this.beforePath, listener);

        if (zkClient.exists(this.beforePath)) {
            try{
                cdl = new CountDownLatch(1);
                cdl.await();
            }catch (Exception e) {
                e.printStackTrace();
            }
        }

        // 取消订阅前面的节点的变化
        zkClient.unsubscribeDataChanges(this.beforePath, listener);
    }

    @Override
    public boolean tryLock() {
        // 判断当前节点是否存在
        if (this.selfPath == null || this.selfPath.length() == 0) {
            // 在当前节点下创建临时顺序节点，例如0000000034,
            // 生成的节点应为 /lock/0000000034
            this.selfPath = this.zkClient.createEphemeralSequential(LOCK_ROOT_NODE + "/", data);
            logger.info("当前节点为 ————> " + this.selfPath);
        }

        // 获取所有的临时顺序节点，并进行排序
        List<String> allESNodes = zkClient.getChildren(LOCK_ROOT_NODE);
        Collections.sort(allESNodes);
        logger.info("0  ————> "+ allESNodes.get(0));
        // 判断当前节点是否为最小节点
        if (this.selfPath.equals(LOCK_ROOT_NODE + "/" + allESNodes.get(0))) {
            // 如果当前结点为最小节点，说明当前可以加锁
            return true;
        } else {
            // 如果当前临时节点并非最小，代表当前客户端没有获取锁，需要继续等待,
            // 此时获取比当前节点序号小的节点（比当前节点小的最大节点, 将此值赋给beforePath
            // 例如： 当前节点是 /lock/000000003, 那么beforePath为 /lock/000000002，
            // 只有当beforePath获得锁并且释放锁后，当前客户端才能去获取锁
            // 这样可以 避免羊群效应
            int wz = Collections.binarySearch(allESNodes, this.selfPath.substring(6));
            this.beforePath = LOCK_ROOT_NODE + "/" + allESNodes.get(wz - 1);
        }

        return false;
    }

    @Override
    public void unlock() {
        // 删除当前节点，释放锁
        zkClient.delete(this.selfPath);
    }

    @Override
    public boolean tryLock(long time, TimeUnit unit) throws InterruptedException {
        // TODO Auto-generated method stub
        return false;
    }

    @Override
    public void lockInterruptibly() throws InterruptedException {
        // TODO Auto-generated method stub

    }

    @Override
    public Condition newCondition() {
        // TODO Auto-generated method stub
        return null;
    }

}

```


上面的代码还有其他实现方式，代码如下（网上的）：

```java
package pers.mingshan.ZookeeperLock;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.apache.zookeeper.*;
import org.apache.zookeeper.data.Stat;
import java.util.List;
import java.io.IOException;
import java.util.Collections;
import java.util.concurrent.CountDownLatch;

public class ZookeeperOptimizedDistributedLock implements Watcher{
    private int threadId;
    private ZooKeeper zk = null;
    private String selfPath;
    private String waitPath;
    private String LOG_PREFIX_OF_THREAD;
    private static final int SESSION_TIMEOUT = 10000;
    private static final String GROUP_PATH = "/disLocks";
    private static final String SUB_PATH = "/disLocks/sub";
    private static final String CONNECTION_STRING = "localhost:2181";

    private static final int THREAD_NUM = 10;
    //确保连接zk成功
    private CountDownLatch connectedSemaphore = new CountDownLatch(1);
    //确保所有线程运行结束
    private static final CountDownLatch threadSemaphore = new CountDownLatch(THREAD_NUM);
    private static final Logger LOG = LoggerFactory.getLogger(ZookeeperOptimizedDistributedLock.class);

    public ZookeeperOptimizedDistributedLock(int id) {
        this.threadId = id;
        LOG_PREFIX_OF_THREAD = "【第"+threadId+"个线程】";
    }

    public static void main(String[] args) {
        for(int i = 0; i < THREAD_NUM; i++) {
            final int threadId = i + 1;
            new Thread() {
                @Override
                public void run() {
                    try {
                        ZookeeperOptimizedDistributedLock dc = new ZookeeperOptimizedDistributedLock(threadId);
                        dc.createConnection(CONNECTION_STRING, SESSION_TIMEOUT);
                        //GROUP_PATH不存在的话，由一个线程创建即可；
                        synchronized (threadSemaphore){
                            dc.createPath(GROUP_PATH, "该节点由线程" + threadId + "创建", true);
                        }
                        dc.getLock();
                    } catch (Exception e) {
                        LOG.error("【第"+threadId+"个线程】 抛出的异常：");
                        e.printStackTrace();
                    }
                }
            }.start();
        }

        try {
            threadSemaphore.await();
            LOG.info("所有线程运行结束!");
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }

    /**
     * 获取锁
     * @return
     */
    private void getLock() throws KeeperException, InterruptedException {
        selfPath = zk.create(SUB_PATH, null, ZooDefs.Ids.OPEN_ACL_UNSAFE, CreateMode.EPHEMERAL_SEQUENTIAL);
        LOG.info(LOG_PREFIX_OF_THREAD+"创建锁路径:"+selfPath);
        if(checkMinPath()){
            getLockSuccess();
        }
    }

    /**
     * 创建节点
     * @param path 节点path
     * @param data 初始数据内容
     * @return
     */
    public boolean createPath( String path, String data, boolean needWatch)
                throws KeeperException, InterruptedException {
        if(zk.exists(path, needWatch)==null){
            LOG.info( LOG_PREFIX_OF_THREAD + "节点创建成功, Path: "
                    + this.zk.create( path,
                    data.getBytes(),
                    ZooDefs.Ids.OPEN_ACL_UNSAFE,
                    CreateMode.PERSISTENT )
                    + ", content: " + data );
        }

        return true;
    }

    /**
     * 创建ZK连接
     * @param connectString  ZK服务器地址列表
     * @param sessionTimeout Session超时时间
     */
    public void createConnection( String connectString, int sessionTimeout )
                throws IOException, InterruptedException {
            zk = new ZooKeeper( connectString, sessionTimeout, this);
            connectedSemaphore.await();
    }

    /**
     * 获取锁成功
    */
    public void getLockSuccess() throws KeeperException, InterruptedException {
        if (zk.exists(this.selfPath,false) == null) {
            LOG.error(LOG_PREFIX_OF_THREAD+"本节点已不在了...");
            return;
        }

        LOG.info(LOG_PREFIX_OF_THREAD + "获取锁成功，赶紧干活！");
        Thread.sleep(2000);
        LOG.info(LOG_PREFIX_OF_THREAD + "删除本节点："+selfPath);
        zk.delete(this.selfPath, -1);
        releaseConnection();
        threadSemaphore.countDown();
    }

    /**
     * 关闭ZK连接
     */
    public void releaseConnection() {
        if ( this.zk !=null ) {
            try {
                this.zk.close();
            } catch ( InterruptedException e ) {}
        }

        LOG.info(LOG_PREFIX_OF_THREAD + "释放连接");
    }

    /**
     * 检查自己是不是最小的节点
     * @return
     */
    public boolean checkMinPath() throws KeeperException, InterruptedException {
         List<String> subNodes = zk.getChildren(GROUP_PATH, false);
         Collections.sort(subNodes);
         int index = subNodes.indexOf( selfPath.substring(GROUP_PATH.length() + 1));
         switch (index){
             case -1:{
                 LOG.error(LOG_PREFIX_OF_THREAD+"本节点已不在了..."+selfPath);
                 return false;
             }
             case 0:{
                 LOG.info(LOG_PREFIX_OF_THREAD+"子节点中，我果然是老大"+selfPath);
                 return true;
             }
             default:{
                 this.waitPath = GROUP_PATH +"/"+ subNodes.get(index - 1);
                 LOG.info(LOG_PREFIX_OF_THREAD+"获取子节点中，排在我前面的"+waitPath);
                 try{
                     zk.getData(waitPath, true, new Stat());
                     return false;
                 }catch(KeeperException e){
                     if(zk.exists(waitPath,false) == null){
                         LOG.info(LOG_PREFIX_OF_THREAD+"子节点中，排在我前面的"+waitPath+"已失踪，幸福来得太突然?");
                         return checkMinPath();
                     }else{
                         throw e;
                     }
                 }
             }
         }
    }

    @Override
    public void process(WatchedEvent event) {
        if(event == null){
            return;
        }
        Event.KeeperState keeperState = event.getState();
        Event.EventType eventType = event.getType();
        if ( Event.KeeperState.SyncConnected == keeperState) {
            if ( Event.EventType.None == eventType ) {
                LOG.info( LOG_PREFIX_OF_THREAD + "成功连接上ZK服务器" );
                connectedSemaphore.countDown();
            }else if (event.getType() == Event.EventType.NodeDeleted && event.getPath().equals(waitPath)) {
                LOG.info(LOG_PREFIX_OF_THREAD + "收到情报，排我前面的家伙已挂，我是不是可以出山了？");
                try {
                    if(checkMinPath()){
                        getLockSuccess();
                    }
                } catch (KeeperException e) {
                    e.printStackTrace();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }else if ( Event.KeeperState.Disconnected == keeperState ) {
            LOG.info( LOG_PREFIX_OF_THREAD + "与ZK服务器断开连接" );
        } else if ( Event.KeeperState.AuthFailed == keeperState ) {
            LOG.info( LOG_PREFIX_OF_THREAD + "权限检查失败" );
        } else if ( Event.KeeperState.Expired == keeperState ) {
            LOG.info( LOG_PREFIX_OF_THREAD + "会话失效" );
        }
    }
}

```


## 参考
http://blog.csdn.net/desilting/article/details/41280869


[<font size=3 color="#409EFF">向本文提出修改或勘误建议</font>](https://github.com/mstao/mstao.github.io/blob/hexo/source/_posts/distributed-lock-with-zookeeper.md)