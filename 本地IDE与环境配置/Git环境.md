Git全局设置如下：

```shell
#查看当前git用户名： 
git config user.name
#查看当前git邮箱： 
git config user.email
#切换git用户名、邮箱、保存: 
git config --global user.name "liuyanntes"
git config --global user.email "liuyanntes@163.com"
git config --global credential.helper store

git config --global user.name "liuyan3"
git config --global user.email "liuyan@isrc.iscas.ac.cn"
```

# 代码推送

```Bash
Command line instructions
#Create a new repository克隆远程项目
git clone http://10.45.47.10/ai/dve-frontend.git  
cd dve-frontend
touch README.md
git add README.md
git commit -m "add README"
git push -u origin master

#Existing folder本地存在项目
cd existing_folder
git init
git remote add origin http://10.45.47.10/ai/dve-frontend.git
git add .
git commit -m "Initial commit"
git push -u origin master
```

# 版本回退

## 撤销Add

```Bash
git reset HEAD <file>
```

## 撤销Commit

仅仅是撤回commit操作，您写的代码仍然保留。

```Bash
#HEAD^的意思是上一个版本，也可以写成HEAD~1
#如果你进行了100次commit，想都撤回，可以使用HEAD~100
git reset --soft HEAD^
#reflog记录每一次Commit的命令的ID
git reflog
e475afc HEAD@{1}: reset: moving to HEAD^
1094adb (HEAD -> master) HEAD@{2}: commit: append GPL
e475afc HEAD@{3}: commit: add distributed
eaadf4e HEAD@{4}: commit (initial): wrote a readme file
#通过commitID进行回退
git reset --hard 1094a
```

# 分支创建

```Java
#1.首先在远程创建远程dev分支
#2.查看本地分支只有master
git branch
#git切换至本地已有分支
git checkout mybranch
#git本地创建分支并切换到当前新创建的分支上
git checkout -b dev
#查看本地分支有两个了master和dev
git branch
#推送到dev分支
git push -u origin dev
```

## 分支合并

假如我们在分支version-20220331-notice上，刚开发完项目，并且提交了分支代码

```Bash
git  add .
git  commit -m '提交的备注信息'
git  push -u origin ersion-20220331-notice
```

后面，想将分支version-20220331-notice合并到主分支version-20220331，操作如下：

1. 切换到主分支version-20220331
    
2. 主分支执行pull，并执行`git merge version-20220331-notice`命令
    
3. 主分支提交到远程master：`git push origin master`
    

![[Pasted image 20240127170159.png]]
![[Pasted image 20240127170205.png]]
![[Pasted image 20240127170211.png]]

```Bash
#首先切换到主分支version-20220331
git  checkout version-20220331
```

然后我们把分支version-20220331-notice的代码合并主分支version-20220331

```Bash
##如果是自己一个开发就没有必要了，为了保险期间需要把远程主分支version-20220331上的代码pull下来
git pull origin version-20220331
git  merge version-20220331-notice 
```

然后查看状态及执行提交命令

```Bash
git status
#下面的意思就是你有12个commit，需要push到远程master上 
On branch master
Your branch is ahead of 'origin/master' by 12 commits.
(use "git push" to publish your local commits)
nothing to commit, working tree clean
#最后执行下面提交命令
git push origin master
```

## 分支管理

```Bash
更新远程分支列表
git remote update origin --prune
查看所有分支
git branch -a
删除远程分支Chapater6
git push origin --delete Chapater6
删除本地分支 Chapater6
git branch -d  Chapater6
```

# Rebase

当我们合并分支时，git会自动替我们生成一个merge commit，记录此次的merge。

记录merge操作没有什么问题，问题是当你merge了一个巨大改动的分支之后，你的git log基本上就不用看了，肯定全都是其他人的各种乱七八糟你也不清楚的提交记录。为了解决这个头疼的问题，我们就需要用到变基操作rebase
![[Pasted image 20240127170220.png]]

```Bash
#bugFix分支提交之后执行rebase命令
git checkout bugFix
git  commit -m 'bugFix'
git  push -u origin bugFix
git rebase master
#主分支master执行merge
git checkout master
git merge bugFix
git status
git push origin master
```

# 错误排查

## error: failed to push some refs to

问题描述：在git bash中键入 $ git push -u origin master 进行提交的时候出现异常：`error: failed to push some refs to`

问题原因：远程库与本地库不一致造成的，需要先把远程库同步到本地库

```Java
git pull --rebase origin master
```
![[Pasted image 20240127170228.png]]

```Java
#重新提交
git push -u origin master
```
![[Pasted image 20240127170235.png]]

等待上传....上传成功

## fatal: remote origin already exists错误

执行`git remote add origin` `[http://10.45.47.10/ai/dve-frontend.git](http://10.45.47.10/ai/dve-frontend.git)`出现如下错误：

![[Pasted image 20240127170242.png]]

解决办法如下：我们可以先 git remote -v 查看远程库信息
![[Pasted image 20240127170248.png]]

```Bash
#我们可以先 git remote -v 查看远程库信息：
git remote -v
#
git remote rm origin
#重新添加
git remote add origin https://gitee.com/liuyanntes/datamining.git
```

## OpenSSL SSL_read: Connection was reset, errno 10054

这个错误是我们通过梯子访问GitHub，需要额外给git配置本地代理端口

```Shell
git config --global http.proxy http://127.0.0.1:7897
git config --global https.proxy http://127.0.0.1:7897
```

## fatal: The remote end hung up unexpectedly

在使用git更新或提交项目时候出现 "fatal: The remote end hung up unexpectedly "

那就简单了，要么是缓存不够，要么是网络不行，要么墙的原因。

我这里是因为缓存不够，解决方案：

修改提交缓存大小为500M，或者更大的数字

```Shell
git config --global http.postBuffer 524288000
#some comments below report having to double the value:
git config --global http.postBuffer 1048576000
```

或者在克隆/创建版本库生成的 .git目录下面修改生成的`config`文件增加如下：

```Shell
[http]postBuffer = 524288000
```