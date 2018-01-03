---
layout: post
categories:
  - Git
title:  "Git工具回滚线上错误提交"
date: 2015-01-27 22:21:35 +0800
comments: true
---


## <a id="Intro">前言</a>

应用场景：自动部署系统发布后发现问题，需要回滚到某一个commit，再重新发布

原理：先将本地分支退回到某个commit，删除远程分支，再重新push本地分支

操作步骤：

``` bash

1. git checkout the_branch

2. git pull

3. git branch the_branch_backup # 备份一下这个分支当前的情况

4. git reset --hard the_commit_id # 把the_branch本地回滚到the_commit_id

5. git push origin :the_branch # 删除远程 the_branch

6. git push origin the_branch # 用回滚后的本地分支重新建立远程分支

7. git push origin :the_branch_backup # 如果前面都成功了，删除这个备份分支

```

>> Tips：获取`the_commit_id` 使用`git log`，然后找到需要回滚到的那个提交的id hash值。

<!-- more -->
