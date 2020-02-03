---
title: Windows下安装RabbitMQ无法启用管理界面
tags: [RabbitMQ]
author: Mingshan
categories: [RabbitMQ]
date: 2020-02-0
---

很久没有用现在的笔记本连过RabbitMQ，今天在Windows上装了一下RabbitMQ，然后启用管理界面，发现怎么都启用不了，一直提示：`Plugin configuration unchanged.`，查看服务发现MQ的服务都关闭了，为什么会自动关闭服务呢？我们来查看下MQ的日志，路径是：`C:\Users\明山\AppData\Roaming\RabbitMQ\log\rabbit@DESKTOP-Q1D3TT5.log`，在文件末尾，发现有下面的错误：

<!-- more -->

```
2020-02-03 18:51:50.250 [error] <0.8.0> FORMAT ERROR: "~nError description:~s~nLog file(s) (may contain more information):~n   ~s~n   ~s~n" ["\n    init:do_boot/3 line 817\n    init:start_em/1 line 1109\n    rabbit:start_it/1 line 474\n    rabbit:broker_start/1 line 350\n    rabbit:start_loaded_apps/2 line 600\n    app_utils:manage_applications/6 line 126\n    lists:foldl/3 line 1263\n    rabbit:'-handle_app_error/1-fun-0-'/3 line 723\nthrow:{could_not_start,rabbitmq_management_agent,\n       {rabbitmq_management_agent,\n        {{shutdown,\n          {failed_to_start_child,rabbit_mgmt_agent_sup,\n           {shutdown,\n            {failed_to_start_child,rabbit_mgmt_external_stats,\n             {badarg,\n              [{erlang,list_to_binary,\n                [[67,58,47,85,115,101,114,115,47,26126,23665,47,65,112,112,68,\n                  97,116,97,47,82,111,97,109,105,110,103,47,82,97,98,98,105,\n                  116,77,81,47,108,111,103,47,114,97,98,98,105,116,64,68,69,\n                  83,75,84,79,80,45,81,49,68,51,84,84,53,46,108,111,103]],\n                []},\n               {rabbit_mgmt_external_stats,'-i/2-lc$^0/1-2-',1,\n                [{file,\"src/rabbit_mgmt_external_stats.erl\"},{line,212}]},\n               {rabbit_mgmt_external_stats,'-infos/2-lc$^0/1-0-',2,\n                [{file,\"src/rabbit_mgmt_external_stats.erl\"},{line,184}]},\n               {rabbit_mgmt_external_stats,'-infos/2-lc$^0/1-0-',2,\n                [{file,\"src/rabbit_mgmt_external_stats.erl\"},{line,184}]},\n               {rabbit_mgmt_external_stats,emit_update,1,\n                [{file,\"src/rabbit_mgmt_external_stats.erl\"},{line,390}]},\n               {rabbit_mgmt_external_stats,init,1,\n                [{file,\"src/rabbit_mgmt_external_stats.erl\"},{line,366}]},\n               {gen_server,init_it,2,[{file,\"gen_server.erl\"},{line,374}]},\n               {gen_server,init_it,6,\n                [{file,\"gen_server.erl\"},{line,342}]}]}}}}},\n         {rabbit_mgmt_agent_app,start,[normal,[]]}}}}",[67,58,47,85,115,101,114,115,47,26126,23665,47,65,112,112,68,97,116,97,47,82,111,97,109,105,110,103,47,82,97,98,98,105,116,77,81,47,108,111,103,47,114,97,98,98,105,116,64,68,69,83,75,84,79,80,45,81,49,68,51,84,84,53,46,108,111,103],[67,58,47,85,115,101,114,115,47,26126,23665,47,65,112,112,68,97,116,97,47,82,111,97,109,105,110,103,47,82,97,98,98,105,116,77,81,47,108,111,103,47,114,97,98,98,105,116,64,68,69,83,75,84,79,80,45,81,49,68,51,84,84,53,95,117,112,103,114,97,100,101,46,108,111,103]]
2020-02-03 18:51:50.361 [error] emulator Error in process <0.8.0> on node 'rabbit@DESKTOP-Q1D3TT5' with exit value:
{badarg,[{io,format,[standard_error,"~nBOOT FAILED~n===========~n~nError description:~s~nLog file(s) (may contain more information):~n   ~s~n   ~s~n~n",["\n    init:do_boot/3 line 817\n    init:start_em/1 line 1109\n    rabbit:start_it/1 line 474\n    rabbit:broker_start/1 line 350\n    rabbit:start_loaded_apps/2 line 600\n    app_utils:manage_applications/6 line 126\n    lists:foldl/3 line 1263\n    rabbit:'-handle_app_error/1-fun-0-'/3 line 723\nthrow:{could_not_start,rabbitmq_management_agent,\n       {rabbitmq_management_agent,\n        {{shutdown,\n          {failed_to_start_child,rabbit_mgmt_agent_sup,\n           {shutdown,\n            {failed_to_start_child,rabbit_mgmt_external_stats,\n             {badarg,\n              [{erlang,list_to_binary,\n                [[67,58,47,85,115,101,114,115,47,26126,23665,47,65,112,112,68,\n                  97,116,97,47,82,111,97,109,105,110,103,47,82,97,98,98,105,\n                  116,77,81,47,108,111,103,47,114,97,98,98,105,116,64,68,69,\n                  83,75,84,79,80,45,81,49,68,51,84,84,53,46,108,111,103]],\n                []},\n               {rabbit_mgmt_external_stats,'-i/2-lc$^0/1-2-',1,\n                [{file,\"src/rabbit_mgmt_external_stats.erl\"},{line,212}]},\n               {rabbit_mgmt_external_stats,'-infos/2-lc$^0/1-0-',2,\n                [{file,\"src/rabbit_mgmt_external_stats.erl\"},{line,184}]},\n               {rabbit_mgmt_external_stats,'-infos/2-lc$^0/1-0-',2,\n                [{file,\"src/rabbit_mgmt_external_stats.erl\"},{line,184}]},\n               {rabbit_mgmt_external_stats,emit_update,1,\n                [{file,\"src/rabbit_mgmt_external_stats.erl\"},{line,390}]},\n               {rabbit_mgmt_external_stats,init,1,\n                [{file,\"src/rabbit_mgmt_external_stats.erl\"},{line,366}]},\n               {gen_server,init_it,2,[{file,\"gen_server.erl\"},{line,374}]},\n               {gen_server,init_it,6,\n                [{file,\"gen_server.erl\"},{line,342}]}]}}}}},\n         {rabbit_mgmt_agent_app,start,[normal,[]]}}}}",[67,58,47,85,115,101,114,115,47,26126,23665,47,65,112,112,68,97,116,97,47,82,111,97,109,105,110,103,47,82,97,98,98,105,116,77,81,47,108,111,103,47,114,97,98,98,105,116,64,68,69,83,75,84,79,80,45,81,49,68,51,84,84,53,46,108,111,103],[67,58,47,85,115,101,114,115,47,26126,23665,47,65,112,112,68,97,116,97,47,82,111,97,109,105,110,103,47,82,97,98,98,105,116,77,81,47,108,111,103,47,114,97,98,98,105,116,64,68,69,83,75,84,79,80,45,81,49,68,51,84,84,53,95,117,112,103,114,97,100,101,46,108,111,103]]],[]},{rabbit,log_boot_error_and_exit,3,[{file,"src/rabbit.erl"},{line,1081}]},{rabbit,start_it,1,[{file,"src/rabbit.erl"},{line,477}]},{init,start_em,1,[{file,"init.erl"},{line,1109}]},{init,do_boot,3,[{file,"init.erl"},{line,817}]}]}
```

