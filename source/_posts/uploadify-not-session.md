---
title: 利用uploadify插件上传文件时java后台获取不到当前session问题
categories: Java
tags: java
author: Mingshan
date: 2017-07-10
---
我在利用uploadify插件上传文件时发现java后台获取不到当前用户的session值，即当前的用户的session保存的信息失效，导致拦截器将上传请求拦截，这里的拦截器主要对登录的信息进行拦截验证，正因为此原因，导致上传文件失败，后来在利用firebug查找请求的时候，发现uploadify插件会自动生成一个新的session，导致原来的session失效，解决方法是将jsessionid通过url传到后台，这样后台就能识别当前session，问题也就解决了。代码如下：

```javascript
$("#uploadify").uploadify({
	debug			: false,
	swf 			:  CTPPATH+'/admin/static/uploadify/js/uploadify.swf',	//swf文件路径
	method			: 'get',	// 提交方式
	uploader		:  CTPPATH+'/processUpload.ado;jsessionid=${pageContext.session.id}', // 服务器端处理该上传请求的程序(servlet, struts2-Action)   )};

```
  代码中有许多属性这里没有贴出来，这里主要看uploader属性，uploader属性为CTPPATH+'/processUpload.ado;jsessionid=${pageContext.session.id}'，即在请求url中附上
;jsessionid=${pageContext.session.id}，这样上传就没问题了。
