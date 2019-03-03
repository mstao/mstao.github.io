---
title: Jenkins自动构建SpringBoot应用部署到Docker
tags: [Jenkins,SpringBoot,Docker]
author: Mingshan
categories: Docker
date: 2018-8-18
---

在开发SpringCloud微服务时，各个服务需要独立部署，手动部署比较麻烦，所以需要引入持续集成，Jenkins是一个非常好用的持续集成工具，十分好用。在使用Jenkins自动构建源码时，需要按照一定的步骤进行，步骤如下：

<!-- more -->

1. 从Git获取源代码
2. 代码审查
3. 编译代码
4. 打包文件
5. 部署到Docker

以上步骤我们手动也可以完成，但我们都喜欢自动化，自己点一点就能搞定，干嘛还要一步步来呢。

## 配置Jenkins环境
我们直接使用Docker安装Jenkins，我是直接安装Jenkins每日更新的镜像，[Jenkins地址](https://hub.docker.com/r/jenkinsci/jenkins/)，操作如下：

拉取镜像：

```
docker pull jenkinsci/jenkins
```

修改文件夹的归属者和组：

```
chown -R 1000:1000 jenkins_home/
```

启动Jenkins容器：

```
docker run -itd -p 8088:8080 -p 50000:50000 --name jenkins --privileged=true  -v /usr/jenkins_home:/var/jenkins_home docker.io/jenkinsci/jenkins
```

然后再配置一些个人信息和插件。

## 新建任务

首先我们需要新建Jenkins任务，选择Pipeline类型的任务，如下

![image](/images/jenkins_springbootdemo_start.png)

然后配置下git信息和构建参数信息

构建参数主要有以下几个：

- **BRANCH_NAME** 分支名称
- **ENV** 环境
- **TARGET_MACHINE_IP** 部署集群IP
- **SSH_PORT** 部署机器SSH端口
- **TARGET_PORT** 项目启动端口

说到SSH，不要忘了让jenkins服务器能够免密SSH访问目标机器，配置下authorized_key等信息。

接下来就需要编写Jenkinsfile文件了，这个文件需要放在项目主目录下，构建源码时Jenkins会读取该文件，按照该文件配置的步骤进行源码构建。

## 编写Jenkinsfile文件

在Jenkinsfile文件中，利用pipeline来编写步骤。pipeline支持将连续输送Pipeline实施和整合到Jenkins，所以我们可以将上面提到的流程写到pipeline中。这里主要分以下五部：

- git clone
- check code
- build code
- release package
- deploy

前两步就不提了，编译用Maven构建代码，release package这一步为打包代码，将一些脚本和编译好的jar文件打包成压缩文件，等待部署。deploy需要将打包的压缩文件拷贝到目标机器上，再执行部署脚本。

详细的Jenkinsfile如下：

```
pipeline {
    agent any

    environment {
        REPOSITORY="https://github.com/mstao/spring-boot-learning.git"
    }

    stages {
        stage('git clone') {
            steps {
                echo "<<< Starting fetch code from git:${REPOSITORY}"
                git credentialsId: "bd8b65ce-7fd2-4e57-a9fe-45d4fe2924bf", url: "${REPOSITORY}", branch: "$BRANCH_NAME"
            }
        }

        stage('check code') {
            steps {
                // chek code
                // Jenkins 集成 sonar 进行代码审查
                echo "<<<Starting check and analytic code"
            }
        }

        stage('build code') {
            steps {
                echo "<<< Starting build code and docker image"
                withMaven (
                    maven: 'Maven 3.5.4',
                    mavenLocalRepo: '.repository') {
                        sh 'mvn clean package -Dmaven.test.skip=true'
                    }
            }
        }

        stage('release package') {
            steps {
                echo "<<< release package to /var/jenkins_home/release/demo/$ENV/$BRANCH_NAME"
                sh '''
                    if [ -d target/demo ]
                        then
                        rm -rf target/demo
                    fi
                    mkdir -p target/demo
                '''
                sh '''
                    cp target/*.jar target/demo/demo-api.jar
                    cp deploy.sh target/demo/
                    cp Dockerfile target/demo/
                    cd target
                    tar zcvf demo.tar.gz demo
                    exit
                '''
                sh '''
                    if [ ! -d /var/jenkins_home/release/demo/$ENV/$BRANCH_NAME ]
                        then
                        mkdir -p /var/jenkins_home/release/demo/$ENV/$BRANCH_NAME
                    fi
                '''
                sh 'cp -f target/*.tar.gz /var/jenkins_home/release/demo/$ENV/$BRANCH_NAME'
            }
        }

        stage('deploy') {
            steps {
                echo "<<< Starting deploy code to docker"
                 script {
                    try {
                        sh '''
                            ssh -p $SSH_PORT -o StrictHostKeyChecking=no mingshan@$TARGET_MACHINE_IP rm -rf /app/tmp/demo/package
                            ssh -p $SSH_PORT mingshan@$TARGET_MACHINE_IP mkdir -p /app/tmp/demo/package
                        '''
                    }
                    catch(e) {
                        echo e
                    }
                }
                sh '''
                    scp -P $SSH_PORT /var/jenkins_home/release/demo/$ENV/$BRANCH_NAME/*.tar.gz mingshan@$TARGET_MACHINE_IP:/app/tmp/demo/package/
                    ssh -p $SSH_PORT mingshan@$TARGET_MACHINE_IP "tar -zxf /app/tmp/demo/package/demo.tar.gz -C /app/tmp/demo/package"
                    ssh -p $SSH_PORT mingshan@$TARGET_MACHINE_IP "cd /app/tmp/demo/package/demo && chmod +x * && sh deploy.sh $TARGET_PORT"
                    exit
                '''
                echo "deploy done"
            }
        }
    }
}
```


## 部署到Docker

由于需要将打包的文件部署到Docker，所以需要将部署脚本和压缩文件发送到目标机器，在目标机器上制作Docker镜像，然后运行Docker 容器。

由于Docker需要root用户进行运行，而对于一些机器而言，拿不到root权限，所以需要进行一些操作，参考：
[免sudo使用docker命令](https://blog.csdn.net/baidu_36342103/article/details/69357438)

在Jenkinsfile中deploy步骤中，执行了以下命令：

```shell
 ssh -p $SSH_PORT mingshan@$TARGET_MACHINE_IP "cd /app/tmp/demo/package/demo && chmod +x * && sh deploy.sh $TARGET_PORT"
```

这是到目标机器上执行deploy.sh脚本，该脚本的内容如下：

```
#!/bin/bash

port=11001
image=demo
name=demo-service

if [ -z $1 ]; then
    echo "<<< Using default port $port"
else
    echo "<<< Set port to $1"
    $port=$1
fi

echo "<<< Fetch container id of $name"
CID=$(docker ps | grep "demo-service" | awk '{print $1}')

if [ "$CID" != "" ];then
  echo "<<< Stop and remove old container for $name"
  docker stop $CID
  docker rm $CID
fi

echo "<<< Remove old image $image"
docker rmi $image

echo "<<< Start to build docker image $image"
docker build -t $image .

echo "<<< Start new container port: $port for $name"
docker run -d -p $port:8080 --name $name $image
```

在`deploy.sh`脚本中，主要分为以下几步：

1. 停止和删除旧容器
2. 删除旧镜像
3. 构建新镜像
4. 运行容器

这个脚本比较简单，按照上面的几步编写就行了。

最后附上项目的Dockerfile，也没啥说的。

**Dockerfile**
```
FROM openjdk:8-jdk-alpine
VOLUME /tmp
ADD spring-boot-docker-0.0.1-SNAPSHOT.jar app.jar
ENTRYPOINT ["java","-Djava.security.egd=file:/dev/./urandom","-jar","/app.jar"]
```


[<font size=3 color="#409EFF">向本文提出修改或勘误建议</font>](https://github.com/mstao/mstao.github.io/blob/hexo/source/_posts/build-springboot-by-jenkins-to-deploy-into-docker.md)