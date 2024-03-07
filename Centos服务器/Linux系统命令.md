# 查系统

```Shell

#查看系统版本
cat /etc/redhat-release
```

# 查CPU
计算逻辑cpu线程总数
```shell
# 查看CPU信息（型号）
cat /proc/cpuinfo | grep name | cut -f2 -d: | uniq -c
24    Intel(R) Xeon(R) CPU E5-2630 0 @ 2.30GHz


# 查看物理CPU块数
cat /proc/cpuinfo| grep "physical id"| sort| uniq| wc -l
2

# 查看每个物理CPU中核数
cat /proc/cpuinfo| grep "cpu cores"| uniq
cpu cores    : 6

# 查看逻辑CPU的个数=cpu块数*核数*线程数，可以推断出线程数为2
cat /proc/cpuinfo| grep "processor"| wc -l
24


```
查看cpu占用情况
```shell
top
```
# 查磁盘

```Shell
#查看磁盘
df -hl
```

# 查内存

1. 使用 free 命令
    

1. free 命令是Linux系统中最简单和最常用的内存查看命令， 示例如下:
    

```Shell
$ free -m
              total        used        free      shared  buff/cache   available
Mem:           7822         321         324         377        7175        6795
Swap:          4096           0        4095


$ free -h
              total        used        free      shared  buff/cache   available
Mem:           7.6G        322M        324M        377M        7.0G        6.6G
Swap:          4.0G        724K        4.0G
```

其中， -m 选项是以MB为单位来展示内存使用信息; -h 选项则是以人类(human)可读的单位来展示。

上面的示例中, Mem: 这一行：

- total 表示总共有 7822MB 的物理内存(RAM)，即7.6G。
    
- used 表示物理内存的使用量，大约是 322M。
    
- free 表示空闲内存;
    
- shared 表示共享内存?;
    
- buff/cache 表示缓存和缓冲内存量; Linux 系统会将很多东西缓存起来以提高性能，这部分内存可以在必要时进行释放，给其他程序使用。
    
- available 表示可用内存;
    

输出结果很容易理解。 Swap 这一行表示交换内存，从示例中的数字可以看到，基本上没使用到交换内存。

2. 查看 /proc/meminfo
    

1. 另一种方法是读取 /proc/meminfo 文件。 我们知道， /proc 目录下都是虚拟文件，包含内核以及操作系统相关的动态信息。
    

```Shell
$ cat /proc/meminfo
MemTotal:        8010408 kB
MemFree:          323424 kB
MemAvailable:    6956280 kB
Buffers:          719620 kB
Cached:          5817644 kB
SwapCached:          132 kB
Active:          5415824 kB
Inactive:        1369528 kB
Active(anon):     385660 kB
Inactive(anon):   249292 kB
Active(file):    5030164 kB
Inactive(file):  1120236 kB
Unevictable:           0 kB
Mlocked:               0 kB
SwapTotal:       4194304 kB
SwapFree:        4193580 kB
Dirty:                60 kB
Writeback:             0 kB
AnonPages:        247888 kB
Mapped:            61728 kB
Shmem:            386864 kB
Slab:             818320 kB
SReclaimable:     788436 kB
SUnreclaim:        29884 kB
KernelStack:        2848 kB
PageTables:         5780 kB
NFS_Unstable:          0 kB
Bounce:                0 kB
WritebackTmp:          0 kB
CommitLimit:     8199508 kB
Committed_AS:     942596 kB
VmallocTotal:   34359738367 kB
VmallocUsed:       22528 kB
VmallocChunk:   34359707388 kB
HardwareCorrupted:     0 kB
AnonHugePages:     88064 kB
HugePages_Total:       0
HugePages_Free:        0
HugePages_Rsvd:        0
HugePages_Surp:        0
Hugepagesize:       2048 kB
DirectMap4k:      176000 kB
DirectMap2M:     6115328 kB
DirectMap1G:     4194304 kB
```

重点关注这些数据:

- MemTotal, 总内存
    
- MemFree, 空闲内存
    
- MemAvailable, 可用内存
    
