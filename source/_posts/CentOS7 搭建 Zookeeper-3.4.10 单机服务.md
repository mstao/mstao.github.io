---
title: CentOS7 搭建 Zookeeper-3.4.10 单机服务
tags: [Linux, Zookeeper]
author: Mingshan
categories: [Linux]
date: 2018-5-12
---

由于项目用到了Dubbo,而Dubbo的服务的注册与发现我用了Zookeeper，所以我在部署Dubbo服务的时候，就必须安装Zookeeper,本文记录下我在CentOS7 搭建 Zookeeper-3.4.10 单机服务的过程。

<!-- more -->

### Zookeeper的安装

1. 下载Zookeeper

下载最新版本的ZooKeeper，这里有两个镜像可以选择

> 清华镜像:https://mirrors.tuna.tsinghua.edu.cn/apache/zookeeper/
> 
> 阿里镜像:https://mirrors.aliyun.com/apache/zookeeper/

我就选择阿里镜像进行安装。

首先在**/usr/local**目录下创建**services** 文件夹和其子文件**zookeeper**,然后进入该文件夹，
最后用wget进行下载

```
mkdir -p /usr/local/services/zookeeper

cd /usr/local/services/zookeeper

wget --no-check-certificate  https://mirrors.aliyun.com/apache/zookeeper/zookeeper-3.4.10/zookeeper-3.4.10.tar.gz

```

2. 提取tar文件

接下来解压**zookeeper-3.4.10.tar.gz**， 首先进入到该文件夹, 然后进行解压

```
cd /usr/local/services/zookeeper

tar -zxf  zookeeper-3.4.10.tar.gz

cd zookeeper-3.4.10
```

进入到解压后的文件夹后，创建**data**文件夹 用于存储数据文件


```
mkdir data
```

创建**logs**文件夹 用于存储日志


```
mkdir logs
```

3. 创建配置文件

使用命令 vim conf/zoo.cfg 创建配置文件并打开, 其实该文件夹下有了一个zoo_sample.cfg示例配置文件，我们还是新创建一个吧。

编辑内容如下：


```
tickTime = 2000
dataDir = /usr/local/services/zookeeper/zookeeper-3.4.10/data
dataLogDir = /usr/local/services/zookeeper/zookeeper-3.4.10/logs
tickTime = 2000
clientPort = 2181
initLimit = 5
syncLimit = 2
```


### Zookeeper服务

1. 启动服务


```
/usr/local/services/zookeeper/zookeeper-3.4.10/bin/zkServer.sh start
```

2. 连接服务


```
/usr/local/services/zookeeper/zookeeper-3.4.10/bin/zkCli.sh
```

3. 查看服务状态


```
/usr/local/services/zookeeper/zookeeper-3.4.10/bin/zkServer.sh status
```

4. 停止服务


```
/usr/local/services/zookeeper/zookeeper-3.4.10/bin/zkServer.sh stop
```


### 参考：

https://segmentfault.com/a/1190000010791627
