---
title: Git 命令
date: 2018-1-14 12:52:28
tags: Git
categories: Tools
---

{% blockquote Documentation https://git-scm.com/ git-scm.com %}
Git is a free and open source distributed version control system designed to handle everything from small to very large projects with speed and efficiency.
{% endblockquote %}

Git 是一个开源的分布式版本控制系统，用于敏捷高效地处理任何或小或大的项目。它是由 Linux 之父 Linus Torvalds 为了帮助管理 Linux 内核开发而开发的一个开放源码的版本控制软件。本文介绍了 Git 的常用命令。

## 三种状态

在学习 Git 命令之前，首先要理解它的三种状态：已提交（committed）、已修改（modified）和已暂存（staged）。已提交表示数据已经安全的保存在本地数据库中；已修改表示修改了文件，但还没保存到数据库中，增加、删除文件也相当于已修改；已暂存表示对一个已修改文件的当前版本做了标记，使之包含在下次提交的快照中。

由此引入 Git 项目的三个工作区域的概念：Git 仓库、工作目录以及暂存区域。

![Working tree, staging area, and Git directory](https://git-scm.com/book/en/v2/images/areas.png)

<!-- more -->

它们之间的关系可以参考 [Git 工作区、暂存区和版本库](http://www.runoob.com/git/git-workspace-index-repo.html)。

在阅读下面的内容之前，最好在自己的电脑上[安装 Git](https://git-scm.com/)，然后按照顺序操作。如果你想先感受一下 Git 的魅力， [Try Git](https://try.github.io/) 是一个不错的选择。

## Git 配置

安装完 Git，初次运行前需要做一些配置，比如用户信息：

```bash
git config --global user.name "Your Name"
git config --global user.email email@example.com
```

Windows 环境下，推荐使用文本编辑器 [Notepad++](https://notepad-plus-plus.org/)：

```bash
# 注意更改为自己的安装目录
git config --global core.editor "'C:\Program Files\Notepad++\notepad++.exe' -multiInst -nosession"
```

配置完成后，可以通过 `git config --list` 查看所有的配置信息，或者使用 `git config user.name` 查看单个信息。

另外，还可以自定义配置一些命令的别名，方便记忆。

```bash
git config --global alias.last 'log -1'
```

这样 `git last` 就相当于 `git log -1`，用于查看最后一次的提交记录。我比较喜欢这样配置，用于查看提交历史：

```bash
git config --global alias.lg "log --oneline --decorate --graph --all"
```

## 创建版本库

创建版本库有两种方式，一种是使用 `git clone` 从现有 Git 仓库中拷贝项目，格式如下：

```bash
git clone <repo> <directory>
```

另一种是通过 `git init` 初始化一个 Git 仓库，省略 directory 会在当前文件夹中创建。

```bash
git init <directory>
```

例如，在 `D:\test` 文件夹下执行 `git init` 命令，这样会生成一个 **隐藏** 的 .git 目录。

```bash
$ git init
Initialized empty Git repository in D:/test/.git/
```

## Git 工作流程

基本的 Git 工作流程如下：

1. 在工作目录中修改文件。

2. 暂存文件，将文件的快照放入暂存区域。

3. 提交更新，找到暂存区域的文件，将快照永久性存储到 Git 仓库目录。

使用 Git 时文件的生命周期如下：

![The lifecycle of the status of your files](https://git-scm.com/book/en/v2/images/lifecycle.png)

上图来源于 [Pro Git](https://git-scm.com/book/en/v2/Git-Basics-Recording-Changes-to-the-Repository)，这里的 `Add the file` 应该理解为使用 `git add` 命令，`Reomve the file` 则是手动删除文件。

### First Commit

在 `D:\test` 中手动添加 `a.txt` 文件，使用 Notepad++ 编辑（[不要用记事本](https://www.liaoxuefeng.com/wiki/0013739516305929606dd18361248578c67b8067c8c017b000/0013743256916071d599b3aed534aaab22a0db6c4e07fd0000)），然后运行 `git status` 命令，查看当前状态：

```bash
$ git status
On branch master

No commits yet

Untracked files:
  (use "git add <file>..." to include in what will be committed)

        a.txt

nothing added to commit but untracked files present (use "git add" to track)
```

Git 的提示十分人性化，可以看出 `a.txt` 处于 `Untracked` 状态。执行 `git add` 命令：

```bash
$ git add a.txt

$ git status
On branch master

No commits yet

Changes to be committed:
  (use "git rm --cached <file>..." to unstage)

        new file:   a.txt
```

此时 `a.txt` 处于 `Staged` 状态，可以通过 `git rm --cached <file>...` 使其回到 `Untracked` 状态。最后执行 `git commit` 命令，进行第一次提交。

```bash
$ git commit -m "Add a.txt"
[master (root-commit) 3fbc25c] Add a.txt
 1 file changed, 1 insertion(+)
 create mode 100644 a.txt
```

其中 `-m` 是参数，后面跟着提交信息。如果配置了文本编辑器，执行不带参数的 `git commit` 后，可在弹出的编辑器中填写提交信息。注意只有 **保存文件** 并 **退出编辑器**，commit 才会生效。

另外，在只 **修改文件** 时，使用 `-a` 可以跳过 `Staged` 状态直接提交，可以和 `-m` 一起使用：

```bash
git commit -am "Update file"
```

### Second Commit

添加 `b.txt`，然后修改 `a.txt`，查看此时的状态：

```bash
$ git status
On branch master
Changes not staged for commit:
  (use "git add <file>..." to update what will be committed)
  (use "git checkout -- <file>..." to discard changes in working directory)

        modified:   a.txt

Untracked files:
  (use "git add <file>..." to include in what will be committed)

        b.txt

no changes added to commit (use "git add" and/or "git commit -a")
```

此时 `a.txt` 处于 `Modified` 状态，可通过 `git checkout -- <file>...` 放弃更改，但是要 **慎用**，这些更改是找不回来的。 而 `b.txt` 处于 `Untracked` 状态。

`git diff` 命令用于比较工作目录中当前文件和暂存区域快照之间的差异，也就是修改之后还没有暂存起来的变化内容。

`git diff --cached`（Git 1.6.1 及更高版本还允许使用 `git diff --staged`，效果是相同的，但更好记些）可以查看已暂存的将要添加到下次提交里的内容。

```
$ git diff
diff --git a/a.txt b/a.txt
index 69dd9b9..b0c1f18 100644
--- a/a.txt
+++ b/a.txt
@@ -1 +1,2 @@
 aaaaaaaaaa    # a.txt 原本的内容
+AAAAAAAAAA    # a.txt 添加的内容
$ git diff --staged # nothing

```

添加这两个文件到暂存区：

```bash
$ git add "*.txt" # git add .(一个点，表示添加所有文件)

$ git status
On branch master
Changes to be committed:
  (use "git reset HEAD <file>..." to unstage)

        modified:   a.txt
        new file:   b.txt

```

同理，`git reset HEAD <file>...` 命令可使文件回到 add 之前的状态。此时再次执行 diff：

```bash
$ git diff # nothing

$ git diff --staged
diff --git a/a.txt b/a.txt
index 69dd9b9..b0c1f18 100644
--- a/a.txt
+++ b/a.txt
@@ -1 +1,2 @@
 aaaaaaaaaa
+AAAAAAAAAA
diff --git a/b.txt b/b.txt
new file mode 100644
index 0000000..817e5ca
--- /dev/null
+++ b/b.txt
@@ -0,0 +1 @@
+bbbbbbbbbb # b.txt 中添加的内容
```

以上对比可以看出不同 diff 的差别。执行 commit 命令进行第二次提交：

```bash
$ git commit -m "Update a.txt and add b.txt"
[master 24e0903] Update a.txt and add b.txt
 2 files changed, 2 insertions(+)
 create mode 100644 b.txt
```

### 删除

为了演示删除操作，先添加 `c.txt`：

```bash
$ git add c.txt

$ git commit -m "Add c.txt"
[master 9d8751a] Add c.txt
 1 file changed, 1 insertion(+)
 create mode 100644 c.txt
```

手动删除后的状态为 `Untracked`。

```bash
$ git status
Changes not staged for commit:
  (use "git add/rm <file>..." to update what will be committed)
  (use "git checkout -- <file>..." to discard changes in working directory)

        deleted:    c.txt

no changes added to commit (use "git add" and/or "git commit -a")
```

使用 `git checkout -- <file>...` 撤销，然后执行 `git rm` 命令，此时的状态为 `Staged`。这就是两者的差别吧。

```bash
$ git rm c.txt
rm 'c.txt'
$ git status
On branch master
Changes to be committed:
  (use "git reset HEAD <file>..." to unstage)

        deleted:    c.txt

```

提交删除：

```bash
$ git commit -m "Delete c.txt"
[master 95d6e7e] Delete c.txt
 1 file changed, 1 deletion(-)
 delete mode 100644 c.txt
```

如果删除之前修改过并且已经放到暂存区域的话，则必须要用强制删除 `git rm -f <file>`。 这是一种安全特性，用于防止误删还没有添加到快照的数据，这样的数据不能被 Git 恢复。

而 `git rm --cached` 命令只会将文件从 Git 仓库中删除，但仍然保留在当前工作目录中。当你忘记添加 `.gitignore` 文件，不小心把一个很大的日志文件或一堆无关的文件添加到暂存区时，这一做法尤其有用。

### 版本回退

在 Git 中，用 HEAD 表示当前版本，上一个版本就是 `HEAD^`（`HEAD~`）。有关 `~` 和 `^` 的区别，请参考 [What's the difference between HEAD^ and HEAD~ in Git?](https://stackoverflow.com/questions/2221658/whats-the-difference-between-head-and-head-in-git)

注意，Windows 环境下 `^` 识别不了，必须加 **双引号** 才行，像这样 `"HEAD^"`。

假如又要用到 `c.txt`，想反悔，怎么办？Git 允许我们在版本的历史之间穿梭，使用 `git reset --hard <commit_id>` 命令。如果不知道 commit_id，`git log` 可以查看提交历史。

```bash
$ git lg # 自定义的 git log
* 95d6e7e (HEAD -> master) Delete c.txt
* 9d8751a Add c.txt
* 24e0903 Update a.txt and add b.txt
* 3fbc25c Add a.txt

$ git reset --hard HEAD~
HEAD is now at 9d8751a Add c.txt

$ git lg
* 9d8751a (HEAD -> master) Add c.txt
* 24e0903 Update a.txt and add b.txt
* 3fbc25c Add a.txt
```

其中 `3fbc25c` 为版本号（commit_id），它是一个由 SHA-1 计算出来的校验和，用十六进制表示，而且每次都不一样。因为我使用了自定义的 `git lg`， 这里只显示 7 位，其实它是 `3fbc25c7d58e06169a45b587a9c6164234efd43c`。

`git log` 功能十分强大，可参考 [Git Basics - Viewing the Commit History](https://git-scm.com/book/en/v2/Git-Basics-Viewing-the-Commit-History)。

另外，可以使用命令 `git reflog` 查看命令历史。如果想回到 `Delete c.txt` 的版本，直接 reset 对应的 commit_id 即可。

```bash
$ git reflog
9d8751a (HEAD -> master) HEAD@{0}: reset: moving to HEAD~
95d6e7e HEAD@{1}: commit: Delete c.txt
9d8751a HEAD@{2}: commit: Add c.txt
24e0903 HEAD@{3}: commit: Update a.txt and add b.txt
3fbc25c HEAD@{4}: commit (initial): Add a.txt
```

### Reset

其实 reset 分三类，分别为 `--soft`、`--mixed`（默认，可不加）和 `--hard`，它们之间到底有什么区别呢？我们做个试验。注意此时是 `Add c.txt` 的版本。

#### soft

```bash
$ git reset --soft HEAD~

$ git status
On branch master
Changes to be committed:
  (use "git reset HEAD <file>..." to unstage)

        new file:   c.txt

```

`--soft` 参数使文件回到了 `Staged` 的状态。

#### mixed

重新回到 `Add c.txt` 的版本，执行 `--mixed` 命令：

```bash
$ git reset --mixed HEAD~

$ git status
On branch master
Untracked files:
  (use "git add <file>..." to include in what will be committed)

        c.txt

nothing added to commit but untracked files present (use "git add" to track)
```

`--mixed` 参数使文件回到了 `Untracked` 状态。

#### hard

重新回到 `Add c.txt` 的版本，执行 `--hard` 命令：

```bash
$ git reset --hard HEAD~
HEAD is now at 24e0903 Update a.txt and add b.txt
$ git status
On branch master
nothing to commit, working tree clean
```

而 `--hard` 参数直接回到了上一个版本。

想要了解更多关于 Reset 的知识，请参考 [Git Tools - Reset Demystified](https://git-scm.com/book/en/v2/Git-Tools-Reset-Demystified)。

## Git 分支

几乎每一种版本控制系统都以某种形式支持分支。使用分支意味着你可以从开发主线上分离开来，然后在不影响主线的同时继续工作。
有人把 Git 的分支模型称为它的“必杀技特性”，也正因为这一特性，使得 Git 从众多版本控制系统中脱颖而出。为何 Git 的分支模型如此出众呢？Git 处理分支的方式可谓是难以置信的轻量，创建新分支这一操作几乎能在瞬间完成，并且在不同分支之间的切换操作也是一样便捷。

下面演示了 Git 分支的工作流程。创建并切换到 dev 分支：

```bash
$ git branch dev

$ git checkout dev
Switched to branch 'dev'
```

简单地，这两个命令可以合并为一个命令：

```bash
$ git checkout -b dev
Switched to a new branch 'dev'
```

在 dev 分支添加 `d.txt`，修改 `c.txt`，提交：

```bash
$ git add .

$ git commit -m "Add d.txt and update c.txt"
[dev 7f5d2b1] Add d.txt and update c.txt
 2 files changed, 2 insertions(+), 1 deletion(-)
 create mode 100644 d.txt
```

切换到 master 分支，合并 dev 分支：

```bash
$ git checkout master

$ git merge dev
Updating 9d8751a..7f5d2b1
Fast-forward
 c.txt | 2 +-
 d.txt | 1 +
 2 files changed, 2 insertions(+), 1 deletion(-)
 create mode 100644 d.txt
```

最后删除 dev 分支：

```
$ git branch -d dev
Deleted branch dev (was 7f5d2b1).
```

## Git 远程仓库

为了能在任意 Git 项目上协作，需要知道如何管理自己的远程仓库。远程仓库是指托管在因特网或其他网络中的你的项目的版本库。 你可以有好几个远程仓库，通常有些仓库对你只读，有些则可以读写。与他人协作涉及管理远程仓库以及根据需要推送或拉取数据。

这里以 GitHub 为例，演示如何使用远程仓库。在 GitHub 上创建一个新的 Repository，不要添加任何内容，完成后如下图所示：

{% qnimg git/github.png  %}

添加远程仓库：

```bash
git remote add origin git@github.com:muwednesday/git-learning.git
```

使用命令 `git push` 将本地仓库推送到 GitHub，其中 `-u` 为设置当前本地分支的默认远程分支。

```bash
$ git push -u origin master
Counting objects: 14, done.
Delta compression using up to 8 threads.
Compressing objects: 100% (7/7), done.
Writing objects: 100% (14/14), 913 bytes | 304.00 KiB/s, done.
Total 14 (delta 2), reused 0 (delta 0)
remote: Resolving deltas: 100% (2/2), done.
To github.com:muwednesday/git-learning.git
 * [new branch]      master -> master
Branch 'master' set up to track remote branch 'master' from 'origin'.
```

刷新页面后即可看到文件。然后在 GitHub 上创建一个 `README.md` 的文件，提交。

返回本地仓库，查看状态，这里居然显示 `up to date`。本来应该落后才对，为什么呢？原因参见 [Why does git status show branch is up-to-date when changes exist upstream?](https://stackoverflow.com/questions/27828404/why-does-git-status-show-branch-is-up-to-date-when-changes-exist-upstream)

```bash
$ git status
On branch master
Your branch is up to date with 'origin/master'.

nothing to commit, working tree clean
```

最后使用命令 `git pull` 来自动的抓取然后合并远程分支到当前分支。

```
$ git pull
remote: Counting objects: 3, done.
remote: Compressing objects: 100% (2/2), done.
remote: Total 3 (delta 1), reused 0 (delta 0), pack-reused 0
Unpacking objects: 100% (3/3), done.
From github.com:muwednesday/git-learning
   7f5d2b1..ad27849  master     -> origin/master
Updating 7f5d2b1..ad27849
Fast-forward
 README.md | 1 +
 1 file changed, 1 insertion(+)
 create mode 100644 README.md
```

## 推荐阅读

* [Pro Git](https://git-scm.com/book/en/v2)

* [GitHub Cheat Sheet](https://services.github.com/on-demand/downloads/github-git-cheat-sheet.pdf)

* [Reference Manual](https://git-scm.com/doc)

* [廖雪峰的 Git 教程](https://www.liaoxuefeng.com/wiki/0013739516305929606dd18361248578c67b8067c8c017b000)

* [What's the difference between HEAD^ and HEAD~ in Git?](https://stackoverflow.com/questions/2221658/whats-the-difference-between-head-and-head-in-git)

* [Reset, Checkout, and Revert](https://www.atlassian.com/git/tutorials/resetting-checking-out-and-reverting)

* [What is the difference between 'git pull' and 'git fetch'?](https://stackoverflow.com/questions/292357/what-is-the-difference-between-git-pull-and-git-fetch)