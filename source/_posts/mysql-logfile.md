---
title: MySQL日志文件
tags: [Database, MySQL]
author: Mingshan
categories: [Database, MySQL]
date: 2020-04-30
---

MySQL常见的日志类型包括：

- 错误日志（error log）
- 二进制日志（binlog）
- 慢查询日志（slow query log）
- 查询日志（log）

# 错误日志

错误日志文件对MySQL的启动、运行、关闭过程进行了记录。MySQL DBA在遇到问题时应该首先查看该文件以便定位问题。该文件不仅记录了所有的错误信息，也记录一些警告信息或正确的信息。通过`SHOW VARIABLES LIKE '%log_error%'`来查看错误的路径：如下所示：

Variable_name      |Value              |
-------------------|-------------------|
binlog_error_action|ABORT_SERVER       |
log_error          |/var/log/mysqld.log|
log_error_verbosity|3                  |

<!-- more -->

# 慢查询日志

https://dev.mysql.com/doc/refman/5.7/en/slow-query-log.html

查看慢查询的日志是否开启，以及日志文件位置：

```
SHOW VARIABLES LIKE '%slow_query_log%'
```

Variable_name      |Value                            |
-------------------|---------------------------------|
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
---------------|---------|
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
-----------------------------|-----|
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
--------------------------------------|-----|
log_throttle_queries_not_using_indexes|0    |

默认是0，代表没有限制。

# 二进制日志

查看binlog是否开启：

```
show variables like 'log_bin';
```

Variable_name|Value|
-------------|-----|
log_bin      |OFF  |

默认是关闭的，我们先查询下MySQL数据是在哪个目录进行存储的，输入如下命令：

```
SHOW VARIABLES LIKE '%datadir%'
```

Variable_name|Value          |
-------------|---------------|
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

https://dev.mysql.com/doc/refman/5.7/en/mysqlbinlog-row-events.html


```
- 创建表
CREATE TABLE test.test1 (
	id INT NOT NULL AUTO_INCREMENT,
	name VARCHAR(30) NOT NULL,
	date DATA NULL,
	PRIMARY KEY (`id`)
)
ENGINE=InnoDB
DEFAULT CHARSET=utf8mb4
COLLATE=utf8mb4_general_ci;

- 开启事务，执行sql
START TRANSACTION;
INSERT INTO test.test1 VALUES(1, 'apple', NULL);
UPDATE test.test1 SET name = 'pear', date = '2009-01-01' WHERE id = 1;
DELETE FROM test.test1 WHERE id = 1;
COMMIT;
```

