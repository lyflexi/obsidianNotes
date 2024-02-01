# Ceph基础介绍

Ceph是一个可靠地、自动重均衡、自动恢复的分布式存储系统，根据场景划分可以将Ceph分为三大块，分别是**对象存储、块设备存储和文件系统服务**。在虚拟化领域里，比较常用到的是Ceph的块设备存储，比如在OpenStack项目里，Ceph的块设备存储可以对接OpenStack的cinder后端存储、Glance的镜像存储和虚拟机的数据存储，比较直观的是Ceph集群可以提供一个raw格式的块存储来作为虚拟机实例的硬盘。

Ceph相比其它存储的优势点在于它不单单是存储，同时还充分利用了存储节点上的计算能力，在存储每一个数据时，都会通过计算得出该数据存储的位置，尽量将数据分布均衡，同时由于Ceph的良好设计，采用了CRUSH算法、HASH环等方法，使得它不存在传统的单点故障的问题，且随着规模的扩大性能并不会受到影响。

# Ceph的核心组件

Ceph的核心组件包括Ceph OSD、Ceph Monitor和Ceph MDS。

**Ceph OSD**：OSD的英文全称是Object Storage Device，它的主要功能是存储数据、复制数据、平衡数据、恢复数据等，与其它OSD间进行心跳检查等，并将一些变化情况上报给Ceph Monitor。一般情况下一块硬盘对应一个OSD，由OSD来对硬盘存储进行管理，当然一个分区也可以成为一个OSD。

Ceph OSD的架构实现由物理磁盘驱动器、Linux文件系统和Ceph OSD服务组成，对于Ceph OSD Deamon而言，Linux文件系统显性的支持了其拓展性，一般Linux文件系统有好几种，比如有BTRFS、XFS、Ext4等，BTRFS虽然有很多优点特性，但现在还没达到生产环境所需的稳定性，一般比较推荐使用XFS。

伴随OSD的还有一个概念叫做Journal盘，一般写数据到Ceph集群时，都是先将数据写入到Journal盘中，然后每隔一段时间比如5秒再将Journal盘中的数据刷新到文件系统中。一般为了使读写时延更小，Journal盘都是采用SSD，一般分配10G以上，当然分配多点那是更好，Ceph中引入Journal盘的概念是因为Journal允许Ceph OSD功能很快做小的写操作；一个随机写入首先写入在上一个连续类型的journal，然后刷新到文件系统，这给了文件系统足够的时间来合并写入磁盘，一般情况下使用SSD作为OSD的journal可以有效缓冲突发负载。

**Ceph Monitor**：由该英文名字我们可以知道它是一个监视器，负责监视Ceph集群，维护Ceph集群的健康状态，同时维护着Ceph集群中的各种Map图，比如OSD Map、Monitor Map、PG Map和CRUSH Map，这些Map统称为Cluster Map，Cluster Map是RADOS的关键数据结构，管理集群中的所有成员、关系、属性等信息以及数据的分发，比如当用户需要存储数据到Ceph集群时，OSD需要先通过Monitor获取最新的Map图，然后根据Map图和object id等计算出数据最终存储的位置。

**Ceph MDS**：全称是Ceph MetaData Server，主要保存的文件系统服务的元数据，但对象存储和块存储设备是不需要使用该服务的。

# Ceph基础架构组件
![[Pasted image 20240201103229.png]]

从架构图中可以看到最底层的是RADOS，RADOS自身是一个完整的分布式对象存储系统，它具有可靠、智能、分布式等特性，Ceph的高可靠、高可拓展、高性能、高自动化都是由这一层来提供的，用户数据的存储最终也都是通过这一层来进行存储的，RADOS可以说就是Ceph的核心。

RADOS系统主要由两部分组成，分别是OSD和Monitor。

基于RADOS层的上一层是LIBRADOS，LIBRADOS是一个库，它允许应用程序通过访问该库来与RADOS系统进行交互，支持多种编程语言，比如C、C++、Python等。

基于LIBRADOS层开发的又可以看到有三层，分别是RADOSGW、RBD和CEPH FS。

RADOSGW：RADOSGW是一套基于当前流行的RESTFUL协议的网关，并且兼容S3和Swift。

RBD：RBD通过Linux内核客户端和QEMU/KVM驱动来提供一个分布式的块设备。

CEPH FS：CEPH FS通过Linux内核客户端和FUSE来提供一个兼容POSIX的文件系统。

# Ceph数据分布算法

在分布式存储系统中比较关注的一点是如何使得数据能够分布得更加均衡，常见的数据分布算法有一致性Hash和Ceph的Crush算法。Crush是一种伪随机的控制数据分布、复制的算法，Ceph是为大规模分布式存储而设计的，数据分布算法必须能够满足在大规模的集群下数据依然能够快速的准确的计算存放位置，同时能够在硬件故障或扩展硬件设备时做到尽可能小的数据迁移，Ceph的CRUSH算法就是精心为这些特性设计的，可以说CRUSH算法也是Ceph的核心之一。

在说明CRUSH算法的基本原理之前，先介绍几个概念和它们之间的关系。

存储数据与object的关系：当用户要将数据存储到Ceph集群时，存储数据都会被分割成多个object，每个object都有一个object id，每个object的大小是可以设置的，默认是4MB，object可以看成是Ceph存储的最小存储单元。

object与pg的关系：由于object的数量很多，所以Ceph引入了pg的概念用于管理object，每个object最后都会通过CRUSH计算映射到某个pg中，一个pg可以包含多个object。

pg与osd的关系：pg也需要通过CRUSH计算映射到osd中去存储，如果是二副本的，则每个pg都会映射到二个osd，比如[osd.1,osd.2]，那么osd.1是存放该pg的主副本，osd.2是存放该pg的从副本，保证了数据的冗余。

