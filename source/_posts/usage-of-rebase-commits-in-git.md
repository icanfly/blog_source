title: 关于Git合并多个提交的做法：rebase
date: 2017-06-15
tags:
 - git
categories:
 - 原创文章
---

在使用Git作为版本管理的过程中，在一个分支上开发很久了以后，当你回顾以前的提交时，会发现以前的提交非常杂乱而且提交的日志标注不清时，这个时候你就会非常想重新整理一下你的提交了。将多个提交日志合并成一个提交，而这个操作Git提供了这样的方法： `git rebase -i`

<!-- more -->

一个简单的例子：

对于一个已经存在的Git仓库，我们向其中添加一个文件：
```shell
$touch a.txt
$git add a.txt
$git commit -m "add a.txt"
$touch b.txt
$git add b.txt
$git commit -m "add b.txt"
$git add c.txt
$git commit -m "add c.txt"
$git push
```
那么提交历史是这样的：

```
commit f03350c8981c2df08d9443a9befb6f47aeedceb5 (HEAD -> test, origin/test)
Author: lpwork <lpwork@foxmail.com>
Date:   Mon Jun 15 11:58:37 2017 +0800

    add c.txt

commit 239c69ef6e5ac8a2716bd7f561a6b6aa8fd52074
Author: lpwork <lpwork@foxmail.com>
Date:   Mon Jun 15 11:58:17 2017 +0800

    add b.txt

commit 5008e5acb3b437276e3f1796de09071e1ccd7532
Author: lpwork <lpwork@foxmail.com>
Date:   Mon Jun 15 11:57:33 2017 +0800

    add a.txt
```

假如我们想要合并提交文件c.txt和b.txt的两次提交，我们的做法是：

 - 取得合并提交的前一个提交的id，这里我们取得的是`5008e5acb3b437276e3f1796de09071e1ccd7532`
 - 执行git rebase

```shell
git rebase -i 5008e5acb3b437276e3f1796de09071e1ccd7532
```
显示如下：
```
pick 239c69e add b.txt
pick f03350c add c.txt

# Rebase 5008e5a..f03350c onto 5008e5a (2 commands)
#
# Commands:
# p, pick = use commit
# r, reword = use commit, but edit the commit message
# e, edit = use commit, but stop for amending
# s, squash = use commit, but meld into previous commit
# f, fixup = like "squash", but discard this commit's log message
# x, exec = run command (the rest of the line) using shell
# d, drop = remove commit
#
# These lines can be re-ordered; they are executed from top to bottom.
#
# If you remove a line here THAT COMMIT WILL BE LOST.
#
# However, if you remove everything, the rebase will be aborted.
#
# Note that empty commits are commented out
```

这里我们选择保留如下：

```
pick 239c69e add b.txt and c.txt
squash f03350c add c.txt

# Rebase 5008e5a..f03350c onto 5008e5a (2 commands)
#
# Commands:
# p, pick = use commit
# r, reword = use commit, but edit the commit message
# e, edit = use commit, but stop for amending
# s, squash = use commit, but meld into previous commit
# f, fixup = like "squash", but discard this commit's log message
# x, exec = run command (the rest of the line) using shell
# d, drop = remove commit
#
# These lines can be re-ordered; they are executed from top to bottom.
#
# If you remove a line here THAT COMMIT WILL BE LOST.
#
# However, if you remove everything, the rebase will be aborted.
#
# Note that empty commits are commented out
```

:wq保存之后，得到如下：

```
# This is a combination of 2 commits.
# This is the 1st commit message:

add b.txt

# This is the commit message #2:

add c.txt

# Please enter the commit message for your changes. Lines starting
# with '#' will be ignored, and an empty message aborts the commit.
#
# Date:      Mon Aug 21 16:43:14 2017 +0800
#
# interactive rebase in progress; onto 5008e5a
# Last commands done (2 commands done):
#    reword 239c69e add b.txt and c.txt
#    squash f03350c add c.txt
# No commands remaining.
# You are currently rebasing branch 'test2' on '5008e5a'.
#
# Changes to be committed:
#       new file:   b.txt
#       new file:   c.txt
#
```
这里就是我们要修改的合并的提交日志了。我们修改成如下：
```
add b.txt and c.txt

# Please enter the commit message for your changes. Lines starting
# with '#' will be ignored, and an empty message aborts the commit.
#
# Date:      Mon Aug 21 16:43:14 2017 +0800
#
# interactive rebase in progress; onto 5008e5a
# Last commands done (2 commands done):
#    reword 239c69e add b.txt and c.txt
#    squash f03350c add c.txt
# No commands remaining.
# You are currently rebasing branch 'test2' on '5008e5a'.
#
# Changes to be committed:
#       new file:   b.txt
#       new file:   c.txt
#
```
保存并提交。再次查看git提交日志时：
```
commit 14d341cd6eddf4e8b04502bcf261d9becfbedd55 (HEAD -> test, origin/test)
Author: lpwork <lpwork@foxmail.com>
Date:   Mon Jun 15 12:23:14 2017 +0800

    add b.txt and c.txt

commit 5008e5acb3b437276e3f1796de09071e1ccd7532
Author: lpwork <lpwork@foxmail.com>
Date:   Mon Jun 15 11:57:33 2017 +0800

    add a.txt
```

注意此时我们的rebase还只是在我们自己的本地进行了合并，最后还需要push到远程仓库中。

> 注意： 这里需要说明的是，在很大部分情况下，在本地进行了rebase操作后，push远程仓库都有可能会造成冲突而无法提交。这里的一个做法是使用强制push `git push --force`。
注意，强制push可能会造成别人的提交丢失，请确保你在push前合并完别人的push结果`git pull`。

通过这样的方式，可以让你重新整理杂乱无章的提交历史，让提交历史更加清晰，同时可以清除一些无用的提交信息，提高历史查询的效率。

最后需要关注一下rebase提供的一些可选项：

```
#
# Commands:
# p, pick = use commit
# r, reword = use commit, but edit the commit message
# e, edit = use commit, but stop for amending
# s, squash = use commit, but meld into previous commit
# f, fixup = like "squash", but discard this commit's log message
# x, exec = run command (the rest of the line) using shell
# d, drop = remove commit
#
# These lines can be re-ordered; they are executed from top to bottom.
#
# If you remove a line here THAT COMMIT WILL BE LOST.
#
# However, if you remove everything, the rebase will be aborted.
#
# Note that empty commits are commented out
```