- Buffers, 缓冲
    
- Cached, 缓存
    
- SwapTotal, 交换内存
    
- SwapFree, 空闲交换内存
    

提供的信息和 free 命令看到的差不多。

3. 使用 top 命令
    

1. top 命令一般用于查看进程的CPU和内存使用情况；当然也会报告内存总量，以及内存使用情况，所以可用来监控物理内存的使用情况。
    

1. 重点关注顶部的 KiB Mem 和 KiB Swap 这两行。 表示内存的总量、使用量，以及可用量。
    

buffer 和 cache 部分，和 free 命令展示的差不多。

```Shell

top - 15:20:30 up  6:57,  5 users,  load average: 0.64, 0.44, 0.33
Tasks: 265 total,   1 running, 263 sleeping,   0 stopped,   1 zombie
%Cpu(s):  7.8 us,  2.4 sy,  0.0 ni, 88.9 id,  0.9 wa,  0.0 hi,  0.0 si,  0.0 st
KiB Mem:   8167848 total,  6642360 used,  1525488 free,  1026876 buffers
KiB Swap:  1998844 total,        0 used,  1998844 free,  2138148 cached

PID USER      PR  NI  VIRT  RES  SHR S  %CPU %MEM    TIME+  COMMAND 2986 enlighte  20   0  584m  42m  26m S  14.3  0.5   0:44.27 yakuake 1305 root      20   0  448m  68m  39m S   5.0  0.9   3:33.98 Xorg 7701 enlighte  20   0  424m  17m  10m S   4.0  0.2   0:00.12 kio_thumbnail
```

各种操作系统提供的参数略有不同，一般来说都可以根据CPU和内存来排序。例如:

```Shell
#CentOS
top -o %MEM
top -o %CPU

#mac
top -o mem
top -o cpu
```

碰到不清楚的，请使用 top -h 查看帮助信息。

# 防火墙

```Java
关闭防火墙：
$ systemctl stop firewalld
$ systemctl disable firewalld
```

# 查进程

```Shell
#查看进程方法一：根据端口号##################
lsof -i:37017
COMMAND     PID USER   FD   TYPE   DEVICE SIZE/OFF NODE NAME
docker-pr 17735 root    4u  IPv6 69885239      0t0  TCP :37017 (LISTEN)
kill -9 {PID}    
PID=17735
#查看进程方法二：根据端口号#################
netstat -anp|grep 37017
netstat -npl|grep 37017
tcp6       0      0 :::37017                :::                    LISTEN      17735/docker-proxy
查看具体端口时候，必须要看到tcp，端口号，LISTEN那一行，才表示端口被占用了。LISTENING并不表示端口被占用，不要和LISTEN混淆哦
kill -9 17735   
#查看进程方法三：根据服务名#################
ps -ef | grep xxxname 
ps -ef | grep wefe
UID       PID       PPID      C     STIME    TTY       TIME         CMD
root      5148     5061      0     14:32    pts/0    00:00:01     python3 /opt/welab/wefe/flow/app_launcher.py
root      5196     3147      0     14:37    pts/0    00:00:00     grep --color=auto wefe
UID      ：程序被该 UID 所拥有
PID      ：就是这个程序的 ID 
PPID    ：则是其上级父程序的ID
C          ：CPU使用的资源百分比
STIME ：系统启动时间
TTY     ：登入者的终端机位置
TIME   ：使用掉的CPU时间。
CMD   ：所下达的是什么指令
```

# 查网络

```Java

#查看GATEWAY，DNS
cat /etc/sysconfig/network-scripts/ifcfg-eth0
# Generated by VMWare customization engine.
HWADDR=00:50:56:af:5d:f4
NAME=eth0
GATEWAY=10.10.178.254
DNS1=10.45.40.7
DEVICE=eth0
ONBOOT=yes
USERCTL=no
BOOTPROTO=static
NETMASK=255.255.255.0
IPADDR=10.10.178.148
PEERDNS=no
IPV6INIT=yes
IPV6_AUTOCONF=yes

check_link_down() {
 return 1; 
}

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


#内网地址:10.10.178.147     
ifconfig -a(eth0)
#外网地址:222.95.251.228    
curl http://httpbin.org/ip   
curl cip.cc  
curl ifconfig.me

#telnet查看端口是否通，注意中间是个空格###
telnet 10.10.178.147 3305
```

