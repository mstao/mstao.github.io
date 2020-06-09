---
title: SpringBoot项目添加外部Jar包及配置多数据源
tags: [java,SpringBoot]
author: Mingshan
categories: Java
date: 2020-06-09
---

最近项目需要和Oracle数据库进行交互，然后我从Maven中央仓库下载数据库驱动jar包，但怎么都下不下来，我到Oracle官网上一看，我去，居然不让用Maven直接下（大学时候用过Oracle，很久远的事情了0rz），没办法我还是直接下载jar包放到我的项目里面吧。SpringBoot项目引入外部jar包是非常方便的，包含打引入外部jar等操作。

<!-- more -->

我的做法如下：

1. 首先在src同级目录建一个`lib`文件夹，将第三方jar包放到这个文件内，比如我将`ojdbc6.jar` 这个jar包放到这个地方。
2. 接着我们需要在`pom.xml`文件里配置jar的maven坐标，不过这个坐标比较特殊，我们需要直接定位到我们上一步添加的文件，而不是从Maven仓库里面去下载，以`ojdbc6.jar`为例，配置依赖如下：

```
<dependency>
	<groupId>com.oracle</groupId>
	<artifactId>ojdbc6</artifactId>
	<version>12.1.0.2.0</version>
	<scope>system</scope>
	<systemPath>${project.basedir}/lib/ojdbc6.jar</systemPath>
</dependency>
```

这里比较特殊的是`systemPath`，常见的Maven坐标是没有这个的，这里面直接指定该jar的相对路径（相对项目的根目录），这样Maven在编译的时候就不会从中央仓库里面去下载该jar包了。但只配置这个还不行，还需要配置SpringBoot编译时插件属性`includeSystemScope`，具体如下：

```
<plugin>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-maven-plugin</artifactId>
	<configuration>
		<includeSystemScope>true</includeSystemScope>
	</configuration>
</plugin>
```

上面配置完毕，我们就可以直接执行`mvn clean install`进行打包，然后我们查看打好的jar包里面包含的jar包，会发现`ojdbc6.jar`这个包已经正确被包含进去了。

由于对接的项目比较老，要与其数据库进行交互，而且数据库类型不一致，所以我们的项目需要支持多数据源（接口平台），这个还是非常好配置的，SpringBoot给我们提供了多数据源配置的方案，并且每个数据源对应一个JdbcTemplate，这样就方便很多，具体配置如下：

**application.properties文件内配置多数据源信息**

首先在application.properties或者yml文件内配置多数据源信息，具体配置如下：

```
# ds1数据源配置
spring.datasource.ds1.type=com.alibaba.druid.pool.DruidDataSource
spring.datasource.ds1.driver-class-name=com.mysql.jdbc.Driver
spring.datasource.ds1.url=jdbc:mysql://localhost:3306/zz?useUnicode=true&characterEncoding=utf8
spring.datasource.ds1.username=zz
spring.datasource.ds1.password=zz

# ds2数据源配置
spring.datasource.ds2.driver-class-name=oracle.jdbc.driver.OracleDriver
spring.datasource.ds2.url=jdbc:oracle:thin:@localhost:1521:orcl
spring.datasource.ds2.username=system
spring.datasource.ds2.password=050508
```

**指定数据源与配置信息**

上面我们配置好了数据源，但是已经不是SpringBoot默认的数据源配置信息了，所以我们还要指定不同的数据源实例对应哪个配置信息，配置如下：


```Java
/**
 * 多数据源配置
 */
@Configuration
public class DataSourceConfig {

  /**
   * ds1数据源配置
   *
   * @return 配置信息
   */
  @Primary
  @Bean(name = "ds1DataSourceProperties")
  @ConfigurationProperties(prefix = "spring.datasource.ds1")
  public DataSourceProperties ds1DataSourceProperties() {
    return new DataSourceProperties();
  }

  /**
   * ds1数据源
   *
   * @param dataSourceProperties 配置信息
   * @return 数据源实例
   */
  @Primary
  @Bean(name = "ds1DataSource")
  public DataSource ds1DataSource(@Qualifier("ds1DataSourceProperties") DataSourceProperties dataSourceProperties) {
    return dataSourceProperties.initializeDataSourceBuilder().build();
  }

  /**
   * ds2数据源配置
   *
   * @return 配置信息
   */
  @Primary
  @Bean(name = "ds2DataSourceProperties")
  @ConfigurationProperties(prefix = "spring.datasource.ds2")
  public DataSourceProperties ds2DataSourceProperties() {
    return new DataSourceProperties();
  }

  /**
   * ds2数据源
   *
   * @param dataSourceProperties 配置信息
   * @return 数据源实例
   */
  @Primary
  @Bean(name = "ds2DataSource")
  public DataSource ds2DataSource(@Qualifier("ds2DataSourceProperties") DataSourceProperties dataSourceProperties) {
    return dataSourceProperties.initializeDataSourceBuilder().build();
  }

}
```

**配置JdbcTemplate与数据源关系**

配置完数据源信息，我们想直接用不同的JdbcTemplate来操作不同的数据库，所以我们还要创建几个`JdbcTemplate`实例，并且这些实例与不同的数据源进行绑定，配置信息如下：

```Java
/**
 * JdbcTemplate 多数据源配置
 *
 * @author 明山
 * @see DataSourceConfig
 */
@Configuration
public class JdbcTemplateDataSourceConfig {

  /**
   * ds1 JdbcTemplate 配置
   *
   * @param dataSource 数据源
   * @return JdbcTemplate
   */
  @Primary
  @Bean(name = "ds1JdbcTemplate")
  public JdbcTemplate jdbcTemplate(@Qualifier("ds1DataSource") DataSource dataSource) {
    return new JdbcTemplate(dataSource);
  }

  /**
   *ds2 JdbcTemplate 配置
   *
   * @param dataSource 数据源
   * @return JdbcTemplate
   */
  @Bean(name = "ds2JdbcTemplate")
  public JdbcTemplate hdwmsJdbcTemplate(@Qualifier("ds2DataSource") DataSource dataSource) {
    return new JdbcTemplate(dataSource);
  }
}
```

**使用JdbcTemplate**

配置完后，我们可以直接在具体的类中使用了，使用方式如下：


```Java
@Autowired
@Qualifier("ds1JdbcTemplate")
private JdbcTemplate ds1JdbcTemplate;

@Autowired
@Qualifier("ds2JdbcTemplate")
private JdbcTemplate ds2JdbcTemplate;
```
