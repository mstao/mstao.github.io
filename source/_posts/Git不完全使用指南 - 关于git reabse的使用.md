---
title: Git不完全使用指南 - 关于git reabse的使用
categories: Git
tags: [Git, rebase]
author: Mingshan
date: 2018-3-15
---

由于在工作中用到了git rebase命令，所以记录一下。

比如当前在location-scan 分支做一个新功能，当新功能做完了，然后发pull request请求合并到develop分支，但在你提交pull request 之前，有人改动了develop分支的代码，导致你的代码与develop分支的代码发生了冲突， 由于有冲突，需要从develop分支将代码拉到location-scan 分支，进行代码的合并，然后再进行提交，此时的提交没有合并过的痕迹，所以此时我们就需要用到了git rebase命令了，具体的使用的流程：

<!-- more -->

1. 拉取远程develop分支代码，并与当前分支的代码合并


```
git fetch
```

```
git rebase origin/develop
```


2. 添加代码到暂存区


```
git add .
```


3. 如果让git继续应用(apply)余下的补丁，那么就用--continue参数


```
git rebase --continue
```


4. 如果想让git放弃此次合并，那么就用--abort参数来终止rebase的动作


```
git rebase --abort
```


5. 如果你想多次的提交都有第一次的提交合并，那么就用--amend参数


```
git commit --amend (合并commit)
```


6. 如果需要 将文件从暂存区取消，那么执行以下命令


```
git reset HEAD <file>
```