pg和pgp的关系：pg是用来存放object的，pgp相当于是pg存放osd的一种排列组合，我举个例子，比如有3个osd，osd.1、osd.2和osd.3，副本数是2，如果pgp的数目为1，那么pg存放的osd组合就只有一种，可能是[osd.1,osd.2]，那么所有的pg主从副本分别存放到osd.1和osd.2，如果pgp设为2，那么其osd组合可以两种，可能是[osd.1,osd.2]和[osd.1,osd.3]，是不是很像我们高中数学学过的排列组合，pgp就是代表这个意思。一般来说应该将pg和pgp的数量设置为相等。这样说可能不够明显，我们通过一组实验来体会下：

先创建一个名为testpool包含6个PG和6个PGP的存储池

ceph osd pool create testpool 6 6

通过写数据后我们查看下pg的分布情况，使用以下命令：

ceph pg dump pgs | grep ^1 | awk '{print $1,$2,$15}'

dumped pgs in format plain

1.1 75 [3,6,0]

1.0 83 [7,0,6]

1.3 144 [4,1,2]

1.2 146 [7,4,1]

1.5 86 [4,6,3]

1.4 80 [3,0,4]

第1列为pg的id，第2列为该pg所存储的对象数目，第3列为该pg所在的osd

我们扩大PG再看看

ceph osd pool set testpool pg_num 12

再次用上面的命令查询分布情况：

1.1 37 [3,6,0]

1.9 38 [3,6,0]

1.0 41 [7,0,6]

1.8 42 [7,0,6]

1.3 48 [4,1,2]

1.b 48 [4,1,2]

1.7 48 [4,1,2]

1.2 48 [7,4,1]

1.6 49 [7,4,1]

1.a 49 [7,4,1]

1.5 86 [4,6,3]

1.4 80 [3,0,4]

我们可以看到pg的数量增加到12个了，pg1.1的对象数量本来是75的，现在是37个，可以看到它把对象数分给新增的pg1.9了，刚好是38，加起来是75，而且可以看到pg1.1和pg1.9的osd盘是一样的。

而且可以看到osd盘的组合还是那6种。

我们增加pgp的数量来看下，使用命令：

ceph osd pool set testpool pgp_num 12

再看下

1.a 49 [1,2,6]

1.b 48 [1,6,2]

1.1 37 [3,6,0]

1.0 41 [7,0,6]

1.3 48 [4,1,2]

1.2 48 [7,4,1]

1.5 86 [4,6,3]

1.4 80 [3,0,4]

1.7 48 [1,6,0]

1.6 49 [3,6,7]

1.9 38 [1,4,2]

1.8 42 [1,2,3]

再看pg1.1和pg1.9，可以看到pg1.9不在[3,6,0]上，而在[1,4,2]上了，该组合是新加的，可以知道增加pgp_num其实是增加了osd盘的组合。

通过实验总结：

（1）PG是指定存储池存储对象的目录有多少个，PGP是存储池PG的OSD分布组合个数

（2）PG的增加会引起PG内的数据进行分裂，分裂相同的OSD上新生成的PG当中

（3）PGP的增加会引起部分PG的分布进行变化，但是不会引起PG内对象的变动

pg和pool的关系：**pool也是一个逻辑存储概念，我们创建存储池pool的时候，都需要指定pg和pgp的数量，逻辑上来说pg是属于某个存储池的，就有点像object是属于某个pg的。**

以下这个图表明了存储数据，object、pg、pool、osd、存储磁盘的关系
![[Pasted image 20240201103242.png]]

  

本质上CRUSH算法是根据存储设备的权重来计算数据对象的分布的，权重的设计可以根据该磁盘的容量和读写速度来设置，比如根据容量大小可以将1T的硬盘设备权重设为1，2T的就设为2，在计算过程中，CRUSH是根据Cluster Map、数据分布策略和一个随机数共同决定数组最终的存储位置的。

Cluster Map里的内容信息包括存储集群中可用的存储资源及其相互之间的空间层次关系，比如集群中有多少个支架，每个支架中有多少个服务器，每个服务器有多少块磁盘用以OSD等。

数据分布策略是指可以通过Ceph管理者通过配置信息指定数据分布的一些特点，比如管理者配置的故障域是Host，也就意味着当有一台Host起不来时，数据能够不丢失，CRUSH可以通过将每个pg的主从副本分别存放在不同Host的OSD上即可达到，不单单可以指定Host，还可以指定机架等故障域，除了故障域，还有选择数据冗余的方式，比如副本数或纠删码。

下面这个式子简单的表明CRUSH的计算表达式：CRUSH(X)  -> (osd.1,osd.2.....osd.n)，式子中的X就是一个随机数。

下面通过一个计算PG ID的示例来看CRUSH的一个计算过程：

（1）Client输入Pool ID和对象ID；

（2）CRUSH获得对象ID并对其进行Hash运算；

（3）CRUSH计算OSD的个数，Hash取模获得PG的ID，比如0x48；

（4）CRUSH取得该Pool的ID，比如是1；

（5）CRUSH预先考虑到Pool ID相同的PG ID，比如1.48。

  

# 环境搭建

## 基础环境准备

主机IP 主机名 部署服务 备注

|   |   |   |   |
|---|---|---|---|
|主机IP|主机名|部署服务|备注|
|192.168.0.91|admin-node|ceph、ceph-deploy、mon|mon节点又称为master节点|
|192.168.0.92|ceph01|ceph|osd|
|192.168.0.93|ceph02|ceph|osd|

Ceph版本 10.2.11 Ceph-deploy版本 7.6.1810 内核版本 3.10.0-957.el7.x86_64

  

