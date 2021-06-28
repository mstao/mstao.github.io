---
title: MySQL隐式转换问题
tags: [Database, MySQL]
author: Mingshan
categories: [Database, MySQL]
date: 2021-06-28
---

MySQL隐式转换在实际开发中比较隐蔽，如果不清楚其隐式转换规则，在测试环境很有可能测不出来问题，但生产环境可能就爆发了，造成生产事故。下面我们来详细了解下MySQL隐式转换问题。

<!-- more -->

# 问题复现

下面有几个SQL，思考下分别会输出什么？
```java
SELECT 1 + '1';
SELECT 1 + '1C';
SELECT 1 + 'C1';
```
结果是：
```java
2
2
1
```
对于第一个语句来说，大概可以猜得到结果，对于第二个第三个感觉很莫名其妙，第二个好像是截取了第一个数字，第三个好像是把 `C1` 这个字符串给忽略了。这个就是MySQL隐式转换的例子，当然我们现实中不会这样写，下面来看一个与查询相关的例子。

下面有一个表
```sql
CREATE TABLE `account` (
  `id` int(11) DEFAULT NULL,
  `amount` int(11) DEFAULT NULL,
  `state` varchar(100) DEFAULT NULL
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci
```
有两行数据：

| id | amount | state |
| --- | --- | --- |
| 1 | 10 | 1 |
| 2 | 11 | ZZZ |



请问下面查询SQL会输出什么？
```sql
select * from test.account where state = 0;
```
**上面的sql语句会把上面两行数据都查出来！！！**


注意 `state` 字段在DDL中定义为字符串，我们查询时用的是`state = 0`，是数字，这里两者的类型不相同。所以这里有一个隐式转换过程。

# 问题原因
在MySQL 官方文档中，有如下说明：

> When an operator is used with operands of different types, type conversion occurs to make the operands compatible. 


即如果两个操作的类型不一致(传入的数据类型与MySQL表存储的类型)，会进行隐形的数据类型转换，这种转换往往是隐蔽的，大部分只有发生了问题才被发现。

针对上面的场景而言（字符串转数字）MySQL有以下原则：


- 如果字符串的第一个字符就是非数字的字符，那么转换为数字就是0。比如如果是`ab`，那么转换后就是0
- 如果字符串以数字开头
   - 如果字符串中都是数字，那么转换为数字就是整个字符串对应的数字。比如是字符串`123`, 转换后是数字123 
   - 如果字符串中存在非数字，那么转换为的数字就是开头的那些数字对应的值。比如字符串`12ab2`，转换后是数字12



这里还有一个非常重要的一点，就是索引问题，如果表的某列是字符串类型，并且给建立了索引，那么进行隐式转换后，索引是会失效，如下面sql：
```
SELECT * FROM tbl_name WHERE str_col=1;
```


其他隐式转换可以参考MySQL官方文档，遇到了再补充吧。

官方提供的隐式转换列表：


- If one or both arguments are NULL, the result of the comparison is NULL, except for the NULL-safe [<=>](https://dev.mysql.com/doc/refman/5.7/en/comparison-operators.html#operator_equal-to) equality comparison operator. For NULL <=> NULL, the result is true. No conversion is needed.
- If both arguments in a comparison operation are strings, they are compared as strings.
- If both arguments are integers, they are compared as integers.
- Hexadecimal values are treated as binary strings if not compared to a number.
- If one of the arguments is a [TIMESTAMP](https://dev.mysql.com/doc/refman/5.7/en/datetime.html) or [DATETIME](https://dev.mysql.com/doc/refman/5.7/en/datetime.html) column and the other argument is a constant, the constant is converted to a timestamp before the comparison is performed. This is done to be more ODBC-friendly. This is not done for the arguments to [IN()](https://dev.mysql.com/doc/refman/5.7/en/comparison-operators.html#operator_in). To be safe, always use complete datetime, date, or time strings when doing comparisons. For example, to achieve best results when using [BETWEEN](https://dev.mysql.com/doc/refman/5.7/en/comparison-operators.html#operator_between) with date or time values, use [CAST()](https://dev.mysql.com/doc/refman/5.7/en/cast-functions.html#function_cast) to explicitly convert the values to the desired data type.A single-row subquery from a table or tables is not considered a constant. For example, if a subquery returns an integer to be compared to a [DATETIME](https://dev.mysql.com/doc/refman/5.7/en/datetime.html) value, the comparison is done as two integers. The integer is not converted to a temporal value. To compare the operands as [DATETIME](https://dev.mysql.com/doc/refman/5.7/en/datetime.html) values, use [CAST()](https://dev.mysql.com/doc/refman/5.7/en/cast-functions.html#function_cast) to explicitly convert the subquery value to [DATETIME](https://dev.mysql.com/doc/refman/5.7/en/datetime.html).
- If one of the arguments is a decimal value, comparison depends on the other argument. The arguments are compared as decimal values if the other argument is a decimal or integer value, or as floating-point values if the other argument is a floating-point value.
- In all other cases, the arguments are compared as floating-point (real) numbers. For example, a comparison of string and numeric operands takes place as a comparison of floating-point numbers.


# 如何避免
之所以有隐式转换，一大部分是因为表设计的不够规范，如果我们可以在设计表结构时规避这些问题，比如一些枚举值，就不要用0,1,2这种魔法字符串代替，而是用真实有意义的枚举来代替，就可以规避该问题，不至于写sql 时搞不清字段的类型。


如果我们在进行sql生成时严格定义字段的类型，比如用mybatis时，不要在SQL语句里面写死这种预定义值，而是由参数传入，大部分就可以避免该情况的发生。
# 参考：

- [https://blog.csdn.net/HaHa_Sir/article/details/93666147](https://blog.csdn.net/HaHa_Sir/article/details/93666147)
- [https://dev.mysql.com/doc/refman/5.7/en/type-conversion.html](https://dev.mysql.com/doc/refman/5.7/en/type-conversion.html)
