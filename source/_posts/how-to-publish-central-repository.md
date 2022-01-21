---
title: 如何发包到中央仓库-完整教程
categories: Maven
tags: [Maven]
author: Mingshan
date: 2022-01-21
---

相信很多小伙伴自己都会有一些包，我们可以将包发布中央仓库，然后直接通过maven来使用这些包，下面就整理下如何将包发到中央仓库。

<!-- more -->

## 新建项目

第一步，我们需要在[sonatype](https://issues.sonatype.org/)上注册一个账号，这一步就不再演示了。

注册好账户之后，然后我们来创建项目。点击**新建**按钮，问题类型 选择 **New Project**，如下所示：

![image](https://user-images.githubusercontent.com/23411433/150465809-2b2f0628-bf0a-49ab-b717-c139e2fff60d.png)

在该页面中，注意需要添加**groupId**，项目的描述信息等，填写完毕后进行保存。用过jira的同学对这个系统肯定十分熟悉，因为就是jira。

注意**如果groupId以前没有在这个上面使用过，那么第一次使用需进行注册，然后才可以发包**，新建好的项目会收到以下评论：

![image](https://user-images.githubusercontent.com/23411433/150466057-4169ec4c-6a6d-4700-aa6a-21528a6d8542.png)

翻译过来有以下几点：

如果你有

- 