```
每个节点关闭防火墙和selinux
[root@admin-node ~]# systemctl stop firewalld
[root@admin-node ~]# systemctl disable firewalld
[root@admin-node ~]# sed -i 's/enforcing/disabled/' /etc/selinux/config

每个节点设置时间同步
yum install ntpdate -y
ntpdate time.windows.com
每个节点修改主机名
[root@admin-node ~]# hostnamectl set-hostname admin-node
[root@ceph01 ~]# hostnamectl set-hostname ceph01
[root@ceph02 ~]# hostnamectl set-hostname ceph02
每个节点修改hosts
[root@admin-node ~]# cat >> /etc/hosts <<EOF
192.168.0.91 admin-node
192.168.0.92 ceph01
192.168.0.93 ceph02
EOF
设置yum源
[root@admin-node ~]# wget -O /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-7.repo
[root@admin-node ~]# wget -O /etc/yum.repos.d/epel.repo http://mirrors.aliyun.com/repo/epel-7.repo
[root@admin-node ~]# yum clean all && yum makecache
添加ceph源
[root@admin-node ~]# cat > /etc/yum.repos.d/ceph.repo <<EOF
[ceph]
name=ceph
baseurl=http://mirrors.aliyun.com/ceph/rpm-jewel/el7/x86_64/
gpgcheck=0
priority =1
[ceph-noarch]
name=cephnoarch
baseurl=http://mirrors.aliyun.com/ceph/rpm-jewel/el7/noarch/
gpgcheck=0
priority =1
[ceph-source]
name=Ceph source packages
baseurl=http://mirrors.aliyun.com/ceph/rpm-jewel/el7/SRPMS
gpgcheck=0
priority=1
EOF
缓存yum源
[root@admin-node ~]# yum clean all && yum makecache
每个节点创建cephuser用户并设置sudo权限，密码都设置的是cephuser
[root@admin-node ~]# useradd -d /home/cephuser -m cephuser
[root@admin-node ~]# echo "cephuser"|passwd --stdin cephuser
[root@admin-node ~]# echo "cephuser ALL = (root) NOPASSWD:ALL" | sudo tee /etc/sudoers.d/cephuser
[root@admin-node ~]# chmod 0440 /etc/sudoers.d/cephuser
[root@admin-node ~]# sed -i s'/Defaults requiretty/#Defaults requiretty'/g /etc/sudoers
测试cephuser的sudo权限
[root@admin-node ~]# su - cephuser
[cephuser@admin-node ~]$ sudo su -
因为ceph-deploy不支持输入密码，所以必须要配置master节点与每个Ceph节点的ssh无密码登录。
[root@admin-node ~]# su - cephuser
[cephuser@admin-node ~]$ ssh-keygen -t rsa
[cephuser@admin-node ~]$ ssh-copy-id cephuser@admin-node
[cephuser@admin-node ~]$ ssh-copy-id cephuser@ceph01
[cephuser@admin-node ~]$ ssh-copy-id cephuser@ceph02

三台机子重启
```

  

## 准备磁盘

三台主机上都添加一块30G大小的硬盘，我这里是在vmware上添加，比较简单。
![[Pasted image 20240201103252.png]]

  

```
添加好硬盘之后，用fdisk查看下
[cephuser@admin-node ~]$ sudo fdisk -l
Disk /dev/sda: 53.7 GB, 53687091200 bytes, 104857600 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disk label type: dos
Disk identifier: 0x000a93ad
   Device Boot      Start         End      Blocks   Id  System
/dev/sda1   *        2048   104857599    52427776   83  Linux
Disk /dev/sdb: 32.2 GB, 32212254720 bytes, 62914560 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
对磁盘进行分区，分别输入“n”，“p”，“1”，然后会两次回车，再输入“w”。
[cephuser@admin-node ~]$ sudo fdisk /dev/sdb
Welcome to fdisk (util-linux 2.23.2).
Changes will remain in memory only, until you decide to write them.
Be careful before using the write command.
Device does not contain a recognized partition table
Building a new DOS disklabel with disk identifier 0x8aec5f5a.
Command (m for help): n
Partition type:
   p   primary (0 primary, 0 extended, 4 free)
   e   extended
Select (default p): p
Partition number (1-4, default 1): 1
First sector (2048-62914559, default 2048): 
Using default value 2048
Last sector, +sectors or +size{K,M,G} (2048-62914559, default 62914559): 
Using default value 62914559
Partition 1 of type Linux and of size 30 GiB is set
Command (m for help): w
The partition table has been altered!
Calling ioctl() to re-read partition table.
Syncing disks.
然后用fdisk -l就可以看到已经分区的磁盘了
[cephuser@admin-node ~]$ sudo fdisk -l
Disk /dev/sdb: 32.2 GB, 32212254720 bytes, 62914560 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disk label type: dos
Disk identifier: 0x8aec5f5a
   Device Boot      Start         End      Blocks   Id  System
/dev/sdb1            2048    62914559    31456256   83  Linux
格式化分区
[cephuser@admin-node ~]$ sudo mkfs.xfs /dev/sdb1
meta-data=/dev/sdb1              isize=512    agcount=4, agsize=1966016 blks
         =                       sectsz=512   attr=2, projid32bit=1
         =                       crc=1        finobt=0, sparse=0
data     =                       bsize=4096   blocks=7864064, imaxpct=25
         =                       sunit=0      swidth=0 blks
naming   =version 2              bsize=4096   ascii-ci=0 ftype=1
log      =internal log           bsize=4096   blocks=3839, version=2
         =                       sectsz=512   sunit=0 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0

切换到root用户
sudo su -
#添加到/etc/fstab
[root@admin-node ~]# echo '/dev/sdb1 /ceph xfs defaults 0 0' >> /etc/fstab
[root@admin-node ~]# mkdir /ceph
[root@admin-node ~]# mount -a
[root@admin-node ~]# df -h
Filesystem      Size  Used Avail Use% Mounted on
/dev/sda1        50G  1.7G   49G   4% /
devtmpfs        1.9G     0  1.9G   0% /dev
tmpfs           1.9G     0  1.9G   0% /dev/shm
tmpfs           1.9G   12M  1.9G   1% /run
tmpfs           1.9G     0  1.9G   0% /sys/fs/cgroup
tmpfs           378M     0  378M   0% /run/user/0
/dev/sdb1        30G   33M   30G   1% /ceph
```

# 部署

使用ceph-deploy工具部署ceph集群，在master节点上新建一个ceph集群，使用ceph-deploy来管理三个节点。

## 启用ceph-deploy来管理三个节点

```
[root@admin-node ~]# su - cephuser
[cephuser@admin-node ~]$ sudo yum -y install ceph ceph-deploy
```

  

创建cluster目录

