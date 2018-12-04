---
title: markdown-01
date: 2017-06-07 15:44:07
tags: markdown
categories: markdown
---
*试试一下markdowm*
## 这是一个标题
1. 这是第一行列表项
2. 这是第二行列表项

## 列表

Markdown 支持有序列表和无序列表。

无序列表使用星号、加号或是减号作为列表标记：
<pre>
*   Red
*   Green
*   Blue
</pre>
等同于：
<pre>
+   Red
+   Green
+   Blue
</pre>
也等同于：
<pre>
-   Red
-   Green
-   Blue
</pre>
有序列表则使用数字接着一个英文句点：
<pre>
1.  Bird
2.  McHale
3.  Parish
</pre>

## 代码区块
markdown使用<pre></pre><code></code> 来将代码包裹起来

## 分割线

* * *
***
*****
- - -
## 链接
```
System.out.println("helllo world");
```
```
[This link](http://example.net/) has no title attribute.
```
[This link](http://example.net/) has no title attribute.

## 强调
Markdown 使用星号（*）和底线（_）作为标记强调字词的符号，被 * 或 _ 包围的字词会被转成用 <em> 标签包围，用两个 * 或 _ 包起来的话，则会被转成 <strong>，例如：
```
*single asterisks*

_single underscores_

**double asterisks**

__double underscores__
```
会转成：
```
<em>single asterisks</em>

<em>single underscores</em>

<strong>double asterisks</strong>

<strong>double underscores</strong>
```
