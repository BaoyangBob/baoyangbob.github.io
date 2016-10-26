---
layout: post
title: Git技巧之添加远程仓库
category: 技术
date:   2016-9-30 22:20:28
catalog:  true
tags:
    - Git
category: 技术
description: 多系统间同步
---




因为经常在 Win10 和 Ubuntu 之间切换作业，以后可能还有苹果机，同一个项目的代码需要在修改后快捷地同步到另一个设备上。网盘、云同步、GitHub 等使用体验受网速、服务商的影响常常不稳定（如近期的360云盘关闭事件），免费的午餐不长久，私以为当下备份数据最稳妥的方式还是放到自己的硬盘上。作为程序员，我们可以利用 Git 来达成目标。

大体思路是：

- 将一份干净的代码放到硬盘的仓库目录，
- 将代码建为 Git仓，
- 在本地设备的工作空间下clone该仓
- 每次修改完代码commit、push到硬盘仓库
- 每次作业前先同步代码

## 添加远程仓库的具体命令



在`clone`到本地的仓库中，可以添加用于同步的远程仓库，给远程仓库起“remote”、“origin”、“upstream”等约定俗成的昵称。接着可以`fetch`远程仓库的所有分支，再用`rebase`就可以继续在远程仓库中的版本上工作了。



```shell

# 添加远程仓库，称之为 "upstream":

git remote add upstream https://github.com/whoever/whatever.git

# 添加本地文件夹中的仓库。获取当前目录的地址可用`pwd`。添加仓库除了可以用`git remote add`,也可以用`git clone`，不过`git clone`会把远程仓库自动命名为 origin

git remote add upstream /M/GitRepo/quicksettingtile/  
git clone /M/GitRepo/quicksettingtile/


# 查看已添加的远程仓库

git remote -v
  
# 移除远程仓库

git remote remove upstream
  
# 重命名远程仓库

git remote rename upstream

# 抓取远程仓库的所有分支到本地，但还没有合并

git fetch upstream

# 切到 master 主分支

git checkout master

# Rewrite your master branch so that any commits of yours that
# aren't already in upstream/master are replayed on top of that
# other branch:

git rebase upstream/master
  
# 本地仓库有修改后，通过`push`将修改同步到远程仓库。但远程仓库默认会拒绝更新。解决方案有两种：设置`receive.denyCurrentBranch`为ignore或warn，或者在初始化远程仓库时就设置为裸露仓库。

Solution1
		git config receive.denyCurrentBranch ignore

Solution2
		git init --bare project.git
		
# 上传更新

git push upstream master
```

