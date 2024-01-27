# SSH生成与配置
![[Pasted image 20240127170312.png]]
![[Pasted image 20240127170320.png]]

注意鼠标要不停移动产生随机序列，然后分别保存公钥（貌似用不上）和私钥文件，使用ppk后缀命名
![[Pasted image 20240127170327.png]]

如果本地早存在有git bash的密钥，可以导入直接使用，一般在C:\Users\Luo\.ssh目录下.
![[Pasted image 20240127170334.png]]

然后复制SSH公钥序列增加到gitee上面
![[Pasted image 20240127170345.png]]

## 全局putty客户端私钥配置

接下来，配置TortoiseGit客户端的密钥（个人理解是全局的密钥），添加生成的私钥
![[Pasted image 20240127170352.png]]
![[Pasted image 20240127170358.png]]

到这里就已经可以进行推送和大文件克隆了

## 局部putty客户端私钥配置

如果不想进行全局putty的密钥设置，也可以单独设置仓库的密钥 右键进入对选中仓库的设置
![[Pasted image 20240127170405.png]]
选中之前保存的私钥文件，然后进行提交推送，也能成功