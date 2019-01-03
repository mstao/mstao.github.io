---
title: Linux搭建Java Web环境(JDK8, Tomcat8.5, Nginx)
tags: [Linux]
author: Mingshan
categories: [Linux]
date: 2018-1-1
---

### 安装JDK

首先下载jdk-8u172-linux-x64.tar.gz，然后上传到服务器**/usr/local/java**目录

解压jdk-8u172-linux-x64.tar.gz

<!-- more -->

```
tar -zxvf jdk-8u172-linux-x64.tar.gz
```

设置环境变量


```
vim /etc/profile
```

在最前面添加：

```
export JAVA_HOME=/usr/local/java/jdk1.8.0_172 
export JRE_HOME=${JAVA_HOME}/jre  
export CLASSPATH=.:${JAVA_HOME}/lib:${JRE_HOME}/lib  
export  PATH=${JAVA_HOME}/bin:$PATH
```

输入:wq!保存退出

执行profile文件

```
source /etc/profile
```

这样可以使配置不用重启即可立即生效。

检查新安装的jdk


```
java -version
```

显示


```
java version "1.8.0_172"
Java(TM) SE Runtime Environment (build 1.8.0_172-b11)
Java HotSpot(TM) 64-Bit Server VM (build 25.172-b11, mixed mode)

```

### 安装Tomcat8.5

下载tomcat到/usr/local/tomcat

```
$ wget http://redrockdigimark.com/apachemirror/tomcat/tomcat-8/v8.5.23/bin/apache-tomcat-8.5.23.tar.gz
```

解压：

```
tar -zxvf apache-tomcat-8.5.23.tar.gz
```

启动：

```
sh startup.sh
```


### 安装Nginx

开始前，请确认gcc g++开发类库是否装好，默认已经安装。

centos平台编译环境使用如下指令

安装make：


```
yum -y install gcc automake autoconf libtool make
```

安装g++:

```
yum install gcc gcc-c++
```

下面正式开始安装

选择安装目录


```
cd /usr/local/nignx
```

**安装PCRE库**


```
cd /usr/local/nignx
wget ftp://ftp.csx.cam.ac.uk/pub/software/programming/pcre/pcre-8.39.tar.gz 
tar -zxvf pcre-8.39.tar.gz
cd pcre-8.39
./configure
make
make install
```

**安装zlib库**


```
cd /usr/local/nignx
 
wget http://zlib.net/zlib-1.2.11.tar.gz
tar -zxvf zlib-1.2.11.tar.gz
cd zlib-1.2.11
./configure
make
make install
```

**安装openssl**


```
cd /usr/local/nignx
wget https://www.openssl.org/source/openssl-1.0.1t.tar.gz
tar -zxvf openssl-1.0.1t.tar.gz
```

**启动nginx**


```
/usr/local/nginx/sbin/nginx
```

**查看进程**

```
ps -ef|grep nginx
```

**其他命令**

```
nginx -s reload ：修改配置后重新加载生效
nginx -s reopen ：重新打开日志文件
nginx -t -c /usr/local/nginx/conf/nginx.conf  测试nginx配置文件是否正确
```

**firewalld的基本使用**

启动： `systemctl start firewalld`

查看状态： `systemctl status firewalld`

停止： `systemctl disable firewalld`

禁用： `systemctl stop firewalld`

**查看firewall是否运行**

```
systemctl status firewalld.service
```

**添加开放端口**

```
firewall-cmd --zone=public --add-port=80/tcp --permanent    （--permanent永久生效，没有此参数重启后失效）
```
**查看所有打开的端口：**

```
firewall-cmd --zone=public --list-ports
```

**更新防火墙规则：**

```
firewall-cmd --reload
```
