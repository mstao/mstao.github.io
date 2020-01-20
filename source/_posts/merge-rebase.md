---
title: Git不完全使用指南 - merge和rebase
categories: Git
tags: [Git, merge, rebase]
author: Mingshan
date: 2018-12-23
---

我们知道Git 的分支模型称为它的‘必杀技特性’，很多团队有自己的Git Flow来管理分支。Git 鼓励在工作流程中频繁地使用分支与合并，所以我们就无法避免在开发中合并分支，这也是git的魅力之一，下面就探讨一下git的分支合并（这里先不涉及规范的Git Flow）。

<!-- more -->

## git merge

`git merge`命令用于合并指定分支到当前分支。比如你现在在开发一个新功能，理论上是不能直接在master上进行开发的，所以我们需要基于master分支创建一个新分支`myfeature1`，命令如下：

```
$ git checkout -b myfeature1 #创建+切换分支
```

上面这个命令作用为创建和切换`myfeature1`分支，它等同于下面两条命令：


```
$ git branch myfeature1 创建分支

$ git checkout myfeature1 #切换分支
```

创建完分支后，我们查看下分支树，命令如下：

```
$ git log --graph --decorate --oneline --simplify-by-decoration --all
```
然后显示：

```
$ git log --graph --decorate --oneline --simplify-by-decoration --all
* bf8370f (HEAD -> myfeature1, origin/master, master) My Update 1.md
| * 846b1a0 (refs/stash) WIP on master: 409be61 Initial commit
|/
* 409be61 Initial commit
```

接下来修改我们的代码，然后提交到本地`myfeature1`分支，
```
$ git add .
$ git commit -m 'My Update3 1.md
```

功能开发完了，我们可以选择先提交到远程的`myfeature1`分支
```
$ git push -f origin myfeature1
```

这时再看下分支树：


```
$ git log --graph --decorate --oneline --simplify-by-decoration --all
* 43fa02b (HEAD -> myfeature1, origin/myfeature1) My Update3 1.md
* bf8370f (origin/master, master) My Update 1.md
| * 846b1a0 (refs/stash) WIP on master: 409be61 Initial commit
|/
* 409be61 Initial commit
```

下面就要将刚开发的`myfeature1`分支上的功能合并到master分支，这里模拟下文件的冲突，刚才我们在`myfeature1`修改`1.md`文件的时候，有人已经在`master`分支修改了`1.md`文件，这样我们将代码合并到`master`的时候必然就会冲突了。

首先切换到`master`分支

```
$ git checkout master
```

然后拉取远程的`master`分支的代码

```
$ git pull origin master
```

拉完代码，我们再看下分支树：

```
$ git log --graph --decorate --oneline --simplify-by-decoration --all
* 43fa02b (origin/myfeature1, myfeature1) My Update3 1.md
| * 2b43486 (HEAD -> master, origin/master) Other Update3 1.md
|/
| * 846b1a0 (refs/stash) WIP on master: 409be61 Initial commit
|/
* 409be61 Initial commit

```
发现别人已经在`master`分支提交了`Other Update3 1.md`，这时我们尝试合并`myfeature1`分支的代码，这时会提示代码冲突：

```
$ git merge myfeature1
Auto-merging 1.md
CONFLICT (content): Merge conflict in 1.md
Automatic merge failed; fix conflicts and then commit the result.
```

这时我们就执行如下命令来解决冲突：