# 用户

su username：**切换用户后，不改变原用户的工作目录，及其他环境变量目录**

su - username: **切换用户后，同时切换到新用户的工作环境中**

# 主机名

```Java
hostnamectl set-hostname <hostname>
```

# 后台运行jar包

```Java
Linux运行jar包
要运行java的项目需要先将项目打包成war包或者jar包，打包成war包需要将war包部署到tomcat服务器上才能运行。而打包成jar包可以直接使用java命令执行。
 
在linux系统中运行jar包主要有以下几种方式。
 
一、java -jar XXX.jar
 
这是最基本的jar包执行方式，但是当我们用ctrl+c中断或者关闭窗口时，程序也会中断执行。
 
二、java -jar XXX.jar &
 
&代表在后台运行，使用ctrl+c不会中断程序的运行，但是关闭窗口会中断程序的运行。
 
三、nohup java -jar XXX.jar &
 
使用这种方式运行的程序日志会输出到当前目录下的nohup.out文件，使用ctrl+c中断或者关闭窗口都不会中断程序的执行。
 
四、nohup java -jar XXX.jar >temp.out &
 
>temp.out的意思是将日志输出重定向到temp.out文件，使用ctrl+c中断或者关闭窗口都不会中断程序的执行。
```

# 后台运行命令

当我们在使用putty/xshell等软件进行ssh远程访问服务器时，进行远程访问的界面往往不能关掉，否则，程序将不再运行。而且，程序在运行的过程中，还必须时刻保证网络的通常，这些条件都很难得到满足。为了解决上述问题，可以使用Linux下的screen命令，即使网络连接中断，用户也不会失去对已经打开的命令行会话的控制。下面介绍一些常用的screen命令。

具体使用如下：我们可以使用screen -S yourname创建一个叫做yourname的session，这时我们要进入该session，需要使用screen -r yourname进入到该session中，此时就可以在该session里进行操作了，如运行程序。之后我们可以使用Ctrl + a +d命令将该session丢到后台进行处理。

```Bash
screen -S yourname -> 新建一个叫yourname的session
screen -ls         -> 列出当前所有的session
screen -r yourname -> 回到yourname这个session
Ctrl+a+d           -> Detached记录当前Session，离开当前Session，若想再次进入，再次输入screen -r yourname即可
```

此时在 screen session 里，每个 window 内运行的 process (无论是前台/后台)都在继续执行，即使 logout 也不影响。

# DOS转Unix

对于 RHEL/CentOS 6/7 系统，使用 yum 命令 安装 dos2unix

```Bash
$ sudo yum install -y dos2unix
```

对于 RHEL/CentOS 8 和 Fedora 系统，使用 dnf 命令 安装 dos2unix

```Bash
$ sudo yum install -y dos2unix
```

覆盖原文件

```Bash
# dos2unix windows.txt
dos2unix: converting file windows.txt to Unix format …
```

如果你想保留原始文件，请使用以下命令。这将把转换后的输出保存为一个新文件。

```Bash
# dos2unix -n windows.txt unix.txt
dos2unix: converting file windows.txt to file unix.txt in Unix format …
```

# Cron表达式

常用表达式例子

