---
title: InnoDB Locking
tags: [Database, InnoDB]
author: Mingshan
categories: [Database, InnoDB]
date: 2019-9-26
---

在InnoDB中，锁的类型有如下几种：

- Shared and Exclusive Locks(共享S或独占X锁)
- Intention Locks(意向锁)
- Record Locks(记录锁)
- Gap Locks(间隙锁)
- Next-Key Locks
- Insert Intention Locks(插入意向锁)
- AUTO-INC Locks(自增锁)

<!-- more -->

## Shared and Exclusive Locks
InnoDB实现了标准的行级锁(row-level locking)，其中有两种类型的锁，共享锁(**shared (S) locks**)和独占锁(**exclusive (X) locks**)。

- 共享锁(S)允许持有锁的事务读取一行
- 独占锁(X)允许持有锁的事务更新或者删除一行

如果事务T1在第r行上持有一个共享锁，那么来自某个不同事务T2的请求对第r行上的锁的处理方法如下:
- T2可以立即获取S锁。因此，T1和T2在r上都有一个S锁
- T2对X锁的请求不能立即被批准

从上面对S锁和X锁的处理情形来看，对于一个事务T1，如果它持有第r行的X锁，那么其他事务不能获取第r行的任何锁，直至T1释放掉S锁。

排它锁和共享锁控制方式用如下表格显示(Y: 相容；N: 不相容)：

\  | X | S 
---|---|---
X  | N | N  
S  | N | Y  


注意普通的查询语句在InnoDB中属于快照读，不会加任何锁；如果查询加`lock in share mode`，那么会将查询出来的行加上**S锁**；如果查询加`for update`， 那么会将查询出来的行加上**X锁**。如下SQL所示：

```SQL
select * from table where ? lock in share mode;
select * from table where ? for update;
```

## Intention Locks

InnoDB支持多粒度锁定(multiple granularity locking)，允许行锁和表锁共存。为了在多个粒度级别上实现锁定，InnoDB使用了Intention Locks(意向锁)。Intention Locks是表级别(table-level)锁，表示事务稍后对表中的一行数据加哪种类型的锁(共享或独占)。有两种类型的Intention Locks：

- 在事务可以获取表中某一行上的共享锁之前，它必须首先获取表上的IS锁或更强的**IS(intention shared lock )**锁。
- 在事务可以获得表中某一行的独占锁之前，它必须首先获得表上的**IX(intention exclusive lock)**锁。

表级锁（**Table-level lock**）的类型兼容性总结如下（Compatible可共存，Conflict不可共存）：

| \ | X	 | IX | S |	IS
---|---|---|---|---
X  |	Conflict  |	Conflict   | 	Conflict   |	Conflict
IX |	Conflict  |	Compatible |	Conflict   |	Compatible
S  |	Conflict  |	Conflict   |	Compatible |	Compatible
IS |	Conflict  |	Compatible |	Compatible |	Compatible

如果请求事务与现有锁兼容，则授予该事务锁，但如果与现有锁冲突，则不授予该事务锁。事务等待冲突的现有锁被释放。如果锁请求与现有锁冲突，并且由于会导致死锁而无法被授予，则会发生错误。

意向锁不会阻塞除全表扫描请求之外的任何请求。意向锁的主要目的是显示某人锁定了一行，或者准备锁定表中的一行（**The main purpose of intention locks is to show that someone is locking a row, or going to lock a row in the table.**）。

上面的是MySQL官方文档对意向锁的描述，看过之后是不是对意向锁的作用不太理解？X 居然与IX、IS冲突，那么意向锁是加在那个地方呢？我们来实操一把，直接用例子来看看意向锁是干啥的，然后总结一下意向锁的作用（以下例子是基于**MySQL8.0**的，低于此版本情况有所不同）。

下面有一个表`t1`，有如下两条数据（字段i和name都没有索引）：

```
mysql> select * from t1;
+------+--------+
| i    | name   |
+------+--------+
|    1 | WALKER |
|    2 | Bob    |
+------+--------+
```

