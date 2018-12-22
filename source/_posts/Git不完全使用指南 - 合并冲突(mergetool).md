---
title: Git不完全使用指南 - 合并冲突(mergetool)
categories: Git
tags: [Git, mergetool]
author: Mingshan
date: 2018-12-22
---

对于刚开始使用Git的小伙伴来说，看到冲突的代码手就会莫名的抖一下，心里会很紧张。不过代码冲突是比较正常的，特别是多人合作开发的时候，所以遇到代码冲突不要慌张，选择顺手的工具合并代码也是很快的（GUI 推荐使用Sourcetree），下面以[`Beyond Compare 4`](https://www.scootersoftware.com/download.php)为例来合并冲突。 

<!-- more -->

## 配置 Git mergetool & difftool 

首先下载[Beyond Compare 4](https://www.scootersoftware.com/download.php) )的Windows的中文版本，选择一个目录安装，完成后需要将Beyond Compare 4作为git的difftool 和 mergetool，配置如下：

```
#difftool 配置

git config --global diff.tool bc3
git config --global difftool.bc3.path "F:/develope/Beyond Compare 4/bcomp.exe"

#mergeftool 配置
git config --global merge.tool bc3
git config --global mergetool.bc3.path "F:/develope/Beyond Compare 4/bcomp.exe"

#让git mergetool不再生成备份文件（*.orig）
git config --global mergetool.keepBackup false

```

上面的安装路径替换为自己的即可，注意是`bcomp.exe`这个exe可执行文件而不是其他的，不要配错了。

## 解决冲突


### 模拟冲突情况

这里以一个示例仓库`git-test`为例，该仓库目前只有一个文件`1.md`，内容如下：

```
我添加的1
```

我们将这个仓库克隆到本地，在本地将`1.md`修改为：

```
我添加的1
我添加的2
```
注意上面的修改只是在本地修改的，并未push到远程。

然后我们去远程仓库将`1.md`修改为（可以通过github的界面修改，或者是其他人修改的）：

```
# 我添加的1
# 其他人添加的2
```

此时我们本地的代码和远程的代码就存在冲突，我们如果想将本地代码push到远程仓库，就必须要解决冲突了。

### 解决冲突步骤

1. 首先我们先将本地代码储藏起来，使工作目录变干净，执行如下命令：

```
git stash
```
2. 然后拉取远程代码：

```
git pull origin master
```
3. 接着应用刚才储藏的代码，与刚刚从远程拉下来的代码合并：

```
git stash apply
```
注意此时会提示文件冲突了，我们接着需要调用`mergetool`来解决冲突：

```
git mergetool
```
4. 接着就会一个一个地合并代码，你会看到命令行如下输出：
```
$ git mergetool
Merging:
1.md

Normal merge conflict for '1.md':
  {local}: modified file
  {remote}: modified file
```
然后git帮你调出`Beyond Compare`的合并界面了，如下图所示：

![image](https://github.com/ZZULI-TECH/interview/blob/master/images/git/bc_overview.png?raw=true)

上面的图是不是有点眼花，别急，Beyond Compare的界面主要分为三个部分，最上面的是工具栏，包含了很多操作；

中间部分又分三部分：依次是“LOCAL”、“BASE”、“REMOTE”，它们只是提供解决冲突需要的信息，是无法编辑的，那么，这几个是啥意思呢？


名字 | 含义
---|---
BASE | 需要合并的两个文件的最近的共同祖先版本
LOCAL | 要合并的分支上的文件
Remote | 我们从服务器将上一次提交拉到本地的文件


最下方为合并之后的文件输出区域。

注意上面用红色框圈出来的部分，我们可以使用这几个按钮来快速合并,操作如下：

- 点击“采用左边部分”按钮，即冲突部分选择左侧窗格的内容作为最终输出文本。
- 点击“采用右边部分”按钮，即冲突部分选择右侧窗格的内容作为最终输出文本。
- 点击“采用中心选择内容”按钮，即冲突部分选择中心窗格的内容作为最终输出文本。
- 在文本合并输出窗格存在冲突行，单击工具栏“编辑”按钮，直接手动修改差异文本。

将冲突的部分逐行合并完成后，点击最终文件输出区域的右上角的保存按钮，保存后退出，如果有冲突可以继续合并。

接下来把代码提交到远程仓库就可以了。

## Refreneces:

- [BC version 4](http://www.scootersoftware.com/support.php?zz=kb_vcs#gitwindows)
- [冲突部分Beyond Compare如何解决](http://www.beyondcompare.cc/jiqiao/chongtu-bufen.html)
- [Beyond Compare如何进行文本合并](http://www.beyondcompare.cc/jiqiao/wen-ben-hebing.html)
- [Git 工具 - 储藏（Stashing）](https://git-scm.com/book/zh/v1/Git-%E5%B7%A5%E5%85%B7-%E5%82%A8%E8%97%8F%EF%BC%88Stashing%EF%BC%89)
- [In a git merge conflict, what are the BACKUP, BASE, LOCAL, and REMOTE files that are generated?](https://stackoverflow.com/questions/20381677/in-a-git-merge-conflict-what-are-the-backup-base-local-and-remote-files-that/20382333)
- [Which version of the git file will be finally used: LOCAL, BASE or REMOTE?](https://stackoverflow.com/questions/11133290/which-version-of-the-git-file-will-be-finally-used-local-base-or-remote)
- [Git Branching - Basic Branching and Merging](https://git-scm.com/book/en/v2/Git-Branching-Basic-Branching-and-Merging)