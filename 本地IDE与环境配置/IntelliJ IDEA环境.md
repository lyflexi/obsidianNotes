# 控制台乱码

windows10电脑设置的语言选项，管理语言设置中找到更改系统区域设置打开找到一个Beta版的UFT-8的选项✔上
![[Pasted image 20240127171002.png]]

# idea高效快捷键

## 代码格式化

```Shell
ctrl+alt+L
```

## 实现树/继承树列表展示

```Java
ctrl+h
```

三个小按钮功能各不相同：
![[Pasted image 20240127171011.png]]

## 实现树/继承树图表展示

选中类名，右键Show Diagram
![[Pasted image 20240127171017.png]]
默认展示的是向上关系

但我们要找向下关系怎么办？右键`TransactionDefinition`，show implements，ctrl+a选择全部，回车：
![[Pasted image 20240127171021.png]]
![[Pasted image 20240127171029.png]]
## 展示当前类所有的方法

```Shell
5.快捷键Alt+7,注意必须先进入当前类才能展示
```

## 展示此方法在其他什么地方被调用

```Shell
6.快捷键Alt+f7,
```

## 全局查找/替换

```Shell
ctrl+shift+大写的F    或者    CTRL+shift+r  ：全文件内容查找
13.两个shift ：查找类

CTRL+r替换
CTRL+shift+r全局替换
```

## 查看该类的源码

```Shell
8.CTRL+鼠标点击：查看该类的源码
9.CTRL+ALT+鼠标点击：查看该类的实现类源码

10.Ctrl+Alt+ left/right  ：返回至上次源码浏览的位置
```

## 快速复制当前行

```Shell
：CTRL+D
```

# idea插件

## JRebel

安装和使用JRebel需要注意两点：激活和设置

安装JRebel和JRebel mybatisplus

安装好之后重启IDEA
![[Pasted image 20240127171104.png]]
**激活JRebel**

JRebel并非免费的插件，需要激活之后才能使用。

1、首先到github上去下载一个反向代理软件，我下载的是windows x64版本。

[https://github.com/ilanyu/ReverseProxy/releases/tag/v1.4](https://github.com/ilanyu/ReverseProxy/releases/tag/v1.4)

[https://github.com/ilanyu/ReverseProxy/releases/tag/v1.4](https://github.com/ilanyu/ReverseProxy/releases/tag/v1.4)
![[Pasted image 20240127171126.png]]

2、双击运行我们下载的程序
![[Pasted image 20240127171132.png]]

3、在IDEA中一次点击 File->Settings->JRebel 并找到激活界面(因为我的已经激活了，点击change liense进入的激活界面，记不清一开始怎么进入的了)
![[Pasted image 20240127171137.png]]

4、选择JRebel activated中的 connect to online licensing service

第一行输入 [http://127.0.0.1:8888/d3545f42-7b88-4a77-a2da-5242c46d4bc2](http://127.0.0.1:8888/d3545f42-7b88-4a77-a2da-5242c46d4bc2)

第二行输入正确的邮箱格式，例如： test@123.com

再点击以下change liense 按钮验证激活

提示：d3545f42-7b88-4a77-a2da-5242c46d4bc2为UUID,可以自己生成，并且必须是UUID才能通过验证

![[Pasted image 20240127171148.png]]

5、最后别忘了把JRebel设置为offline模式 点一下work offline
![[Pasted image 20240127171154.png]]

**相关设置**

此时虽然安装好了JRebel并成功激活了，但是我们使用JRebel debug的时候，发现修改代码后，热部署不起作用。因为还需要设置两个地方

1、设置项目自动编译
![[Pasted image 20240127171203.png]]

2、设置 compiler.automake.allow.when.app.running

ctrl+shift+A 或者 help->find action…打开

搜索registry

找到 compiler.automake.allow.when.app.running 并✔
![[Pasted image 20240127171210.png]]

**点击边框TOOL栏JRebel选中想要热部署的项目**
![[Pasted image 20240127171216.png]]

  

这样，热部署就成功了。

  

**热部署快捷键ctrl shift F9**


## Sequence Diagram

时序图效果：最上面一行代表顶级父类
![[Pasted image 20240127171304.png]]

- 调用别的类的实现类方法
    
- 调用别的类的构造方法 `<<create>>`
    
- 调用自己类的实现类方法 `1.6.2:handle`
    

---

设置
![[Pasted image 20240127171312.png]]

![[Pasted image 20240127171326.png]]



# idea配置远程
![[Pasted image 20240127171425.png]]

![[Pasted image 20240127171434.png]]


# 中文注释unicode转码

默认中文不会自动进行unicode转码。如下

`#\u5f00\u53d1\u73af\u5883\u5173\u95ed\u7f13\u5b58 \u751f\u4ea7\u73af\u5883\u5f00\u542f`

在project settings - File Encoding，在标红的选项上打上勾，确定即可
![[Pasted image 20240127171526.png]]
  

# Reset Head撤销Commit

撤销commit
![[Pasted image 20240127171632.png]]

Reset Type：参数详解 首先了解：

|   |   |   |
|---|---|---|
|工作区|－暂存区|－本地仓库|
|代码编写及修改是在工作区|－ git add 将本地修改添加到暂存区|－git commit 将暂存区中的内容提交到本地仓库|

## --mixed

--mixed （git reset的默认参数，即不添加参数的默认值）最常用 意思是：不删除工作空间改动代码，撤销commit 和 撤销git add . 操作，回退到工作区

这个为默认参数,git reset --mixed HEAD^ 和 git reset HEAD^ 效果是一样的。

|   |   |   |
|---|---|---|
|工作区|－暂存区|－本地仓库|
|代码编写及修改是在工作区|－ git add 将本地修改添加到暂存区|－git commit 将暂存区中的内容提交到本地仓库|

## --soft

意思是：不删除工作空间改动代码，撤销commit，不撤销git add . 操作，

回退到git commit之前，此时处在暂存区。（即执行git add 命令后）

|   |   |   |
|---|---|---|
|工作区|－暂存区|－本地仓库|
|代码编写及修改是在工作区|－ git add 将本地修改添加到暂存区|－git commit 将暂存区中的内容提交到本地仓库|

## --hard

意思是：删除本地改动代码，撤销commit，撤销git add .

（三者的改变全都丢失，即代码的修改内容丢失，直接回退到某个版本；因此我们修改过的代码就没了，需要谨慎使用）