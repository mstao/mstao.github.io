---
title: Docker容器传递Spring Profile
tags: [Docker]
author: Mingshan
categories: Docker
date: 2018-8-22
---

我们在利用Docker启动SpringBoot应用时，需要在启动时指定Profile来确定运行环境，如果再需要指定其他参数，参数的传递就十分有必要了。下面是stackoverflow关于这方面的讨论，链接如下：

<!-- more -->

[Passing env variables to DOCKER Spring Boot](https://stackoverflow.com/questions/46715072/passing-env-variables-to-docker-spring-boot)


**通过Dockerfile定义Spring Profile**

通常在命令行中我们可以使用`java -jar` 运行 Spring Boot应用。
而Profiles信息可以作为额外参数传递，比如`-Dspring.profiles.active=dev`

```
java -Djava.security.egd=file:/dev/./urandom -Dspring.profiles.active=dev -jar app.jar
```

相似的，我们可以在Dockerfile中将Profile的信息作为参数传递进去，例如：

```
FROM openjdk:8-jdk-alpine
VOLUME /tmp
ADD spring-boot-docker-0.0.1-SNAPSHOT.jar app.jar
ENTRYPOINT ["java", "-Djava.security.egd=file:/dev/./urandom","-Dspring.profiles.active=dev","-jar","/app.jar"]
```

需要注意最后的ENTRYPOINT一行，在这行中我们传递java命令以执行jar文件，所有需要的参数和值以逗号方式分隔传递。
`-Dspring.profiles.active=dev` 是我们定义dev profile的地方，我们可以替换dev为任何需要的名字。

**通过Docker run命令定义Spring Profile**

可以将spring profile作为环境变量传递给docker run命令，使用 -e 标记。
例如` -e “SPRING_PROFILES_ACTIVE=dev”会将dev profile`传递给Docker容器

```
docker run -d -p 8080:8080 -e "SPRING_PROFILES_ACTIVE=dev" --name rest-api dockerImage:latest
```

Dockerfile中改为

```
ENTRYPOINT ["java", "-Djava.security.egd=file:/dev/./urandom","-Dspring.profiles.active=${SPRING_PROFILES_ACTIVE}","-jar","/app.jar"]
```


[<font size=3 color="#409EFF">向本文提出修改或勘误建议</font>](https://github.com/mstao/mstao.github.io/blob/hexo/source/_posts/docker-enables-spring-profiles.md)