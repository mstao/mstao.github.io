---
title: HikariPool校验连接异常排查
tags: [Hikari, MySQL]
author: Mingshan
categories: [Java, Hikari]
date: 2021-07-02
---

最近在本地控制台经常可以看如下异常：
```
HikariPool-1 - Failed to validate connection com.mysql.cj.jdbc.ConnectionImpl@7d2b135f (No operations allowed after connection closed.).
Possibly consider using a shorter maxLifetime value.
```

<!-- more -->

# 复现问题


该异常出现的原因是什么？我们需要从源码来看下。


找到报这段异常的源码，位置在：com.zaxxer.hikari.pool 包下的PoolBase类的isConnectionAlive方法，源码如下：
```java
   boolean isConnectionAlive(final Connection connection)
   {
      try {
         try {
            setNetworkTimeout(connection, validationTimeout);

            final int validationSeconds = (int) Math.max(1000L, validationTimeout) / 1000;

            if (isUseJdbc4Validation) {
               return connection.isValid(validationSeconds);
            }

            try (Statement statement = connection.createStatement()) {
               if (isNetworkTimeoutSupported != TRUE) {
                  setQueryTimeout(statement, validationSeconds);
               }

               statement.execute(config.getConnectionTestQuery());
            }
         }
         finally {
            setNetworkTimeout(connection, networkTimeout);

            if (isIsolateInternalQueries && !isAutoCommit) {
               connection.rollback();
            }
         }

         return true;
      }
      catch (Exception e) {
         lastConnectionFailure.set(e);
         logger.warn("{} - Failed to validate connection {} ({}). Possibly consider using a shorter maxLifetime value.",
                     poolName, connection, e.getMessage());
         return false;
      }
   }
```
从上面代码可以看出，在校验连接是否存活时发生了异常，我们打断点跟踪下，可以看到会发生如下异常：No operations allowed after connection closed
![image.png](https://cdn.nlark.com/yuque/0/2021/png/189999/1625215708969-33448e13-a530-49bc-8e8e-7b3a37b2fa08.png#align=left&display=inline&height=625&margin=%5Bobject%20Object%5D&name=image.png&originHeight=625&originWidth=1430&size=217472&status=done&style=none&width=1430)


从上面的堆栈信息来看，在设置连接的超时时间发生了异常：
```java
private void setNetworkTimeout(final Connection connection, final long timeoutMs) throws SQLException
{
   if (isNetworkTimeoutSupported == TRUE) {
      connection.setNetworkTimeout(netTimeoutExecutor, (int) timeoutMs);
   }
}
```
其实我们看到 `No operations allowed after connection closed` 就应该猜测到怎么回事了，一旦出现这个异常，就是说明当前我们从数据库连接池拿到的连接已经被MySQL关闭了，为什么会这样呢？原因是我们在数据库连接池设置连接的最大存活时间超过了MySQL 中设置的超时时间，就会出现这个问题。


在Hikari中，默认连接最大存活时间为30分钟，即`maxLifetime` 属性。
查看下MySQL 的 `wait_timeout` 字段，`show variables like '%wait_timeout%` ，目前是300秒。


明显30分钟大于300秒，所以就出现了上面的问题。


# 解决方式


根据上面的分析，我们知道需要知道有如下两种方式：


1. 增加 wait_timeout 的时间。
1. 减少 数据库连接池中 connection 的 maxLifeTime。



针对Hikari在SpringBoot中配置如下：
```yaml
spring:
  ########-spring datasource-########
  datasource:
     #账号配置
     url: xx
     username: xx
     password: xx
     driver-class-name: com.mysql.cj.jdbc.Driver
     #hikari数据库连接池
     hikari:
      pool-name: Retail_HikariCP
      minimum-idle: 5 #最小空闲连接数量
      idle-timeout: 180000 #空闲连接存活最大时间，默认600000（10分钟）
      maximum-pool-size: 10 #连接池最大连接数，默认是10
      auto-commit: true  #此属性控制从池返回的连接的默认自动提交行为,默认值：true
      max-lifetime: 1800000 #此属性控制池中连接的最长生命周期，值0表示无限生命周期，默认1800000即30分钟
      connection-timeout: 30000 #数据库连接超时时间,默认30秒，即30000
```
# 参考：

- [https://www.cnblogs.com/aeolian/p/12517841.html](https://www.cnblogs.com/aeolian/p/12517841.html)
- [https://docs.spring.io/spring-boot/docs/2.3.12.RELEASE/reference/html/appendix-application-properties.html#common-application-properties-data-migration](https://docs.spring.io/spring-boot/docs/2.3.12.RELEASE/reference/html/appendix-application-properties.html#common-application-properties-data-migration)