这个错误乱七八糟的也看不懂，看着似乎是启动时解析某些东西出错了，然后我发现我的路径（用户名）居然带中文，是不是这个问题呢？在网上搜了一下，发现可以将这个mq产生数据的路径给改了，在mq的server的sbin目录下，执行如下命令：

```
set RABBITMQ_BASE=E:\develope\rabbitmq\data
```

然后重启服务就好了，相当之坑啊。具体的命令如下：

```
E:\develope\rabbitmq\rabbitmq_server-3.8.2\sbin>rabbitmq-service.bat remove
E:\develope\erl10.6\erts-10.6\bin\erlsrv: Service RabbitMQ removed from system.

E:\develope\rabbitmq\rabbitmq_server-3.8.2\sbin>set RABBITMQ_BASE=E:\develope\rabbitmq\data

E:\develope\rabbitmq\rabbitmq_server-3.8.2\sbin>rabbitmq-service.bat install
E:\develope\erl10.6\erts-10.6\bin\erlsrv: Service RabbitMQ added to system.

E:\develope\rabbitmq\rabbitmq_server-3.8.2\sbin>rabbitmq-plugins enable rabbitmq_management
Enabling plugins on node rabbit@DESKTOP-Q1D3TT5:
rabbitmq_management
The following plugins have been configured:
  rabbitmq_management
  rabbitmq_management_agent
  rabbitmq_web_dispatch
Applying plugin configuration to rabbit@DESKTOP-Q1D3TT5...
The following plugins have been enabled:
  rabbitmq_management
  rabbitmq_management_agent
  rabbitmq_web_dispatch

set 3 plugins.
Offline change; changes will take effect at broker restart.

E:\develope\rabbitmq\rabbitmq_server-3.8.2\sbin>rabbitmq-service.bat stop
没有启动 RabbitMQ 服务。

请键入 NET HELPMSG 3521 以获得更多的帮助。

E:\develope\rabbitmq\rabbitmq_server-3.8.2\sbin>rabbitmq-service.bat start
RabbitMQ 服务正在启动 .
RabbitMQ 服务已经启动成功。
```



