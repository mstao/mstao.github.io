---
title: javaweb下载文件时IE浏览器下报错问题
categories: Java
tags: java
author: Mingshan
---
最近做的网站的一个下载功能出现了问题，在firefox浏览器以及360浏览器下下载均正常，也能将中文正常转换，但在IE浏览器下却出现了问题，当点击下载链接的时候，后台直接报错：
![2017-03-17_113722.png][1]
后台我怎么兼容也不能解决问题，我下载的部分java代码：

![QQ图片20170317113901.png][2]

  [1]: http://www.mingzhiwen.cn/usr/uploads/2017/03/1999726111.png
  [2]: http://www.mingzhiwen.cn/usr/uploads/2017/03/95431087.png
进过我仔细查找，发现我前台通过get方式提交的文件名包含一下字符，导致浏览器解析url不一致，所以需要将url通过javascript进行转码，即用encodeURIComponent函数进行编码，代码如下


```html
 <a href="javascript:location.href='${pageContext.request.contextPath}/download.do?realname='+encodeURIComponent('${file_list.real_name}')+'&filename='+encodeURIComponent('${file_list.file_name}');" class="file-name">${file_list.file_name}</a>

 ```
通过将文件名编码之后就能解决问题了