```
[cephuser@admin-node ~]$ mkdir cluster
[cephuser@admin-node ~]$ cd cluster/
```

  

### master节点创建monitor

  

```
[cephuser@admin-node cluster]$ ceph-deploy new admin-node
[ceph_deploy.conf][DEBUG ] found configuration file at: /home/cephuser/.cephdeploy.conf
[ceph_deploy.cli][INFO  ] Invoked (1.5.39): /bin/ceph-deploy new admin-node
[ceph_deploy.cli][INFO  ] ceph-deploy options:
[ceph_deploy.cli][INFO  ]  username                      : None
[ceph_deploy.cli][INFO  ]  func                          : <function new at 0x7f27b483f5f0>
[ceph_deploy.cli][INFO  ]  verbose                       : False
[ceph_deploy.cli][INFO  ]  overwrite_conf                : False
[ceph_deploy.cli][INFO  ]  quiet                         : False
[ceph_deploy.cli][INFO  ]  cd_conf                       : <ceph_deploy.conf.cephdeploy.Conf instance at 0x7f27b3fbc5a8>
[ceph_deploy.cli][INFO  ]  cluster                       : ceph
[ceph_deploy.cli][INFO  ]  ssh_copykey                   : True
[ceph_deploy.cli][INFO  ]  mon                           : ['admin-node']
[ceph_deploy.cli][INFO  ]  public_network                : None
[ceph_deploy.cli][INFO  ]  ceph_conf                     : None
[ceph_deploy.cli][INFO  ]  cluster_network               : None
[ceph_deploy.cli][INFO  ]  default_release               : False
[ceph_deploy.cli][INFO  ]  fsid                          : None
[ceph_deploy.new][DEBUG ] Creating new cluster named ceph
[ceph_deploy.new][INFO  ] making sure passwordless SSH succeeds
[admin-node][DEBUG ] connection detected need for sudo
[admin-node][DEBUG ] connected to host: admin-node 
[admin-node][DEBUG ] detect platform information from remote host
[admin-node][DEBUG ] detect machine type
[admin-node][DEBUG ] find the location of an executable
[admin-node][INFO  ] Running command: sudo /usr/sbin/ip link show
[admin-node][INFO  ] Running command: sudo /usr/sbin/ip addr show
[admin-node][DEBUG ] IP addresses found: [u'192.168.0.91']
[ceph_deploy.new][DEBUG ] Resolving host admin-node
[ceph_deploy.new][DEBUG ] Monitor admin-node at 192.168.0.91
[ceph_deploy.new][DEBUG ] Monitor initial members are ['admin-node']
[ceph_deploy.new][DEBUG ] Monitor addrs are ['192.168.0.91']
[ceph_deploy.new][DEBUG ] Creating a random mon key...
[ceph_deploy.new][DEBUG ] Writing monitor keyring to ceph.mon.keyring...
[ceph_deploy.new][DEBUG ] Writing initial config to ceph.conf...
执行这条命令之后，admin-node节点作为了monitor节点，如果想多个mon节点可以实现互备，需要加上其他节点并且节点需要安装ceph-deploy
```

  

### 修改ceph.conf文件

当上面的步骤执行完成之后，在cluster目录下会产生ceph.conf文件，这里需要修改ceph.conf文件（注意：mon_host必须和public network网络是同网段的），添加下面4行：

```
osd pool default size = 3
rbd_default_features = 1
public network = 192.168.0.0/24
osd journal size = 2000

解释如下：
# osd pool default size = 3
设置默认副本数为3份，有一个副本故障，其他2个副本的osd可正常提供服务，需要注意的是如果设置为副本数为3，osd总数量需要是3的倍数。
# rbd_default_features = 1
增加 rbd_default_features 配置可以永久的改变默认值。注意：这种方式设置是永久性的，要注意在集群各个node上都要修改。
# public network = 192.168.0.0/24
设置集群所在的网段
# osd journal size = 2000
OSD日志大小，单位是MB
```

  

  

  

### master节点安装ceph

这个过程有点长，需要等待一段时间

```
[cephuser@admin-node cluster]$ ceph-deploy install admin-node ceph01 ceph02
```

  

### master节点初始化mon节点

master节点初始化mon节点及收集秘钥信息

```
cephuser@admin-node cluster]$ ceph-deploy mon create-initial
[ceph_deploy.conf][DEBUG ] found configuration file at: /home/cephuser/.cephdeploy.conf
[ceph_deploy.cli][INFO  ] Invoked (1.5.39): /bin/ceph-deploy mon create-initial
[ceph_deploy.cli][INFO  ] ceph-deploy options:
[ceph_deploy.cli][INFO  ]  username                      : None
[ceph_deploy.cli][INFO  ]  verbose                       : False
[ceph_deploy.cli][INFO  ]  overwrite_conf                : False
[ceph_deploy.cli][INFO  ]  subcommand                    : create-initial
[ceph_deploy.cli][INFO  ]  quiet                         : False
[ceph_deploy.cli][INFO  ]  cd_conf                       : <ceph_deploy.conf.cephdeploy.Conf instance at 0x7fbc00e79ef0>
[ceph_deploy.cli][INFO  ]  cluster                       : ceph
[ceph_deploy.cli][INFO  ]  func                          : <function mon at 0x7fbc00e566e0>
[ceph_deploy.cli][INFO  ]  ceph_conf                     : None
[ceph_deploy.cli][INFO  ]  default_release               : False
[ceph_deploy.cli][INFO  ]  keyrings                      : None
[ceph_deploy.mon][DEBUG ] Deploying mon, cluster ceph hosts admin-node
[ceph_deploy.mon][DEBUG ] detecting platform for host admin-node ...
[admin-node][DEBUG ] connection detected need for sudo
[admin-node][DEBUG ] connected to host: admin-node 
......
[admin-node][INFO  ] Running command: sudo /usr/bin/ceph --connect-timeout=25 --cluster=ceph --name mon. --keyring=/var/lib/ceph/mon/ceph-admin-node/keyring auth get client.bootstrap-rgw
[ceph_deploy.gatherkeys][INFO  ] Storing ceph.client.admin.keyring
[ceph_deploy.gatherkeys][INFO  ] Storing ceph.bootstrap-mds.keyring
[ceph_deploy.gatherkeys][INFO  ] Storing ceph.bootstrap-mgr.keyring
[ceph_deploy.gatherkeys][INFO  ] keyring 'ceph.mon.keyring' already exists
[ceph_deploy.gatherkeys][INFO  ] Storing ceph.bootstrap-osd.keyring
[ceph_deploy.gatherkeys][INFO  ] Storing ceph.bootstrap-rgw.keyring
[ceph_deploy.gatherkeys][INFO  ] Destroy temp directory /tmp/tmpmLTa1b
```

  

  