```Java

  （1）0/2 * * * * ?   表示每2秒 执行任务

  （1）0 0/2 * * * ?    表示每2分钟 执行任务

  （1）0 0 2 1 * ?   表示在每月的1日的凌晨2点调整任务

  （2）0 15 10 ? * MON-FRI   表示周一到周五每天上午10:15执行作业

  （3）0 15 10 ? 6L 2002-2006   表示2002-2006年的每个月的最后一个星期五上午10:15执行作

  （4）0 0 10,14,16 * * ?   每天上午10点，下午2点，4点 

  （5）0 0/30 9-17 * * ?   朝九晚五工作时间内每半小时 

  （6）0 0 12 ? * WED    表示每个星期三中午12点 

  （7）0 0 12 * * ?   每天中午12点触发 

  （8）0 15 10 ? * *    每天上午10:15触发 

  （9）0 15 10 * * ?     每天上午10:15触发 

  （10）0 15 10 * * ?    每天上午10:15触发 

  （11）0 15 10 * * ? 2005    2005年的每天上午10:15触发 

  （12）0 * 14 * * ?     在每天下午2点到下午2:59期间的每1分钟触发 

  （13）0 0/5 14 * * ?    在每天下午2点到下午2:55期间的每5分钟触发 

  （14）0 0/5 14,18 * * ?     在每天下午2点到2:55期间和下午6点到6:55期间的每5分钟触发 

  （15）0 0-5 14 * * ?    在每天下午2点到下午2:05期间的每1分钟触发 

  （16）0 10,44 14 ? 3 WED    每年三月的星期三的下午2:10和2:44触发 

  （17）0 15 10 ? * MON-FRI    周一至周五的上午10:15触发 

  （18）0 15 10 15 * ?    每月15日上午10:15触发 

  （19）0 15 10 L * ?    每月最后一日的上午10:15触发 

  （20）0 15 10 ? * 6L    每月的最后一个星期五上午10:15触发 

  （21）0 15 10 ? * 6L 2002-2005   2002年至2005年的每月的最后一个星期五上午10:15触发 

  （22）0 15 10 ? * 6#3   每月的第三个星期五上午10:15触发
```

# 分区合并

Centos7把/home分区合并到/root

  

当/home分区和/不是一块硬盘或者挂载成不同分区的时候，我们有时候往往只大量使用了其中一个分区。

那么如何把这两个分区合并成一个。

首先看下当前分区大小分布

```Java
[root@localhost ~]# df -lh
Filesystem              Size  Used Avail Use% Mounted on
/dev/mapper/centos-root  925G  47G  879G  6% /
devtmpfs                1.9G    0  1.9G  0% /dev
tmpfs                    1.9G  116K  1.9G  1% /dev/shm
tmpfs                    1.9G  191M  1.7G  11% /run
tmpfs                    1.9G    0  1.9G  0% /sys/fs/cgroup
/dev/sda1                494M  97M  398M  20% /boot
tmpfs                    376M    0  376M  0% /run/user/0
/dev/mapper/centos-home  2.0G  33M  2.0G  2% /home
```

看到此时的home分区是占用2G空间的，那我们操作它，把它合并到root分区。

```Java
# 把/home内容备份，然后将/home文件系统所在的逻辑卷删除，扩大/root文件系统，新建/home：
tar cvf /tmp/home.tar /home    #备份/home  没东西可以不备份
# 记录一下 home下有多少可用空间  ，比如2G
umount /home    #卸载/home，如果无法卸载，先终止使用/home文件系统的进程
lvremove /dev/centos/home    # 删除/home所在的lv
lvextend -L +2G /dev/centos/root    # 扩展/root所在的lv，增加/home的大小
xfs_growfs /dev/centos/root    #扩展/root文件系统
mkdir -p /home && cd /home # 重新创建home目录
tar xvf /tmp/home.tar # 恢复备份的文件
# 这里一般需要重新设置下逻辑分区的大小
xfs_growfs /dev/centos/root  # 重新设置root对应分区大小
df -h # 查询新分区
```

现在如下：

```Java
[root@localhost ~]# df -lh
Filesystem              Size  Used Avail Use% Mounted on
/dev/mapper/centos-root  927G  47G  881G  6% /
devtmpfs                1.9G    0  1.9G  0% /dev
tmpfs                    1.9G  120K  1.9G  1% /dev/shm
tmpfs                    1.9G  191M  1.7G  11% /run
tmpfs                    1.9G    0  1.9G  0% /sys/fs/cgroup
/dev/sda1                494M  97M  398M  20% /boot
tmpfs                    376M    0  376M  0% /run/user/0
```