---
title: MySQL日志文件
tags: [Database, MySQL]
author: Mingshan
categories: [Database, MySQL]
date: 2020-04-30
---

MySQL常见的日志类型包括：

- 错误日志（error log）
- 二进制日志（binary log）
- 慢查询日志（slow query log）
- 查询日志（general query log）

# [错误日志](https://dev.mysql.com/doc/refman/5.7/en/error-log.html)

错误日志文件对MySQL的启动、运行、关闭过程进行了记录。MySQL DBA在遇到问题时应该首先查看该文件以便定位问题。该文件不仅记录了所有的错误信息，也记录一些警告信息或正确的信息。通过`SHOW VARIABLES LIKE '%log_error%'`来查看错误的路径：如下所示：

Variable_name      |Value              |
-------------------|-------------------
binlog_error_action|ABORT_SERVER       |
log_error          |/var/log/mysqld.log|
log_error_verbosity|3                  |

<!-- more -->

# [慢查询日志](https://dev.mysql.com/doc/refman/5.7/en/slow-query-log.html)

如果MySQL有语句查询比较慢，在数据库层面有没有办法直接去定位这些SQL呢？MySQL提供了记录慢查询的文件，该文件会记录查询时间超过**long_query_time**的SQL语句，默认是关闭的。

查看慢查询的日志是否开启，以及日志文件位置：

```
SHOW VARIABLES LIKE '%slow_query_log%'
```

Variable_name      |Value                            |
-------------------|---------------------------------
slow_query_log     |OFF                              |
slow_query_log_file|/var/lib/mysql/localhost-slow.log|


慢查询日志默认是关闭的，通过如下命令将其开启：

```
SET GLOBAL slow_query_log=ON
```

一个查询消耗多长时间被定义为慢查询呢？MySQL是通过**long_query_time**这个参数来控制的：

```
SHOW VARIABLES LIKE '%long_query_time%'
```

Variable_name  |Value    |
---------------|---------
long_query_time|10.000000|

long_query_time单位是秒，默认十秒，可以通过以下命令进行设置：

```
SET GLOBAL long_query_time=1
```

如果有SQL查询时间超过设置的阈值，就会记录到慢查询日志里面，我们可以用 `tail -f xx-slow.log`来查看慢查询日志信息，比如：

```
$ tail -f db2-slow.log 
# Query_time: 1.290560  Lock_time: 0.001066 Rows_sent: 0  Rows_examined: 77
use iwmsfacility;
SET timestamp=1588212896;
SELECT DISTINCT r.*
FROM iwmsreceivebill AS r
WHERE (r.companyUuid = '0000001') AND (r.dcUuid = '000000110000029') AND (EXISTS (SELECT 1
FROM iwmsreceivebillitem AS i
WHERE (r.uuid = i.billUuid) AND (i.articleCode IN ('q'))))
ORDER BY r.billNumber DESC
LIMIT 61;
```

除此之外，如果查询没有使用索引，那么被记录到慢查询的日志概率性较大，MySQL 提供了 `log_queries_not_using_indexes`参数来控制是否将没有使用索引的查询记录到慢查询日志里面：

```
SHOW VARIABLES LIKE '%log_queries_not_using_indexes%'
```

Variable_name                |Value|
-----------------------------|-----
log_queries_not_using_indexes|OFF  |

`log_queries_not_using_indexes`参数默认是关闭的，我们可以将其打开，

```
SET GLOBAL log_queries_not_using_indexes=ON
```

如果`log_queries_not_using_indexes`打开了，MySQL提供了`log_throttle_queries_not_using_indexes`参数来控制每分钟最好写入多少条这样的记录到慢查询日志：

```
SHOW VARIABLES LIKE '%log_throttle_queries_not_using_indexes%'
```

Variable_name                         |Value|
--------------------------------------|-----
log_throttle_queries_not_using_indexes|0    |

默认是0，代表没有限制。

# [二进制日志](https://dev.mysql.com/doc/refman/5.7/en/binary-log.html)

二进制日志包含描述数据库更改的“事件”，如表创建操作或表数据更改。除非使用基于行的日志记录，否则它还包含可能已进行更改的语句的事件(例如，不匹配任何行的DELETE)。二进制日志还包含关于每个语句花费多长时间更新数据的信息。二进制日志有以下几个的用途:

- 恢复（recovery）：某些数据的恢复需要二进制日志，例如，在一个数据库全备文件恢复后，用户可以通过二进制日志进行point-in-time的恢复。
- 复制（replication）：其原理与恢复类似，通过复制和执行二进制日志使一台远程的MySQL数据库（一般称为slave或standby）与一台MySQL数据库（一般称为master或primary）进行实时同步。
- 审计（audit）：用户可以通过二进制日志中的信息来进行审计，判断是否有对数据库进行注入的攻击。

二进制文件默认是关闭的，开启后官方说会稍微影响性能。

查看binlog是否开启：

```
SHOW VARIABLES LIKE 'log_bin';
```

Variable_name|Value|
-------------|-----
log_bin      |OFF  |

默认是关闭的，我们先查询下MySQL数据是在哪个目录进行存储的，输入如下命令：

```
SHOW VARIABLES LIKE '%datadir%'
```

Variable_name|Value          |
-------------|---------------
datadir      |/var/lib/mysql/|

