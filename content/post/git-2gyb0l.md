---
title: git
slug: git-2gyb0l
url: /post/git-2gyb0l.html
date: '2024-07-11 09:29:35+08:00'
lastmod: '2024-08-08 13:27:42+08:00'
toc: true
tags:
  - git
categories:
  - 工具
keywords: git
isCJKLanguage: true
---

# git

​![image](https://raw.githubusercontent.com/arukasxy/notes/main/content/post/image-viewer/image-20240711093024-uwlve1i.png)​

​![image](https://raw.githubusercontent.com/arukasxy/notes/main/content/post/image-viewer/image-20240711093030-u4flwxj.png)​

## 远程仓库

* 查看远程仓库地址

  ```shell
  git remote -v
  ```

## clone代码

* git clone 指定分支

  ```shell
  git clone -b master http://gitslab.yiqing.com/declare/about.git
  ```

### 相关问题（+git 更新）

1. Failed to connect to github.com port 443 after 21007 ms

    方法：使用镜像网站

    * https://ghproxy.com/
    * http://raw.githubsercontent.com
    * https://toolwa.com/github/

    ```shell
    http://github.com/
    替换为
    http://gitclone.com/github.com/

    git config --global url."https://gitclone.com/github.com".insteadOf https://github.com/
    # 全局配置
    ```

2. fatal: unable to access 'https://github.com/google/googletest.git/': GnuTLS recv error (-54): Error in the pull function

    方法：更新git

    ```shell
    sudo add-apt-repository ppa:git-core/ppa
    sudo apt update
    sudo apt install git
    ```

## 新建仓库

1. 如果是复制他人仓库，先clone代码
2. 删除.git文件夹
3. ```shell
    git init
    ```
4. 提交代码

    ```shell
    git add ./
    git commit -m "first commit"
    ```

5. 先在gitee上新建仓库，再参考命令关联远程仓库

    ```shell
    git remote add origin https://github.com/YYY/SimpleUI
    git push -u origin master
    ```

## 分支

|命令|功能|
| ---------------------------------------| ---------------------------------------------|
|git branch -a|查看所有远程分支|
|git branch|查看当前分支|
|git branch dev|创建分支|
|git switch dev<br />git checkout 本地分支|切换分支|
|git checkout -b 新本地分支 [远程分支]|创建与远程分支关联的本地分支|
|git branch -d 本地分支|删除分支，需要先合并分支<sup>（使用-D 强制删除分支）</sup><br />|
|git push origin --delete 远程分支||
|||

### 分支图

```shell
alias graph="git log --oneline --graph --decorate --all"
```

### 合并分支

* 获取另一分支的**所有**代码修改

```shell
git merge
# 将被合并分支的所有commit合并为一个commit后再merge到目标分支
git merge --squash
git rebase
```

### 合并指定提交

* 获取另一分支的**部分**代码修改到当前分支

```shell
git cherry-pick commitHash
# 合并多个提交
git cherry-pick HashA HashB   
# 合并一系列连续提交,不包括A  
git cherry-pick A..B  
# 合并一系列连续提交，包括A 
git cherry-pick A^..B
+ 冲突解决后
git cherry-pick --continue // 继续执行
git cherry-pick --abort    // 回到操作前的样子
git cherry-pick --quit     // 不回到操作前的样子
+
git push
```

## 查看修改（历史版本）

|命令|功能|
| :---------------------------------------: | :----------------------------------------: |
|git status|查看本地修改文件|
|git diff|**本地修改**​和​**暂存区<sup>（add文件后，又对该文件进行修改）</sup>**​对应文件的差异<sup>（add文件后，又对该文件进行修改）</sup>|
|git diff --cached<br />git diff --staged<br />|已提交和暂存区的差异<sup>（仅对比相同文件名）</sup>|
|git diff head|已提交和未提交的差异<sup>（add到暂存区的文件也会和已提交的对比）</sup>|
|**查看分支差异**<br />||
|git diff branch1 branch2 --stat|显示branch1和branch2中的差异|
|git diff branch1 branch2 文件|显示指定文件的详细差异|
|git log branch1 \^branch2|查看branch1分支有，而branch2中没有的|
|git log branch1..branch2|查看branch2中比branch1中多提交了哪些内容|
|**查看commit修改**<br />||
|git log -3 --stat|查看最近3次commit log|
|git reflog|查看所有历史版本信息，常用于版本恢复|
|git show commitId|查看commitId的修改|
|git show commitId 文件|查看commitId中指定文件的修改|
|git diff commitId1 commitId --name-only|查看两次commit间的所有修改|

## 提交修改

1. add + commit 本地提交修改
2. 获取更新
3. 冲突解决 + git pull获取更新
4. 推送更新

### 本地提交修改

```shell
git status
# 查看修改的文件
git add fileName
# 将指定文件放到暂存区
git commit -m ""
# 提交
```

### 获取更新

1. fetch获取更新，用户检查后merge

```shell
git fetch origin master
# 获取远程的master分支的更新
git fetch origin  
# 获取远程的全部更新

git merge
```

2. pull 获取更新后自动merge

```shell
git pull origin 远程分支名:本地分支名
# 获取远程分支名，与本地分支名合并，默认为当前分支
git pull
```

### 推送更新push

```shell
git push 本地分支 远程分支
git push
# 默认推送到当前分支绑定的远程分支
```

## 放弃修改

|命令|功能|
| :-------------------------------------------------------------------------| :----------------------------------------------------------|
|                                     工作区<br />||
|git checkout .|放弃工作区中全部修改|
|git checkout -- fileName|放弃工作区中某个文件的修改|
|git checkout -f|放弃工作区和暂存区的全部修改|
|                                     暂存区<br />||
|git reset fileName|撤销暂存区中的某个文件|
|git reset|撤销暂存区中的全部修改|
|                                    提交点<br />||
|git revert commitId<br />-m 1|放弃某次commitId<br />如果commitId为merge节点，需-m指定提交点|

## 冲突解决

修改文件后，git add + git commit

## 保存工作现场

作用：切换分支时不需要先提交

1. 保存未提交的修改

    ```shell
    git stash
    ```

2. 恢复现场

    ```shell
    git stash pop
    ```

# github

## git@github.com: Permission denied (publickey)

```C++
ssh-keygen -t rsa -C "1026565513@qq.com"
```

‍
