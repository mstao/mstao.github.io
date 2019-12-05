---
title: 构建一个Tomcat的Docker镜像
tags: [Docker]
author: Mingshan
categories: Docker
date: 2018-7-28
---

最近在看Docker，对一个事物由陌生到熟悉需要一个过程，而这个过程需要从动手实践开始，下面记录一下我从零开始构建centos7+jdk8+tomcat8.5的docker镜像。

<!-- more -->

![image](https://github.com/mstao/static/blob/master/blog/docker.png?raw=true)

## CentOS7 Docker 安装

Docker 要求 CentOS 系统的内核版本高于 3.10 ，查看本页面的前提条件来验证你的CentOS 版本是否支持 Docker。
通过 uname -r 命令查看你当前的内核版本 

利用yum安装docker

```
yum -y install docker-io
```

安装完成后，启动docker服务


```
service docker start
```

测试一下

```
docker run hello-world
```

## 准备文件和构建环境

本次需要构建Tomcat的Docker镜像，所以需要Tomcat和JDK安装包，版本分别是


```
apache-tomcat-8.5.32.tar.gz

jdk-8u172-linux-x64.tar.gz
```

创建`/usr/docker-tomcat`文件夹，将以上两个文件解压到该文件夹中，分别重命名为tomcat和jdk文件夹。

创建`Dockerfile`和`run.sh`两个文件，最终`/usr/docker-tomcat`文件夹内容如下图所示：

![image](https://github.com/mstao/static/blob/master/blog/docker-tomcat-folder.png?raw=true)

编辑`Dockerfile`文件


```
vim Dockerfile
```
向该文件添加如下内容：

```
# 设置继承的镜像
FROM registry.cn-hangzhou.aliyuncs.com/repos_zyl/centos:0.0.1

# 创建者信息
MAINTAINER mingshan "walkerhan@126.com"

# 设置环境变量，所有操作都是非交互式的
ENV DEBIAN_FRONTEND noninteractive

# 设置tomcat的环境变量
ENV CATALINA_HOME /tomcat
ENV JAVA_HOME /jdk

# 复制tomcat和jdk文件到镜像中
ADD apache-tomcat /tomcat
ADD jdk /jdk

# 复制启动脚本至镜像，并赋予脚本可执行权限
ADD run.sh /run.sh
RUN chmod +x /*.sh
RUN chmod +x /tomcat/bin/*.sh

# 暴露接口8080，这是我的tomcat接口，默认为8080
EXPOSE 8080

# 设置自启动命令
CMD ["/run.sh"]
```
保存后，然后编辑`run.sh`文件

```
vim run.sh
```
添加如下内容：

```
#!/bin/bash
# 启动tomcat
exec ${CATALINA_HOME}/bin/catalina.sh run
```

保存后接下来就开始构建docker镜像文件

## 构建docker镜像文件

我们可以利用`docker build`来构建Docker镜像，
-t 设置tag名称, 命名规则registry/image:tag
. 表示使用当前目录下的Dockerfile文件

```
docker build -t tomcat:test . 
```

执行该命令后，Docker就会按照Dockerfile文件顺序执行，会有很多步骤，下面是输出日志：
 
```
Sending build context to Docker daemon 403.1 MB
Step 1/12 : FROM registry.cn-hangzhou.aliyuncs.com/repos_zyl/centos:0.0.1
 ---> e1e65df66640
Step 2/12 : MAINTAINER mingshan "walkerhan@126.com"
 ---> Using cache
 ---> f030a7c09868
Step 3/12 : ENV DEBIAN_FRONTEND noninteractive
 ---> Using cache
 ---> ef3f61db3034
Step 4/12 : ENV CATALINA_HOME /tomcat
 ---> Running in 5145fe0de0d1
 ---> 9a4af98c3434
Removing intermediate container 5145fe0de0d1
Step 5/12 : ENV JAVA_HOME /jdk
 ---> Running in 8bac05b87945
 ---> f5b7eb8d180e
Removing intermediate container 8bac05b87945
Step 6/12 : ADD tomcat /tomcat
 ---> d68a1754c19e
Removing intermediate container 29f7625a6b95
Step 7/12 : ADD jdk /jdk
 ---> df1669bd68de
Removing intermediate container dc7de6c936fa
Step 8/12 : ADD run.sh /run.sh
 ---> 90a13284ec01
Removing intermediate container 5bf2c6666a03
Step 9/12 : RUN chmod +x /*.sh
 ---> Running in 7f22e7927ffb

 ---> 10fff91bfa16
Removing intermediate container 7f22e7927ffb
Step 10/12 : RUN chmod +x /tomcat/bin/*.sh
 ---> Running in 163f103fcc0a

 ---> da593ceeaa49
Removing intermediate container 163f103fcc0a
Step 11/12 : EXPOSE 8080
 ---> Running in d66d686928fe
 ---> e2d7f915335a
Removing intermediate container d66d686928fe
Step 12/12 : CMD /run.sh
 ---> Running in a7f9d1e9c9a0
 ---> a21030de3aac
Removing intermediate container a7f9d1e9c9a0
Successfully built a21030de3aac
```

构建完成后，就可以启动容器啦

### 启动容器

我们利用`docker run`来新建并启动容器，这个命令有很多参数，-d: 后台运行容器，并返回容器ID；-p: 端口映射，格式为：主机(宿主)端口:容器端口


```
docker run -d -p 11001:8080 tomcat:test
```

然后使用 `docker ps` 命令查看正在运行的容器，如下图所示：

![image](https://github.com/mstao/static/blob/master/blog/docker-tomcat-ps.png?raw=true)

### 开启端口

如果让外网访问的话，需要开放11001端口，在CentOS7中，可以利用firewall-cmd来开放端口


```
#开放11001端口  permanent为永久开放
firewall-cmd --zone=public --add-port=11001/tcp --permanent
#重新读取配置
firewall-cmd --reload
#查看全部开放端口
firewall-cmd --list-all
```

接下来就可在浏览器中看到Tomcat界面了，大功告成！

![image](https://github.com/mstao/static/blob/master/blog/docke-tomcat-showpage.png?raw=true)

### 镜像保存为本地离线文件

将docker image保存为离线的本地文件，执行`docker save image_name > ./save.tar `或者 `docker save -o filepath image_name`


```
[root@VM_0_6_centos docker-tocmat]# docker images
REPOSITORY                                           TAG                 IMAGE ID            CREATED             SIZE
tomcat                                               test                a21030de3aac        2 hours ago         593 MB
registry.cn-hangzhou.aliyuncs.com/repos_zyl/centos   0.0.1               e1e65df66640        19 months ago       192 MB
[root@VM_0_6_centos docker-tocmat]# docker save a21030de3aac > ./up_tomcat.tar
[root@VM_0_6_centos docker-tocmat]# ls
Dockerfile  jdk  run.sh  tomcat  up_tomcat.tar
```


### 其他操作

1. 停止容器


```
docker stop tomcat:test 
```

2. 启动容器

```
docker start tomcat:test 
```

3. 删除容器


删除一个容器：
```
docker rm 278636f577a6 
```

停用全部运行中的容器:

```
docker stop $(docker ps -q)
```

删除全部容器：

```
docker rm $(docker ps -aq)
```

一条命令实现停用并删除容器：

```
docker stop $(docker ps -q) & docker rm $(docker ps -aq)
```

4. 删除镜像

删除镜像需要先删除使用该镜像的容器，然后再删除镜像


```
docker rmi 3ae3626adcfa
```

### 参考

- https://www.jianshu.com/p/369e75f6303b
- https://www.cnblogs.com/zhouyalei/p/6390963.html
- http://www.dockerinfo.net/docker%e5%ae%b9%e5%99%a8-2