```
$ git mergetool
```
然后进入到解决冲突的GUI界面（具体操作细节参考：[Git不完全使用指南 - 合并冲突(mergetool)](https://mingshan.fun/2018/12/22/git%E4%B8%8D%E5%AE%8C%E5%85%A8%E4%BD%BF%E7%94%A8%E6%8C%87%E5%8D%97%20-%20%E5%90%88%E5%B9%B6%E5%86%B2%E7%AA%81%28mergetool%29/)）：

![image](https://github.com/ZZULI-TECH/interview/blob/master/images/git/bc_overview2.png?raw=true)


合并完冲突后，提交代码就是了


```
$ git add .
$ git commit -m 'Merge myfeature1 into master'
$ git push origin master
```

此时我们再看一下分支树：

```
$ git log --graph --pretty=oneline --abbrev-commit
*   1506ce8 (HEAD -> master, origin/master) Merge myfeature1 into master
|\
| * 43fa02b (origin/myfeature1, myfeature1) My Update3 1.md
* | 2b43486 Other Update3 1.md
|/
* bf8370f My Update 1.md
* 9ef0adf Update 1.md
* 409be61 Initial commit
```

**参考命令：**

```
git branch #查看分支

git branch <name> 创建分支

git checkout <name> #切换分支

git checkout -b <name> #创建+切换分支

git merge <name> #合并某分支到当前分支

git branch -d <name> # 删除分支
```

## git rebase

上面我们用了`git merge` 合并了代码，那么`git rebase`是干嘛的？`rebase`被称为变基，在 Git 中整合来自不同分支的修改主要有两种方法：merge 以及 rebase。那么接下来我们使用下rebase吧，看看它与merge有什么不同。

首先基于`master`分支创建一个`myfeature2`分支，

```
$ git checkout -b myfeature2
```

此时其他人在master分支又提交了代码，而我们的`myfeature2`分支是没有master分支的代码，所以我们先将修改的代码贮藏，执行如下命令：

```
$ git stash
```

然后执行变基操作将master的代码合并到`myfeature2`

```
$ git fetch origin

$ git rebase origin/master
```

接下来会发生什么呢？我们来看看分支树


```
$ git log --graph --pretty=oneline --abbrev-commit
* d7160d5 (HEAD -> myfeature2, origin/master) Create 2.md
*   1506ce8 (master) Merge myfeature1 into master
|\
| * 43fa02b (origin/myfeature1, myfeature1) My Update3 1.md
* | 2b43486 Other Update3 1.md
|/
* bf8370f My Update 1.md
* 9ef0adf Update 1.md
* 409be61 Initial commit
```

发现了什么，`d7160d5 Create 2.md`这个提交居然被整到了`myfeature2`分支上，

注意我们本地的修改还没有commit， 下面我们把我们的代码commit到本地

```
$ git stash apply

$ git add . 

$ git commit -m 'My Update4 1.md'
```

再查看分支树

```
$ git log --graph --pretty=oneline --abbrev-commit
* e581488 (HEAD -> myfeature2) My Update4 1.md
* d7160d5 (origin/master) Create 2.md
*   1506ce8 (master) Merge myfeature1 into master
|\
| * 43fa02b (origin/myfeature1, myfeature1) My Update3 1.md
* | 2b43486 Other Update3 1.md
|/
* bf8370f My Update 1.md
* 9ef0adf Update 1.md
* 409be61 Initial commit
```

现在我们的代码已经提交到了本地仓库，假设当我commit之前又有人向master提交了代码，即在d7160d5 之后，e581488 之前 有了一个新的提交（master分支），此时我们需要再进行rebase，

```
$ git fetch origin

$ git rebase origin/master
```

此时又发生了什么？ 我们还是查看分支树：

```
$ git log --graph --pretty=oneline --abbrev-commit
* 72f4523 (HEAD -> myfeature2) My Update4 1.md
* 71a0090 (origin/master) Create 3.md
* d7160d5 Create 2.md
*   1506ce8 (master) Merge myfeature1 into master
|\
| * 43fa02b (origin/myfeature1, myfeature1) My Update3 1.md
* | 2b43486 Other Update3 1.md
|/
* bf8370f My Update 1.md
* 9ef0adf Update 1.md
* 409be61 Initial commit
```

有没有发现我们的`My Update4 1.md` 的提交的祖先变了，不再基于`d7160d5 Create 2.md`, 而是基于 `71a0090 (origin/master) Create 3.md` , 并且commit id也变了，即Git把我们本地的提交“挪动”了位置，现在是一条干净的直线。

好了，到这里我们就知道了rebase是干嘛的了，**它的作用将一个分支的提交历史，在当前分支的基础上应用一次，即将提交到某一分支上的所有修改都移至另一分支上，并且可能修改当前分支的提交信息（commit id）。**

### Golden Rule of Rebase

所以rebase有一个黄金法则：

> “No one shall rebase a shared branch” — Everyone about rebase

这句话告诉我们：绝不要在公共的分支上使用它！在你运行 git rebase 之前，一定要问问你自己「**有没有别人正在这个分支上工作？**」。如果答案是肯定的，那就不能rebase。

为啥呢？刚才我们也看到了，rebase会把本地提交的commit 移动位置，修改commit id， 如果在一个大家都用的分支上，岂不是乱了套，没办法追踪原始的提交历史了。

### 收尾

我们将修改推送到远程`myfeature2`分支，命令如下：

```
$ git push -f origin myfeature2
```

然后创建个pull request

![image](https://github.com/ZZULI-TECH/interview/blob/master/images/git/git_test_merge_request.png?raw=true)

合并后我们切换到master分支，并拉取远程代码:

```
$ git checkout master
$ git pull
```

然后我们再看看分支树：


```
$ git log --graph --pretty=oneline --abbrev-commit
*   1d165e9 (HEAD -> master, origin/master) Merge pull request #1 from mstao/myfeature2
|\
| * 72f4523 (origin/myfeature2, myfeature2) My Update4 1.md
|/
* 71a0090 Create 3.md
* d7160d5 Create 2.md
*   1506ce8 Merge myfeature1 into master
|\
| * 43fa02b (origin/myfeature1, myfeature1) My Update3 1.md
* | 2b43486 Other Update3 1.md
|/
* bf8370f My Update 1.md
* 9ef0adf Update 1.md
* 409be61 Initial commit
```


是我们想要的效果呢。。

完结，撒花花(〃’▽’〃)


## References：

- [git-merge](https://git-scm.com/docs/git-merge)
- [git-rebase](https://git-scm.com/docs/git-rebase)
- [解决冲突](https://www.liaoxuefeng.com/wiki/0013739516305929606dd18361248578c67b8067c8c017b000/001375840202368c74be33fbd884e71b570f2cc3c0d1dcf000)
- [Git 分支 - 变基](https://git-scm.com/book/zh/v2/Git-%E5%88%86%E6%94%AF-%E5%8F%98%E5%9F%BA)
- [Git Rebase原理以及黄金准则详解](https://segmentfault.com/a/1190000005937408)


[<font size=3 color="#409EFF">向本文提出修改或勘误建议</font>](https://github.com/mstao/mstao.github.io/blob/hexo/source/_posts/merge-rebase.md)
