---
title: Git使用不完全指南 - 恢复被Reset掉的提交
categories: Git
tags: Git
author: Mingshan
date: 2017-12-04
---

记得有一次我用github的桌面客户端提交代码时，提交了我不想提交的内容，于是我就点了Revert按钮，进行了**Revert**操作，这个操作会撤销一个提交的同时会新建一个提交，这也不是我想要的效果，所以我就用了Reset命令，最后操作失误，提交的代码都丢了，这让我很忧伤，但我之前进行提交操作了，git会有提交记录，所以我查了查命令，发现了**git reflog**命令，这个命令用于记录对git仓库进行的各种操作，输入命令显示如下：


```
D:\code\LightBlog [master ≡ +0 ~2 -0 !]> git reflog
4ca2fb7 HEAD@{0}: commit: Add spring-redis for cache
ddcc61a HEAD@{1}: commit: Add light blog ui template
1e9f0b9 HEAD@{2}: pull --progress --prune origin master: Fast-forward
93d6519 HEAD@{3}: commit: Add encryption util with MD5
1d158ff HEAD@{4}: commit: fIX(#2) HttpStatus 204 ->no content
ef8cb50 HEAD@{5}: revert: Revert "Refine exception handler"
51a7b00 HEAD@{6}: rebase: updating HEAD
51a7b00 HEAD@{7}: rebase: aborting
e20b30f HEAD@{8}: pull --progress --rebase --prune origin master: checkout e20b30f0a282cb4b1229a5ebcf220422d3685c40
51a7b00 HEAD@{9}: commit: Refine exception handler
294b1e3 HEAD@{10}: pull --progress --prune origin master: Merge made by the 'recursive' strategy.
927ff21 HEAD@{11}: commit: Add exception handler for RESTful
4e2592c HEAD@{12}: commit: Add authorization for web with redis.
```
可以看见每次操作都会有记录的（没有全部贴出来），这是我们可以用**git reset ID**来恢复内容，ID指的第一列的东西，比如**4ca2fb7**，这样我们就可以找回丢失的内容了。

下面是我常用的一套命令，用来更新本地代码，这也是我经过好几次拉取远程代码冲突一大片得到的经验，**git mergetool**指的是当发生代码冲突时，输入此命令就会进行冲突的解决，由于我用的VS，输入此命令后就会跳到VS中来解决冲突了，对于一个文件如果冲突实在是太多，推荐**winmerge**工具，一个轻量且免费的冲突合并工具，我平时也用。

```
git status

git commit -m 'temp commit'

git stash save appconfig

git log

git pull -r origin develop

git mergetool(回车合并冲突)

git reset HEAD~

git stash apply
```