### master节点创建osd

组概念：[root@localhost ~]# chown -R mysql:mysql ./

这两个mysql谁是用户名谁是用户组呢？chown将指定文件的拥有者改为指定的用户或组，

用户可以是用户名或者用户ID；

组可以是组名或者组ID；

文件是以空格分开的要改变权限的文件列表，支持通配符。

系统管理员经常使用chown命令，在将文件拷贝到另一个用户的名录下之后，让用户拥有使用该文件的权限。

```
创建osd
[cephuser@admin-node cluster]$ ceph-deploy osd prepare admin-node:/ceph ceph01:/ceph ceph02:/ceph
激活osd
在激活之前需要先将/ceph目录的权限改为ceph。因为从I版本起，ceph的守护进程以ceph用户而不是root用户运行。而osd在prepare和activate之前需要将硬盘挂载到指定目录去。而创建指定目录（例如/ceph）时，用户可能是非ceph用户例如root用户等。
[cephuser@admin-node cluster]$ sudo chown -R ceph:ceph /ceph
[cephuser@admin-node cluster]$ ceph-deploy osd activate admin-node:/ceph ceph01:/ceph ceph02:/ceph
```

注意还会报错：

[ceph_deploy][ERROR ] RuntimeError: Failedto execute command: /usr/sbin/ceph-disk -v activate --mark-init systemd --mount/var/local/osd0

原因是：创建节点的目录权限不够；解决办法：进入到各节点上运行chmod 777  /var/local/osd0

```
sudo su -
chmod 777  /ceph
```

  

  

  

### master创建mon节点

master创建mon节点-监控集群状态-同时管理集群及msd

```
[cephuser@admin-node cluster]$ ceph-deploy mon create admin-node
如果需要管理集群节点，需要在node节点安装ceph-deploy，因为ceph-deploy是管理ceph节点的工具。
如果需要增加其他的mon机器，可以使用 ceph-deploy mon add  主机名
```

  

  

修改秘钥权限

```
[cephuser@admin-node cluster]$ sudo chmod 644 /etc/ceph/ceph.client.admin.keyring
```

  

## 状态检查

### 检查ceph状态

```
[cephuser@admin-node cluster]$ sudo ceph health
HEALTH_OK
[cephuser@admin-node cluster]$ sudo ceph -s
    cluster e76f5c31-89d6-4d5f-a302-8f1faf17655e
     health HEALTH_OK
     monmap e1: 1 mons at {admin-node=192.168.0.91:6789/0}
            election epoch 4, quorum 0 admin-node
     osdmap e15: 3 osds: 3 up, 3 in
            flags sortbitwise,require_jewel_osds
      pgmap v31: 64 pgs, 1 pools, 0 bytes data, 0 objects
            6321 MB used, 85790 MB / 92112 MB avail
                  64 active+clean
[cephuser@admin-node cluster]$ ceph osd stat
     osdmap e15: 3 osds: 3 up, 3 in
            flags sortbitwise,require_jewel_osds
```

  

### 检查osd运行状态

```
[cephuser@admin-node cluster]$ ceph osd stat
     osdmap e15: 3 osds: 3 up, 3 in
            flags sortbitwise,require_jewel_osds
```

  

### 查看ceph集群磁盘

```
[cephuser@admin-node cluster]$ ceph df
GLOBAL:
    SIZE       AVAIL      RAW USED     %RAW USED 
    92112M     85790M        6321M          6.86 
POOLS:
    NAME     ID     USED     %USED     MAX AVAIL     OBJECTS 
    rbd      0         0         0        27061M           0 
这里3台服务器每台30G硬盘组成90G
```

  

### 查看osd节点状态

```
[cephuser@admin-node cluster]$ ceph osd tree
ID WEIGHT  TYPE NAME           UP/DOWN REWEIGHT PRIMARY-AFFINITY 
-1 0.08789 root default                                          
-2 0.02930     host admin-node                                   
 0 0.02930         osd.0            up  1.00000          1.00000 
-3 0.02930     host ceph01                                       
 1 0.02930         osd.1            up  1.00000          1.00000 
-4 0.02930     host ceph02                                       
 2 0.02930         osd.2            up  1.00000          1.00000
```

  

### 查看osd的状态

查看osd的状态-负责数据存放的位置

