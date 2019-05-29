---
title: git学习笔记
date: 2019-05-28 16:00:32
tags:
 - git branch
 - git revoke
 - git confict
---

初学Git命令中遇到的一些坑和一些笔记，希望自己能够回过头来看看吧。
## 1.无法链接到文件（Unlink of file '.git/objects/pack/pack...pack/.idx）
 由于git pull时候，此时的文件是被占用的状态，所以需要退出编辑器或者git命令（有时候一次退出没有效果，需要再重试一次,或者在任务管理器中找到对应的进程，直接杀死也是可以的），然后输入以下在对于的仓库目录下输入以下git命令.
```
git gc --auto

git repack -d -l 
```
## 2.git创建本地分支，并且推送到远程不同的分支

```
 git branch -r  //查看所有的分支
```
创建没有映射关系的分支，适用于本地分支需要推送到远程的多个分支
```
 git checkout -b 分支名称  //本地会切换到创建的分支

 git branch 分支名称 //仅仅创建分支

 git fetch origin 远程分支名x:本地分支名x   //使用该方式会在本地新建分支x，但是不会自动切换到该本地分支x，需要手动checkout。采用此种方法建立的本地分支不会和远程分支建立映射关系。

```
创建适用于单一分支，有映射关系，方便推送，直接使用git push 
```
git checkout -b 本地分支名x origin/远程分支名x  //使用该方式会在本地新建分支x，并自动切换到该本地分支x。采用此种方法建立的本地分支会和远程分支建立映射关系。

git branch -u origin/addFile //创建当前分支，和远程addFile分支的映射关系

git branch --set-upstream-to origin/addFile //同上
```
撤销映射关系
```
git branch --unset-upstream //适用于当前分支
```
推送本地分支到远程分支
```
1.本地分支名称和远程相同

git push 库名称 本地分支名称

2.本地分支名称和远程的分支名称不相同

git push [远程库名] [本地分支名] : [远程分支名]

```
也可以利用merge
```
git checkout 远程分支master

git merge 本地分支名称

git push orgin master //推送到远程master

```
## 3.git的撤销
git是有工作区，暂存区，本地分支的概念，我们的通常提交的顺序是从工作区中修改对应的资源文件（不管是在哪个分支都是在对应分支的工作区开始修改的）。
此时如果想要撤销的话：
```
git checkout .//撤销所有已经添加的文件到原始状态
git checkout filename //撤销单一文件到原始状态
```
修改完成之后，使用 ```git add . ``` 或者是``` git add file```就会把对应的文件添加到git本地电脑的暂存区，这是对应你这个分支的暂存区。

此时如果想要撤销的话，可以直接使用命令 ：
```
git reset .//撤销所有已经添加的文件到工作区
git reset filename //撤销单一文件到工作区
```
如果此时add之后还是使用了```git commit ```的时候发现了错误，想要撤销这一次的更改的话，就需要使用：
```
git log 
```
git log 查询到对应刚才的那一次提交的**上一次commitId**自己记录下来
```
git reset commitId
```
然后Git就会告诉你已经到了对应的上一次commitId的版本了（注意此时reset要保证工作区和暂存区没有修改的文件，不然可能会失败）

## 4.错误使用git config global命令
往往在工作中或者自己的电脑上使用git命令的时候会便于简化命令操作，使用了git config global命令来设置用户，这就导致在电脑如果使用了其他的仓库就会出错，使用以下命令来查看自己的config配置。

config配置命令
```
git config
```
在git里面config有三层分别是system，global，local三个级别配置，分别使用system、global、local可以定位到对应的级别，以及设置相关的配置。

在对应级别后添加--list，即可查看对应的配置信息,例如查看用户配置信息。
```
git config --global --list
```
在查看了自己的配置信息之后，可以使用git config命令直接把对应的仓库用户修改成自己所需要的用户。
```
admin@admin-PC MINGW64 /f/momo733.github.io (master)
$ git config --local user.name "Momo733"
$ git config  --local user.email "1550526230@qq.com"
$ git config  --global --list
user.name=chengzhang
user.email=Xxx@jubao56.com
uwe.email=xxx@163.com
gui.encoding=utf-8
i18n.commitencoding=GB2312
svn.pathnameencoding=GB2312
filter.lfs.process=git-lfs filter-process
filter.lfs.required=true
filter.lfs.clean=git-lfs clean -- %f
filter.lfs.smudge=git-lfs smudge -- %f
credential.helper=store

$ git config  --local --list
core.repositoryformatversion=0
core.filemode=false
core.bare=false
core.logallrefupdates=true
core.symlinks=false
core.ignorecase=true
remote.origin.url=https://github.com/Momo733/momo733.github.io.git
remote.origin.fetch=+refs/heads/*:refs/remotes/origin/*
branch.master.remote=origin
branch.master.merge=refs/heads/master
user.name=Momo733
user.email=1550526230@qq.com
```
## 5.撤销合并的冲突（处于merging状态）
当发现合并代码错误分支，但是需要撤销的太多，此时可以使用：
```
git reset --hard HEAD
```
