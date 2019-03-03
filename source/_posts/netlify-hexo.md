---
title: 使用Netlify部署Hexo博客并升级为HTTPS
tags: [Netlify, Hexo]
author: Mingshan
categories: [Hexo]
date: 2018-12-05
---

用了一年多的[Github pages](https://pages.github.com/)，其实挺方便的，但在国内访问太慢。前几天突然发现了[Netlify](https://www.netlify.com/)，恰逢我博客更换域名，想把博客整理一下，所以我就部署，试试还挺快，并且是自动部署的，so easy阿。下面记录下部署Hexo博客并升级HTTPS的过程，方便同学迁移博客。。

<!-- more -->

## 部署到Netlify

### 创建项目

首先利用Github帐号登录Netlify，点击`New site from Git`，然后选择`github` 按钮进入到选择项目界面，如下：

![image](https://github.com/ZZULI-TECH/interview/blob/master/images/Netlify/Netlify_new_site.png?raw=true)


这里选择你自己github上的项目，然后进入到设置部署选项的页面，分支选择hexo源码的分支，命令为`hexo g`，让netlify帮我们自动编译和部署项目，部署目录输入`public/`，这是因为hexo 编译后输出的静态页面在这个目录，如下图所示：

![image](https://github.com/ZZULI-TECH/interview/blob/master/images/Netlify/Netlify_deploy_setting.png?raw=true)

设置完后，进入刚才创建站点的概览界面，如下图所示：

![image](https://github.com/ZZULI-TECH/interview/blob/master/images/Netlify/Netlify_site_overview.png?raw=true)

这时，我们选择站点设置按钮，进入到站点设置界面，在左边的栏目列表中选择`Domain & deploy` 此时界面如下：

![image](https://github.com/ZZULI-TECH/interview/blob/master/images/Netlify/Netlify_site_management.png?raw=true)

### 配置站点

在这个界面，你可以点击上面的站点的名称(这个可以修改，我的是改过的)访问刚才部署过的网站，没问题的话可以直接访问的，然后可以添加自定义域名，比如你自己有一个域名(我的是`mingshan.fun`)，先添加不带www的域名作为主域名，它会自动添加一个`www.mingshan.fun`重定向到`mingshan.fun`，我的已经添加过了。

接下来去域名提供商配置域名解析，我的域名是在阿里云买的，所示在阿里云控制台进行域名解析，需要添加两条解析记录，一条A记录，一条CNAME记录，如下图：

![image](https://github.com/ZZULI-TECH/interview/blob/master/images/Netlify/domain_parse.png?raw=true)

过一会就可以你的域名来访问了。

## 升级HTTPS

如果我们的域名有申请的HTTPS证书，在哪里进行配置呢？Netlify为我们提供配置HTTPS证书的方法。你需要先下载证书，并且是部署到apache的证书（估计Netlify是用apache部署的），我的证书目录如下：

![image](https://github.com/ZZULI-TECH/interview/blob/master/images/Netlify/mingshan_fun_zheng.png?raw=true)

- `1582749_mingshan.fun.key` 是证书的私匙
- `1582749_mingshan.fun_public.crt` 是证书的公匙
- `1582749_mingshan.fun_chain.crt` 是证书的Certificate Chain

接下来复制这些数据到如下页面：

![image](https://github.com/ZZULI-TECH/interview/blob/master/images/Netlify/Netlify_custom_certificate.png?raw=true)


`Certificate` 对应`1582749_mingshan.fun_public.crt`， `Private key` 对应`1582749_mingshan.fun.key`，`Intermediate certs` 对应 `1582749_mingshan.fun_chain.crt`，注意要复制文件里面的全部信息，不要自己截取。

添加后，等待一会，会显示当前站点Https可用了哦，如下图所示：

![image](https://github.com/ZZULI-TECH/interview/blob/master/images/Netlify/Netlify_https_enable.png?raw=true)

完美，撒花花(〃'▽'〃)


[<font size=3 color="#409EFF">向本文提出修改或勘误建议</font>](https://github.com/mstao/mstao.github.io/blob/hexo/source/_posts/netlify-hexo.md)