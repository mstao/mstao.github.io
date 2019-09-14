---
title: 定位INFORMATION_SCHEMA INNODB_TRX事务长时间处于RUNNING状态
categories: Database
tags: [Database]
author: Mingshan
date: 2019-9-1
---

INFORMATION_SCHEMA提供对数据库元数据的访问、关于MySQL服务器的信息，如数据库或表的名称、列的数据类型或访问权限。其中有一个关于InnoDB数据库引擎表的集合，里面有记录数据库事务和锁的相关表，InnoDB INFORMATION_SCHEMA表可以用来监视正在进行的InnoDB活动，在它们变成问题之前检测低效，或者对性能和容量问题进行故障排除。在实际开发和应用中，会碰到和数据库事务相关的问题，比如事务一直未结束，出现行锁，表锁以及死锁等情况，这时我们就需要有一个快速定位问题行之有效的方法，所以我们来系统了解下INFORMATION_SCHEMA和定位事务问题。

<!-- more -->

## 事务基本要素（ACID）及并发问题

### ACID

[ACID](https://en.wikipedia.org/wiki/ACID)，是指数据库管理系统（DBMS）在写入或更新资料的过程中，为保证事务（transaction）是正确可靠的，所必须具备的四个特性：原子性（atomicity，或称不可分割性）、一致性（consistency）、隔离性（isolation，又称独立性）、持久性（durability）:

- Atomicity（原子性）：一个事务（transaction）中的所有操作，或者全部完成，或者全部不完成，不会结束在中间某个环节。事务在执行过程中发生错误，会被回滚（Rollback）到事务开始前的状态，就像这个事务从来没有执行过一样。即，事务不可分割、不可约简。
- Consistency（一致性）：在事务开始之前和事务结束以后，数据库的完整性没有被破坏。这表示写入的资料必须完全符合所有的预设约束、触发器、级联回滚等。
- Isolation（隔离性）：数据库允许多个并发事务同时对其数据进行读写和修改的能力，隔离性可以防止多个事务并发执行时由于交叉执行而导致数据的不一致。事务隔离分为不同级别，包括未提交读（Read uncommitted）、提交读（read committed）、可重复读（repeatable read）和串行化（Serializable）。
- Durability（持久性）：事务处理结束后，对数据的修改就是永久的，即便系统故障也不会丢失。


事务是恢复和并发控制的基本单位，上面ACID被破坏的可能因素有：

- 多个事务并行运行时，不同事务的操作交叉执行
- 事务在运行过程中强行终止

### 并发问题

如果事务是串行执行的，即每个时刻都只有一个事务在运行，那么很明显事务之间没有交叉，没有抢夺共享资源，我们平时遇到的数据库并发问题将统统不存在，但在实际应用中，为了充分利用系统资源，多个事务并行执行是非常有必要的，但为了保证事务的隔离性和一致性，数据库肯定会有一些必要的牺牲来支撑并发调度，下面我们来了解下并发带来的问题。

```
 事务串行执行                            事务并发执行

     |                              T1      T2       T3
     | T1                           T1  | ⟶   
     |                                      T2 |
     | T2                           T1  | ⟵ 
     |                                    ⟶   ⟶    T3|
     | T3
     | 

```

#### 丢失修改/丢失更新(lose update)

两个事务T1和T2读入统一数据并修改，T2的提交结果破坏了T1的提交结果，导致T1的修改被丢失。


#### 不可重复读(non-repeatable read)

不可重复度是指事务T1读取数据后，事务T2执行更新操作，使事务T1无法再现前一次读取结果。分三种情况：

1. 事务T1读取某一数据后，事务T2对其进行了修改，当事务T1再次读取该数据时，得到的值与前一次不同；
2. 事务T1按照某一条件读取某些数据后，事务T2删除了其中部分记录，当事务T1再次以相同条件读取数据时，发现某些记录神秘消失了；
3. 事务T1按照某一条件读取某些数据后，事务T2在符合该条件下插入了部分记录，当事务T1再次以相同条件读取数据时，发现多了一些记录。

第二条和第三条好像发生幻觉一样，数据莫名其妙多了或者少了，有时也被称为**幻影(phantom row)** 或者 **幻读**，其注重插入和删除操作，而第一种注重修改操作。所以对于第一种，对象是一条记录或者多条记录，解决此问题需要**锁行**(row locks)，其他的记录不用关心；而对于幻读，操作对象是整个表，所以需要锁表(table locks)。

#### 脏读(dirty read)

脏读是指事务T1读取了事务T2更新后的数据(已写入磁盘)，然后T2回滚操作，那么T1读取到的数据是脏数据。


## 数据库封锁协议

产生上面三种数据不一致的主要原因是破坏了事务的隔离性。如果我们学过数据库相关理论，那么可能对封锁这个概念有所了解。封锁是数据库实现并发控制的一个非常重要的技术。基本的封锁类型有两种：`排它锁（exclusive locks 简称X锁）` 和 `共享锁（share locks 简称S锁）`。

`排它锁又称写锁`。若事务T对数据对象A加上X锁，则只允许T读取和修改A，其他任何事务都不能再对A加任何类型的锁，直到事务T释放A上的锁为止。

`共享锁又称读锁`。如事务T对数据对象A加上S锁，则事务T可以读A但不能修改A，其他事务只能再对A家S锁，而不能再加X锁，直至T释放A上的S锁为止。

排它锁和共享锁控制方式用如下表格显示(Y: 相容；N: 不相容)：


\  | X | S | - 
---|---|---|---
X  | N | N | Y 
S  | N | Y | Y
-  | Y | Y | Y

了解了排它锁和共享锁，那么何时申请X锁或S锁、持续时间、何时释放等。这些规则被称为封锁协议，下面介绍下是三种封锁协议。

### 一级封锁协议

一级封锁协议是指，事务T在修改数据R之前必须先对其加X锁，直到事务结束才释放。事务结束包括正常结束（COMMIT）和 非正常结束（ROLLBACK）。一级封锁协议可防止丢失修改，并保证事务是可恢复的，但对于仅仅是读数据是不需要加锁的，所以不能保证可重复读和不读脏数据。

### 二级封锁协议

二级封锁协议是指，在一级封锁协议基础上增加事务T在读取数据R之前必须先对其加上S锁，读完后即可释放S锁。它除防止了丢失修改，还可进一步防止读脏数据。由于读完即可释放S锁，所以不可保证重复读。

### 三级封锁协议

三级封锁协议是指，在一级封锁协议基础上增加事务T在读取数据R之前必须先对其加S锁，直到事务结束后才释放。它除防止了丢失修改和读脏数据，还可防止重复读。

## 活锁和死锁

上面的封锁协议有可能导致活锁或死锁问题。

### 活锁

如果事务T1封锁了事务R，事务T2又请求封锁R，于是T2等待；T3也要封锁数据R，T3也要等待；当事务T1释放了R上的封锁之后，首先批准了T3的请求，T2依然等待。。。就这样每次都把T2给忽略了，这样T2可能一直等待，这就是**活锁**的情形。

如果我们有了解Java的可重复锁，对这种情形是比较熟悉的，可重复锁分两种模式，公平锁和不公平锁，不公平锁对应的是活锁的情形，但至于具体的数据库是如何处理的，那么就不能一概而论了。

### 死锁

如果事务T1封锁了R1，事务T2封锁了R2，然后T1又请求封锁T2，因为R2已经被T2封锁了，所以事务T1只能等待T2释放R2上的锁；接着事务T2又请求封锁R1，R1此时已经被T1封锁了，T2只能等待T1释放R1上的锁。**这就形成了T1在等待T2，而T2又在等待T1的局面，形成了死锁**。

判断是否可能发生死锁，可以用等待图检测。事务等待图是一个有向图$G=(T, U)$，T为结点的集合，每个结点表示正在运行的事务；U为边的集合，每个边表示事务等待情况。如T1等待T2，那么在T1和T2之间画一条有向边，由T1指向T2，如下图所示：

```
graph LR;
  T1-->T2
```

对于死锁情况来说，如果两个结点之间出现回路，那么系统就是出现了死锁，如下图所示：

```
graph LR;
  T1-->T2
  T2-->T1
```

对于多个事务的检测也是如此：

```
graph LR;
  T1-->T2
  T2-->T3
  T3-->T2
  T3-->T4
  T4-->T1
```


## MySQL的InnoDB四种事务隔离级别

上面的封锁协议是在数据库概论级别阐述的，但对于具体的数据库，实现封锁协议可能有所不同，现在以MySQL为例，了解MySQL的四种事务隔离级别。

InnoDB引擎的四种事务隔离级别分别是`READ UNCOMMITTED`, `READ COMMITTED`, `REPEATABLE READ`, and `SERIALIZABLE`。InnoDB引擎默认隔离级别是 `REPEATABLE READ `。

### REPEATABLE READ

这是InnoDB的默认隔离级别。在数据库系统对`REPEATABLE READ`的说明中，禁止脏读和不可重复读，但可能会出现幻读。InnoDB存储引擎与标准SQL有所不同，使用Next-Key 和 Gap-Key 锁的算法，避免幻读的产生。简单来说，可以分为以下几方面：

1. 对于普通的(非锁定的)SELECT语句，不会加锁（S锁也不加），而是读取数据的快照（read the snapshot）
2. 对于锁定读取(选择with For UPDATE或For SHARE)、UPDATE和DELETE语句，锁定取决于该语句是使用具有唯一搜索条件的惟一索引，还是使用范围类型搜索条件。
    1. 对于具有唯一搜索条件的唯一索引，InnoDB只锁定找到的索引记录，而不锁定它之前的空白。
    2. 对于其他搜索条件，InnoDB锁定扫描的索引范围，使用gap-lock锁或next-key锁来阻止其他会话插入到范围覆盖的间隙中

### READ COMMITTED

在`READ COMMITTED`级别下，除了唯一性的约束检查(duplicate-key checking)和外键约束检查(foreign-key constraint checking)需要使用gap lock，InnoDB不会使用此算法。所以由于gap lock算法禁用，该级别下是会出现幻读的。

只读提交隔离级别仅支持基于行的二进制日志记录。如果使用`read committed with binlog_format=mixed`，服务器将自动使用基于行的日志记录。


1. 对于update或delete语句，innodb只对其更新或删除的行持有锁。在mysql评估where条件之后，将释放非匹配行的记录锁。这大大降低了死锁的可能性，但它们仍然可能发生。
2. 对于update语句，如果一行已经被锁定，innodb执行“semi-consistent”读取，将最新提交的版本返回给mysql，这样mysql就可以确定该行是否与更新的where条件匹配。如果行匹配（必须更新），mysql会再次读取行，这次innodb要么锁定它，要么等待对它的锁定。


### READ UNCOMMITTED

可以读取未提交记录。没有解决脏读问题，不建议使用。

### SERIALIZABLE

解决上面说的所有事务并发问题，但事务是串行执行的。

## INFORMATION_SCHEMA中与事务和锁相关的表

INFORMATION_SCHEMA中与事务和锁相关的表有如下几个：

- INFORMATION_SCHEMA.INNODB_TRX 
- INFORMATION_SCHEMA.INNODB_LOCKS
- INFORMATION_SCHEMA.INNODB_LOCK_WAITS
- INFORMATION_SCHEMA.PROCESSLIST

下面分别介绍下这几个表及其字段。

### INFORMATION_SCHEMA.INNODB_TRX

INNODB_TRX表提供了关于当前在InnoDB中执行的每个事务(不包括只读事务)的信息，包括事务是否等待锁、事务何时启动以及事务正在执行的SQL语句(如果有的话)。INNODB_TRX表有以下字段：


Field | Comment
---|---
TRX_ID | 自增id
TRX_WEIGHT | 事务权重，反映(但不一定是准确的计数)事务更改的行数和锁定的行数。为了解决死锁，InnoDB选择权重最小的事务作为要回滚的“受害者”
TRX_STATE | 事务执行状态。允许的值包括运行(RUNNING)、锁等待(LOCK WAIT)、回滚(ROLLING BACK)和提交(COMMITTING)。
TRX_STARTED | 事务开始时间
TRX_REQUESTED_LOCK_ID | 事务当前等待的锁的ID，如果TRX_STATE为LOCK WAIT;否则无效。要获取关于锁的详细信息，请将此列与INNODB_LOCKS表的LOCK_ID列关联
TRX_WAIT_STARTED | 事务开始等待锁的时间，如果TRX_STATE为锁等待(LOCK WAIT);否则无效。
TRX_MYSQL_THREAD_ID | MySql事务线程id，要获取关于线程的详细信息，与INFORMATION_SCHEMA PROCESSLIST表的ID列关联
TRX_QUERY | 事务正在执行的SQL语句
TRX_OPERATION_STATE | 事务当前操作
TRX_TABLES_IN_USE | 处理此事务的当前SQL语句使用的InnoDB表的数量
TRX_TABLES_LOCKED | 当前SQL语句具有行锁(row locks)的InnoDB表的数量(因为这些是行锁(row locks)，而不是表锁(table locks)，所以表通常仍然可以由多个事务读写，尽管有些行被锁定了)
TRX_LOCK_STRUCTS | 事务保留的锁的数量
TRX_LOCK_MEMORY_BYTES | 此事务在内存中的锁结构占用的总大小
TRX_ROWS_LOCKED | 此事务锁定的近似数目或行。该值可能包括物理上存在但对事务不可见的删除标记行
TRX_ROWS_MODIFIED | 此事务中修改和插入的行数量
TRX_CONCURRENCY_TICKETS | 指示当前事务在换出之前可以做多少工作的值，由innodb_concurrency_tickets系统变量指定
TRX_ISOLATION_LEVEL | 事务隔离级别
TRX_UNIQUE_CHECKS | 是否为当前事务打开或关闭唯一性检查
TRX_FOREIGN_KEY_CHECKS | 是否为当前事务打开或关闭外键检查
TRX_ADAPTIVE_HASH_LATCHED | 自适应哈希索引是否被当前事务锁定
TRX_ADAPTIVE_HASH_TIMEOUT | 是否立即放弃自适应哈希索引的搜索锁存器，还是在来自MySQL的调用之间保留它
TRX_IS_READ_ONLY | 值为1表示只读事务
TRX_AUTOCOMMIT_NON_LOCKING | 值1表示事务是一个SELECT语句，它不使用FOR UPDATE或LOCK IN SHARED MODE子句，并且在执行时启用了autocommit，因此事务将只包含这一条语句。当这个列和TRX_IS_READ_ONLY都为1时，InnoDB优化事务，以减少与更改表数据的事务相关的开销。


### INFORMATION_SCHEMA.INNODB_LOCKS

INNODB_LOCKS表提供有关innodb事务已请求但尚未获取的每个锁的信息，以及事务持有以阻止另一个事务的每个锁的信息。(该表在MySQL8.0被废弃) INNODB_LOCKS表有以下字段：


Field | Comment
---|---
LOCK_ID | 自增id
LOCK_TRX_ID | 持有锁的事务的ID。要获取关于事务的详细信息，请将此列与INNODB_TRX表的TRX_ID列关联。
LOCK_MODE | 描述如何获取锁. 可以是 S, X, IS, IX, GAP, AUTO_INC, and UNKNOWN.
LOCK_TYPE | 锁的类型。 RECORD for a row-level lock, TABLE for a table-level lock
LOCK_TABLE | 已锁定或包含锁定记录的表的名称
LOCK_INDEX | 索引名称，仅当LOCK_TYPE为RECORD时；否则无效
LOCK_SPACE | 如果LOCK_TYPE为RECORD，则锁定记录的表空间ID；否则无效
LOCK_PAGE | 如果LOCK_TYPE为RECORD，则锁定记录的页码；否则无效
LOCK_REC | 如果LOCK_TYPE为RECORD，则页内锁定的记录的堆号；否则无效

### INFORMATION_SCHEMA.INNODB_LOCK_WAITS

The INNODB_LOCK_WAITS table contains one or more rows for each blocked InnoDB transaction, indicating the lock it has requested and any locks that are blocking that request.

Field | Comment
---|---
REQUESTING_TRX_ID | 被阻塞的正在请求事务id
REQUESTED_LOCK_ID | 事务正在等待的锁的ID。要获取有关锁的详细信息，请将此列与INNODB_LOCKS表的LOCK_ID列关联
BLOCKING_TRX_ID | 阻塞事务的ID
BLOCKING_LOCK_ID | 事务持有的阻止另一个事务继续进行的锁的ID。要获取关于锁的详细信息，请将此列与INNODB_LOCKS表的LOCK_ID列关联。


### INFORMATION_SCHEMA.PROCESSLIST

PROCESSLIST表记录了每个MySql线程的用户，地址以及操作的表等其他信息。具体字段如下：

Field | Comment
---|---
ID | 标识ID。这与在SHOW PROCESSLIST语句的Id列、Performance Schema threads表的PROCESSLIST_ID列中显示的值类型相同，并由CONNECTION_ID()函数返回
USER | 发出该语句的mysql用户
HOST | 发出该语句的客户机的主机名(系统用户除外，没有主机)。
DB | 默认数据库
COMMAND | 线程正在执行的命令的类型
TIME | 线程处于当前状态的时间(以秒为单位)
STATE | 指示线程正在执行的操作、事件或状态
INFO | 线程正在执行的语句，如果没有执行任何语句，则为NULL


## Performance Schema

在INFORMATION_SCHEMA.PROCESSLIST表的字段描述中，我们了解到PROCESSLIST的ID字段与Performance Schema中threads表有关联（performance_schema.threads），那么Performance Schema在MySql中有什么用呢？

Performance Schema是用于在较低级别监视MySQL服务运行过程中的资源消耗、资源等待等情况。按照官方的描述，有一大堆特性，详情请参考：[MySQL Performance Schema](https://dev.mysql.com/doc/refman/5.7/en/performance-schema.html)。我们主要来看`events_statements_current` 和 `threads` 这两个表。

### events_statements_current

events_statements_current表包含当前语句的事件。表为每个线程存储一行，显示线程最近监视的语句事件的当前状态。具体字段请参考：[The events_statements_current Table](https://dev.mysql.com/doc/refman/5.7/en/events-statements-current-table.html)

### threads

threads表为每一个服务线程创建一条记录。每一行包含关于线程的信息，并指示是否为线程启用监视和历史事件日志记录。具体字段请参考：[The threads Table](https://dev.mysql.com/doc/refman/5.7/en/threads-table.html)


## 定位事务一直处于RUNNING状态

### 前提条件

#### 1. 开启MySQL的general log

general log会记录下发送给MySQL服务器的所有SQL记录，因为SQL的量大，默认是不开启的。如果一个问题反复出现（经常出现事务不结束），这个时候需要把general log打开，事务没有提交，一样会写到general_log，这样来定位出现问题的SQL。

MySQL有三个参数用于设置general log：

- general_log：用于开启general log。ON表示开启，OFF表示关闭。
- log_output：日志输出的模式。FILE表示输出到文件，TABLE表示输出到mysq库的general_log表，NONE表示不记录general_log。
- general_log_file：日记输出文件的路径，这是log_output=FILE时才会输出到此文件。

**查看general_log有没有开启**

```SQL
show variables like '%general%';
```
默认general_log是OFF的，general_log_file是日志输出路径：

```
mysql> show variables like '%general%';
+------------------+---------------------+
| Variable_name    | Value               |
+------------------+---------------------+
| general_log      | OFF                 |
| general_log_file | DESKTOP-Q1D3TT5.log |
+------------------+---------------------+
```

如果general_log是关闭的，执行下面SQL，开启之：

```
set global general_log=1;
或
set global general_log=ON;
```

**查看日志输出模式**


```
show variables where Variable_name="log_output";
```

默认是FILE，如下：

```
mysql> show variables where Variable_name="log_output";
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| log_output    | FILE  |
+---------------+-------+
```

**关闭general log**

大多数情况是临时开启general log，需要记得关闭，并把日志的输出模式恢复为FILE。

```
set global general_log=OFF;
set global log_output='FILE'
```

### 定位步骤

1. 使用如下语句查看事务，找到状态为RUNNING的记录

```
SELECT * FROM information_schema.INNODB_TRX;
```

2. 通过trx_mysql_thread_id: xxx的去查询information_schema.processlist找到执行事务的客户端请求的SQL线程

```
select * from information_schema.PROCESSLIST WHERE ID = 'xxx';
```

3. 查看到端口和host以后，再在对应的服务器查看相关的应用和日志

```
netstat -nlatp | grep 23452
ps -eaf | grep 12059
```

4. 如果无法定位，此时我们需要从performance_schema来寻找特定线程的信息

查看事件

```
select * from performance_schema.events_statements_current
```

根据我们拿到的线程id去查，可以获取到具体的执行sql

```
select * from performance_schema.events_statements_current
where THREAD_ID in (select THREAD_ID from performance_schema.threads where PROCESSLIST_ID=15844)
```

5. 如果还无法定位，此时我们已经有了事务的线程id(trx_mysql_thread_id)，那么就去general_log去查一下，找到执行sql，再具体分析：


```
cat /usr/software/mysql/data/mysql/db2.log | grep 4121
```

如果以上根据SQL分析不出来问题，我们需要从我们系统来进行定位，此时需要保存“案发现场”，数据库中处于RUNNING的事务先不要结束掉，然后根据上面定位的进程对应的项目来跟踪线程的执行情况，可以利用jconsole或者jmc来跟踪线程的执行活动，或者用jstack来跟踪。

## References：

- 王珊，萨师煊 《数据库系统概论》
- [INFORMATION_SCHEMA InnoDB Tables](https://dev.mysql.com/doc/refman/5.7/en/innodb-i_s-tables.html)
- [The INFORMATION_SCHEMA INNODB_TRX Table](https://dev.mysql.com/doc/refman/5.7/en/innodb-trx-table.html)
- [Using InnoDB Transaction and Locking Information](https://dev.mysql.com/doc/refman/5.7/en/innodb-information-schema-examples.html)
- [MySQL Performance Schema](https://dev.mysql.com/doc/refman/5.7/en/performance-schema.html)
- [The threads Table](https://dev.mysql.com/doc/refman/5.7/en/threads-table.html)
- [The events_statements_current Table](https://dev.mysql.com/doc/refman/5.7/en/events-statements-current-table.html)
- [MySQL开启general_log查看执行的SQL语句](https://majing.io/posts/10000005371184)
- [SET TRANSACTION Syntax](https://dev.mysql.com/doc/refman/8.0/en/set-transaction.html)
- [Transaction Isolation Levels](https://dev.mysql.com/doc/refman/8.0/en/innodb-transaction-isolation-levels.html)
- [MySQL的四种事务隔离级别](https://www.cnblogs.com/huanongying/p/7021555.html)
- [READ-COMMITED 与 REPEATABLE-READ](https://segmentfault.com/a/1190000008677275)
- [Getting “Lock wait timeout exceeded; try restarting transaction” even though I'm not using a transaction](https://stackoverflow.com/questions/5836623/getting-lock-wait-timeout-exceeded-try-restarting-transaction-even-though-im?r=SearchResults)
- [How to debug Lock wait timeout exceeded on MySQL?](https://stackoverflow.com/questions/6000336/how-to-debug-lock-wait-timeout-exceeded-on-mysql?r=SearchResults)
- [InnoDB and the ACID Model](https://dev.mysql.com/doc/refman/5.7/en/mysql-acid.html)
- [ACID - 维基百科](https://en.wikipedia.org/wiki/ACID)
- [InnoDB Locking](https://dev.mysql.com/doc/refman/8.0/en/innodb-locking.html)
- [Mysql加锁过程详解（1）-基本知识](https://www.cnblogs.com/crazylqy/p/7611069.html)
- [Isolation_(database_systems) - 维基百科](https://en.wikipedia.org/wiki/Isolation_(database_systems))
