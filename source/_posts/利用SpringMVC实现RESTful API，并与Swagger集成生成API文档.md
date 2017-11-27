---
title: 利用SpringMVC实现RESTful API，并与Swagger集成生成API文档
categories: Java
tags: [java, Swagger, SpringMVC, RESTful-API]
author: Mingshan
---
## 认识RESTful API
RESTful API是目前比较成熟的API设计理论，它通过统一的API接口来对外提供服务，这样对其他调用者来说比较友好，更加容易实现前后端分离。那么如果要使用RESTful API来写我们的代码，那么就需要先知道RESTful API规范。
## 参考RESTful API规范
下面是两篇文章讲解RESTful API的，推荐：
1. [RESTful API 设计指南](http://www.ruanyifeng.com/blog/2014/05/restful_api.html)
2. [RESTful API 设计最佳实践](http://www.csdn.net/article/2013-06-13/2815744-RESTful-API)

## SpringMVC实现RESTful API
SpringMVC提供了一些注解来实现RESTful API, 例如**@RestController**，同时我们用Swagger来生成API文档，这样更加利于测试API。
### 常见swagger注解一览与使用
**最常用的5个注解**

@Api：修饰整个类，描述Controller的作用
@ApiOperation：描述一个类的一个方法，或者说一个接口
@ApiParam：单个参数描述
@ApiModel：用对象来接收参数
@ApiProperty：用对象接收参数时，描述对象的一个字段

**其它若干**

@ApiResponse：HTTP响应其中1个描述
@ApiResponses：HTTP响应整体描述
@ApiClass
@ApiError
@ApiErrors
@ApiParamImplicit
@ApiParamsImplicit

**其中@ApiOperation和@ApiParam参数说明**

@ApiOperation和@ApiParam为添加的API相关注解，参数说明如下：
@ApiOperation(value = “接口说明”, httpMethod = “接口请求方式”, response = “接口返回参数类型”, notes = “接口发布说明”；其他参数可参考源码；
@ApiParam(required = “是否必须参数”, name = “参数名称”, value = “参数具体描述”

### 添加依赖
首先在pom.xml文件中添加swagger依赖

```xml
<!-- swagger -->
<dependency>
  <groupId>com.mangofactory</groupId>
  <artifactId>swagger-springmvc</artifactId>
  <version>1.0.2</version>
</dependency>
<dependency>
  <groupId>com.fasterxml.jackson.core</groupId>
  <artifactId>jackson-core</artifactId>
  <version>2.5.1</version>
</dependency>
<dependency>
  <groupId>com.fasterxml.jackson.core</groupId>
  <artifactId>jackson-databind</artifactId>
  <version>2.5.1</version>
</dependency>
<dependency>
  <groupId>com.fasterxml.jackson.core</groupId>
  <artifactId>jackson-annotations</artifactId>
  <version>2.5.1</version>
</dependency>
```
### Swagger-UI配置
首先从[Swagger-UI下载地址](https://github.com/swagger-api/swagger-ui)下载Swagger-UI文件，然后将其拷贝到webapp目录下，我这里新建了一个swagger文件夹，然后解压后的文件拷贝到这个文件夹里面了。

修改swagger/index.html文件，默认是从连接http://petstore.swagger.io/v2/swagger.json获取 API 的JSON，这里需要将url值修改为http://{ip}:{port}/{projectName}/api-docs的形式，{}中的值根据自身情况填写。比如我的url值为：http://localhost:8080/lightblog/api-docs

### 编写swagger配置文件
配置完Swagger-UI后，我们需要配置Swagger，并将其交给Spring进行管理。
<!-- more -->
SwaggerConfig类代码如下：

```java
package com.lightblog.swagger;

import com.mangofactory.swagger.configuration.SpringSwaggerConfig;
import com.mangofactory.swagger.models.dto.ApiInfo;
import com.mangofactory.swagger.plugin.EnableSwagger;
import com.mangofactory.swagger.plugin.SwaggerSpringMvcPlugin;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.ComponentScan;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.servlet.config.annotation.EnableWebMvc;

/**
 * @Description:
 * @Author: Minsghan
 * @Date: Created in 16:40 2017/10/3
 * @Modified By:
 */
@Configuration
@EnableWebMvc//如果没加这个会报错
@EnableSwagger//上面三个注释都是必要的
@ComponentScan(basePackages="com.lightblog.controller")//添加这个注释，会自动扫描该类中的每一个方法自动生成api文档
public class SwaggerConfig {
    private SpringSwaggerConfig springSwaggerConfig;

    /**
     * Required to autowire SpringSwaggerConfig
     */
    @Autowired
    public void setSpringSwaggerConfig(SpringSwaggerConfig springSwaggerConfig)
    {
        this.springSwaggerConfig = springSwaggerConfig;
    }

    /**
     * Every SwaggerSpringMvcPlugin bean is picked up by the swagger-mvc
     * framework - allowing for multiple swagger groups i.e. same code base
     * multiple swagger resource listings.
     */
    @Bean
    public SwaggerSpringMvcPlugin customImplementation()
    {
        return new SwaggerSpringMvcPlugin(this.springSwaggerConfig)
                .apiInfo(apiInfo())
                .includePatterns(".*?");
    }

    private ApiInfo apiInfo()
    {
        ApiInfo apiInfo = new ApiInfo(
                "springmvc搭建swagger",
                "spring-API swagger测试",
                "My Apps API terms of service",
                "499445428@qq.com",
                "web app",
                "My Apps API License URL");
        return apiInfo;
    }
}

```

将 springSwaggerConfig加载到spring容器，配置如下：

```
<!-- 将 springSwaggerConfig加载到spring容器 -->
<bean class="com.mangofactory.swagger.configuration.SpringSwaggerConfig" />
<!-- 将自定义的swagger配置类加载到spring容器 -->
<bean class="com.lightblog.swagger.SwaggerConfig" />

```
### Controller实现REST API以及与Swagger集成

在Controller中，我们不需要返回页面了，而是要返回json格式的数据

```java
package com.lightblog.controller;

import com.lightblog.model.User;
import com.lightblog.service.UserService;
import com.wordnik.swagger.annotations.Api;
import com.wordnik.swagger.annotations.ApiOperation;
import com.wordnik.swagger.annotations.ApiParam;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.http.HttpHeaders;
import org.springframework.http.HttpStatus;
import org.springframework.http.MediaType;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;
import org.springframework.web.util.UriComponentsBuilder;

import java.util.List;

/**
 * @Description: The api of user.
 * @Author: Minsghan
 * @Date: Created in 15:27 2017/10/3
 * @Modified By:
 */
@Api(value="user")
@RestController
@RequestMapping("/api/user")
public class UserController extends BaseController {
    @Autowired
    private UserService userService;

    /**
     * @Author: Mingshan
     * @Description: Get list of user.
     * @param:  null
     * @Date: 15:16 2017/10/3
     */
    @RequestMapping(value = "", method = RequestMethod.GET)
    @ApiOperation(value="获取所有用户信息", httpMethod="GET", notes="Get users", response=ResponseEntity.class)
    public ResponseEntity<List<User>> listAllUsers() {
        List<User> users = userService.findAll();
        if(users.isEmpty()){
            // You many decide to return HttpStatus.NOT_FOUND
            return new ResponseEntity<List<User>>(HttpStatus.NO_CONTENT);
        }

        return new ResponseEntity<List<User>>(users, HttpStatus.OK);
    }

    /**
     * @Author: Mingshan
     * @Description: Get information of user by id.
     * @param:  * @param id
     * @Date: 15:34 2017/10/3
     */
    @RequestMapping(value = "/{id}", method = RequestMethod.GET, produces = MediaType.APPLICATION_JSON_VALUE)
    @ApiOperation(value="获取用户信息", httpMethod="GET", notes="Get user by id", response=User.class)
    public ResponseEntity<User> getUser(@ApiParam(required=true,value="用户ID",name="id")@PathVariable("id") long id) {
        logger.info("Fetching User with id " + id);
        User user = userService.findById(id);
        if (user == null) {
            logger.info("User with id " + id + " not found");
            return new ResponseEntity<User>(HttpStatus.NOT_FOUND);
        }
        return new ResponseEntity<User>(user, HttpStatus.OK);
    }

    /**
     * @Author: Mingshan
     * @Description: Create a user.
     * @param:  * @param null
     * @Date: 15:34 2017/10/3
     */
    @RequestMapping(value = "", method = RequestMethod.POST)
    @ApiOperation(value="新增用户", httpMethod="POST", notes="Create user", response=ResponseEntity.class)
    public ResponseEntity<Void> createUser(@ApiParam(required=true,value="用户信息",name="User")
                                               @RequestBody User user, UriComponentsBuilder ucBuilder) {
        logger.info("Creating User " + user.getName());

        if (userService.isUserExist(user)) {
            System.out.println("A User with name " + user.getName() + " already exist");
            return new ResponseEntity<Void>(HttpStatus.CONFLICT);
        }

        userService.insert(user);

        HttpHeaders headers = new HttpHeaders();
        headers.setLocation(ucBuilder.path("/user/{id}").buildAndExpand(user.getId()).toUri());
        return new ResponseEntity<Void>(headers, HttpStatus.CREATED);
    }

    /**
     * @Author: Mingshan
     * @Description: Update a user.
     * @param:  * @param null
     * @Date: 15:33 2017/10/3
     */
    @RequestMapping(value = "/{id}", method = RequestMethod.PUT)
    @ApiOperation(value="更新用户信息", httpMethod="PUT", notes="Update user", response=User.class)
    public ResponseEntity<User> updateUser(@ApiParam(required=true,value="用户ID",name="id")@PathVariable("id") long id,
                                           @RequestBody User user) {
        logger.info("Updating User " + id);

        User currentUser = userService.findById(id);

        if (currentUser == null) {
            logger.info("User with id " + id + " not found");
            return new ResponseEntity<User>(HttpStatus.NOT_FOUND);
        }

        currentUser.setName(user.getName());
        currentUser.setAge(user.getAge());

        userService.update(currentUser);
        return new ResponseEntity<User>(currentUser, HttpStatus.OK);
    }

    /**
     * @Author: Mingshan
     * @Description: Delete a user by id.
     * @param:  * @param null
     * @Date: 15:32 2017/10/3
     */
    @RequestMapping(value = "/{id}", method = RequestMethod.DELETE)
    @ApiOperation(value="删除用户", httpMethod="DELETE", notes="Delete user by id", response=ResponseEntity.class)
    public ResponseEntity<Void> deleteUser(@ApiParam(required=true,value="用户ID",name="id")@PathVariable("id") long id) {
        logger.info("Fetching & Deleting User with id " + id);

        User user = userService.findById(id);
        if (user == null) {
            logger.info("Unable to delete. User with id " + id + " not found");
            return new ResponseEntity<Void>(HttpStatus.NOT_FOUND);
        }

        userService.delete(id);
        return new ResponseEntity<Void>(HttpStatus.NO_CONTENT);
    }
}

```
### 运行截图
![image](https://ip.freep.cn/590836/snipaste20171008_163312.png)
![image](https://ip.freep.cn/590836/snipaste20171008_163351.png)

### 参考
1. http://blog.csdn.net/fansunion/article/details/51923720
2. http://blog.csdn.net/w605283073/article/details/51338765

官网：http://swagger.io/

GitHub：

swagger-springmvc:https://github.com/martypitt/swagger-springmvc

swagger-ui:https://github.com/swagger-api/swagger-ui

swagger-core:https://github.com/swagger-api/swagger-core

swagger-spec：https://github.com/swagger-api/swagger-spec