```
/*!50530 SET @@SESSION.PSEUDO_SLAVE_MODE=1*/;
/*!50003 SET @OLD_COMPLETION_TYPE=@@COMPLETION_TYPE,COMPLETION_TYPE=0*/;
DELIMITER /*!*/;
# at 4
#200429  9:11:16 server id 1  end_log_pos 123 CRC32 0xaaafe6c0 	Start: binlog v 4, server v 5.7.30-log created 200429  9:11:16 at startup
# Warning: this binlog is either in use or was not closed properly.
ROLLBACK/*!*/;
BINLOG '
9HypXg8BAAAAdwAAAHsAAAABAAQANS43LjMwLWxvZwAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
AAAAAAAAAAAAAAAAAAD0fKleEzgNAAgAEgAEBAQEEgAAXwAEGggAAAAICAgCAAAACgoKKioAEjQA
AcDmr6o=
'/*!*/;
# at 123
#200429  9:11:16 server id 1  end_log_pos 154 CRC32 0xfed7364e 	Previous-GTIDs
# [empty]
# at 154
#200429 10:08:11 server id 1  end_log_pos 219 CRC32 0xb45e4cec 	Anonymous_GTID	last_committed=0	sequence_number=1	rbr_only=no
SET @@SESSION.GTID_NEXT= 'ANONYMOUS'/*!*/;
# at 219
#200429 10:08:11 server id 1  end_log_pos 421 CRC32 0x0ea7428a 	Query	thread_id=3	exec_time=0	error_code=0
SET TIMESTAMP=1588169291/*!*/;
SET @@session.pseudo_thread_id=3/*!*/;
SET @@session.foreign_key_checks=1, @@session.sql_auto_is_null=0, @@session.unique_checks=1, @@session.autocommit=1/*!*/;
SET @@session.sql_mode=1436549152/*!*/;
SET @@session.auto_increment_increment=1, @@session.auto_increment_offset=1/*!*/;
/*!\C utf8 *//*!*/;
SET @@session.character_set_client=33,@@session.collation_connection=33,@@session.collation_server=33/*!*/;
SET @@session.lc_time_names=0/*!*/;
SET @@session.collation_database=DEFAULT/*!*/;
/* ApplicationName=DBeaver 6.1.5 - Main */ CREATE SCHEMA `test`
DEFAULT CHARACTER SET utf8mb4
DEFAULT COLLATE utf8mb4_general_ci
/*!*/;
# at 421
#200429 10:12:23 server id 1  end_log_pos 486 CRC32 0x08527478 	Anonymous_GTID	last_committed=1	sequence_number=2	rbr_only=no
SET @@SESSION.GTID_NEXT= 'ANONYMOUS'/*!*/;
# at 486
#200429 10:12:23 server id 1  end_log_pos 779 CRC32 0xe1082ae6 	Query	thread_id=3	exec_time=0	error_code=0
SET TIMESTAMP=1588169543/*!*/;
/* ApplicationName=DBeaver 6.1.5 - Main */ CREATE TABLE test.test1 (
	id INT NOT NULL AUTO_INCREMENT,
	name VARCHAR(30) NOT NULL,
	PRIMARY KEY (`id`)
)
ENGINE=InnoDB
DEFAULT CHARSET=utf8mb4
COLLATE=utf8mb4_general_ci
/*!*/;
# at 779
#200429 11:15:02 server id 1  end_log_pos 844 CRC32 0xd80b70b0 	Anonymous_GTID	last_committed=2	sequence_number=3	rbr_only=no
SET @@SESSION.GTID_NEXT= 'ANONYMOUS'/*!*/;
# at 844
#200429 11:15:02 server id 1  end_log_pos 1000 CRC32 0xf218a9f6 	Query	thread_id=3	exec_time=0	error_code=0
SET TIMESTAMP=1588173302/*!*/;
/* ApplicationName=DBeaver 6.1.5 - Main */ ALTER TABLE test.test1 ADD `date` DATE NULL
/*!*/;
# at 1000
#200429 11:16:06 server id 1  end_log_pos 1065 CRC32 0xc1887469 	Anonymous_GTID	last_committed=3	sequence_number=4	rbr_only=yes
/*!50718 SET TRANSACTION ISOLATION LEVEL READ COMMITTED*//*!*/;
SET @@SESSION.GTID_NEXT= 'ANONYMOUS'/*!*/;
# at 1065
#200429 11:16:06 server id 1  end_log_pos 1133 CRC32 0x2fa0666d 	Query	thread_id=3	exec_time=0	error_code=0
SET TIMESTAMP=1588173366/*!*/;
BEGIN
/*!*/;
# at 1133
#200429 11:16:06 server id 1  end_log_pos 1185 CRC32 0xe3ff3dea 	Table_map: `test`.`test1` mapped to number 101
# at 1185
#200429 11:16:06 server id 1  end_log_pos 1231 CRC32 0xd342352b 	Write_rows: table id 101 flags: STMT_END_F

BINLOG '
NpqpXhMBAAAANAAAAKEEAAAAAGUAAAAAAAEABHRlc3QABXRlc3QxAAMDDwoCeAAE6j3/4w==
NpqpXh4BAAAALgAAAM8EAAAAAGUAAAAAAAEAAgAD//wBAAAABWFwcGxlKzVC0w==
'/*!*/;
# at 1231
#200429 11:16:06 server id 1  end_log_pos 1283 CRC32 0x0e85e500 	Table_map: `test`.`test1` mapped to number 101
# at 1283
#200429 11:16:06 server id 1  end_log_pos 1343 CRC32 0x503c42f3 	Update_rows: table id 101 flags: STMT_END_F

BINLOG '
NpqpXhMBAAAANAAAAAMFAAAAAGUAAAAAAAEABHRlc3QABXRlc3QxAAMDDwoCeAAEAOWFDg==
NpqpXh8BAAAAPAAAAD8FAAAAAGUAAAAAAAEAAgAD///8AQAAAAVhcHBsZfgBAAAABHBlYXIhsg/z
QjxQ
'/*!*/;
# at 1343
#200429 11:16:06 server id 1  end_log_pos 1395 CRC32 0x3a133eb0 	Table_map: `test`.`test1` mapped to number 101
# at 1395
#200429 11:16:06 server id 1  end_log_pos 1443 CRC32 0xfb4ca372 	Delete_rows: table id 101 flags: STMT_END_F

BINLOG '
NpqpXhMBAAAANAAAAHMFAAAAAGUAAAAAAAEABHRlc3QABXRlc3QxAAMDDwoCeAAEsD4TOg==
NpqpXiABAAAAMAAAAKMFAAAAAGUAAAAAAAEAAgAD//gBAAAABHBlYXIhsg9yo0z7
'/*!*/;
# at 1443
#200429 11:16:06 server id 1  end_log_pos 1474 CRC32 0x6d6703d1 	Xid = 93
COMMIT/*!*/;
SET @@SESSION.GTID_NEXT= 'AUTOMATIC' /* added by mysqlbinlog */ /*!*/;
DELIMITER ;
# End of log file
/*!50003 SET COMPLETION_TYPE=@OLD_COMPLETION_TYPE*/;
/*!50530 SET @@SESSION.PSEUDO_SLAVE_MODE=0*/;
```

