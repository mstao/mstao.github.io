---
title: CentOS7安裝Redis遇到问题及解决
tags: [Linux, Redis]
author: Mingshan
categories: [Linux]
date: 2018-5-23
---

前段时间我在我的CentOS7服务器上安装了Redis，遇到了一些问题忘记记录下来了ヽ(￣▽￣)ﾉ，而且我参考网上的教程配置一些脚本的时候发现有错误，根据我对shell的简单理解会介绍一下，毕竟这个东西很常用。

<!-- more -->

## 安装Redis

首先在**/usr/local**目录下创建services 文件夹和其子文件redis,然后进入该文件夹。

下载redis可以通过wget，我这里下载的是**redis-4.0.2**版本


```
wget http://download.redis.io/releases/redis-4.0.2.tar.gz
```

将Redis下载到该文件夹下后，将压缩文件解压

```
cd /usr/local/services/redis

tar -xzvf redis-4.0.2.tar.gz
```

进入到解压后的文件夹，进行编译即可


```
cd /usr/local/services/redis/redis-4.0.2

make
```

编译完后，我们需要将**redis-cli**，**redis-server**，**redis.conf**拷贝到一个单独的文件夹，这里我们在**/usr/local/services/redis** 文件夹下新建一个文件夹
**redisroot**


```
mkdir -p /usr/local/services/redis/redisroot
```
然后将编译后的**redis-cli**，**redis-server**，**redis.conf**拷贝到**redisroot**文件夹下


```
cp /usr/local/services/redis/redis-4.0.2/src/redis-server /usr/local/services/redis/redisroot

cp /usr/local/services/redis/redis-4.0.2/src/redis-cli /usr/local/services/redis/redisroot

cp /usr/local/services/redis/redis-4.0.2/redis.conf /usr/local/services/redis/redisroot
```

拷贝完后的文件夹结构如下：

![image](https://github.com/mstao/static/blob/master/blog/redisroot-folder.png?raw=true)

## 编辑配置文件

编辑Redis配置文件

进入到**redisroot**文件夹，输入下列命令进入编辑模式

```
vim redis.conf
```
然后修改以下配置：

1. 在bind 127.0.0.1前加“#”将其注释掉
2. 默认为保护模式，把 protected-mode yes 改为 protected-mode no
3. 默认为不守护进程模式，把daemonize no 改为daemonize yes
4. 将 requirepass foobared前的“#”去掉，密码改为你想要设置的密码

设置完后，**ESC**切换模式后输入**:wq!**保存退出

## 编辑Redis开机启动脚本

输入以下命令编辑脚本

```
vim /etc/init.d/redis
```
打开后在这个文件里添加如下脚本


```shell
#!/bin/sh
# chkconfig: 2345 80 90
# description: Start and Stop redis
#PATH=/usr/local/bin:/sbin:/usr/bin:/bin
REDISPORT=6379
EXEC=/usr/local/services/redis/redisroot/redis-server     
REDIS_CLI=/usr/local/services/redis/redisroot/redis-cli     
PIDFILE=/var/run/redis_6379.pid
CONF="/usr/local/services/redis/redisroot/redis.conf"     
RESDISPASSWORD=123456

case "$1" in
    start)
        if [ -f $PIDFILE ]
        then
                echo "$PIDFILE exists, process is already running or crashed"
        else
                echo "Starting Redis server..."
                $EXEC $CONF
        fi
        if [ "$?"="0" ]
        then
              echo "Redis is running..."
        fi
        ;;
    stop)
        if [ ! -f $PIDFILE ]
        then
                echo "$PIDFILE does not exist, process is not running"
        else
                PID=$(cat $PIDFILE)
                echo "Stopping ..."
                $CLIEXEC -a $RESDISPASSWORD -p$REDISPORT shutdown
                while [ -x ${PIDFILE} ]
               do
                    echo "Waiting for Redis to shutdown ..."
                    sleep 1
                done
                echo "Redis stopped"
        fi
        ;;
   restart|force-reload)
        ${0} stop
        ${0} start
        ;;
  *)
    echo "Usage: /etc/init.d/redis {start|stop|restart|force-reload}" >&2
        exit 1
esac
```

脚本中声明的路径常量需要根据自己的安装路径进行配置。

上面这个脚本是我参考网上的，但网上的有错误，比如我设置了Redis的密码，那么在执行停止命令时是需要验证密码的，所以要这样写**$CLIEXEC -a $RESDISPASSWORD -p$REDISPORT shutdown**。


## 后续配置

下面的一些操作是从网上拷贝的，为后续配置Redis，基本相同

### 添加开机启动服务

编辑**/etc/rc.local**

```
vim /etc/rc.local
```
增加启动代码

```
service redis start
```
编辑后的配置文件如下：

![image](https://github.com/mstao/static/blob/master/blog/service-redis-start.png?raw=true)

### 设置权限


```
chmod 755 /etc/init.d/redis
```

### 注册系统服务


```
chkconfig --add redis
```
### 测试redis服务

启动服务

```
service redis start
```

启动日志如下：

```
[root@VM_37_72_centos redisroot]# service redis start
Starting Redis server...
21416:C 23 May 00:24:19.666 # oO0OoO0OoO0Oo Redis is starting oO0OoO0OoO0Oo
21416:C 23 May 00:24:19.666 # Redis version=4.0.2, bits=64, commit=00000000, modified=0, pid=21416, just started
21416:C 23 May 00:24:19.666 # Configuration loaded
Redis is running...

```


停止服务


```
service redis stop
```

停止日志如下：

```
[root@VM_37_72_centos redisroot]#  service redis stop
Stopping ...
Redis stopped

```

### 创建redis命令软连接

在linux下很多地方都需要软连接，软连接其实就是windows的快捷方式。 

```
ln -s /usr/local/services/redis/redisroot/redis-cli /usr/bin/redis
```

### 测试Redis

最后可以直接进行测试了

![image](https://github.com/mstao/static/blob/master/blog/redis-test.png?raw=true)

OK, 大功告成。

## 后记

其实这些命令教程啊网上都有，不过有些是错误的，只有自己完完全全测试过一遍后才知道哪些有问题，同时会对Redis有个基本的了解吧，平时都在用Windows，相对来说有些傻瓜式，多敲些Linux命令还是有益处的，哈哈(～￣▽￣)～ 

## 参考

https://blog.csdn.net/lc1010078424/article/details/78295482


[<font size=3 color="#409EFF">向本文提出修改或勘误建议</font>](https://github.com/mstao/mstao.github.io/blob/hexo/source/_posts/centos7-redis.md)