此时我们用两个session来模拟锁冲突的情况，一个加S锁，一个加X锁。如下所示：

\ | TX1 | TX2
---|---|---
1 | begin; | begin; 
2 | SELECT * FROM t1 WHERE i = 1 FOR UPDATE; |
3 | |  UPDATE t1 SET name = 'WALKER1' WHERE i = 1;

由上面的SQL语句执行情况来看，我们不难猜出TX2的`UPDATE t1 SET name = 'WALKER1';`更新语句必被阻塞住，下面来分析具体锁的情况。

首先我们先来查询当前两个**事务**的状态：

```SQL
SELECT * FROM information_schema.INNODB_TRX\G;
```

输出是：

```
mysql> SELECT * FROM information_schema.INNODB_TRX\G;
*************************** 1. row ***************************
                    trx_id: 44300
                 trx_state: LOCK WAIT
               trx_started: 2019-09-26 13:49:07
     trx_requested_lock_id: 44300:40:4:2
          trx_wait_started: 2019-09-26 13:51:34
                trx_weight: 2
       trx_mysql_thread_id: 22
                 trx_query: UPDATE t1 SET name = 'WALKER1' WHERE i = 1
       trx_operation_state: starting index read
         trx_tables_in_use: 1
         trx_tables_locked: 1
          trx_lock_structs: 2
     trx_lock_memory_bytes: 1136
           trx_rows_locked: 3
         trx_rows_modified: 0
   trx_concurrency_tickets: 0
       trx_isolation_level: REPEATABLE READ
         trx_unique_checks: 1
    trx_foreign_key_checks: 1
trx_last_foreign_key_error: NULL
 trx_adaptive_hash_latched: 0
 trx_adaptive_hash_timeout: 0
          trx_is_read_only: 0
trx_autocommit_non_locking: 0
*************************** 2. row ***************************
                    trx_id: 44299
                 trx_state: RUNNING
               trx_started: 2019-09-26 13:47:46
     trx_requested_lock_id: NULL
          trx_wait_started: NULL
                trx_weight: 2
       trx_mysql_thread_id: 21
                 trx_query: NULL
       trx_operation_state: NULL
         trx_tables_in_use: 0
         trx_tables_locked: 1
          trx_lock_structs: 2
     trx_lock_memory_bytes: 1136
           trx_rows_locked: 3
         trx_rows_modified: 0
   trx_concurrency_tickets: 0
       trx_isolation_level: REPEATABLE READ
         trx_unique_checks: 1
    trx_foreign_key_checks: 1
trx_last_foreign_key_error: NULL
 trx_adaptive_hash_latched: 0
 trx_adaptive_hash_timeout: 0
          trx_is_read_only: 0
trx_autocommit_non_locking: 0
```

由上面输出可以看出，`trx_id: 44300`的状态为`LOCK WAIT`，被阻塞住了，而` trx_id: 44299`的状态为`RUNNING`，正常运行，注意两者的事务都未提交。

接着我们来查询**锁**的情况，这个事务被上了什么锁，锁的状态是什么，我们都可以知晓，通过如下sql查询：

```SQL
select * from performance_schema.data_locks\G;
```

输出是：


