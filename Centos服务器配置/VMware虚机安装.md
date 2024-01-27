子网设置：192.168.18.0
网关设置：192.168.18.2
![[Pasted image 20240127151958.png]]

# 配置静态NAT-IP

## 方式一：安装界面设置IP

安装虚拟机过程中注意下面选项的设置：

软件选择：基础设施服务器
![[Pasted image 20240127152024.png]]

分区选择：自动分区

设置主机名：
![[Pasted image 20240127152029.png]]

网络配置：按照下面配置网路地址信息

网络地址：192.168.18.100 （每台主机都不一样 分别为100、101、102）

子网掩码：255.255.255.0

默认网关：192.168.18.2

DNS： 223.5.5.5 阿里DNS
![[Pasted image 20240127152040.png]]

## 方式二：进入系统修改ifcfg-ens33

```Shell
#配置静态IP
cd /etc/sysconfig/network-scripts
vim /etc/sysconfig/network-scripts/ifcfg-ens33
# 修改
TYPE="Ethernet"
PROXY_METHOD="none"
BROWSER_ONLY="no"
BOOTPROTO="static" # 配置IP静态
DEFROUTE="yes"
IPV4_FAILURE_FATAL="no"
IPV6INIT="yes"
IPV6_AUTOCONF="yes"
IPV6_DEFROUTE="yes"
IPV6_FAILURE_FATAL="no"
IPV6_ADDR_GEN_MODE="stable-privacy"
NAME="ens33"
UUID="59c8f94a-fc40-45fe-b39b-a4faa358e99d"
DEVICE="ens33"
ONBOOT="yes"
IPADDR="192.168.18.100" # 指定IP
NETMASK="255.255.255.0" #配置子网掩码
GATEWAY="192.168.18.2" #配置网关
DNS=223.5.5.5 # 域名解析系统
# 使配置生效
systemctl restart network
```

# 设置主机名（可选）
比如设置以下主机名
master节点： k8s-master
node节点：   k8s-node1
node节点：   k8s-node2
```shell
hostnamectl set-hostname <hostname>
```

主机名解析，为了方便后面集群节点间的直接调用，在这配置一下主机名解析，企业中推荐使用内部DNS服务器

```shell
# 主机名成解析 编辑三台服务器的/etc/hosts文件，添加下面内容
192.168.18.100  k8s-master
192.168.18.101  k8s-node1
192.168.18.102  k8s-node2
```

# 图形化界面（可选）

```shell
yum groupinstall "GNOME Desktop" "Graphical Administration Tools"
```

如果没有网络的情况下会报无法连接的错误, 这里就需要打通网络，之前我们没有设置，这里需要修改相关配置.

键入命令:`vi /etc/sysconfig/network-scripts/ifcfg-ens33`

```shell
把文件里的ONBOOT=no改为yes
```

然后重启网络:`systemctl restart network`

然后就是输入安装linux图行化界面的命令了. 中间需要输入两次y，表示确认安装。

## 修改CentOS7默认启动模式为图形化模式：

查看默认启动界面: `systemctl get-default`

然后设置启动界面为图形化界面: `systemctl set-default graphical.target`

输入命令 `systemctl get-default` 可查看当前默认的模式为 `multi-user.target`，即命令行模式，我们要将它修改为图形界面模式，如下图：
![[Pasted image 20240127152232.png]]

此时再次输入命令 `systemctl get-default` 即可查看当前修改后的默认模式为`graphical.target`，即图形界面模式，如下图：
![[Pasted image 20240127152238.png]]

## 重启CentOS，检验GUI界面效果：

输入命令 `reboot` 重启CentOS系统，重启之后就已经切换到GUI图形界面模式，如下图：
![[Pasted image 20240127152244.png]]