```
[cephuser@admin-node cluster]$ ceph-deploy osd list admin-node ceph01 ceph02
[ceph_deploy.conf][DEBUG ] found configuration file at: /home/cephuser/.cephdeploy.conf
[ceph_deploy.cli][INFO  ] Invoked (1.5.39): /bin/ceph-deploy osd list admin-node ceph01 ceph02
[ceph_deploy.cli][INFO  ] ceph-deploy options:
[ceph_deploy.cli][INFO  ]  username                      : None
[ceph_deploy.cli][INFO  ]  verbose                       : False
[ceph_deploy.cli][INFO  ]  overwrite_conf                : False
[ceph_deploy.cli][INFO  ]  subcommand                    : list
[ceph_deploy.cli][INFO  ]  quiet                         : False
[ceph_deploy.cli][INFO  ]  cd_conf                       : <ceph_deploy.conf.cephdeploy.Conf instance at 0x7f614b4ba170>
[ceph_deploy.cli][INFO  ]  cluster                       : ceph
[ceph_deploy.cli][INFO  ]  func                          : <function osd at 0x7f614b4fdf50>
[ceph_deploy.cli][INFO  ]  ceph_conf                     : None
[ceph_deploy.cli][INFO  ]  default_release               : False
[ceph_deploy.cli][INFO  ]  disk                          : [('admin-node', None, None), ('ceph01', None, None), ('ceph02', None, None)]
[admin-node][DEBUG ] connection detected need for sudo
[admin-node][DEBUG ] connected to host: admin-node 
[admin-node][DEBUG ] detect platform information from remote host
[admin-node][DEBUG ] detect machine type
[admin-node][DEBUG ] find the location of an executable
[admin-node][DEBUG ] find the location of an executable
[admin-node][INFO  ] Running command: sudo /bin/ceph --cluster=ceph osd tree --format=json
[admin-node][DEBUG ] connection detected need for sudo
[admin-node][DEBUG ] connected to host: admin-node 
[admin-node][DEBUG ] detect platform information from remote host
[admin-node][DEBUG ] detect machine type
[admin-node][DEBUG ] find the location of an executable
[admin-node][INFO  ] Running command: sudo /usr/sbin/ceph-disk list
[admin-node][INFO  ] ----------------------------------------
[admin-node][INFO  ] ceph-0
[admin-node][INFO  ] ----------------------------------------
[admin-node][INFO  ] Path           /var/lib/ceph/osd/ceph-0
[admin-node][INFO  ] ID             0
[admin-node][INFO  ] Name           osd.0
[admin-node][INFO  ] Status         up
[admin-node][INFO  ] Reweight       1.0
[admin-node][INFO  ] Active         ok
[admin-node][INFO  ] Magic          ceph osd volume v026
[admin-node][INFO  ] Whoami         0
[admin-node][INFO  ] Journal path   /ceph/journal
[admin-node][INFO  ] ----------------------------------------
[ceph01][DEBUG ] connection detected need for sudo
[ceph01][DEBUG ] connected to host: ceph01 
[ceph01][DEBUG ] detect platform information from remote host
[ceph01][DEBUG ] detect machine type
[ceph01][DEBUG ] find the location of an executable
[ceph01][INFO  ] Running command: sudo /usr/sbin/ceph-disk list
[ceph01][INFO  ] ----------------------------------------
[ceph01][INFO  ] ceph-1
[ceph01][INFO  ] ----------------------------------------
[ceph01][INFO  ] Path           /var/lib/ceph/osd/ceph-1
[ceph01][INFO  ] ID             1
[ceph01][INFO  ] Name           osd.1
[ceph01][INFO  ] Status         up
[ceph01][INFO  ] Reweight       1.0
[ceph01][INFO  ] Active         ok
[ceph01][INFO  ] Magic          ceph osd volume v026
[ceph01][INFO  ] Whoami         1
[ceph01][INFO  ] Journal path   /ceph/journal
[ceph01][INFO  ] ----------------------------------------
[ceph02][DEBUG ] connection detected need for sudo
[ceph02][DEBUG ] connected to host: ceph02 
[ceph02][DEBUG ] detect platform information from remote host
[ceph02][DEBUG ] detect machine type
[ceph02][DEBUG ] find the location of an executable
[ceph02][INFO  ] Running command: sudo /usr/sbin/ceph-disk list
[ceph02][INFO  ] ----------------------------------------
[ceph02][INFO  ] ceph-2
[ceph02][INFO  ] ----------------------------------------
[ceph02][INFO  ] Path           /var/lib/ceph/osd/ceph-2
[ceph02][INFO  ] ID             2
[ceph02][INFO  ] Name           osd.2
[ceph02][INFO  ] Status         up
[ceph02][INFO  ] Reweight       1.0
[ceph02][INFO  ] Active         ok
[ceph02][INFO  ] Magic          ceph osd volume v026
[ceph02][INFO  ] Whoami         2
[ceph02][INFO  ] Journal path   /ceph/journal
[ceph02][INFO  ] ----------------------------------------
```

  

### 查看集群mon选举状态

```
[cephuser@admin-node cluster]$ ceph quorum_status --format json-pretty
{
    "election_epoch": 4,
    "quorum": [
        0
    ],
    "quorum_names": [
        "admin-node"
    ],
    "quorum_leader_name": "admin-node",
    "monmap": {
        "epoch": 1,
        "fsid": "e76f5c31-89d6-4d5f-a302-8f1faf17655e",
        "modified": "2020-09-28 15:18:13.294522",
        "created": "2020-09-28 15:18:13.294522",
        "mons": [
            {
                "rank": 0,
                "name": "admin-node",
                "addr": "192.168.0.91:6789\/0"
            }
        ]
    }
}
```

  

## 创建文件的系统

```
先查看管理节点状态，默认是没有管理节点的
[cephuser@admin-node ~]$ ceph mds stat
e1:
创建管理节点（admin-node作为管理节点）
需要注意：如果不创建mds管理节点的，client客户端将不能正常挂载到ceph集群！！
[cephuser@admin-node ~]$ cd /home/cephuser/cluster/
[cephuser@admin-node cluster]$ ceph-deploy mds create admin-node
再次查看管理节点状态，发现已经在启动中了
[cephuser@admin-node cluster]$ ceph mds stat
e2:, 1 up:standby
创建pool池，pool是ceph存储数据时的逻辑分区，它起到namespace的作用
[cephuser@admin-node cluster]$ ceph osd lspools #先查看pool
0 rbd,
新创建的ceph集群只有rdb一个pool。这时需要创建一个新的pool
[cephuser@admin-node cluster]$ ceph osd pool create cephfs_data 128 #后面的数据是pg的数量
pool 'cephfs_data' created 

##命令格式##
#创建pool
ceph osd pool create [pool池名称] 128
#删除pool
ceph osd pool delete [pool池名称]
#调整副本数量
ceph osd pool set [pool池名称] size 2
###

[cephuser@admin-node cluster]$ ceph osd pool create cephfs_metadata 128 #pool的元数据
pool 'cephfs_metadata' created
[cephuser@admin-node cluster]$ ceph fs new mycephfs cephfs_metadata cephfs_data
new fs with metadata pool 2 and data pool 1
再次查看pool状态
[cephuser@admin-node cluster]$ ceph osd lspools
0 rbd,1 cephfs_data,2 cephfs_metadata,
检查mds管理节点状态
[cephuser@admin-node cluster]$ ceph mds stat
e5: 1/1/1 up {0=admin-node=up:active}
检查ceph集群状态
[cephuser@admin-node cluster]$ ceph -s
    cluster e76f5c31-89d6-4d5f-a302-8f1faf17655e
     health HEALTH_WARN
            too many PGs per OSD (320 > max 300)
     monmap e1: 1 mons at {admin-node=192.168.0.91:6789/0}
            election epoch 5, quorum 0 admin-node
      fsmap e5: 1/1/1 up {0=admin-node=up:active}
     osdmap e29: 3 osds: 3 up, 3 in
            flags sortbitwise,require_jewel_osds
      pgmap v248: 320 pgs, 3 pools, 2068 bytes data, 20 objects
            6323 MB used, 85788 MB / 92112 MB avail
                 320 active+clean
```

  

