---
layout:     post
title:      "Git撤销合并"
date:       2017-01-23
author:     "秀川"
header-img: "img/post-bg-re-vs-ng2.jpg"
header-mask: 0.3
catalog:    true
tags:
    - git
    - 翻译
    - undo merge
    - 撤销
    - 还原
---

# 撤销合并

[原文地址](https://git-scm.com/blog/2010/03/02/undoing-merges.html)


我想开始在这里写更多关于git的一般的注意事项，技巧和即将实现的功能。自从这本书（<Pro Git>）发布以来，已经实现了很多很酷的东西，以及很多有趣的东西在这本书上没有涉及到。我想如果我开始把这些有趣的东西记录到博客上，它可以作为非常便利的入门指南。

对于第一篇类似的文章，我想解释我最近在做训练的时候被问到的问题。这个问题是关于开发了很长时间的分支的偶尔合并的流程问题，很像我在书上描述的[很大的合并](https://progit.org/book/ch5-3.html#largemerging_workflows)的流程问题。他们问到怎么临时撤销对分支的合并，之后再合并回来。

实际上，你可以通过一些步骤做这个事情。假设你的历史提交记录是这个样子的：
![图1](/img/in-post/1701/image1.png)

你有好几个已经开发好的不同主题的分支，以及通过一系列的合并整合到了一起。现在你想把历史记录中的某些内容回滚掉，假设要回滚到‘C10’。

第一个要解决的问题是把master分支倒回到C3，然后再次合并剩下的两个分支既可以了。这需要你的同事都知道怎么回退heads，如果那不是问题，那么这是完美可行的方案。这基本上就是Git项目本身如何处理‘pu’分支的流程。

```
$ git checkout master
$ git reset --hard [sha_of_C3]
$ git merge jk/post-checkout
$ git merge db/push-cleanup

```

一旦你回退并且重新合并了，那么你的历史提交记录可能是下面这个样子的：
![图2](/img/in-post/1701/unmerge2.png)

现在你可以退回去，并且在新的没有合并的分支上开发，然后再合并，或者考虑忽略这个分支。

## 回滚合并

然而，倘若你过了一段时间才发现在一系列的合并之后你或者你的同事已经做了一些提交，如果你的历史记录像下面这样的：
![图3](/img/in-post/1701/unmerge3.png)
现在你要么还原所有合并中的一个，要么退到所有合并之前，再重新合并，然后挑选剩余的变更（这个例子中的C10和C11），这种方法使人感到迷惑，困难，尤其是在那些合并之后又有很多提交。

事实上Git可以非常好的还原一个完整的合并。尽管我们很可能使用`git revert`命令来还原一个提交（如果你用过），你同样可以用于还原一个合并中的所有提交。

你所需要做的是指定你想还原的那个合并生成的commit-id以及你想保留哪一个父提交（使用`git cat-file -p commit-id`命令可以查看到这个commit-id的相应的信息，parent有两个，第一个就是1，第二个就是2）。假设我们想还原`jk/post-checkout`这个合并的分支，我们可以按照以下步骤来操作：

```
$ git revert -m 1 [sha_of_C9]
Finished one revert.
[master 88edd6d] Revert "Merge branch 'jk/post-checkout'"
 1 files changed, 0 insertions(+), 2 deletions(-)
```
这将引入一个新的提交，撤销起初合并分支的时候引起的变更-有点像从所有的提交中反向选择出属于那个分支的提交，非常酷。

![图4](/img/in-post/1701/unmerge4.png)
可是，我们还没有完成。

## 撤销还原
假设你想再重新合并那个分支。如果你尝试再次合并这个分支，Git会认为那个分支上的所有的提交已经在历史记录中并且会认为你的这次尝试合并已经存在的分支是一次误操作。

```
$ git merge jk/post-checkout
Already up-to-date.
```

哎呀，它什么都没有做。更让人迷惑的是如果你退回去了，并且在那个分支做了新的提交，然后尝试合并这个分支，它只会指明你最初合并之后更新的内容。（这段内容比较难以理解，如果master合并了A分支，然后又回退了，A分支有了新的提交，这个时候master再合并A分支，因为A分支已经在master的历史记录中了，所以master只会合并A分支最新提交的内容）
![图5](/img/in-post/1701/unmerge5.png)

现在这真是一个奇怪的状态并且很可能引发一堆的冲突和令人困惑的错误。然而，你可能想还原对合并的还原：

```
$ git revert 88edd6d
Finished one revert.
[master 268e243] Revert "Revert "Merge branch 'jk/post-checkout'""
 1 files changed, 2 insertions(+), 0 deletions(-)
```

![图6](/img/in-post/1701/unmerge6.png)
酷，我们现在基本上重新引入了这个分支之前还原出去的所有内容。现在如果我们在此期间在这个分支上做了很多工作，我们只要重新合并就可以了。

```
$ git merge jk/post-checkout
Auto-merging test.txt
Merge made by recursive.
 test.txt |    1 +
 1 files changed, 1 insertions(+), 0 deletions(-)
```
![图7](/img/in-post/1701/unmerge7.png)
所以，我希望这会很有帮助。如果我们有一个重合并的发展过程，这会特别有用。事实上，通常如果你在主题分支上工作，在你为了整合而合并之前，你可能想用`git merge --no-ff`选项，这样第一次合并就不是快进模式，可以用这种方式还原出分支信息。

下次见。

对于翻译有争议的地方，请在评论处留言，一起斟酌
