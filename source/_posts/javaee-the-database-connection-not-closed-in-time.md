---
title: javaee 配置数据源后数据库连接未及时关闭出现的问题
categories: Java
tags: java
author: Mingshan
date: 2017-06-06
---
我在配置要数据源后没有仔细检查我的代码，有些数据库连接没有及时关闭，报以下异常:

>org.apache.tomcat.dbcp.dbcp.SQLNestedException: Cannot get a connection, pool error Timeout waiting for idle objec

这个异常产生的原因是在使用完数据库连接后没有及时关闭，导致数据库连接池的连接没有可供使用的连接，进而报异常。
解决的方法是检查代码，将数据库连接及时关闭，并且在context.xml文件中加上

> removeAbandoned="true" removeAbandonedTimeout="60"
logAbandoned="true"

这样就解决问题了。