```
mysql> select * from performance_schema.data_locks\G;
*************************** 1. row ***************************
               ENGINE: INNODB
       ENGINE_LOCK_ID: 44300:1100
ENGINE_TRANSACTION_ID: 44300
            THREAD_ID: 60
             EVENT_ID: 9
        OBJECT_SCHEMA: test
          OBJECT_NAME: t1
       PARTITION_NAME: NULL
    SUBPARTITION_NAME: NULL
           INDEX_NAME: NULL
OBJECT_INSTANCE_BEGIN: 2671434678088
            LOCK_TYPE: TABLE
            LOCK_MODE: IX
          LOCK_STATUS: GRANTED
            LOCK_DATA: NULL
*************************** 2. row ***************************
               ENGINE: INNODB
       ENGINE_LOCK_ID: 44300:40:4:2
ENGINE_TRANSACTION_ID: 44300
            THREAD_ID: 60
             EVENT_ID: 10
        OBJECT_SCHEMA: test
          OBJECT_NAME: t1
       PARTITION_NAME: NULL
    SUBPARTITION_NAME: NULL
           INDEX_NAME: GEN_CLUST_INDEX
OBJECT_INSTANCE_BEGIN: 2671434675648
            LOCK_TYPE: RECORD
            LOCK_MODE: X
          LOCK_STATUS: WAITING
            LOCK_DATA: 0x000000000300
*************************** 3. row ***************************
               ENGINE: INNODB
       ENGINE_LOCK_ID: 44299:1100
ENGINE_TRANSACTION_ID: 44299
            THREAD_ID: 59
             EVENT_ID: 7
        OBJECT_SCHEMA: test
          OBJECT_NAME: t1
       PARTITION_NAME: NULL
    SUBPARTITION_NAME: NULL
           INDEX_NAME: NULL
OBJECT_INSTANCE_BEGIN: 2671434673112
            LOCK_TYPE: TABLE
            LOCK_MODE: IX
          LOCK_STATUS: GRANTED
            LOCK_DATA: NULL
*************************** 4. row ***************************
               ENGINE: INNODB
       ENGINE_LOCK_ID: 44299:40:4:1
ENGINE_TRANSACTION_ID: 44299
            THREAD_ID: 59
             EVENT_ID: 7
        OBJECT_SCHEMA: test
          OBJECT_NAME: t1
       PARTITION_NAME: NULL
    SUBPARTITION_NAME: NULL
           INDEX_NAME: GEN_CLUST_INDEX
OBJECT_INSTANCE_BEGIN: 2671434670328
            LOCK_TYPE: RECORD
            LOCK_MODE: X
          LOCK_STATUS: GRANTED
            LOCK_DATA: supremum pseudo-record
*************************** 5. row ***************************
               ENGINE: INNODB
       ENGINE_LOCK_ID: 44299:40:4:2
ENGINE_TRANSACTION_ID: 44299
            THREAD_ID: 59
             EVENT_ID: 7
        OBJECT_SCHEMA: test
          OBJECT_NAME: t1
       PARTITION_NAME: NULL
    SUBPARTITION_NAME: NULL
           INDEX_NAME: GEN_CLUST_INDEX
OBJECT_INSTANCE_BEGIN: 2671434670328
            LOCK_TYPE: RECORD
            LOCK_MODE: X
          LOCK_STATUS: GRANTED
            LOCK_DATA: 0x000000000300
*************************** 6. row ***************************
               ENGINE: INNODB
       ENGINE_LOCK_ID: 44299:40:4:4
ENGINE_TRANSACTION_ID: 44299
            THREAD_ID: 59
             EVENT_ID: 7
        OBJECT_SCHEMA: test
          OBJECT_NAME: t1
       PARTITION_NAME: NULL
    SUBPARTITION_NAME: NULL
           INDEX_NAME: GEN_CLUST_INDEX
OBJECT_INSTANCE_BEGIN: 2671434670328
            LOCK_TYPE: RECORD
            LOCK_MODE: X
          LOCK_STATUS: GRANTED
            LOCK_DATA: 0x000000000301
6 rows in set (0.00 sec)
```


居然有六条记录，注意后四条都是事务44297的，我只给`i = 1`的记录加了X锁，为什么有三条[4,5,6]X锁（LOCK_TYPE为RECORD）？注意有三条X锁的记录`LOCK_DATA`字段数据是不同的，
原因是字段i并没有添加索引，所以MySQL就利用聚簇索引（Cluster Index）来加锁了，所以两条记录都被上了X锁，第三条记录的`LOCK_DATA`是 `supremum pseudo-record`，这个是啥玩意？这个和后面说到的`Next-Key Locks`有关，这个先暂放不表。

