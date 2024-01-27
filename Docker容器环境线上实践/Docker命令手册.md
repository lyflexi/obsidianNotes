![[Pasted image 20240127165512.png]]

# 镜像

images_name 表示镜像名

con_name表示容器名

```Java
#获取镜像
docker pull images_name 
#查看已有的docker镜像
docker images
#删除镜像
docker rmi image_name
#修改镜像名
docker tag imageid name:tag
#导出镜像
docker save -o image_name.tar image_name
```

## 提交改变

将自己修改好的镜像提交到本地

```Java
docker commit [OPTIONS] CONTAINER [REPOSITORY[:TAG]]

docker commit -a "leifengyang"  -m "首页变化" 341d81f7504f guignginx:v1.0
#这样启动之后,运行的就是我们修改过的定制镜像
docker run -d -p 88:80 guignginx:v1.0
```
![[Pasted image 20240127165521.png]]

## 镜像传输-离线安装

```Java
# 将镜像保存成压缩包
docker save -o abc.tar guignginx:v1.0
#远程传输
scp abc.tar root@另外一台主机ip:/目录位置



# 别的机器加载这个镜像
docker load -i abc.tar
```

docker images
![[Pasted image 20240127165529.png]]

```Java
# 离线安装
docker run -d -p 88:80 guignginx:v1.0
```

# 镜像传输-在线安装

# 容器

con_name表示容器名

image_name 表示镜像名

```Java
#启动一个容器
docker run image_name

#查看正在运行的容器
docker ps

#查看所有的容器
docker ps -a

#停止/启动/重启 一个容器
docker stop/start/restart con_name

#删除容器
docker stop 容器
docker rm 
docker rmi

#编辑容器名称
docker rename 原容器名 新容器名

#看容器的端口映射情况
docker port con_name

#动态查看容器日志
docker logs -f con_name

#动态查看最后100行的容器日志
docker logs -f –tail 100 con_name

#查看容器pid
docker top con_name

#查看docker容器IP
docker inspect con_name| grep IPAddress

#进入容器
docker exec -it con_name  /bin/bash

#重启 docker 服务
systemctl restart docker

#重启应用
docker restart xxx
```

# 网络

net_name 网络名

```Java
#查看所有网络
docker network ls
#创建一个docker网络
docker network create -d bridge net_name
#创建一个docker网络
docker run –network= net_name -itd –name=con_name image_name
#创建一个docker网络
docker network connect net_name con_name
```

  

# 文件

host_path 主机文件路径

container_path 容器文件路径

containerID 容器id，用容器名也可以

```Java
#从主机复制到容器
sudo docker cp host_path containerID:container_path
sudo docker cp /wefe/krb5.conf e7b8b9fd56e6:/etc/
sudo docker cp /root/wefe_board_website/resources/mount/html/board-website/img/x-logo.b4403526.png dad8ed0737a8:/opt/website/html/board-website/img/
#从容器复制到主机
sudo docker cp containerID:container_path host_path

:/data/projects/fate/fateflow/examples/toy
```

**kubenetes类似**

1.kubectl cp /主机目录/文件路径 podName:/容器路径/xxx.datasource -n namespaces

这样可以把主机目录文件拷贝到容器内

2.kubectl cp podName:容器路径/xxx.datasource -n namespaces /主机目录

这样可以把容器内文件cp到主机目录

  

  

# 日志

```Java
#刚运行就退出的容器，查看日志
docker start con_name -ia
#动态查看容器日志
docker logs -f con_name
#动态查看最后100行的容器日志
docker logs -f –tail=100 con_name
```

# 配置

```Java
#查看容器配置
docker inspect con_name
```

# 数据卷挂载

以Redis为例：