## 挂载ceph服务器

由于我这里的cephfs存储是用来给pod中的java应用存放日志的，所以可以将cephfs存储挂载到一台服务器上，方便查看日志。

```
查看并记录admin用户的key，admin用户默认就存在，不需要创建
[cephuser@admin-node cluster]$ ceph auth get-key client.admin
AQA2jnFfBT8QBRAAd504A08U+Jlk/pt6TkcS4Q==
查看用户权限
[cephuser@admin-node cluster]$ ceph auth get client.admin
exported keyring for client.admin
[client.admin]
        key = AQA2jnFfBT8QBRAAd504A08U+Jlk/pt6TkcS4Q==
        caps mds = "allow *"
        caps mon = "allow *"
        caps osd = "allow *"
查看ceph授权
[cephuser@admin-node cluster]$ ceph auth list
installed auth entries:
mds.admin-node
        key: AQCj+XNfUd6qLBAAERbdVdkbeV3AmKyxvDLDyA==
        caps: [mds] allow
        caps: [mon] allow profile mds
        caps: [osd] allow rwx
osd.0
        key: AQAZqHJfsM7fCxAAX4BRw+HxNAPMezGpKKTafw==
        caps: [mon] allow profile osd
        caps: [osd] allow *
osd.1
        key: AQDMqHJfJxAIAhAAyccoTUr5PciteZ5X/vTRLw==
        caps: [mon] allow profile osd
        caps: [osd] allow *
osd.2
        key: AQDXqHJfkilwCBAAtw2QqKZ6LATMijK/8Ggi7Q==
        caps: [mon] allow profile osd
        caps: [osd] allow *
client.admin
        key: AQA2jnFfBT8QBRAAd504A08U+Jlk/pt6TkcS4Q==
        caps: [mds] allow *
        caps: [mon] allow *
        caps: [osd] allow *
client.bootstrap-mds
        key: AQA2jnFfRdOMLRAA9LuyGs5YsZYfH+Mu4wQrsA==
        caps: [mon] allow profile bootstrap-mds
client.bootstrap-mgr
        key: AQA5jnFfp87tDxAAtx4rvkE7OIobER2iKQDyew==
        caps: [mon] allow profile bootstrap-mgr
client.bootstrap-osd
        key: AQA2jnFfocEJFRAArMJYW9Bjd8B3zT/PpsRZJg==
        caps: [mon] allow profile bootstrap-osd
client.bootstrap-rgw
        key: AQA2jnFfrOQqIRAAmH3v7pvVNBmy5CqxtxeNsg==
        caps: [mon] allow profile bootstrap-rgw
```

  

### 创建第四台机子-客户机

注意添加第四台机子作为客户机。

### 客户机配置ceph镜像源

客户机需要安装ceph，具体yum源上面已经写过，这里不再赘述:

```
设置yum源
[root@admin-node ~]# wget -O /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-7.repo
[root@admin-node ~]# wget -O /etc/yum.repos.d/epel.repo http://mirrors.aliyun.com/repo/epel-7.repo
[root@admin-node ~]# yum clean all && yum makecache
添加ceph源
[root@admin-node ~]# cat > /etc/yum.repos.d/ceph.repo <<EOF
[ceph]
name=ceph
baseurl=http://mirrors.aliyun.com/ceph/rpm-jewel/el7/x86_64/
gpgcheck=0
priority =1
[ceph-noarch]
name=cephnoarch
baseurl=http://mirrors.aliyun.com/ceph/rpm-jewel/el7/noarch/
gpgcheck=0
priority =1
[ceph-source]
name=Ceph source packages
baseurl=http://mirrors.aliyun.com/ceph/rpm-jewel/el7/SRPMS
gpgcheck=0
priority=1
EOF
缓存yum源
[root@admin-node ~]# yum clean all && yum makecache
```

赘述完毕。

---

### 客户机安装ceph包

```
[root@localhost ~]# yum -y install ceph
[root@localhost ~]# mkdir /mnt/cephfs
root@localhost ~]# mount.ceph 192.168.0.91:6789:/  /mnt/cephfs/ -o name=admin,secret=AQA2jnFfBT8QBRAAd504A08U+Jlk/pt6TkcS4Q==
[root@localhost ~]# df -h
Filesystem           Size  Used Avail Use% Mounted on
/dev/sda1             50G  2.0G   48G   4% /
devtmpfs             1.9G     0  1.9G   0% /dev
tmpfs                1.9G     0  1.9G   0% /dev/shm
tmpfs                1.9G   12M  1.9G   1% /run
tmpfs                1.9G     0  1.9G   0% /sys/fs/cgroup
tmpfs                378M     0  378M   0% /run/user/0
192.168.0.91:6789:/   90G  6.2G   84G   7% /mnt/cephfs
```

### 客户机挂载ceph服务器

  