我们注意第三条记录的LOCK_MODE为IX，且LOCK_TYPE为TABLE，说明这是个表级锁，且是意向写锁，由上表可知，IX锁与IX锁是相容的，所以可以看到，第一条记录的IX锁的状态也是GRANTED的。

再看第二条纪录，LOCK_TYPE为RECORD，LOCK_MODE为X， LOCK_STATUS为WAITING，说明被阻塞了，等待其他事务释放锁。


最后我们想看**锁等待**的情况，可以用如下sql：

```sql
select * from performance_schema.data_lock_waits\G;
```

输出如下：

```
mysql> select * from performance_schema.data_lock_waits\G;
*************************** 1. row ***************************
                          ENGINE: INNODB
       REQUESTING_ENGINE_LOCK_ID: 44300:40:4:2
REQUESTING_ENGINE_TRANSACTION_ID: 44300
            REQUESTING_THREAD_ID: 60
             REQUESTING_EVENT_ID: 11
REQUESTING_OBJECT_INSTANCE_BEGIN: 2671434675992
         BLOCKING_ENGINE_LOCK_ID: 44299:40:4:2
  BLOCKING_ENGINE_TRANSACTION_ID: 44299
              BLOCKING_THREAD_ID: 59
               BLOCKING_EVENT_ID: 7
  BLOCKING_OBJECT_INSTANCE_BEGIN: 2671434670328
1 row in set (0.00 sec)
```

这个表记录的是一个锁被哪个锁阻塞了，其中`REQUESTING_ENGINE_LOCK_ID`代表当前被阻塞的锁ID，`REQUESTING_ENGINE_TRANSACTION_ID`代表当前被阻塞的事务ID；`BLOCKING_ENGINE_TRANSACTION_ID`代表当前持有锁的锁ID，`BLOCKING_ENGINE_TRANSACTION_ID`代表持有锁的事务ID。所以我们可以明显地看到，事务44300被事务44299阻塞了。


通过上面的实操，我们知道了：

- 意向锁是**表级锁**（Table-level），不会和行级的X，S锁发生冲突。只会和表级的X，S发生冲突
- 表级别的IX与表级别的 X、S均不相容，表级别的IS只与表级别的S相容。 而表级别的IX、IS是相互相容的，而IX、IS只是表明申请更低层次级别元素（比如 page、record）的X、S操作

假设此时有一个事务T需要申请表的X锁

- 如果没有意向锁的话，则需要遍历所有整个表判断是否有行锁的存在，以免发生冲突
- 如果有了意向锁，只需要判断该意向锁与即将添加的表级锁是否兼容即可。因为意向锁的存在代表了，有行级锁的存在或者即将有行级锁的存在。因而无需遍历整个表，即可获取结果

## Record Locks

Record Locks总是锁定索引记录，即使表没有定义索引。对于这种情况，InnoDB创建一个隐藏的聚簇索引（[Cluster Index](https://dev.mysql.com/doc/refman/8.0/en/innodb-index-types.html)），并使用这个索引来锁定记录。

## Gap Locks

## Next-Key Locks

## Insert Intention Locks

## AUTO-INC Locks

参考：

- InnoDB Locking https://dev.mysql.com/doc/refman/8.0/en/innodb-locking.html
- Deadlocks in InnoDB https://dev.mysql.com/doc/refman/5.6/en/innodb-deadlocks.html
- How to Minimize and Handle Deadlocks https://dev.mysql.com/doc/refman/5.6/en/innodb-deadlocks-handling.html
- START TRANSACTION, COMMIT, and ROLLBACK Syntax https://dev.mysql.com/doc/refman/8.0/en/commit.html
- DeadLock Examples https://github.com/aneasystone/mysql-deadlocks
- MySQL 死锁与日志二三事 https://www.cnblogs.com/cnsanshao/p/7252825.html
- InnoDB 的意向锁有什么作用？ https://www.zhihu.com/question/51513268
- Mysql死锁分析案例（一）https://www.cnblogs.com/chenshouchang/p/11266138.html
- 一个最不可思议的MySQL死锁分析 http://hedengcheng.com/?p=844