接着将binlog开启，需要更改MySQL的配置文件，输入命令`vim /etc/my.cnf`，然后添加如下配置：

```
[mysqld]
# binlog
log-bin=/var/lib/mysql/mysql-bin.log
expire-logs-days=14
max-binlog-size=500M
server-id=1
```
- log-bin：指明binlog的位置
- expire-logs-days：设置binlog清理时间
- max-binlog-size：设置每个文件的大小
- server-id：指定mysql集群的服务id

配置完后进行保存，然后执行`systemctl restart mysqld.service`重启MySQL，我们再去看`log_bin`参数，已经开启了。

我们来查看下MySQL数据存储都有哪些文件，在MySQL命令窗口执行命令：`system ls -lh /var/lib/mysql`，显示结果如下：

```
mysql> system ls -lh /var/lib/mysql
total 121M
-rw-r-----. 1 mysql mysql   56 Apr 29 04:00 auto.cnf
-rw-------. 1 mysql mysql 1.7K Apr 29 04:00 ca-key.pem
-rw-r--r--. 1 mysql mysql 1.1K Apr 29 04:00 ca.pem
-rw-r--r--. 1 mysql mysql 1.1K Apr 29 04:00 client-cert.pem
-rw-------. 1 mysql mysql 1.7K Apr 29 04:00 client-key.pem
-rw-r-----. 1 mysql mysql  387 Apr 29 09:11 ib_buffer_pool
-rw-r-----. 1 mysql mysql  12M Apr 29 09:11 ibdata1
-rw-r-----. 1 mysql mysql  48M Apr 29 09:11 ib_logfile0
-rw-r-----. 1 mysql mysql  48M Apr 29 04:00 ib_logfile1
-rw-r-----. 1 mysql mysql  12M Apr 29 09:11 ibtmp1
-rw-r-----. 1 mysql mysql  179 Apr 29 07:07 localhost-slow.log
drwxr-x---. 2 mysql mysql 4.0K Apr 29 04:00 mysql
-rw-r-----. 1 mysql mysql  154 Apr 29 09:11 mysql-bin.000001
-rw-r-----. 1 mysql mysql   32 Apr 29 09:11 mysql-bin.index
srwxrwxrwx. 1 mysql mysql    0 Apr 29 09:11 mysql.sock
-rw-------. 1 mysql mysql    6 Apr 29 09:11 mysql.sock.lock
drwxr-x---. 2 mysql mysql 8.0K Apr 29 04:00 performance_schema
-rw-------. 1 mysql mysql 1.7K Apr 29 04:00 private_key.pem
-rw-r--r--. 1 mysql mysql  452 Apr 29 04:00 public_key.pem
-rw-r--r--. 1 mysql mysql 1.1K Apr 29 04:00 server-cert.pem
-rw-------. 1 mysql mysql 1.7K Apr 29 04:00 server-key.pem
drwxr-x---. 2 mysql mysql 8.0K Apr 29 04:00 sys
```

其中**mysql-bin.index**是一个文本文件，该文件内记录真实二进制文件的位置，如下所示：

```
[root@localhost mysql]# vim mysql-bin.index 

/var/lib/mysql/mysql-bin.000001
```

**mysql-bin.000001**就是真实的二进制文件，这里只有一个，关于如何解读二进制文件及如何利用二进制文件进行恢复数据，后面再写。

二进制日志有格式的概念，参数为：**binlog_format**，该参数可设的值有STATEMENT、ROW和MIXED。输入如下明细：

```
SHOW VARIABLES LIKE '%binlog_format%'
```

Variable_name|Value|
-------------|-----
binlog_format|ROW  |

可以看到，MySQL默认的**binlog_format**的值为**ROW**。

MySQL的**binlog_format**有三个选项，分别是**STATEMENT**、**ROW**和**MIXED**。

- STATEMENT格式和之前的MySQL版本一样，二进制日志文件记录的是日志的逻辑SQL语句。
- 在ROW格式下，二进制日志记录的不再是简单的SQL语句了，而是记录表的行更改情况。基于ROW格式的复制类似于Oracle的物理Standby（当然，还是有些区别）。同时，对上述提及的Statement格式下复制的问题予以解决。从MySQL 5.1版本开始，如果设置了binlog_format为ROW，可以将InnoDB的事务隔离基本设为READCOMMITTED，以获得更好的并发性。
- 在MIXED格式下，MySQL默认采用STATEMENT格式进行二进制日志文件的记录，但是在一些情况下会使用ROW格式，可能的情况有：
  1. 表的存储引擎为NDB，这时对表的DML操作都会以ROW格式记录。
  2. 使用了UUID（）、USER（）、CURRENT_USER（）、FOUND_ROWS（）、ROW_COUNT（）等不确定函数。
  3. 使用了INSERT DELAY语句。
  4. 使用了用户定义函数（UDF）。
  5. 使用了临时表（temporary table）。

# [查询日志](https://dev.mysql.com/doc/refman/5.7/en/query-log.html)

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

# References：

- https://dev.mysql.com/doc/refman/5.7/en/error-log.html
- https://dev.mysql.com/doc/refman/5.7/en/slow-query-log.html
- https://dev.mysql.com/doc/refman/5.7/en/binary-log.html
- 姜承尧 《MySQL技术内幕:InnoDB存储引擎(第二版)》


