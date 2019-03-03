---
title: 利用Docker搭建Nexus3私服
tags: [Nexus3]
author: Mingshan
categories: Docker
date: 2018-8-5
---

搭建私服对于一个团队来说十分有必要，利用Nexus3搭建私服十分方便，结合Docker，那是相当快速。

<!-- more -->

## 安装Nexus3

拉取Nexus3的镜像

```
docker pull sonatype/nexus3
```

创建文件夹，挂载目录
```
mkdir /var/nexus-data && chown -R 200 /var/nexus-data
```

启动容器

```
docker run -d -p 8081:8081 --name nexus -v /var/nexus-data:/nexus-data --restart=always sonatype/nexus3
```

开启端口

如果让外网访问的话，需要开放8081端口，在CentOS7中，可以利用firewall-cmd来开放端口


```
#开放11001端口  permanent为永久开放
firewall-cmd --zone=public --add-port=8081/tcp --permanent
#重新读取配置
firewall-cmd --reload
#查看全部开放端口
firewall-cmd --list-all
```

接下来在浏览器中输入以下网址，就可以看到Nexus3界面了如下图所示，用户名和密码默认为admin，admin123


```
http://ip:port
```

![image](/images/nexus3-dashboard.png)

## 配置阿里云仓库

启动Nexus后，需要将中央仓库配置为阿里云仓库，提高国内的访问速度。Nexus的仓库如下：

![image](/images/nexus3-repository.png)


由上图看出，Nexus的仓库分为这么几类：

- hosted 宿主仓库：主要用于部署无法从公共仓库获取的包以及自己或第三方的包；
- proxy 代理仓库：代理公共的远程仓库；
- group 仓库组：Nexus 通过仓库组的概念统一管理多个仓库，这样我们在项目中直接请求仓库组即可请求到仓库组管理的多个仓库。

示意图如下：

![image](/images/nexus3-repository-desc.png)

现在点击“Create Repository”按钮，来添加阿里云仓库，Recipe选择`maven2(proxy)`，在具体配置页面取名aliyun-repository，这里建议用a开头(估计按字母排序将它排第一位)，URL输入：`http://maven.aliyun.com/nexus/content/groups/public/`，其他默认值即可。


## 配置maven

接下来我们需要配置maven，打开setting.xml文件，在mirrors节点下加入以下配置（根据自己的配置更改）：

将`<mirror><url>`标签内的地址修改成nexus服务的地址。

```xml
<mirror>
  <id>nexus</id>
  <mirrorOf>*</mirrorOf>
  <name>nexus maven</name>
  <url>http://www.mingzhiwen.cn:8081/repository/maven-public/</url>
</mirror>
```


然后在services节点下加入以下配置，`<servers>`标签内填写nexus服务的账号密码，发布maven项目到nexus时，需要用到。`<server><id>`下id需要跟`<mirror><id>`一致。

```xml
<server>
  <id>nexus</id>
  <username>admin</username>
  <password>admin123</password>
</server>

```

最后在项目中的`pom.xml`文件加入以下配置：


```xml
<distributionManagement> 
	<repository>
		<id>nexus</id>
		<name>Releases</name>
		<url>http://{your-nexus-ip}:port/repository/maven-releases/</url>
	</repository>
	<snapshotRepository>
		<id>nexus</id>
		<name>Snapshot</name>
		<url>http://{your-nexus-ip}:port/repository/maven-snapshots/</url>
	</snapshotRepository>
</distributionManagement>
```

运行maven命令，将编译好的Jar包上传到Nexus私服中。


```
mvn clean deploy
```

上传效果如下图所示：

![image](/images/nexus3-upload.png)


## 参考

- https://www.jianshu.com/p/dbeae430f29d
- https://github.com/sonatype/docker-nexus3


[<font size=3 color="#409EFF">向本文提出修改或勘误建议</font>](https://github.com/mstao/mstao.github.io/blob/hexo/source/_posts/docker-nexus3.md)