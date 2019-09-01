---
title: 定位INFORMATION_SCHEMA INNODB_TRX事务长时间处于RUNNING状态
categories: Database
tags: [Database]
author: Mingshan
date: 2019-9-1
---

INFORMATION_SCHEMA提供对数据库元数据的访问、关于MySQL服务器的信息，如数据库或表的名称、列的数据类型或访问权限。其中有一个关于InnoDB数据库引擎表的集合，里面有记录数据库事务和锁的相关表，InnoDB INFORMATION_SCHEMA表可以用来监视正在进行的InnoDB活动，在它们变成问题之前检测低效，或者对性能和容量问题进行故障排除。在实际开发和应用中，会碰到和数据库事务相关的问题，比如事务一直未结束，出现行锁，表锁以及死锁等情况，这时我们就需要有一个快速定位问题行之有效的方法，所以我们来系统了解下INFORMATION_SCHEMA和定位事务问题。

<!-- more -->

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


## 数据库封锁协议

如果我们学过数据库相关理论，那么可能对封锁这个概念有所了解。封锁是数据库实现并发控制的一个非常重要的技术。基本的封锁类型有两种：`排它锁（exclusive locks 简称X锁）` 和 `共享锁（share locks 简称S锁）`。

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

## 模拟（Using InnoDB Transaction and Locking Information）

请参考：[Using InnoDB Transaction and Locking Information](https://dev.mysql.com/doc/refman/5.7/en/innodb-information-schema-examples.html)

## References：

- 王珊，萨师煊 《数据库系统概论》
- [INFORMATION_SCHEMA InnoDB Tables](https://dev.mysql.com/doc/refman/5.7/en/innodb-i_s-tables.html)
- [The INFORMATION_SCHEMA INNODB_TRX Table](https://dev.mysql.com/doc/refman/5.7/en/innodb-trx-table.html)
- [Using InnoDB Transaction and Locking Information](https://dev.mysql.com/doc/refman/5.7/en/innodb-information-schema-examples.html)
- [MySQL Performance Schema](https://dev.mysql.com/doc/refman/5.7/en/performance-schema.html)
- [The threads Table](https://dev.mysql.com/doc/refman/5.7/en/threads-table.html)
- [The events_statements_current Table](https://dev.mysql.com/doc/refman/5.7/en/events-statements-current-table.html)
- [MySQL开启general_log查看执行的SQL语句](https://majing.io/posts/10000005371184)
