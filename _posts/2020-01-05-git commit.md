---
layout:     post
title:      git基本命令
subtitle:   git使用法则
date:       2020-01-05
author:     shmily
header-img: img/post-bg-re-vs-ng2.jpg
catalog: true
tags:
    - git使用法则
    - 开发工具
---
# git commit




# git rebase



# git stash
git stash 保存当前工作的进度，会把暂存和工作区的改动保存起来。执行完这个命令后，在运行git staus命令，就会发现当前是一个干净的工作区，没有任何的改动。使用git staus save 'message' 可以添加一些注释

## git stash list
显示保存进度的列表，也就意味着，git statsh命令可以执行多次
## git stash pop [-index][statsh_id]

* ```git stash pop``` 恢复最新的进度到工作区，git默认会把工作区和暂存区的改动都恢复到工作区
* ```git stash pop --index``` 恢复最新的进度到工作区和暂存区（尝试将原来的暂存区的改动恢复到暂存区）
* ```git stash pop statsh@{1}```恢复指定的进度到工作区。```stash_id```是通过```git stash list``` 命令得到的通过git stash pop命令恢复进度后，会删除当前进度

## git stash apply [-index][stash_id]
除了不删除恢复的进度之外，其余和git stash pop命令一样

## git stash drop [stash_id]
删除一个存储的进度。如果不指定stash_id 则默认删除最新的存储进度

## git stash clear
删除所有存储的进度

# git push 
## git push --force
当我们使用git reset命令回退本地的git log 显示的commit信息之后，使用git push提交改动到远端服务器会报错，打印类似下面的错误信息：

```
提示：更新被拒绝，因为您当前分支的最新提交落后于其对应的远程分支
```

此时如果想强制用本地git log的commit信息覆盖服务器的git log 可以使用git push -f命令来推送代码到服务器。查看man git -push

![](https://tva1.sinaimg.cn/large/006tNbRwly1gav11fnerfj30yc0g446b.jpg)


## git push origin:<branchName>

### 删除远端服务器的分支

在这个命令中，冒号":"前面的参数是要推送到本地的分支名，这里没有提供，想到你公园推多送一个本地的空的分支冒号":"后面的参数是要推送到服务器的远端分支名

这个命令推送一个空分支到远端分支，相当于远端分支被置为空，从而删除branchname指定的远端分支

## git push origin --delete <branchName>

git push origin --detele <branchName> 命令本质上和git push origin：<branchName>是一样的。官网解释delete命令其实也是推送本地空的分支到远端分支




# git rebase 
git rebase 有三种特别常用的地方

* 拉取远程代码
* 合并多次提交
* 合并分支

1、拉取远程代码
这个使用最为频繁的一种场景中，使用最为频繁的，拉取远程代码的场景

拉取远程代码进一步可以细分为两种场景

1、远程代码中他人的提交与本地我们的提交有重合
2、无重合

## 1.1代码无重合
这个问题在于远程库提交历史的偏离。

举个例子：假设你周一克隆了一个仓库，然后开始研发某个新功能。到周五时，你新功能开发测试完毕，可以发布了。但是 —— 天啊！你的同事这周写了一堆代码，还改了许多你的功能中使用的 API，这些变动会导致你新开发的功能变得不可用。但是他们已经将那些提交推送到远程仓库了，因此你的工作就变成了基于项目旧版的代码，与远程仓库最新的代码不匹配了。

这种情况下，git push会回来星期一那天的状态，还是直接在新代码的基础上添加你的代码，这个时候Git是不会允许直接push，会要求我们先合并远程最新的代码，然后才能push自己的代码。















