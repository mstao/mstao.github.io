---
title: 在Linux(CentOS)下重置MySQL根(Root)密码
tags: [Linux, MySQL]
author: Mingshan
categories: [Linux, MySQL]
date: 2018-5-5
---

我的CentOS服务器的mysql密码忘了，记录一下如何重置mysql的root密码

下面是操作步骤：

<!-- more -->

1. 首先输入“service mysqld status”查看当前mysql服务状态。

2. 输入“killall -TERM mysqld”命令停止所有的mysqld进程。

3. 输入“service mysqld stop”命令停止mysqld服务。

4. 输入“mysqld_safe  --skip-grant-tables &”命令以无密码方式进入MySQL安全模式。

5. 输入“mysql -u root”并按回车键即可。

6. 输入“use mysql;”挂载数据库。(注意：请勿忘记在最后输入分号（;）)

7. 输入"update user set password=password("admin") where user='root';"将Root密码修改为admin。

8. 输入"flush privileges;"更新权限。

9. 输入“quit”并按回车键退出。(注意：此处不需输入分号。)

10. 输入"service mysqld restart"重启mysqld服务。

11. 输入“mysql -u root -p”并按回车键提示输入密码。

12. 输入新密码admin并按回车键，提示已经成功登录。

下面是所有步骤运行截图：

![image](/images/update_mysql_password.png)
