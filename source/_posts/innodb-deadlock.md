---
title: 查看和分析死锁日志
tags: [Database, InnoDB]
author: Mingshan
categories: [Database, InnoDB]
date: 2019-10-12
---

死锁在系统中可能出现的频率比较高，特别是在生产环境中，对于死锁发生原因的定位比较困难，读懂死锁日志是非常有必要的。下面我们来模拟死锁的产生，然后分析死锁日志。

<!-- more -->

## 死锁概念

对于死锁，MySQL官方文档是这样描述的：

>A deadlock is a situation where different transactions are unable to proceed because each holds a lock that the other needs. Because both transactions are waiting for a resource to become available, neither ever release the locks it holds.

简单概括，死锁是两个或两个以上的事务在执行过程中，因争夺资源而造成的一种互相等待的现象。

**查看MySQL错误日志位置：**

```
show variables like 'log_error';
```

**在Windows下**，会显示相对路径：

```
mysql> show variables like 'log_error';
+---------------+--------------------+
| Variable_name | Value              |
+---------------+--------------------+
| log_error     | .\HANJUNTAO-PC.err |
+---------------+--------------------+
```

我们发现是相对路径，这个是相对哪个路径呢？我们找到MySQL的服务，查看属性，如下图所示：

![image](https://github.com/mstao/db-readings/blob/master/img/mysql_errorlog_win.png?raw=true)

找到**可执行文件的路径：**

```
"C:\Program Files\MySQL\MySQL Server 8.0\bin\mysqld.exe" --defaults-file="C:\ProgramData\MySQL\MySQL Server 8.0\my.ini" MySQL80
```

其中有`--defaults-file`参数，错误日志就在路径`C:\ProgramData\MySQL\MySQL Server 8.0\Data`下，仔细查看下就可以找到。

**在Linux下**，直接查就好了，如下所示：

```
mysql> show variables like 'log_error';
+---------------+---------------------------------------------+
| Variable_name | Value                                       |
+---------------+---------------------------------------------+
| log_error     | /usr/software/mysql/data/mysql/rds.iwms.err |
+---------------+---------------------------------------------+
```

**开启死锁日志记录，将死锁日志记录到错误日志里面：**

```
set global innodb_print_all_deadlocks = 1;
```

## 模拟死锁

首先创建一个表，初始化一条数据：

```SQL
CREATE TABLE t (i INT) ENGINE = InnoDB;
INSERT INTO t (i) VALUES(1);
```

然后我们开启两个session，步骤如下：

\ | TX1 | TX2
---|---|---
1 | BEGIN; | 
2 | SELECT * FROM t WHERE i = 1 FOR SHARE; | BEGIN;
3 | |  DELETE FROM t WHERE i = 1;
4 | DELETE FROM t WHERE i = 1; |


当TX2走到3的时候，此时删除语句会被阻塞，然后我们在TX1中也执行删除语句，此时死锁发生，显示在TX2面板上，如下所示：

```
mysql> DELETE FROM t WHERE i = 1;
ERROR 1213 (40001): Deadlock found when trying to get lock; try restarting transaction
```

上面死锁产生的原因是：TX1需要一个X锁来删除行。但是，这个锁请求不能被授予，因为TX2已经有了一个X锁的请求，并且正在等待TX1释放它的S锁。TX1持有的S锁也不能因为TX2之前请求X锁而升级为X锁。这里InnoDB选择回滚其中的一个事务，从操作来看，回滚了事务TX1，TX2报出了死锁。

简单来说，就是TX1的删除请求等待TX2的删除请求完成，而TX2的删除请求又在等待TX1的S锁释放，导致TX1的删除请求在等待TX1的S锁释放，这根本就释放不了的，在同一个事务自己等自己，造成死锁。

## 分析死锁

在TX1执行完第二步的时候，我们来查看此时的锁情况，如下所示：

```
mysql> select * from performance_schema.data_locks\G;
*************************** 1. row ***************************
               ENGINE: INNODB
       ENGINE_LOCK_ID: 283133240155680:1098
ENGINE_TRANSACTION_ID: 283133240155680
            THREAD_ID: 51
             EVENT_ID: 7
        OBJECT_SCHEMA: test
          OBJECT_NAME: t
       PARTITION_NAME: NULL
    SUBPARTITION_NAME: NULL
           INDEX_NAME: NULL
OBJECT_INSTANCE_BEGIN: 1658247672504
            LOCK_TYPE: TABLE
            LOCK_MODE: IS
          LOCK_STATUS: GRANTED
            LOCK_DATA: NULL
*************************** 2. row ***************************
               ENGINE: INNODB
       ENGINE_LOCK_ID: 283133240155680:38:4:1
ENGINE_TRANSACTION_ID: 283133240155680
            THREAD_ID: 51
             EVENT_ID: 7
        OBJECT_SCHEMA: test
          OBJECT_NAME: t
       PARTITION_NAME: NULL
    SUBPARTITION_NAME: NULL
           INDEX_NAME: GEN_CLUST_INDEX
OBJECT_INSTANCE_BEGIN: 1658247669720
            LOCK_TYPE: RECORD
            LOCK_MODE: S
          LOCK_STATUS: GRANTED
            LOCK_DATA: supremum pseudo-record
*************************** 3. row ***************************
               ENGINE: INNODB
       ENGINE_LOCK_ID: 283133240155680:38:4:2
ENGINE_TRANSACTION_ID: 283133240155680
            THREAD_ID: 51
             EVENT_ID: 7
        OBJECT_SCHEMA: test
          OBJECT_NAME: t
       PARTITION_NAME: NULL
    SUBPARTITION_NAME: NULL
           INDEX_NAME: GEN_CLUST_INDEX
OBJECT_INSTANCE_BEGIN: 1658247669720
            LOCK_TYPE: RECORD
            LOCK_MODE: S
          LOCK_STATUS: GRANTED
            LOCK_DATA: 0x000000000200
3 rows in set (0.00 sec)
```

我们可以清晰地看到， INDEX_NAME 为`GEN_CLUST_INDEX`，这是因为这个表既没有主键，也没有索引，如果表中没有PRIMARY KEY，而且也没有合适的UNIQUE index，那么InnoDB内部将生产一个名字叫GEN_CLUST_INDEX的隐藏clustered index，其值为行ID。对记录加了S锁（共享锁），同时对表加了IS锁（意向共享锁）。

在TX1执行完第四步的时候，我们来查看此时的锁情况，如下所示：

```
mysql> select * from performance_schema.data_locks\G;
*************************** 1. row ***************************
               ENGINE: INNODB
       ENGINE_LOCK_ID: 47885:1098
ENGINE_TRANSACTION_ID: 47885
            THREAD_ID: 51
             EVENT_ID: 7
        OBJECT_SCHEMA: test
          OBJECT_NAME: t
       PARTITION_NAME: NULL
    SUBPARTITION_NAME: NULL
           INDEX_NAME: NULL
OBJECT_INSTANCE_BEGIN: 1658247672504
            LOCK_TYPE: TABLE
            LOCK_MODE: IS
          LOCK_STATUS: GRANTED
            LOCK_DATA: NULL
*************************** 2. row ***************************
               ENGINE: INNODB
       ENGINE_LOCK_ID: 47885:38:4:1
ENGINE_TRANSACTION_ID: 47885
            THREAD_ID: 51
             EVENT_ID: 7
        OBJECT_SCHEMA: test
          OBJECT_NAME: t
       PARTITION_NAME: NULL
    SUBPARTITION_NAME: NULL
           INDEX_NAME: GEN_CLUST_INDEX
OBJECT_INSTANCE_BEGIN: 1658247669720
            LOCK_TYPE: RECORD
            LOCK_MODE: S
          LOCK_STATUS: GRANTED
            LOCK_DATA: supremum pseudo-record
*************************** 3. row ***************************
               ENGINE: INNODB
       ENGINE_LOCK_ID: 47885:38:4:2
ENGINE_TRANSACTION_ID: 47885
            THREAD_ID: 51
             EVENT_ID: 7
        OBJECT_SCHEMA: test
          OBJECT_NAME: t
       PARTITION_NAME: NULL
    SUBPARTITION_NAME: NULL
           INDEX_NAME: GEN_CLUST_INDEX
OBJECT_INSTANCE_BEGIN: 1658247669720
            LOCK_TYPE: RECORD
            LOCK_MODE: S
          LOCK_STATUS: GRANTED
            LOCK_DATA: 0x000000000200
*************************** 4. row ***************************
               ENGINE: INNODB
       ENGINE_LOCK_ID: 47885:1098
ENGINE_TRANSACTION_ID: 47885
            THREAD_ID: 51
             EVENT_ID: 8
        OBJECT_SCHEMA: test
          OBJECT_NAME: t
       PARTITION_NAME: NULL
    SUBPARTITION_NAME: NULL
           INDEX_NAME: NULL
OBJECT_INSTANCE_BEGIN: 1658247672592
            LOCK_TYPE: TABLE
            LOCK_MODE: IX
          LOCK_STATUS: GRANTED
            LOCK_DATA: NULL
*************************** 5. row ***************************
               ENGINE: INNODB
       ENGINE_LOCK_ID: 47885:38:4:1
ENGINE_TRANSACTION_ID: 47885
            THREAD_ID: 51
             EVENT_ID: 8
        OBJECT_SCHEMA: test
          OBJECT_NAME: t
       PARTITION_NAME: NULL
    SUBPARTITION_NAME: NULL
           INDEX_NAME: GEN_CLUST_INDEX
OBJECT_INSTANCE_BEGIN: 1658247670064
            LOCK_TYPE: RECORD
            LOCK_MODE: X
          LOCK_STATUS: GRANTED
            LOCK_DATA: supremum pseudo-record
*************************** 6. row ***************************
               ENGINE: INNODB
       ENGINE_LOCK_ID: 47885:38:4:2
ENGINE_TRANSACTION_ID: 47885
            THREAD_ID: 51
             EVENT_ID: 8
        OBJECT_SCHEMA: test
          OBJECT_NAME: t
       PARTITION_NAME: NULL
    SUBPARTITION_NAME: NULL
           INDEX_NAME: GEN_CLUST_INDEX
OBJECT_INSTANCE_BEGIN: 1658247670064
            LOCK_TYPE: RECORD
            LOCK_MODE: X
          LOCK_STATUS: GRANTED
            LOCK_DATA: 0x000000000200
6 rows in set (0.00 sec)
```

看上面的输出，发现只有ENGINE_TRANSACTION_ID为47885的，TX1的事务不在data_locks里面了，我们有没有COMMIT，所以我们推断，MySQL将TX1的事务ROLLBACK了。我们用`show engine innodb status;`来查看此时的死锁日志，如下：

```
=====================================
2019-10-11 16:43:14 0x3c7c INNODB MONITOR OUTPUT
=====================================
Per second averages calculated from the last 31 seconds
-----------------
BACKGROUND THREAD
-----------------
srv_master_thread loops: 9 srv_active, 0 srv_shutdown, 6899 srv_idle
srv_master_thread log flush and writes: 0
----------
SEMAPHORES
----------
OS WAIT ARRAY INFO: reservation count 19
OS WAIT ARRAY INFO: signal count 19
RW-shared spins 0, rounds 0, OS waits 0
RW-excl spins 0, rounds 0, OS waits 0
RW-sx spins 0, rounds 0, OS waits 0
Spin rounds per wait: 0.00 RW-shared, 0.00 RW-excl, 0.00 RW-sx
------------------------
LATEST DETECTED DEADLOCK
------------------------
2019-10-11 16:42:19 0x1cfc
*** (1) TRANSACTION:
TRANSACTION 47884, ACTIVE 16 sec starting index read
mysql tables in use 1, locked 1
LOCK WAIT 2 lock struct(s), heap size 1136, 1 row lock(s)
MySQL thread id 15, OS thread handle 2372, query id 93 localhost ::1 root updating
DELETE FROM t WHERE i = 1
*** (1) WAITING FOR THIS LOCK TO BE GRANTED:
RECORD LOCKS space id 38 page no 4 n bits 72 index GEN_CLUST_INDEX of table `test`.`t` trx id 47884 lock_mode X waiting
Record lock, heap no 2 PHYSICAL RECORD: n_fields 4; compact format; info bits 0
 0: len 6; hex 000000000200; asc       ;;
 1: len 6; hex 000000009910; asc       ;;
 2: len 7; hex 81000001050110; asc        ;;
 3: len 4; hex 80000001; asc     ;;

*** (2) TRANSACTION:
TRANSACTION 47885, ACTIVE 119 sec starting index read, thread declared inside InnoDB 5000
mysql tables in use 1, locked 1
4 lock struct(s), heap size 1136, 3 row lock(s)
MySQL thread id 13, OS thread handle 7420, query id 94 localhost ::1 root updating
DELETE FROM t WHERE i = 1
*** (2) HOLDS THE LOCK(S):
RECORD LOCKS space id 38 page no 4 n bits 72 index GEN_CLUST_INDEX of table `test`.`t` trx id 47885 lock mode S
Record lock, heap no 1 PHYSICAL RECORD: n_fields 1; compact format; info bits 0
 0: len 8; hex 73757072656d756d; asc supremum;;

Record lock, heap no 2 PHYSICAL RECORD: n_fields 4; compact format; info bits 0
 0: len 6; hex 000000000200; asc       ;;
 1: len 6; hex 000000009910; asc       ;;
 2: len 7; hex 81000001050110; asc        ;;
 3: len 4; hex 80000001; asc     ;;

*** (2) WAITING FOR THIS LOCK TO BE GRANTED:
RECORD LOCKS space id 38 page no 4 n bits 72 index GEN_CLUST_INDEX of table `test`.`t` trx id 47885 lock_mode X waiting
Record lock, heap no 2 PHYSICAL RECORD: n_fields 4; compact format; info bits 0
 0: len 6; hex 000000000200; asc       ;;
 1: len 6; hex 000000009910; asc       ;;
 2: len 7; hex 81000001050110; asc        ;;
 3: len 4; hex 80000001; asc     ;;

*** WE ROLL BACK TRANSACTION (1)
------------
TRANSACTIONS
------------
Trx id counter 47890
Purge done for trx's n:o < 47890 undo n:o < 0 state: running but idle
History list length 6
LIST OF TRANSACTIONS FOR EACH SESSION:
---TRANSACTION 283133240156560, not started
0 lock struct(s), heap size 1136, 0 row lock(s)
---TRANSACTION 283133240154800, not started
0 lock struct(s), heap size 1136, 0 row lock(s)
---TRANSACTION 283133240153920, not started
0 lock struct(s), heap size 1136, 0 row lock(s)
---TRANSACTION 47885, ACTIVE 174 sec
4 lock struct(s), heap size 1136, 4 row lock(s), undo log entries 1
MySQL thread id 13, OS thread handle 7420, query id 94 localhost ::1 root
--------
FILE I/O
--------
I/O thread 0 state: wait Windows aio (insert buffer thread)
I/O thread 1 state: wait Windows aio (log thread)
I/O thread 2 state: wait Windows aio (read thread)
I/O thread 3 state: wait Windows aio (read thread)
I/O thread 4 state: wait Windows aio (read thread)
I/O thread 5 state: wait Windows aio (read thread)
I/O thread 6 state: wait Windows aio (write thread)
I/O thread 7 state: wait Windows aio (write thread)
I/O thread 8 state: wait Windows aio (write thread)
I/O thread 9 state: wait Windows aio (write thread)
Pending normal aio reads: [0, 0, 0, 0] , aio writes: [0, 0, 0, 0] ,
 ibuf aio reads:, log i/o's:, sync i/o's:
Pending flushes (fsync) log: 0; buffer pool: 0
1051 OS file reads, 222 OS file writes, 30 OS fsyncs
0.00 reads/s, 0 avg bytes/read, 0.00 writes/s, 0.00 fsyncs/s
-------------------------------------
INSERT BUFFER AND ADAPTIVE HASH INDEX
-------------------------------------
Ibuf: size 1, free list len 0, seg size 2, 0 merges
merged operations:
 insert 0, delete mark 0, delete 0
discarded operations:
 insert 0, delete mark 0, delete 0
Hash table size 2267, node heap has 2 buffer(s)
Hash table size 2267, node heap has 1 buffer(s)
Hash table size 2267, node heap has 7 buffer(s)
Hash table size 2267, node heap has 0 buffer(s)
Hash table size 2267, node heap has 2 buffer(s)
Hash table size 2267, node heap has 0 buffer(s)
Hash table size 2267, node heap has 0 buffer(s)
Hash table size 2267, node heap has 0 buffer(s)
0.00 hash searches/s, 0.00 non-hash searches/s
---
LOG
---
Log sequence number          23994432
Log buffer assigned up to    23994432
Log buffer completed up to   23994432
Log written up to            23994432
Log flushed up to            23994432
Added dirty pages up to      23994432
Pages flushed up to          23994432
Last checkpoint at           23994432
34 log i/o's done, 0.00 log i/o's/second
----------------------
BUFFER POOL AND MEMORY
----------------------
Total large memory allocated 8585216
Dictionary memory allocated 406902
Buffer pool size   512
Free buffers       244
Database pages     256
Old database pages 0
Modified db pages  0
Pending reads      0
Pending writes: LRU 0, flush list 0, single page 0
Pages made young 0, not young 0
0.00 youngs/s, 0.00 non-youngs/s
Pages read 1026, created 133, written 169
0.00 reads/s, 0.00 creates/s, 0.00 writes/s
No buffer pool page gets since the last printout
Pages read ahead 0.00/s, evicted without access 0.00/s, Random read ahead 0.00/s
LRU len: 256, unzip_LRU len: 0
I/O sum[0]:cur[0], unzip sum[0]:cur[0]
--------------
ROW OPERATIONS
--------------
0 queries inside InnoDB, 0 queries in queue
0 read views open inside InnoDB
Process ID=4352, Main thread ID=000000000000170C , state=sleeping
Number of rows inserted 51, updated 313, deleted 2, read 16575
0.00 inserts/s, 0.00 updates/s, 0.00 deletes/s, 0.00 reads/s
----------------------------
END OF INNODB MONITOR OUTPUT
============================
```

从上面的日志我们可以看出是分很多不同的部分，对于死锁日志，目前我们只看BAC**KGROUND THREAD**和 **TRANSACTIONS**这两部分就可以了。

在分析日志前，我们要知道InnoDB中锁在日志中具体显示的数据类型，平时我们常接触到的是Record Locks（记录锁），Gap Locks（间隙锁），Next-Key Locks和Insert Intention Locks（插入意向锁）。这四种锁对应的死锁如下：

- 记录锁（LOCK_REC_NOT_GAP）: **lock_mode X locks rec but not gap**
- 间隙锁（LOCK_GAP）: **lock_mode X locks gap before rec**
- Next-key 锁（LOCK_ORNIDARY）: **lock_mode X**
- 插入意向锁（LOCK_INSERT_INTENTION）: **lock_mode X locks gap before rec insert intention**

**LATEST DETECTED DEADLOCK**表示InnoDB引擎上次发生的死锁，我们知道，死锁是至少需要有两个事务参与的，所以在其内部包含了发生死锁的具体两个事务，我们先看`(1) TRANSACTION`。

在`(1) TRANSACTION`下，首先我们可以看到这样一句话：`TRANSACTION 47884, ACTIVE 16 sec starting index read`，47884是当前事务的ID，`starting index read`当前当前事务正在进行索引读，注意删除和更新InnoDB内部也是要进行读操作的。接着可以看到事务1的状态：`LOCK WAIT`，并且有一条记录被锁上了。再接着我们看到当前事务被阻塞的SQL语句：`DELETE FROM t WHERE i = 1`，申请将`i = 1`的索引记录加上X锁。

事务1的状态看完了，接着在`(1) WAITING FOR THIS LOCK TO BE GRANTED`下，此处表示当前事务1等待获取行锁。在这个小标题下，我们可以看到下面的日志：

```
RECORD LOCKS space id 38 page no 4 n bits 72 index GEN_CLUST_INDEX of table `test`.`t` trx id 47884 lock_mode X waiting
Record lock, heap no 2 PHYSICAL RECORD: n_fields 4; compact format; info bits 0
 0: len 6; hex 000000000200; asc       ;;
 1: len 6; hex 000000009910; asc       ;;
 2: len 7; hex 81000001050110; asc        ;;
 3: len 4; hex 80000001; asc     ;;
```

我们可以看出，事务1走的是隐藏的聚集索引`GEN_CLUST_INDEX`，等待的锁类型是` lock_mode X `，说明是Next-Key Locks。


接着我们来看事务2，在`(2) TRANSACTION`下，和事务1一样的分析，其正在执行的SQL：`DELETE FROM t WHERE i = 1`。`(2) HOLDS THE LOCK(S)`表示事务2正在持有的锁，注意在该小标题下，事务2持有的锁是：`lock mode S`，这是怎么回事呢？我们知道事务1持有的是S锁，S锁与S锁之间是相容的，事务2也能获取S锁。

接着在`(2) WAITING FOR THIS LOCK TO BE GRANTED`下，我们可以看到事务2正在等待被授予Next-Key Locks。

事务1和事务2的信息看完了，接着是`WE ROLL BACK TRANSACTION (1)`，表名InnoDB回滚了事务1，至于为什么回滚事务1，我们用事务等待图来表示事务1和事务2的等待情况，事务等待图是一张有向图$G=(T, U)$，T为结点的集合，每个结点表示正在运行的事务；U为边的集合，每个边表示事务等待情况。如T1等待T2，那么在T1和T2之间画一条有向边，由T1指向T2，如下所示：

![image](https://github.com/mstao/db-readings/blob/master/img/deadlock-graph.png?raw=true)

可以清楚的看到，事务1和事务2在同一个记录上形成了环，也就是造成了死锁。MySQL的InnoDB引擎有死锁检测机制（等待图 wait-for-graph），如果检测到在同一个记录事务之间存在环，就会立即报出死锁，并且回滚相应事务，具体实现细节以后再说吧。

## References：

- Deadlocks in InnoDB https://dev.mysql.com/doc/refman/8.0/en/innodb-deadlocks.html
- An InnoDB Deadlock Example https://dev.mysql.com/doc/refman/8.0/en/innodb-deadlock-example.html
- InnoDB Monitors https://dev.mysql.com/doc/refman/8.0/en/innodb-monitors.html
- InnoDB Standard Monitor and Lock Monitor Output https://dev.mysql.com/doc/refman/8.0/en/innodb-standard-monitor.html
- 了解MySQL死锁日志 https://segmentfault.com/a/1190000018730103?utm_source=tag-newest