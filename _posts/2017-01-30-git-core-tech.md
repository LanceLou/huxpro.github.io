---
layout:     post
title:      "GIT总结之知识点与常用命令"
subtitle:   "Lance2016总结-GIT篇"
date:       2017-01-30 00:00
author:     "LanceLou"
header-img: "img/post-git-banner.png"
catalog: true
tags:
    - git
    - 工程化
    - 2016前端总结集
---

> distributed-is-the-new-centralized	&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;
&nbsp;&nbsp;&nbsp; — git-scm.com

## 序

作为Lance的2016技术总结集的开篇之作，也算是Lance blog的技术博客的处女作，献给伟大的GIT，以彰显其在软件工程管理方面，版本控制方面，附加开源方面所作出的贡献！

本文旨在总结GIT常用命令以及相关的基础知识，主要讨论git三个区间以及各区间之间切换的常用命令！从而可以应付一些基础的业务需求！如有不足之处，欢迎各位指正！！！

<br>

## 正文

#### three stages/sections
首先，我们引入git的三个分区（sections），不同地方可能有不同的汉译名！

* 工作目录（Working Directory）是对项目的某个版本独立提取出来的内容。 这些从 Git 仓库的压缩数据库中提取出来的文件，放在磁盘上供你使用或修改。
* 暂存区域（Staging）是一个文件，保存了下次将提交的文件列表信息，一般在 Git 仓库目录中。 有时候也被称作'索引(index)'，不过一般说法还是叫暂存区域。
* Git 仓库目录（Repo）是 Git 用来保存项目的元数据和对象数据库的地方。 这是 Git 中最重要的部分，从其它计算机克隆仓库时，拷贝的就是这里的数据。

![git three stages](/img/post-git-threeStages.png)


三个sections对应三种不同的文件状态

* 已提交（committed）: 表示数据已经安全的保存在本地数据库中。
* 已修改（modified）: 修改了文件，但还没保存到数据库中。
* 已暂存（staged）: 表示对一个已修改文件的当前版本做了标记，使之包含在下次提交的快照中。

而其实我们的常规的git的操作，就是在这三个sections之间来回的操作：

#### operation in stages/sections

![git three stages](/img/post-git-sectionTrans.png)

那么，依据上面图解，我们来逐一介绍git的常用命令(命令相关的参数在这不做详细介绍):

* ***git add files***: 将工作区中的files文件放入暂存区域，更新或添加暂存区域的对应文件对象的信息（文件名，文件状态，唯一id），将目标文件的内容做快照并保存到Git的对象库(objects)。工作区 -> 暂存区。
* ***git stash***: 在工作区进行了修改并add，但未完成修改，此时需要切换到别的分支(会使得修改被丢弃)，使用git stash暂存修改。
* ***git commit***: 将暂存区的tree添加到版本库，让git真正实现对files进行版本的记录和控制等。暂存区 -> Repo。
* ***git push***: 将修改推送带远程分支。local repo -> remote repo.
* ***git pull***: 取回远程主机某个分支的更新，再与本地的指定分支合并。remote repo -> 工作区。
* ***git checkout HEAD***: 从版本库(Repo)中检出当前HEAD指针指向的分支至工作区。local repo -> 工作区&暂存区。
* ***git checkout***: 对提交的代码进行回滚。local repo -> 工作区(丢弃本地的修改)&暂存区。
* ***git fetch***: 将远程主机的版本库的更新取回本地，通常用来查看其他人的进程，因为它取回的代码对我们本地的开发代码没有影响。remote repo -> local repo。
* ***git rebase***: 分支并入命令，将目标分支移动到master分支后面，直线型合并(重新创建提交)！而merge命令合并的结果是"带节点的直线"。
* ***git diff***: 可用来比较不同分区或相同分区的不同提交之间的差异。
* ***git revert***: 撤销一个分支的同时创建一个新的分支(不会重写提交历史)。
* ***cherry-pick***: 将commit从一个分支移动到另一个分支。

## 结语
2016前端总结集git篇到此就告一段落了，老实说，作者在这一年git的使用上深入的不多，但对其在实际项目中的使用的理解还是有那么一点进步，git确实一个不错的东西，配上github，那就更加完美了！是不，老铁。

## 参考链接
* [Difference between HEAD / Working Tree / Index in Git(stackoverflow)](http://stackoverflow.com/questions/3689838/difference-between-head-working-tree-index-in-git)
* [代码回滚：Reset、Checkout、Revert的选择](https://github.com/geeeeeeeeek/git-recipes/wiki/5.2-%E4%BB%A3%E7%A0%81%E5%9B%9E%E6%BB%9A%EF%BC%9AReset%E3%80%81Checkout%E3%80%81Revert%E7%9A%84%E9%80%89%E6%8B%A9)
* [代码合并：Merge、Rebase的选择](https://github.com/geeeeeeeeek/git-recipes/wiki/5.1-%E4%BB%A3%E7%A0%81%E5%90%88%E5%B9%B6%EF%BC%9AMerge%E3%80%81Rebase%E7%9A%84%E9%80%89%E6%8B%A9)
* [git diff 命令详解](http://www.jianshu.com/p/80542dc3164e)
* [Git远程操作详解](http://www.ruanyifeng.com/blog/2014/06/git_remote.html)
* [Git 工具 - 储藏](https://git-scm.com/book/zh/v1/Git-%E5%B7%A5%E5%85%B7-%E5%82%A8%E8%97%8F%EF%BC%88Stashing%EF%BC%89)
* [Git 常用命令详解（二）](http://blog.csdn.net/ithomer/article/details/7529022)
* [Git笔记(三)——[cherry-pick, merge, rebase]](http://pinkyjie.com/2014/08/10/git-notes-part-3/)

-- Lance 2017-01











