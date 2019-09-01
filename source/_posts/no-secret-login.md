---
title: Linux免密Shell登录
tags: [Linux]
author: Mingshan
categories: [Linux]
date: 2018-9-12
---

现在假设有两台机器 A 和 B，正常情况下在A机器中通过shell登录B机器，输入以下命令：

```
ssh walker@xx.yy.zz.cc
```

此时会提示输入密码。当我们通过脚本执行相关命令时，需要用到免密登录。

<!-- more -->

## 步骤如下：

### 生成ssh key

首先在A机器执行如下命令生成ssh key

```
ssh-keygen -t rsa
```
一路按回车键即可。生成之后会在用户的根目录生成一个 `.ssh`的文件夹，注意该文件夹默认是隐藏的，直接`cd`进去即可。
该文件夹会有两个文件：`id_rsa` 和 `id_rsa.pub` 两个文件，如下所示：
```
[iwms@ecs ~]$ cd .ssh
[iwms@ecs .ssh]$ ls
id_rsa  id_rsa.pub
```

如果有其他的机器想免密登录到A机器上，需要在`.ssh`文件夹再创建两个文件：authorized_keys 和 know_hosts

现在说明这四个文件的意思：

- authorized_keys:存放远程免密登录的公钥,主要通过这个文件记录多台机器的公钥
- id_rsa : 生成的私钥文件
- id_rsa.pub ： 生成的公钥文件
- know_hosts : 已知的主机公钥清单


如果希望ssh公钥生效需满足至少下面两个条件：

1. `.ssh`目录的权限必须是700
2. `.ssh/authorized_keys`文件权限必须是600

### 将产生的公钥复制到 B 机的用户目录下

接着我们要将生成的公钥拷贝到B机器的用户目录下，命令如下：

```
scp id_rsa.pub 登录用户名@IP地址(或域名):/home/用户名/id_rsa.pub
```

例如：

```
scp id_rsa.pub iwms@172.17.10.232:/home/iwms/id_rsa.pub
```

然后切换到B机器上，也是到当前用户的`.ssh`目录下，追加公钥到 authorzied_keys中(注意不要用 `>`，否则会清空原有的内容，使其他人无法使用原有的密钥登录) ：

```
cat id_rsa.pub >> .ssh/authorized_keys
```

此时必须将`/home/imws/.ssh/authorized_keys`的权限改为600, 该文件用于保存ssh客户端生成的公钥，可以修改服务器的ssh服务端配置文件/etc/ssh/sshd_config来指定其他文件名

```
chmod 600 .ssh/authorized_keys
```

### 验证：

直接在A机器输入

```
ssh walker@xx.yy.zz.cc
```

此时没有密码提示，直接进入到B机器，说明没有问题