```Java
# 如果直接挂载的话docker会以为挂载的是一个目录，所以我们先创建一个文件然后再挂载，在虚拟机中。
# 在虚拟机中，Linux如果不提前创建redis.conf，-v /mydata/redis/conf/redis.conf会把redis.conf当作目录
mkdir -p /mydata/redis/conf
touch /mydata/redis/conf/redis.conf

#安装：
docker pull redis

docker run -p 6379:6379 --name redis \
-v /mydata/redis/data:/data \
-v /mydata/redis/conf/redis.conf:/etc/redis/redis.conf \
-d redis redis-server /etc/redis/redis.conf

# 直接进去redis客户端。
docker exec -it redis redis-cli
exit

#配置持久化
默认是不持久化的。在配置文件中输入appendonly yes，就可以aof持久化了。修改完docker restart redis，docker -it redis redis-cli
vim /mydata/redis/conf/redis.conf
# 插入下面内容，aof持久化
appendonly yes
保存
docker restart redis

#设置redis容器在docker启动的时候启动
docker update redis --restart=always
```

这就是Docker挂载目录与挂载文件的区别：

- volume叫做数据卷挂载，默认行为是挂载目录将container内的目录挂载出来，同时自动在宿主机创建好目录以及同步container目录内的文件，`-v /mydata/redis/data:/data`
    
- **但如果想要挂载单一文件的就要在宿主机对应位置提前放好文件，否则****`-v /mydata/redis/conf/redis.conf:/etc/redis/redis.conf`****会把宿主机的redis.conf当作目录**
    

```Java
mkdir -p /mydata/redis/conf
touch /mydata/redis/conf/redis.conf
```

# 修改 Docker镜像的默认存储路径

  

Docker 默认安装的情况下，会使用 /var/lib/docker/ 目录作为存储目录，用以存放拉取的镜像和创建的容器等。不过由于此目录一般都位于系统盘，遇到系统盘比较小，而镜像和容器多了后就容易尴尬，这里说明一下如何修改 Docker 的存储目录。

以我手头的一台 VPS 作为例子，可以看到这台机子本身有两块硬盘，我把数据盘 vdb 挂载到了/www 目录，目标就是将 Docker 存储目录移到/www/docker。

![](https://x3r1317gt9.feishu.cn/space/api/box/stream/download/asynccode/?code=ODQ1NzdkN2Y3M2YzNzI0YzQzZGEzNzg4MDEyOTE0MWVfa1hIVUFRZXZkUkoyUmRSejNBaklBcmNhN3VPbUxwb0RfVG9rZW46Q2dpUGJJY21ab2lLUXF4WHBTY2NLbENVbmllXzE3MDYzNDU3MDY6MTcwNjM0OTMwNl9WNA)

  

输入：

docker info

可以查看程序信息，红框里就是默认的存储目录：

![](https://x3r1317gt9.feishu.cn/space/api/box/stream/download/asynccode/?code=OTNhOThmZTAyZGMzYWFhNjQ1MDI1NzhlY2FlYTJmN2JfYTZUbExGVjNtcHBGNGl0ZWk0dzIyc1IxWFdhaldvZjhfVG9rZW46QXN4cWIwTEZrb09QSmR4RWhVQmN0cHp0bkJ4XzE3MDYzNDU3MDY6MTcwNjM0OTMwNl9WNA)

  

最简单粗暴的办法，当然就是直接把数据盘挂载到/var/lib/docker 目录下，不过这样对整体影响太大，其他程序需要使用数据盘时很不方便，所以还是从 Docker 端的修改入手。

  

官方文档的修改办法是编辑 /etc/docker/daemon.json 文件：

```Java
vi /etc/docker/daemon.json 
```

默认情况下这个配置文件是没有的，这里实际也就是新建一个，然后写入以下内容：

{ "data-root": "/www/docker" }

此文件还涉及默认源的设定，如果设定了国内源，那么实际就是在源地址下方加一行，写成：

{ "registry-mirrors": ["http://hub-mirror.c.163.com"], "data-root": "/www/docker" }

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

![](https://x3r1317gt9.feishu.cn/space/api/box/stream/download/asynccode/?code=OTlhYWVlZThkMDFmZGEwMjI4OTFhMTBkNmM0YmNkYTJfS2UwV2hYQkdSZjhBVURBemoxU0xrdlp6VXpjTlEwcjVfVG9rZW46SUhnVWIycGUxb2MyV3B4cW5leGNmNWZUbmxlXzE3MDYzNDU3MDY6MTcwNjM0OTMwNl9WNA)

发布于 2019-12-05 14:27