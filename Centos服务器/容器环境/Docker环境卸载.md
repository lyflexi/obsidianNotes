使用yum安装docker（安装过程可以参照linux 安装docker），如需卸载docker可以按一下步骤操作：

## 1、查看当前docker状态

![[Pasted image 20240127165607.png]]

如果是运行状态则停掉

systemctl stop docker

![[Pasted image 20240127165612.png]]

## 2、查看yum安装的docker文件包

yum list installed |grep docker
![[Pasted image 20240127165618.png]]

如果执行了上一步的删除操作,则这里的rpm就不会显示

查看docker相关的rpm源文件,rpm -qa |grep docker
![[Pasted image 20240127165624.png]]

## 3、删除所有安装的docker文件包

yum -y remove docker.x86_64
![[Pasted image 20240127165630.png]]

其他的docker相关的安装包同样删除操作，删完之后可以再查看下docker rpm源

rpm -qa |grep docker
![[Pasted image 20240127165636.png]]

## 4、删除docker的镜像文件，默认在/var/lib/docker目录下
![[Pasted image 20240127165641.png]]

删除上述的docker目录

rm -rf /var/lib/docker
![[Pasted image 20240127165647.png]]
到此docker卸载就完成了