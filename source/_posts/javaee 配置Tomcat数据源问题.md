---
title: javaee 配置Tomcat数据源问题
categories: Java
tags: java
author: Mingshan
---
最近用javaee写网站配置数据源时遇到了这个错误:

> java.lang.ClassCastException: org.apache.tomcat.dbcp.dbcp.PoolingDataSource$PoolGuardConnectionWrapper cannot be cast to com.mysql.jdbc.Connection

经过我查看代码发现有些类中包导错了，涉及到数据库的包应该导入java.sql.*这个相关的，而我用ide自动导入为jdbc那个了，发生了类型不匹配问题，改掉就不会报这个错了。
