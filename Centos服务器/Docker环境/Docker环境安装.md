docker hub 的两种仓库
1. 顶层仓库, 由docker hub 自己管理
2. 用户仓库, 由开发人员创建维护, 命名构成: 用户名/仓库名, 例如: zabbix/zabbix-proxy-sqlite3

# Docker客户端执行流程
- Docker_Host：安装Docker的主机
- Docker Daemon：运行在Docker主机上的Docker后台进程
- Client：操作Docker主机的客户端（命令行、UI等）
- Registry：镜像仓库如Docker Hub
- Images：镜像，带环境打包好的程序，可以直接启动运行
- Containers：容器，由镜像启动起来正在运行中的程序
![[Pasted image 20240127164722.png]]
交互逻辑 装好Docker，然后去软件市场寻找镜像，下载并运行，查看容器状态日志等排错

# 安装Docker


```Shell
wget https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo -O /etc/yum.repos.d/docker-ce.repo

yum list  docker-ce --showduplicates|sort -r

yum -y install docker-ce-18.06.3.ce-3.el7

#若报错 Requires: container-selinux >= 2.9
wget -O /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-7.repo
yum install epel-release
yum install container-selinux
```

  
```Shell
systemctl enable docker && systemctl start docker
```

  
查看dockers版本
```Shell
docker version
```

  
设置Docker镜像加速器：

```Shell
sudo mkdir -p /etc/docker
```

  
```Shell
sudo tee /etc/docker/daemon.json <<-'EOF'
{
  "exec-opts": ["native.cgroupdriver=systemd"],        
  "registry-mirrors": ["https://docker.mirrors.ustc.edu.cn","https://b9pmyelo.mirror.aliyuncs.com"],        
  "live-restore": true,
  "log-driver":"json-file",
  "log-opts": {"max-size":"500m", "max-file":"3"}
}
EOF
```

刷新加速器

```Shell
sudo systemctl daemon-reload

sudo systemctl restart docker
```


# 安装 Docker-Compose

```shell
# ---------- 安装 docker compose ----------
# 可能会出现连接超时的情况，请耐心的尝试几次
wget https://github.com/docker/compose/releases/download/1.27.4/docker-compose-Linux-x86_64
mv docker-compose-Linux-x86_64 /usr/local/bin/docker-compose
# 添加执行权限
chmod +x /usr/local/bin/docker-compose
# 检测是否安装成功
docker-compose -version
```

# 修改 Docker镜像的默认存储路径

Docker 默认安装的情况下，会使用 /var/lib/docker/ 目录作为存储目录，用以存放拉取的镜像和创建的容器等。不过由于此目录一般都位于系统盘，遇到系统盘比较小，而镜像和容器多了后就容易尴尬，这里说明一下如何修改 Docker 的存储目录。

以我手头的一台 VPS 作为例子，可以看到这台机子本身有两块硬盘，我把数据盘 vdb 挂载到了/www 目录，目标就是将 Docker 存储目录移到/www/docker。
![[Pasted image 20240127164910.png]]

输入：

docker info

可以查看程序信息，红框里就是默认的存储目录：
![[Pasted image 20240127164918.png]]

最简单粗暴的办法，当然就是直接把数据盘挂载到/var/lib/docker 目录下，不过这样对整体影响太大，其他程序需要使用数据盘时很不方便，所以还是从 Docker 端的修改入手。

官方文档的修改办法是编辑 /etc/docker/daemon.json 文件：

```Java
vi /etc/docker/daemon.json 
```

默认情况下这个配置文件是没有的，这里实际也就是新建一个，然后写入以下内容：

{ "data-root": "/www/docker" }

```Java
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "registry-mirrors": ["https://docker.mirrors.ustc.edu.cn","https://b9pmyelo.mirror.aliyuncs.com"],
  "live-restore": true,
  "log-driver":"json-file",
  "log-opts": {"max-size":"500m", "max-file":"3"},
  "data-root": "/work/docker"
}
```

保存退出，然后重启 docker 服务：

systemctl restart docker

再次查看 docker 信息，可以看到目录已经变成了设定的/www/docker:
![[Pasted image 20240127164938.png]]