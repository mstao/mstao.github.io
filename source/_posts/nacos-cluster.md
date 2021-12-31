---
title: Nacos2.0.3集群搭建踩坑
tags: [Nacos]
author: Mingshan
categories: [Java,Nacos]
date: 2021-10-31
---


Nacos2.0版本相比1.X新增了gRPC的通信方式，如果已经有Nacos集群，那么需要更改集群的配置方式，这里以Nginx为例，来介绍下如何搭建集群。


<!-- more -->

## 配置流程


新增端口是在配置的主端口(server.port)基础上，进行一定偏移量自动生成。

| **端口** | **与主端口的偏移量** | **描述** |
| --- | --- | --- |
| 9848 | 1000 | 客户端gRPC请求服务端端口，用于客户端向服务端发起连接和请求 |
| 9849 | 1001 | 服务端gRPC请求服务端端口，用于服务间同步等 |



假设我们Nacos的服务端口为8848，那么客户端gRPC与服务端进行交互时，会使用9848端口，所以我们必须转发这个9848端口。在转发该端口时，注意使用`stream`方式，这就要求nginx在编译时，必须有`--with-stream`，使用`nginx -V`来查看编译配置：
​

```
[root@173-16-200-97 sbin]# ./nginx -V
nginx version: nginx/1.15.6
built by gcc 4.8.5 20150623 (Red Hat 4.8.5-44) (GCC) 
configure arguments: --prefix=/usr/local/nginx --with-pcre=/usr/local/nginx/pcre-8.42 --with-zlib=/usr/local/nginx/zlib-1.2.11 --with-openssl=/usr/local/nginx/openssl-1.1.1a --with-stream --add-module=/usr/local/nginx/ngx_healthcheck_module-master

```
检查完Nginx信息后，我们先来配置转发**9848**端口：
```
stream {
    upstream NACOS_ADDR_9848 {
      server 173.16.200.98:9848 max_fails=3 fail_timeout=30s;
      server 173.16.200.99:9848 max_fails=3 fail_timeout=30s;
      server 173.16.200.115:9848 max_fails=3 fail_timeout=30s;
    }

    server {
      listen 9848 so_keepalive=on;
      proxy_connect_timeout 3s;
      proxy_pass NACOS_ADDR_9848;
      tcp_nodelay on;
      proxy_buffer_size 32k;
    }
}


```
接着配置转发**8848**端口，使用http配置就可以：
```
http {
    upstream NACOS {
       server 173.16.200.98:8848 max_fails=3 fail_timeout=5s;
       server 173.16.200.99:8848 max_fails=3 fail_timeout=5s;
       server 173.16.200.115:8848 max_fails=3 fail_timeout=5s;
    }
    
    server {
        listen 8848;
        server_name 173.16.200.97;
        location / {
             proxy_pass http://NACOS;
             proxy_connect_timeout 75;
             proxy_read_timeout 400;
             proxy_send_timeout 400;
             client_max_body_size 100m;
             proxy_set_header Host $host;
             proxy_set_header X-Real-IP $remote_addr;
             proxy_set_header X-Forwarded-For $remote_addr;
        }
    }

}

```
## 遇到的问题


1. **应用启动报：UNAVAILABLE: Network closed for unknown reason**

报这个错是由于没有转发9848端口，转发9848端口即可

2. **Nacos服务是2.0.3，客户端是1.x版本，是否可以使用？**

Nacos服务端可以兼容1.x版本的客户端，只不过不会走gRPC模式，建议生产环境服务端与客户端一致