```
并在etc/fstab中加入以下配置：
192.168.0.91:6789:/  /mnt/cephfs/ ceph     name=admin,secret=AQA2jnFfBT8QBRAAd504A08U+Jlk/pt6TkcS4Q==   0   0

使用echo方式：#虚拟机出现问题，写在etc/fstab下客户机无法开机，
echo '192.168.18.100:6789:/  /mnt/cephfs/ ceph     name=admin,secret=AQB18wBhWTATLBAA5QT0KFQpVM0Dgg7VLY2c5Q==   0   0' >> /etc/fstab

#我们每次手动挂载
mount.ceph 192.168.18.100:6789:/  /mnt/cephfs/ -o name=admin,secret=AQB18wBhWTATLBAA5QT0KFQpVM0Dgg7VLY2c5Q==
```

### 客户机配置ceph密钥

下面在ceph-master创建秘钥

```
sudo su -
ceph auth get-key client.admin > /opt/secret
这一步其实有问题：ceph-master的/opt/secret 跟k8s-master的/opt/secret不是一回事，k8s访问不了ceph-master的目录
应该在ceph-master执行完ceph auth get-key client.admin查出来AQA4hAJhJObtGhAAyed8HJTiEs3KeZhXAzbZBw==
拿着AQA4hAJhJObtGhAAyed8HJTiEs3KeZhXAzbZBw==手动复制到k8s-master的/opt/secret，
再执行下面的操作。
```

在k8s-master操作：

```
kubectl create ns cephfs
kubectl create secret generic ceph-secret-admin --from-file=/opt/secret --namespace=cephfs
```

## 客户机配置storage class模式

  

前置注意点：

1.需要在k8s的master和node节点中都安装ceph，因为我后面还要使用storageclass模式，还需要在k8s的master节点中安装ceph-common，否则会报错。

2.因为只有cephfs(即文件类的存储)才能够支持ReadWriteMany，所以上面创建的是ceph的文件系统cephfs。

3.在k8s中使用cephfs存储时，节点中必须安装ceph-fuse，不然会报错。

### 创建Provisioner

  

部署Provisioner

external-storage.git已经下载好：

```
git clone https://github.com/kubernetes-retired/external-storage.git
cd external-storage/ceph/cephfs/deploy
NAMESPACE=cephfs
sed -r -i "s/namespace: [^ ]+/namespace: $NAMESPACE/g" ./rbac/*.yaml
sed -r -i "N;s/(name: PROVISIONER_SECRET_NAMESPACE.*\n[[:space:]]*)value:.*/\1value: $NAMESPACE/" ./rbac/deployment.yaml
kubectl -n $NAMESPACE apply -f ./rbac
检查pod的状态是否正常
kubectl get pods -n cephfs
```

### 创建StorageClass,pvc以及部署pod测试（ceph类型）

返回root根目录：

```
创建Storageclass
cat > storageclass.yaml <<EOF
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: cephfs
provisioner: ceph.com/cephfs
parameters:
    monitors: 192.168.0.91:6789   #换成自己的ceph地址
    adminId: admin
    adminSecretName: ceph-secret-admin
    adminSecretNamespace: cephfs
    claimRoot: /logs
EOF
kubectl apply -f storageclass.yaml 

##############################################################################
创建pvc
cat > pvc.yaml <<EOF
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: claim1
spec:
  storageClassName: cephfs #指定上面的PV（storageclass）名字,在pv的元属性里面写着cephfs
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 2Gi
EOF
kubectl apply -f pvc.yaml
##############################################################################
部署2个pod
apiVersion: v1
kind: Pod
metadata:
  name: pod-testceph-storageclass1
  namespace: default
spec:
  containers:
  - name: nginx
    image: nginx:1.17.1
    command: ["/bin/sh","-c","while true;do echo pod-testceph_storageclass1 >> /var/out.txt; sleep 10; done;"]
    volumeMounts: # 将volume(共享存储)挂载到nginx容器中的对应目录，该目录为/var/
    - name: volume
      mountPath: /var/
  volumes:
    - name: volume
      persistentVolumeClaim:
        claimName: claim1
        readOnly: false

---
apiVersion: v1
kind: Pod
metadata:
  name: pod-testceph-storageclass2
  namespace: default
spec:
  containers:
    - name: nginx
      image: nginx:1.17.1
      command: ["/bin/sh","-c","while true;do echo pod-testceph_storageclass2 >> /var/out.txt; sleep 10; done;"]
      volumeMounts: #将volume(共享存储)挂载到nginx容器中的对应目录，该目录为/var/
        - name: volume
          mountPath: /var/
  volumes:
    - name: volume
      persistentVolumeClaim:
        claimName: claim1
        readOnly: false
```

  

### 查看pvc

```
kubectl get pvc
NAME     STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
claim1   Bound    pvc-7c1740a3-a3ab-4889-9d94-e50c736701fd   2Gi        RWX            cephfs         27m
```

  

  

上面创建好了StorageClass和PVC之后，我这边的java应用就可以直接使用这个pvc进行日志目录的挂载了，并且是多个pod可以同时使用同一个PVC的，这样方便了日志的管理和收集工作。

  

注意：在这里还是要再次强调下，ceph存储的文件系统cephfs才支持ReadWriteMany哦。

  

## 补充说明：

yum安装报错：[Errno 14] curl#6 - "Could not resolve host: mirrors.aliyun.com; Unknown error"

[root@localhost /]# yum install memcached

[http://mirrors.aliyun.com/centos/7/os/x86_64/repodata/repomd.xml:](http://mirrors.aliyun.com/centos/7/os/x86_64/repodata/repomd.xml:) [Errno 14] curl#6 - "Could not resolve host: mirrors.aliyun.com; Unknown error"

Trying other mirror.

[http://mirrors.aliyuncs.com/centos/7/os/x86_64/repodata/repomd.xml:](http://mirrors.aliyuncs.com/centos/7/os/x86_64/repodata/repomd.xml:) [Errno 14] curl#6 - "Could not resolve host: mirrors.aliyuncs.com; Unknown error"

Trying other mirror.

Solution:

[root@localhost /]# service networt restart

Redirecting to /bin/systemctl restart networt.service

Failed to restart networt.service: Unit not found.

[root@localhost /]# service network restart

Restarting network (via systemctl):