---
title: 编写一个自己的SpringBoot Starter
tags: [java,SpringBoot]
author: Mingshan
categories: Java
date: 2019-6-26
---

我们用SpringBoot的时候发现有很多starter，比如`spring-boot-starter-web`等，对于SpringBoot的官方starter，基本上是以`spring-boot-starter-xxx`来命名的，对于非官方的一些包来说，我们该怎样将自己的包与SpringBoot结合起来呢？在SpringBoot的官方文档中，有这样一章，[Creating Your Own Starter](https://docs.spring.io/spring-boot/docs/2.1.6.RELEASE/reference/html/boot-features-developing-auto-configuration.html#boot-features-custom-starter)，来教我们如何编写自己的starter。

<!-- more -->

## 理解`@Conditional`

我们可以控制一个类在满足某一条件才能进行实例化吗？
在Spring中有一个`@Conditional`注解和`Condition`接口，这个接口有一个`matches`方法，使用者可以重写这个方法，如下所示：

```Java
public class TestCondition implements Condition {

    @Override
    public boolean matches(ConditionContext context, AnnotatedTypeMetadata metadata) {
        return true;
    }
}
```

然后我们就可以把`@Conditional(value = TestCondition.class)`标注在类上面，表示该类下面的所有@Bean都会启用配置，也可以标注在方法上面，只是对该方法启用配置。

在[spring-boot-autoconfigure](https://github.com/spring-projects/spring-boot/tree/v2.1.6.RELEASE/spring-boot-project/spring-boot-autoconfigure/src/main/java/org/springframework/boot/autoconfigure/condition)中，SpringBoot官方实现了一系列Condition，用来方便平时的开发，这些实现类都位于`org.springframework.boot.autoconfigure.condition`，并且每个都提供了对应的注解，下面相关注解及其说明：

注解 | 说明
---|---
@ConditionalOnBean | 在BeanFactory已经存在指定Bean
@ConditionalOnMissingBean | 在BeanFactory不存在指定Bean
@ConditionalOnClass | 在classpath下已存在指定class
@ConditionalOnMissingClass | 在classpath下不存在指定class
@ConditionalOnCloudPlatform | 指定的云平台已生效（Cloud Foundry platform，Heroku platform，SAP Cloud platform）
@ConditionalOnExpression | 指定的SPEL表达式为true时
@ConditionalOnJava | 指定的Java版本（前或者后）
@ConditionalOnJndi | 指定的JNDI存在
@ConditionalOnNotWebApplication | 非web应用
@ConditionalOnWebApplication | web环境
@ConditionalOnProperty | 指定的property有指定的值
@ConditionalOnResource | 在classpath下存在指定的资源
@ConditionalOnSingleCandidate | BeanFactory中该类型Bean只有一个或@Primary的只有一个时


## 基本原理

用过SPI机制的同学可能会清楚一个概念，当一个框架需要动态的扩展能力，给使用者给予充分的扩展能力，那么可能会用到SPI机制。在SpringBoot中，如果我们编写了一个Starter，SpringBoot框架怎么会识别我们的项目呢？所以我们就需要告诉SpringBoot，这里是我写的Starter，你运行时加载吧，类似SPI。SpringBoot正好提供了该功能，被称为`Auto-configuration`，下面是SpringBoot的官方文档：

> Spring Boot checks for the presence of a META-INF/spring.factories file within your published jar. The file should list your configuration classes under the EnableAutoConfiguration key, as shown in the following example:

```
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
com.mycorp.libx.autoconfigure.LibXAutoConfiguration,\
com.mycorp.libx.autoconfigure.LibXWebAutoConfiguration
```

从上面可以看出，只要我们在我们自己的Stater的`META-INF/spring.factories`中配置好AutoConfiguration，SpringBoot就可以检测到并且加载啦。这个AutoConfiguration需要用`@Configuration`标识，代表它是一个配置类，并且结合上面提到的`@Conditional`及其相关注解，灵活地读取相应配置信息。

## 实现步骤

SpringBoot官方推荐自定义Starter以`xxx-spring-boot-starter`命名，并且分两个模块，`Autoconfigure模块`和`Starter模块`，主要配置在`Autoconfigure模块`，`Starter模块`是一个空项目。首先在父项目的pom文件加入SpringBoot依赖

```
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-dependencies</artifactId>
    <version>${spring-boot.version}</version>
    <type>pom</type>
    <scope>import</scope>
</dependency>
```

### Autoconfigure模块

Autoconfigure模块加入如下依赖：

```
<!-- Compile dependencies -->
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-autoconfigure</artifactId>
</dependency>

<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-configuration-processor</artifactId>
  <optional>true</optional>
</dependency>
```

其中`spring-boot-configuration-processor`主要是用来生成元数据文件(META-INF/spring-configuration-metadata.json )。如果存在该文件，那么SpringBoot会优先读取该元数据文件，提高启动速度。

**编写自定义Service**

我们编写一个测试Service，其中config是从配置文件读取的：

```Java
public class StarterService {
    private String config;

    public StarterService(String config) {
        this.config = config;
    }

    public String[] split(String separatorChar) {
        return StringUtils.split(this.config, separatorChar);
    }
}
```

**编写配置文件读取类**

我们用SpringBoot时，会在`application.yml`中编写一些配置，如果我们要编写自己的配置，那么肯定要读取的，下面这个类用来读取配置文件信息：

```Java
@ConfigurationProperties("example.service")
public class StarterServiceProperties {
    private boolean enable;
    private String config;

    public boolean isEnable() {
        return enable;
    }

    public void setEnable(boolean enable) {
        this.enable = enable;
    }

    public String getConfig() {
        return config;
    }

    public void setConfig(String config) {
        this.config = config;
    }
}
```

**编写AutoConfigure**

接下来编写AutoConfigure文件。`@ConditionalOnClass`代表classpath 中有StarterService这个class，`@EnableConfigurationProperties`代表启用`StarterServiceProperties`这个配置类，`@ConditionalOnMissingBean`代表BeanFactory中没有StarterService这个Bean才实例化，`@ConditionalOnProperty`根据指定的配置文件指定的值来判断是否加载该Bean。代码如下：

```Java
@Configuration
@ConditionalOnClass(StarterService.class)
@EnableConfigurationProperties(StarterServiceProperties.class)
public class StarterAutoConfigure {

    @Autowired
    private StarterServiceProperties properties;

    @Bean
    @ConditionalOnMissingBean
    @ConditionalOnProperty(prefix = "example.service", value = "enable", havingValue = "true")
    StarterService starterService () {
        return new StarterService(properties.getConfig());
    }

}
```

**配置spring.factories**

最后在`META-INF`文件夹新建`spring.factories`中加入如下配置：

```
org.springframework.boot.autoconfigure.EnableAutoConfiguration=me.mingshan.spring.boot.autoconfigure.StarterAutoConfigure
```

如果有多个，以`,`隔开。

### Starter模块

Starter模块是个空的项目，只有一个pom文件，加入如下依赖：

```Java
  <dependencies>
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter</artifactId>
    </dependency>
    <dependency>
      <groupId>me.mingshan</groupId>
      <artifactId>demo-spring-boot-autoconfigure</artifactId>
      <version>${project.version}</version>
    </dependency>
  </dependencies>
```

### 测试

新建一个测试项目，将stater依赖引入，在application.yml文件加入如下配置：

```
example:
  service:
    enable: true
    config: qqq,www
```

新建一个Runner，实现`ApplicationRunner`接口，当我们启动SpringBoot容器时，就会执行下面该类，输出`qqq www`。

```Java
@Component
public class Runner implements ApplicationRunner {
    @Autowired
    private StarterService starterService;

    @Override
    public void run(ApplicationArguments args) throws Exception {
        String[] splitArray = starterService.split(",");
        for (String str : splitArray) {
            System.out.println(str);
        }
    }
}
```



## References：

- [Creating Your Own Starter](https://docs.spring.io/spring-boot/docs/2.1.6.RELEASE/reference/html/boot-features-developing-auto-configuration.html#boot-features-custom-starter)
- [编写自己的SpringBoot-starter](https://www.cnblogs.com/yuansc/p/9088212.html)