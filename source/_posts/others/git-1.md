---
title: Git实践整理
date: 2019/6/1 14:25
categories:
- [git]
tags:
- 知识点整理
---
&emsp;&emsp;都2019-6-1了，不要再说你只会 add commit push 和 pull 了。
<!--more-->
# 前言
&emsp;&emsp;Git是一个我们最常使用用代码管理工具，它的命令操作快速而又优雅，分支开发特性形成的独特git工作流模式也非常符合实际工作开发需求。这篇文章用于整理汇总我在实际工作过程中遇到的分支操作处理情况。

&emsp;&emsp;整理前汇总几个概念：

1. hash算法
&emsp;&emsp;hash算法是一个开源的文件加密算法，它的特性在于不管多大文件的输入，文件在**没有变动**的情况下，总是得到相同的输出。而文件只要变动一点，它得出的hash值就完全不同。hash 算法具有不可逆性。Git 当中使用 hash 算法作为 git 的 文件变动commit成的快照的版本标识。

2. git的**分支**和 **commit** 的操作实质上是对**文件快照(snapshot)**的处理
&emsp;&emsp;我们在工作区的 add&commit 操作，本质上是保存了当前文件的一组快照。我们所做的分支切换改变工作区内容，本质上是移动本地的HEAD指针指向快照的位置，从而复现不同的工作区内容。
![git三区交互](/blog/public/imgs/git-snapshot.png)

3. 本地与远程 HEAD 和 origin
&emsp;&emsp;前文提过，分支的本地移动，实质上是移动 git 文件当中的 HEAD 指针指向文件的快照。所以HEAD所代表的的就是本地的**当前操作的分支**。我们经常操作过程中提到的 origin 实际上是本地 git 文件**关联**的远程仓库的昵称。origin是你在关联远程仓库时候没有起名字时git默认的名称。因此再使用 git fetch origin/master 意思就是拉取 git 关联的名为 origin 远程仓库下面的master分支到本地。可以使用git remote来查看本地所关联的远程仓库别名。

# 穿梭在 git 时空，reset 的妙用
&emsp;&emsp;`git reset reset方式参数 log-hash值`，可以归置文件到对用的 commit 历史处。对应的 hash 值可以使用`git log`获得。
`git reset` 拥有三个参数 `--hard --soft --mixed`，他们分别代表了reset的三种模式。
- `--hard` 重置index和working tree
- `--soft` 不会触碰working tree(工作区)和暂存区(index),（即不会改变文件内容），只是移动指针
- `--mixed` 重置暂存区，不会触碰过工作区

利用`--soft`不会改变当前工作区文件的特性，当你当前分支开发的 commit 太多想要删除一些commit的时候，可以使用soft重置到想要删除的commit之前，再使用 `git add & git commit` 来规整 commit 历史。
&emsp;&emsp;比如这里我输入一下 git log，发现工作分支残留的 commit 太多：
```bash
commit 7a3e18e3e4ac6a8b3f95fe8c073cf6cc0fcc9d1b
Author: tangding12 <568783010@qq.com>
Date:   Sun May 12 22:07:50 2019 +0800

    change img

commit 31cef3d7f850c3f3cf21d3ae4357f1dee3d145bb
Author: tangding12 <568783010@qq.com>
Date:   Sun May 12 21:58:36 2019 +0800

    add github url

commit da004222bddb1a99f9ed0e02c70b2173ad85149e
Author: tangding12 <568783010@qq.com>
Date:   Sun May 12 21:55:47 2019 +0800

    start new chap of sicp

commit c6c29f8f825d24c4d1a9de0b72c56295022239ab
Author: tangding12 <568783010@qq.com>
```
&emsp;&emsp;上面的commit中 `add github url change img` 这两个小的commit改动，我不想残留，我就直接 `git reset --soft c6c29f8f825d24c4d1a9de0b72c56295022239ab`跳转到 start new chap of sicp 这个 commit 上面，然后使用 `git add & git commit`。这样我就能在保留当前文件变化的同时删除掉不要的一些微小的commit，以便影响我后续的操作。


# 重命名远程分支本质是删除远程分支再重新推送
&emsp;&emsp;远程分支没有重命名操作，只有本地分支才能使用 `git branch -m oldBranchName newBranchName` 来重命名分支。远程分支重命名本质是使用 `git push origin :oldBranchName` 删除你需要重命名的分支，然后在使用 `git push --set-upstream` 推送一个新的分支名到远程仓库。

# 优雅的合入分支，用 rebase 来代替 merge
&emsp;&emsp;首先明确，分支的合入操作，永远是合入到当前所在的分支(HEAD)。因此再处理时候冲突时候，HEAD标识的永远是当前分支的内容，指定合入的分支永远是传入的内容。
&emsp;&emsp;使用 `git merge` 操作的时候，git 会帮你当前所在分支自动创建一个快照文件并用以提交。

![git merge iss53](/blog/public/imgs/git-merg.png)

&emsp;&emsp;如图，当我在master分支的时候使用git merge iss53，git会帮我合入分支并自动自动创建一个到c2的文件快照。此时不难发现一个问题，那就是在iss53上面的提交commit 和 master 上面的log，就像两条平行线一样毫不相关。如果把iss53上面的所有git log能够同步到master分支上面，使得**合入分支的 git log **能够完整的记录和呈现，不是更全面更优雅吗？

&emsp;&emsp;使用 `git rebase` 可以保证提交历史的整洁。——例如贡献代码时，你首先在自己的分支里进行开发，当开发完成时你需要先将你的代码变基到 origin/master 上，然后再向主项目提交修改。 这样的话，该项目的维护者就不再需要进行整合工作，只需要快进合并便可。

&emsp;&emsp;简单来说 merge 和 rebase 最终的结果相同，唯一不同的就是分支合并的处理方式以及当前分支的 git log。温馨提示，git rebase完毕提交的时候，如果是你单人开发的场景，记得使用`git push -f`哦,千万不要照常在push前习惯性的 `git pull` 一下，这是我血与泪的教训。

# 危险的pull，保险的fetch
&emsp;&emsp;众所周知，`git pull = git fetch + git merge`，虽然git pull自动帮你合入当前分支的线上最新代码是一件非常畅快的事情，但是在实际开发过程当中是非常危险的。

```bash
master
develop
    |
    ------ 你所在的分支
    ------ 你的队友
```
&emsp;&emsp;实际开发过程当中，大多数是遵循git工作流的哲学(如上示意)。有一个维稳的 develope 分支来并行 master，而我们自己功能的开发分支均是基于 develope。经常你的队友完成一些**通用模块开发后**，会**先于你合入 develop 分支**。这种情况下你经常需要 merge 或者 rebase 一下最新的 develop 分支。此时如果你使用 git pull 操作去拉取最新的分支，很容易会让线上的分支代码与你新的通用模块产生一大堆冲突。最稳定的方法还是使用 `git fetch 分支` 的操作， 在浏览完代码没有问题之后再手动 merge/rebase 合入。