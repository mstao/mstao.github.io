---
title: 查看和分析死锁日志
tags: [Database, InnoDB]
author: Mingshan
categories: [Database, InnoDB]
date: 2019-10-12
---

死锁在系统中可能出现的频率比较高，特别是在生产环境中，对于死锁发生原因的定位比较困难，读懂死锁日志是非常有必要的。下面我们来模拟死锁的产生，然后分析死锁日志。

<!-- more -->

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

上面死锁产生的原因是：

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
